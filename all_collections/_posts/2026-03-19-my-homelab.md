---
layout: post
title: "The humble homelab monster in my closet"
date: 2026-03-19
categories: homelab
---

# The humble homelab monster in my closet

## Why I Built This Monster

Let's be honest - I could have just run a few Docker containers on a Raspberry Pi and called it a day. But where's the fun in that? The FuryNet started as a weekend project that quickly spiraled into a full-blown infrastructure obsession. What began as "I just want to host my own stuff" turned into "let me rebuild my entire network from scratch using the same tools Netflix uses."

The result? A homelab that's probably overkill for checking email and storing cat photos, but absolutely perfect for scratching that technology itch and learning everything from Kubernetes to AI-powered security cameras.

## The Grand Architecture (Or: How I Learned to Stop Worrying and Love YAML)

### Why Go Full Enterprise at Home?

Because I can, and because breaking things in production at work is frowned upon. My home infrastructure follows the same principles I'd use for a startup's tech stack:

- **Everything Breaks Eventually**: So make it easy to rebuild
- **Git All The Things**: Because `sudo rm -rf /` happens to the best of us
- **Automate Everything**: I'm too lazy to click through setup wizards repeatedly
- **Plan for Chaos**: If it can't survive my weekend "improvements," it's not production-ready

### The Tech Stack (AKA My Digital LEGO Collection)

Here's what's running in my digital empire:

**The Foundation:**
- **Proxmox VE**: Because paying for VMware is for suckers, and I like my hypervisors free and powerful
- **K3s**: Kubernetes, but without the enterprise bloat. Perfect for when you want to feel like a Google engineer but only have a few old PCs
- **Docker**: The container engine that started it all. Still the workhorse of the operation

**The Automation Army:**
- **Terraform**: Turns infrastructure into code, because clicking through web UIs is for mortals
- **Ansible**: My configuration management sidekick. SSH + YAML = infrastructure happiness
- **Packer**: Creates VM templates so I never have to install Ubuntu Server manually again

**The Plumbing:**
- **Nginx Load Balancer**: Dedicated VM handling all k3s API and ingress traffic like a boss
- **Private Docker Registry**: Because pulling from Docker Hub gets old, and I like my images local and fast
- **Proper VLANs**: Network segmentation, because even at home, security matters

## Code Organization (Because Future Me Will Thank Present Me)

I learned the hard way that throwing all your infrastructure code into one giant folder is a recipe for disaster. Here's how I keep my digital chaos organized:

```
furynet-infrastructure/
├── common/                 # The stuff that works everywhere
│   ├── ansible/           # Playbooks I can run anywhere
│   ├── config/            # Templates for the lazy
│   ├── packer/            # VM building recipes
│   └── terraform/modules/ # Infrastructure LEGO blocks
├── development/           # Where I break things safely
├── production/           # The stuff that actually matters
└── _images/              # Pretty diagrams for documentation
```

This isn't just organization porn - it actually serves a purpose:
- **DRY Principle**: Don't Repeat Yourself (because copy-pasting is for amateurs)
- **Safe Experimentation**: I can nuke dev without affecting my actually useful stuff
- **Sane Navigation**: When I'm debugging at 2 AM, logical folders save my sanity

## Dev to Prod: The Sacred Journey

### The "Please Don't Break Production" Strategy

I've got a simple rule: if it works in dev, it *might* work in prod. To make this slightly less terrifying, I built a sync mechanism that's basically `rsync` with trust issues:

```bash
rsync -av --exclude='terraform.tfvars' --exclude='.terraform/' \
  --exclude='.terraform.lock.hcl' development/ production/
```

This magical incantation:
- **Tests Everything First**: Dev environment is my crash test dummy
- **Keeps Secrets Secret**: Production passwords don't accidentally end up in dev
- **Prevents "It Worked on My Machine"**: Well, technically it's all my machine, but you get the idea

### GitHub Actions: My Personal CI/CD Butler

Because manually deploying infrastructure is for people who enjoy pain, I've got GitHub Actions handling the boring stuff. Push to dev branch → robots take over → infrastructure deploys itself. It's like having a very obedient, very fast intern who never asks for coffee breaks.

