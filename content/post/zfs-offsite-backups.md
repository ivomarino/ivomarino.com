+++
title = "ZFS Offsite Backups on FreeBSD: Incremental Replication at Scale"
date = "2026-05-01T00:00:00-00:00"
draft = false
slug = "zfs-offsite-backups"
tags = ["backup-series", "infrastructure", "backups", "operations", "zfs", "freebsd", "replication", "incremental-backup", "point-in-time-recovery", "automation"]
summary = "ZFS snapshots are the foundation for efficient backups. Learn how to replicate them to offsite targets at scale using zfs-autobackup, from simple pairs (lab scale) to hub-and-spoke architectures (production). Includes real numbers and deployment patterns for 1TB+ datasets with minute-level incremental backups."
comments = true
image = "img/zfs-header.jpg"
+++

ZFS snapshots are one of the best tools for backups. They're efficient, point-in-time capable, and idempotent. But most posts only show simple examples: backup one server to one target.

This is about real-world patterns at scale.

## Why ZFS for Backups

ZFS snapshots have two critical advantages:

**1. Incremental by design**

A snapshot is just a point-in-time view of the filesystem. To replicate it, ZFS only sends the changed blocks since the last snapshot—not the entire dataset.

First sync of a 100GB dataset: might take hours on a slow link.

Subsequent daily syncs: minutes (only changed blocks).

**2. Idempotent replication**

Send the same snapshot 100 times, the target gets the exact same result. No corruption, no half-states. Perfect for automation.

## Real Production Patterns

We deploy ZFS snapshots in two patterns:

### Pattern 1: Simple Pair (Lab to Offsite)

**Topology:**
```
Source: Home lab (101GB)
  ↓ (WireGuard tunnel, 600 KB/s)
Target: Offsite hypervisor
```

**Schedule:**
- Snapshot every hour (background)
- Incremental replication nightly (3:15 AM)
- Retention: Keep last 10 snapshots locally, auto-prune older ones offsite
- Recovery time: <5 minutes (clone offsite snapshot)

**Use case:** Single hypervisor, small infrastructure, testing.

### Pattern 2: Hub-and-Spoke (Production)

**Topology:**
```
9 source hypervisors ─┐
                      ├─→ Central offsite target
                     ...
```

**Schedule:**
- Daily snapshots on each source (every 6-12 hours)
- Incremental sync to offsite twice daily (7 AM and 7 PM UTC)
- All sources use same SSH key, push to same offsite target
- Retention: Source keeps 12 hourly + 2 daily; target keeps 7 recent + 1/day/week/month/year
- Storage: 2TB source data → 5TB offsite (with multi-snapshot retention)

**Use case:** Distributed infrastructure, production scale, multiple hypervisors.

## How It Works

