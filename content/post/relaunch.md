+++
comments = true
date = "2026-01-01T00:00:00-00:00"
draft = false
slug = "relaunch"
tags = ["infrastructure", "operations", "lessons", "platform-engineering"]
title = "2026 Relaunch: Infrastructure Lessons from 10+ Years Running Production"
description = "Back online with a focus on pragmatic infrastructure patterns, backup strategies, and Kubernetes operations. Real lessons from building and operating production platforms."
image = "img/main-01.jpg"
summary = "Relaunching with a focus on pragmatic infrastructure: backup strategies, Kubernetes autoscaling, and lessons from 10+ years of production operations."
+++

It’s time to relaunch.

After years of building and operating production platforms, I’m sharing what actually works—not theory, not marketing, but real patterns from infrastructure that handles billions of transactions.

## What You’ll Read Here

**Infrastructure at Scale:**
- How databases fail when they outgrow Kubernetes (and what to do about it)
- Backup strategies that don’t bankrupt you: XtraBackup, ZFS, S3
- High-availability firewalls for private clusters (OpenBSD CARP)

**Kubernetes Operations:**
- Pod autoscaling that works for queue-based systems (spoiler: CPU metrics fail)
- Node autoscaling when your pods can’t find homes
- Real numbers on latency, cost, and what matters

**Enterprise Infrastructure:**
- Three-layer backup strategy for disaster recovery
- Incremental replication across storage nodes
- Production patterns that survive complexity

## Why Now

Most blog posts in tech are either tutorials for beginners or marketing for vendors. This is different. These are pragmatic solutions to real problems: scaling MySQL to TB, autoscaling bursty workloads, protecting S3 buckets, designing resilience.

No buzzwords. No SaaS pitches. Just what we learned running infrastructure at scale.

**Ready?** Start with [Scaling MySQL Beyond Kubernetes](/post/mysql-scaling-beyond-k8s/).
