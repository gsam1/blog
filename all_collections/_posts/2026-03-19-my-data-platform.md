---
layout: post
title: "The my home data platform"
date: 2026-03-19
categories: homelab
---

# Building a Home Data Platform: From Smart Sensors to Iceberg Tables

I've been building a personal data platform to make sense of the sensor data coming out of my home. What started as a simple MQTT listener has grown into something I'd actually call production-grade: a streaming ingestion pipeline, Kafka as a central data bus, Apache Iceberg tables on MinIO, DuckDB for analytics, and Metabase for dashboards — all running on a k3s cluster in my house.

This is a writeup of how it's built and why I made the decisions I did.

---

## The Problem

I have a bunch of sensors and smart home devices feeding data into Home Assistant — temperature, humidity, CO2, tVOC, PM2.5 particulates, power consumption. Home Assistant is great for automation, but it's not a data platform. If I want to answer questions like "what does indoor air quality look like over the past six months, and does it correlate with outdoor weather?" I need somewhere to actually store and query that data.

I also wanted an excuse to work with Apache Iceberg outside of a managed cloud environment.

---

## Architecture Overview

Here's the full data flow at a high level:

```
[Home Assistant / MQTT / InfluxDB / HTTP APIs]
             ↓
   [source-declerative-py]        ← config-driven ingestion
             ↓
         [Kafka]                  ← central data bus ("data_bus" topic)
             ↓
    [sink-kiceberg-rs]            ← Rust consumer
             ↓
   [Iceberg Tables on MinIO]      ← Parquet files, S3-compatible
             ↓
   [DuckDB + Airflow DAGs]        ← staging → refined → enriched
             ↓
        [Metabase]                ← dashboards
```

Everything runs on k3s on hardware I own. No cloud bills.

---

## Ingestion: Declarative ETL

The ingestion layer went through a few generations. Originally I had separate Python services for each source — one for MQTT, one for the Home Assistant REST API, one for InfluxDB. This got messy fast. Every new source meant a new service, a new Dockerfile, a new Kubernetes deployment.

I replaced all of that with `source-declerative-py`, a config-driven ingestion engine where sources are defined in TOML:

```toml
[[sources]]
type = "http"
name = "home_assistant_temperature"
url = "http://<ha-url>/api/history/period"
interval_seconds = 1800
entity_ids = ["sensor.living_room_temperature"]

[[sources]]
type = "mqtt"
name = "air_quality_co2"
broker = "<mqtt-url>"
port = 1883
topics = ["airquality/pi-sensor/CO2eq", "airquality/pi-sensor/tVOC"]
```

Each source runs in its own process. The engine handles scheduling, retries, and routing all output to Kafka. Adding a new source means adding a TOML block — no code changes, no new deployment.

The Kafka transport uses JSON with Snappy compression and round-robin partitioning to the `data_bus` topic.

---

## The Sink: Rust + Apache Iceberg

On the other side of Kafka sits `sink-kiceberg-rs`, a Kafka consumer written in Rust that writes directly to Iceberg tables.

I chose Rust here because this is a long-running streaming service and I didn't want to think about GC pauses or memory leaks. It consumes messages in batches, converts them to Arrow record batches, and commits them to Iceberg via a REST catalog backed by PostgreSQL.

Storage is MinIO with four buckets: `raw`, `lake`, `warehouse`, and `documents`. The Iceberg warehouse lives in `s3://warehouse/`, and the REST catalog is exposed at `iceberg-rest.fury.net` inside the cluster.

Iceberg gives me things I actually care about for this use case:

- **ACID transactions** — no partial writes showing up in queries
- **Time-travel** — I can query historical snapshots if I mess up a transformation
- **Schema evolution** — adding a new sensor doesn't require migrating existing data
- **Native Parquet on S3** — DuckDB can read these directly with zero ETL

---

## Transformation: Airflow + DuckDB

Raw data lands in Iceberg staging tables. From there, Airflow DAGs run nightly transformations through three layers:

1. **Staging** — raw ingestion, minimal processing
2. **Refined** — deduplication, type casting, null handling
3. **Enriched** — joining sensor streams, applying business logic
4. **Presentation** — optimized views for Metabase queries

I wrote a custom Airflow operator, `DuckDBModelOperator`, that runs DuckDB SQL models against Iceberg tables. DuckDB is a surprisingly good fit here — it has native Iceberg support via the `httpfs` extension, it can read Parquet directly from MinIO without any intermediate copies, and it runs in-process so there's nothing else to manage.

A typical transformation looks like this — the sensor enrichment DAG joins temperature, humidity, CO2, and air quality into a single wide table with proper units and timestamps.

I also have a `DuckDBToPostgresOperator` for cases where Metabase needs data from a traditional Postgres table rather than querying Iceberg directly.

---

## One Extra Thing: Document Embeddings

There's an `embeddings_DAG.py` in the pipeline that I added more out of curiosity than necessity. It pulls documents from the `documents` S3 bucket, generates embeddings using a local Ollama instance, and stores them in PostgreSQL with pgvector. Keeping the option open to do semantic search over home documentation, manuals, notes — we'll see.

---

## Infrastructure

Everything runs on k3s. Key services:

| Service | Namespace |
|---|---|
| source-declerative-py | data-platform |
| sink-kiceberg-rs | data-platform |
| iceberg-rest catalog | data-platform |
| Metabase | dwh-infra |

Deployments are defined as Kubernetes manifests. ConfigMaps hold non-sensitive config; Kubernetes Secrets hold credentials. Containers run as non-root (uid 1000).

CI/CD is GitHub Actions with per-component workflows. Pushing a change to `tools/declerative-etl/source-declerative-py/**` triggers a build, push to the local registry and a `kubectl rollout restart`. The whole deploy usually takes under two minutes.

I'm also running a small `flow-monitor-go` service and a `kafka-rest-go` REST API in front of Kafka for debugging, both written in Go.

---

## What I'd Do Differently

**Catalog backend.** I started with SQLite for the Iceberg REST catalog and migrated to PostgreSQL. Should have started with Postgres.

**Schema design.** I'm storing everything as raw JSON in early staging layers. It works, but means transformation SQL is doing a lot of `json_extract()`. Better ingestion-time schema enforcement would help.

**MQTT-to-Kafka is noisy.** Some MQTT topics publish very frequently and the signal-to-noise ratio is low. I want to add a filtering/aggregation step before Kafka rather than doing it downstream.

---

## Stack Summary

| Layer | Tech |
|---|---|
| Ingestion | Python, Paho MQTT, Httpx, Kafka-Python |
| Transport | Apache Kafka |
| Sink | Rust, Apache Iceberg, PyIceberg |
| Storage | MinIO (S3), Apache Parquet |
| Catalog | Iceberg REST + PostgreSQL |
| Transformation | Apache Airflow, DuckDB |
| Analytics | DuckDB, Metabase |
| Infra | k3s, Docker, GitHub Actions |