The setup includes:
- **Git Push = Deploy**: Because clicking buttons is so 2010
- **Secret Sauce Management**: GitHub secrets keep my passwords out of public repos
- **Self-Hosted Runners**: My own hardware doing the heavy lifting (because I don't trust "the cloud" with my precious infrastructure)

## Kubernetes: Where I Pretend to Be Google

### K3s: Kubernetes Without the Kubernetes

Let's be real - full Kubernetes is like using a freight train to deliver pizza. K3s gives me all the orchestration goodness without needing a data center:
- **Actually Fits in RAM**: My poor servers can breathe easy
- **Setup That Doesn't Require a PhD**: One binary, done
- **Still Real Kubernetes**: All the APIs, none of the enterprise complexity

### What's Actually Running (My Digital Zoo)

My cluster is home to a motley crew of applications that serve various purposes from "legitimately useful" to "seemed like a good idea at 3 AM":

**The Productivity Squad:**
- **JupyterLab**: Where I pretend to be a data scientist and mostly just plot random graphs
- **Private Docker Registry**: Because I'm too cool for Docker Hub rate limits

**The Message Passing Mafia:**
- **Apache Kafka**: For when I want to feel like I'm building the next Twitter
- **RabbitMQ**: Because sometimes you need a message broker that just works

### Storage: The Persistent Problem

Nothing ruins a good container party like losing all your data when things restart:
- **NFS Everywhere**: Shared storage because I'm not an animal
- **Persistent Volume Claims**: Data that survives my "improvements"
- **Resource Limits**: Because one runaway Python script shouldn't kill everything

## The Paranoid Homeowner's Paradise: Frigate NVR

### When Your Homelab Becomes Your Security Guard

Sure, I could run some boring web apps, but why not turn my homelab into a full-blown surveillance state? Enter Frigate NVR - because apparently I need to know exactly when the neighbor's cat decides to visit my yard at 3 AM.

The setup is gloriously excessive:
- **7 Cameras**: Covering every inch like I'm protecting Fort Knox (it's actually just my house)
- **AI-Powered Detection**: Because humans are terrible at watching security footage
- **RTSP Streams**: Direct from IP cameras, no cloud nonsense

### The AI Brain: Google Coral TPU

Here's where things get fun - I've got a dedicated AI chip just for staring at video feeds:

```yaml
detectors:
  coral:
    type: edgetpu
    device: pci
```

Why this is awesome:
- **Local AI**: No sending footage to Google/Amazon/SkyNet
- **Fast Detection**: Hardware acceleration means instant "person detected" alerts
- **Privacy First**: What happens in my cameras stays in my cameras

## The "Just Push a Button" Deployment Experience

### The Automated Kubernetes Assembly Line

Building a Kubernetes cluster by hand is like assembling IKEA furniture without instructions - possible, but why torture yourself? I've automated the whole thing into a beautifully orchestrated process:

**Phase 1: The Foundation (Terraform + Ansible)**
1. **Terraform Provisions the Hardware**: Spawns VMs on Proxmox like they're going out of style
   - Master nodes for running the control plane
   - Worker nodes for actual workloads
   - Dedicated nginx load balancer VM (because high availability isn't optional)
2. **Ansible Preps the Environment**: Updates, packages, all the boring-but-necessary stuff
3. **Nginx Gets Configured**: Ansible installs nginx with stream module for TCP load balancing
   - Layer 4 load balancing for k3s API server (port 6443)
   - HTTP/HTTPS ingress traffic distribution (ports 80/443)

**Phase 2: The Cluster Magic (k3sup + Terraform)**
4. **k3sup Bootstraps the First Master**: One command creates the initial control plane
   - TLS certificate includes the load balancer IP (crucial for HA setup)
   - Cluster mode enabled from the start
5. **Additional Masters Join**: More control plane nodes for that sweet, sweet redundancy
   - Each master also gets the load balancer in its TLS SAN
   - They retrieve join tokens from the first master, then register via the LB
6. **Workers Join the Party**: Agent nodes connect and await their workload destiny
7. **Registry Gets Distributed**: Private Docker registry setup because external dependencies are for suckers
8. **Kubeconfig Gets Updated**: Switch from direct node access to load balancer endpoint
9. **Apps Get Deployed**: The fun stuff finally happens (Kafka, RabbitMQ, JupyterLab, etc.)

**Why This Approach Actually Rocks:**
- **True High Availability**: Load balancer means no single point of failure
- **Run It Again and Again**: Idempotent operations mean I can hammer the deploy button without fear
- **Fail Fast, Fix Faster**: Each step can be debugged independently
- **Production-Grade Pattern**: Same approach Netflix/Google use, just smaller scale
- **Future Me Documentation**: When I inevitably forget how this works in 6 months

**The Load Balancer: Unsung Hero**

The nginx load balancer is the secret sauce that makes this whole thing production-worthy:
- **API Server HA**: All kubectl commands route through the LB, not individual masters
- **Automatic Failover**: If a master dies, the LB just routes to the surviving ones
- **Ingress Distribution**: HTTP/HTTPS traffic gets spread across all nodes
- **Single Entry Point**: One IP to rule them all (192.168.10.210 in my case)

## The Accidental Education Platform

### What I've Learned (Besides How to Break Things Creatively)

This whole setup started as "I want to host my own stuff" and turned into an accidental crash course in everything:

**The DevOps Toolbox:**
- **Infrastructure as Code**: Turns out, clicking through web UIs is the wrong way to do everything
- **Container Orchestration**: Now I can sound smart talking about "workload distribution" at parties
- **CI/CD Pipelines**: Robots deploying my code while I sleep? Yes please
- **Configuration Management**: Ansible has saved me from SSH-ing into servers more times than I can count

**The Systems Admin Skills I Didn't Know I Needed:**
- **Virtualization**: Proxmox taught me that VMs are just computers inside computers (mind blown)
- **Networking**: VLANs, subnets, and why my IoT devices can't talk to my servers (by design)
- **Storage**: Why NFS is amazing and why I should have set it up sooner
- **Security**: How to lock things down without locking myself out (mostly)

**The Cool Future Stuff:**
- **Microservices**: Breaking monoliths into tiny pieces for fun and profit
- **Message Queues**: How to make services talk without them knowing about each other
- **Edge AI**: Running machine learning on a tiny chip because the future is now

### Why This Actually Matters for Real Jobs

Plot twist: this "hobby" setup taught me more about infrastructure than most bootcamps:
- **Kubernetes**: Every company wants this, and I've actually broken it enough times to understand it
- **Infrastructure Automation**: Manual deployments are career suicide in 2024
- **Security Mindset**: Thinking about security from day one, not as an afterthought

## The Philosophy Behind the Madness

### Infrastructure as Code: Because Clicking Is for Mortals

Everything in my setup follows a few simple rules:
- **Git Everything**: If it's not in version control, it doesn't exist
- **Modular Design**: Write once, use everywhere (because I'm lazy)
- **Document Everything**: Future me will thank present me
- **State Management**: Terraform state files are sacred - lose them and cry

### Security: Paranoia as a Feature

I may be running this at home, but I'm not an idiot:
- **Network Isolation**: IoT devices live in network jail where they belong
- **Secrets Management**: Passwords in plain text are for amateurs
- **Principle of Least Privilege**: Everything gets exactly the access it needs and nothing more
- **Regular Updates**: Automated patching because manual updates are how you get pwned

### Operations: Planning for Inevitable Disaster

My infrastructure assumes I'm going to break things:
- **Monitoring Ready**: When (not if) things break, I want to know immediately
- **Backup Everything**: Data that's not backed up is just temporary
- **Disaster Recovery**: I can rebuild everything from Git (theoretically)
- **Documentation**: Because 3 AM debugging sessions require good notes

## What I've Learned (The Hard Way)

### Finding the Sweet Spot Between "Cool" and "Actually Useful"

Building this thing taught me that there's a fine line between:
- **Learning New Tech**: Playing with Kubernetes vs. actually needing it
- **Practical Stuff**: Security cameras that actually work vs. showing off AI
- **Resource Reality**: My electricity bill vs. my desire to run everything
- **Maintenance Time**: How much weekend time I'm willing to sacrifice to the server gods

### When Your Homelab Becomes Part of Your Actual Home

The coolest part? My infrastructure isn't just a playground - it's actually useful:
- **Security System**: Real cameras protecting real stuff
- **Development Environment**: JupyterLab for actual data analysis projects
- **Learning Lab**: Safe place to break things and learn from the wreckage
- **Bragging Rights**: "Oh this? Just my personal Kubernetes cluster"

### Growing Pains and Future Plans

The beauty of this modular approach:
- **Add More Nodes**: More hardware = more problems to solve
- **New Services**: Each new app is an excuse to learn something
- **Technology Upgrades**: When something cooler comes along, I can swap it in
- **Inevitable Rewrites**: Because what's the fun in leaving well enough alone?

## The Bottom Line

What started as "I want to self-host some stuff" turned into a full-blown digital empire that:

1. **Teaches Real Skills**: The stuff I've learned here actually matters at work
2. **Solves Real Problems**: Home security, development environments, and more
3. **Provides Endless Entertainment**: There's always something to fix, improve, or completely rebuild
4. **Impresses Absolutely No One**: Except other infrastructure nerds who understand the beauty of well-orchestrated YAML

This isn't just a homelab - it's a digital playground, a learning laboratory, and occasionally, a source of genuine household utility. Sure, I could have just bought a Synology NAS and called it a day, but where's the fun in that?

The real magic happens when your side project becomes your R&D lab, your security system, your development environment, and your excuse to stay up too late tweaking configuration files. It's Infrastructure as Code, but more importantly, it's Infrastructure as Fun.

And yes, I know using Kubernetes for a homelab is overkill. That's exactly the point.