We use [zfs-autobackup](https://github.com/jimklimov/zfs-autobackup), a Python tool that manages the entire workflow:

### 1. Tag Your Datasets

Mark which datasets to back up:

```bash
zfs set autobackup:offsite=true zroot/home
zfs set autobackup:offsite=true zroot/bastille
zfs set autobackup:offsite=true zroot/vm
```

### 2. Create Offsite Receiving Datasets

On the target system, prepare to receive snapshots:

```bash
zfs create -o canmount=off zroot/remote
zfs create zroot/remote/virt-01
zfs create zroot/remote/virt-02
# ... (one per source)
```

### 3. Run Automated Replication

**Simple pair (local SSH):**

```bash
zfs-autobackup -v \
  --clear-mountpoint \
  --keep-source=10 \
  --keep-target=10,1d1w,1w1m,1m1y \
  --ssh-source virt.home offsite zroot/remote
```

**Hub-and-spoke (multiple sources):**

For each source, run a script like this twice daily via cron:

```bash
#!/bin/sh
/usr/local/bin/zfs-autobackup -v \
  --clear-mountpoint \
  --clear-refreservation \
  --ignore-transfer-errors \
  --keep-source=12h2d \
  --keep-target=7,1d1w,1w1m,6m1y \
  --destroy-missing 7d \
  --ssh-source keep@10.61.0.10 offsite-virt-05 \
  tank/remote/virt-01
```

Schedule via cron:
```
0 7,19 * * * /usr/local/bin/zfs-autobackup-virt-01.sh
0 7,19 * * * /usr/local/bin/zfs-autobackup-virt-02.sh
# ... (one per source)
```

## Real Numbers

### Lab Deployment (Pattern 1)

- Data: 101GB (user home, jails, VMs)
- Initial sync: **47 hours** (600 KB/s on 10 Mbit WAN)
- Daily incremental: **10-30 minutes** (typical 5% change = 5GB)
- Offsite retention: ~500GB (multiple snapshots)
- Cron window: Usually completes in 20 minutes

### Hub-and-Spoke Deployment (Pattern 2)

- Sources: 9 hypervisors
- Total data: ~2TB
- Change rate per source: 50-200MB daily
- Offsite storage: ~5TB (data + retention)
- Backup cycle: 7 AM + 7 PM UTC, takes 2-3 hours each
- Network: 10 Mbit private link (handles concurrent pulls)

### Bandwidth Reality

| Scenario | Change/Day | Sync Time | WAN Impact |
|----------|-----------|-----------|-----------|
| 100GB at 5% change | 5GB | 2-3 hours | 0.5-0.7 Mbit/s |
| 1TB at 2% change | 20GB | 8-10 hours | 0.2-0.3 Mbit/s |
| 2TB at 5% change | 100GB | 40-50 hours | 0.5-0.7 Mbit/s |

**Key insight:** It's not about total data, it's about change rate. ZFS incremental scaling is beautiful—100GB and 1TB backup times are nearly identical if they have the same change rate.

## Retention Policy

Retention is where most backup solutions fail. Delete the wrong snapshot and you orphan all dependents.

ZFS-autobackup handles this:

```bash
--keep-source=12h2d          # Keep 12 hourly + 2 daily snapshots locally
--keep-target=7,1d1w,1w1m,6m1y  # 7 recent, 1/day/week/month/year
--destroy-missing 7d         # Delete offsite if missing 7+ days
```

**What this means:**

- Source: Keep enough for local recovery (don't fill disk)
- Target: Keep for compliance and point-in-time (7 recent + 1/day/week/month/year)
- Automatic: Both are auto-pruned, no manual snapshot deletion
- Safe: Nothing gets deleted if it still has dependents

## Operational Procedures

**Check backup status:**

```bash
ssh syseng@offsite "tail -50 /var/log/zfs-autobackup.log"
ssh syseng@offsite "zfs list -t snapshot -r zroot/remote/virt-01 | tail -20"
ssh syseng@offsite "grep -i error /var/log/zfs-autobackup.log | tail -10"
```

**Manual trigger (don't wait for cron):**

```bash
ssh syseng@offsite "sudo -u keep /usr/local/bin/zfs-autobackup-run.sh"
```

**Dry-run (see what would happen):**

```bash
ssh syseng@offsite "sudo -u keep /usr/local/bin/zfs-autobackup --dry-run \
  -v --clear-mountpoint --ssh-source virt.home offsite zroot/remote/virt-home"
```

**Recovery procedure:**

```bash
# Find the snapshot you need
ssh syseng@offsite "zfs list -t snapshot -r zroot/remote/virt-01"

# Clone snapshot to a new dataset
ssh syseng@offsite "sudo zfs clone zroot/remote/virt-01@2026-05-01_12-00 \
  zroot/recovery/virt-01"

# Mount and verify
ssh syseng@offsite "mount -t zfs zroot/recovery/virt-01 /mnt/recovery"
ssh syseng@offsite "ls -lah /mnt/recovery/"

# Copy back to source or use as recovery
```

## What We Learned

**1. SSH authentication is the best choice**

No special networking needed. Works across zones, datacenters, VPNs. SSH keys in chezmoi (encrypted), managed through normal DevOps tooling.

**2. Retention policies prevent runaway snapshot growth**

Without policies: 365+ snapshots per year (disk fills).
With policies: 30-50 snapshots per year (manageable).
Automatic pruning is safer than manual.

**3. Cron scheduling matters for multi-source setups**

Stagger sources if possible (avoid thundering herd).
Use monitoring hooks (healthchecks.io, runitor).
Check logs daily first week, then weekly.

**4. Handle-releasing is critical**

If a transfer fails, ZFS holds prevent cleanup.
Use `zfs release` even on error paths.
Use `--destroy-missing` to prune orphaned snapshots.

**5. WireGuard beats dedicated VPN for backups**

Lower latency, lower CPU overhead, simpler configuration.
SSH over WireGuard is rock-solid even with packet loss.

## Getting Started

Install zfs-autobackup from your OS package manager (FreeBSD, Debian, Alpine):

```bash
# FreeBSD
pkg install zfs-autobackup

# Debian/Ubuntu
apt install zfs-autobackup

# Then set up SSH keys and cron
```

For detailed deployment with Ansible, see [flamelet infrastructure repository](https://github.com/floadsio/flamelet-home).

The key insight: ZFS snapshots are incremental by design. Replication is fast, reliable, and scales beautifully to TB-scale datasets. Automation handles retention so you don't have to.

---

**Next in the backup series:** We'll show the three-layer backup strategy combining ZFS snapshots, Restic backups, and S3 archives for complete infrastructure protection.
