+++
title = "Scaling MySQL Beyond Kubernetes: When Backups Become The Blocker"
date = "2026-03-23T00:00:00-00:00"
draft = false
slug = "mysql-scaling-beyond-k8s"
tags = ["kubernetes", "database", "scaling", "mysql", "backups", "xtrabackup", "operations"]
summary = "We ran MySQL in Kubernetes for years until the data scaled from GB to TB. Kubernetes wasn't the problem—backups were. Kasten (the Kubernetes backup tool) was never designed for databases at our scale. Learn why we moved MySQL off Kubernetes and the backup solution we built to solve this infrastructure challenge."
comments = true
image = "img/mysql-header.jpg"
+++

We ran MySQL in Kubernetes for years. It worked great.

Then the data grew.

From GB to hundreds of GB to TB. And suddenly, Kubernetes wasn't the problem—backups were.

This is the story of why we moved MySQL off Kubernetes, but not because Kubernetes failed. It failed because the backup tool was never designed for databases at our scale.

## The Years It Worked

Kubernetes was fine for MySQL when:

- Data was small (single-digit GB)
- Backups were quick
- Recovery was rare
- Kasten (the Kubernetes backup tool) could handle snapshots

We had everything infrastructure engineers want:

- Infrastructure as code (YAML files)
- Automatic failover (StatefulSets)
- Self-healing (pod restarts)
- Version-controlled infrastructure

It wasn't elegant, but it worked. We weren't thinking about moving it.

## The Scaling Problem

Data grew. Fintech platforms have that problem—transactional data multiplies.

We moved from:

- Single GB backups (minutes to complete, hours to verify recovery)
- To 100GB backups (hours to complete, restore became risky)
- To TB backups (overnight backups, recovery windows measured in days)

Suddenly, backup operations became the critical path for the entire infrastructure.

## Why Kasten Failed at Scale

Kasten is designed for Kubernetes workloads. It's great for:

- Restoring a lost pod
- Backing up application state
- Snapshot-based recovery

It's terrible for:

- Large, consistent database exports
- Incremental backups
- Point-in-time recovery
- Hot backup management

The root cause: **Kasten doesn't understand databases.**

It treats MySQL the same as it treats stateless services—snapshots and restore. But databases need:

- Consistent, logical backups (not raw snapshots)
- Incremental backups (TB of data can't be full-backed daily)
- Recovery verification (you need to test that backups actually work)
- Point-in-time recovery (financial systems need this)

When backups hit TB scale, Kasten couldn't keep up. Recovery testing took days. Backup failures happened often. The team spent cycles babysitting backups instead of shipping features.

## The Decision: Move to VMs

Not because Kubernetes was broken. Because the backup requirement broke Kubernetes.

The solution: **Move MySQL to Debian VMs and implement proper database backups.**

This sounds like a step backward. It's not. It's recognizing that databases need specialized infrastructure.

## The Implementation: XtraBackup + S3

We didn't just move the database. We built a backup solution designed for MySQL at scale.

**Three components:**

1. **Backup script** - Full and incremental backups with S3 sync
2. **Restore script** - Automatic decompression, decryption, preparation
3. **Analysis tool** - Backup chain verification and integrity checking

**Key features:**

- Automatic tool detection (MySQL vs Percona vs MariaDB vs Galera)
- Incremental backup chains that track relationships
- Compression and encryption built-in
- Smart retention (never deletes a chain with recent incrementals)
- Point-in-time recovery (restore up to any incremental in the chain)
- Local or S3 storage modes
- Dry-run support for safety

The full implementation is open source:
[github.com/floadsio/syseng/tree/main/mysql/xtrabackup-s3](https://github.com/floadsio/syseng/tree/main/mysql/xtrabackup-s3)

**Workflow:**

```
Sunday:   Full backup + cleanup of orphaned chains
Mon-Sat:  Incremental backups every 6 hours
Weekly:   Analyze all backup chains
Monthly:  Verify integrity of recent backups
```

**Result:**

TB-scale MySQL with reliable backups. Recovery testing works. The team sleeps.

## What Changed Operationally

**Before (Kubernetes):**
- Backup operations were uncertain
- Recovery testing took days
- Backup failures were common
- Team spent time debugging Kasten
- Scaling meant hoping backups still worked

**After (VMs + XtraBackup):**
- Backups complete reliably every 6 hours
- Recovery testing takes hours, not days
- Backup failures are rare (and when they happen, they're database problems, not infrastructure)
- Team maintains simple shell scripts, not Kubernetes operators
- Scaling means adding more incremental backups, not rearchitecting

## The Real Lesson

**Kubernetes is great for compute. It's not a data platform.**

When people say "run everything on Kubernetes," they usually mean "run stateless services on Kubernetes."

Databases are different. They're stateful, they're I/O sensitive, and they have specialized operational requirements.

For databases:

- Use managed services (RDS, Cloud SQL) if available
- Use dedicated infrastructure (VMs with proper backup tooling) if not
- Don't use container orchestration platforms designed for stateless services

This isn't a failure of Kubernetes. It's recognizing the right tool for the job.

## What We Learned

1. **Backup strategy determines infrastructure choices**
   - For large databases, backup design is the primary constraint
   - Kubernetes has no good answer for this
   - Database-native tools (XtraBackup) exist for a reason

2. **Scaling exposes wrong choices**
   - At GB scale, Kubernetes for MySQL is fine
   - At TB scale, it's untenable
   - If you find yourself fighting your infrastructure, you have the wrong tool

3. **Operational simplicity matters**
   - Simple scripts > complex operators
   - Database-native tools > infrastructure-generic tools
   - Team can maintain shell scripts; Kubernetes expertise isn't required

4. **Cloud-native ≠ Kubernetes**
   - Cloud-native means using cloud resources effectively
   - Sometimes that's Kubernetes, sometimes it's managed services, sometimes it's VMs
   - Choose the tool that solves your problem with the least operational overhead

## The Trade-off

**What we gave up:**

- Cluster elasticity (VMs are fixed-size)
- The feeling of being "modern" (VMs feel old-fashioned)

**What we got:**

- Infrastructure as code via Ansible (VMs fully automated and reproducible)
- Reliable backups that actually work
- Point-in-time recovery capability (validated with full recovery testing for compliance)
- Team doesn't spend cycles on infrastructure
- Operational simplicity
- Significantly fewer production incidents related to data loss

The trade-off was worth it.
