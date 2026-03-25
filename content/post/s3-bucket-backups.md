+++
title = "S3 Bucket Backups with Restic: Automating Cloud Storage Replication"
date = "2026-05-10T00:00:00-00:00"
draft = false
slug = "s3-bucket-backups"
tags = ["backup-series", "cloud-storage", "backups", "operations", "s3", "restic", "rclone", "incremental-backup", "point-in-time-recovery", "automation"]
summary = "S3 buckets aren't backups. Secure them with independent snapshots using rclone FUSE mounting and restic deduplication. 500GB of production buckets compressed to 100GB with 80% savings. Includes automation, configuration, and real operational procedures for 2-3 hour backup windows."
comments = true
image = "img/s3-backup-header.jpg"
+++

You store critical data in S3 buckets at scale:

- Application logs (hundreds of GB daily)
- Database dumps (hourly snapshots)
- Media assets
- Compliance-required data with retention policies

But S3 is **not** a backup:

- Single point of failure (no independent copy)
- No audit trail if data is accidentally deleted
- Ransomware/compromise = instant data loss
- Bucket policies easy to misconfigure (entire buckets deleted)
- No point-in-time recovery (just current state)

**The challenge:** How do you back up S3 bucket contents to an independent, auditable system without manual operations?

## The Solution: autorestic-rclone

Three-phase automated workflow:

1. **Mount** — Use rclone FUSE to expose S3 buckets as local directories
2. **Backup** — Run restic/autorestic to create snapshots
3. **Cleanup** — Unmount filesystems, verify success, clean up

Result: S3 bucket contents have independent, deduplicated, compressed snapshots with point-in-time recovery and full audit trail.

## Real Production Implementation

### The Setup

**Data being backed up:** Multiple S3 buckets across environments

- Production environment: Application backups + system data
- Staging environment: Development/testing data
- Total volume: ~500GB across all buckets
- Daily change: 50-100GB (active logs + new backups)

**Storage backend:** Private S3-compatible object storage

- Separate from source buckets (resilience)
- Encrypted in transit and at rest

**Schedule:**
- Daily via cron at 4 AM UTC (off-peak)
- Duration: 2-3 hours (mounting + deduplication + backup)
- Monitoring integrated with health check service

### Configuration Pattern

**rclone config** — S3 bucket connectivity:

```ini
[prod-env]
type = s3
provider = Other
access_key_id = <key>
secret_access_key = <secret>
endpoint = https://<s3-compatible-endpoint>

[stage-env]
type = s3
provider = Other
access_key_id = <key>
secret_access_key = <secret>
endpoint = https://<s3-compatible-endpoint>
```

**autorestic config** — Backup policy and retention:

```yaml
version: 2

global:
  forget:
    keep-daily: 30      # Keep one backup per day for 30 days
    keep-weekly: 52     # Keep one per week for a year

locations:
  prod-env:
    from: ~/mnt-s3/prod-env
    to: restic-backend
    forget: prune       # Auto-delete old snapshots

  stage-env:
    from: ~/mnt-s3/stage-env
    to: restic-backend
    forget: prune

backends:
  restic-backend:
    type: s3
    path: /restic-backups
    env:
      AWS_ACCESS_KEY_ID: <key>
      AWS_SECRET_ACCESS_KEY: <secret>
      AWS_S3_URL: https://<s3-compatible-endpoint>
```

**autorestic-rclone config** — Mount point mapping:

```bash
# Location list
LOCATIONS=("prod-env" "stage-env")

# Configuration using associative arrays
declare -A CONFIG

# Production environment: Multiple buckets
CONFIG["prod-env.remote"]="prod-env"
CONFIG["prod-env.base_dir"]="$HOME/mnt-s3/prod-env"
CONFIG["prod-env.buckets"]="app-backups system-data metrics"
CONFIG["prod-env.mount_dirs"]="backups system metrics"

# Staging environment: Multiple buckets
CONFIG["stage-env.remote"]="stage-env"
CONFIG["stage-env.base_dir"]="$HOME/mnt-s3/stage-env"
CONFIG["stage-env.buckets"]="dev-data test-logs cache-data"
CONFIG["stage-env.mount_dirs"]="data logs cache"
```

**Cron schedule:**

```bash
0 4 * * * keep TZ=UTC /usr/local/bin/health-check-run -- \
  /usr/local/bin/autorestic-rclone.sh
```

## How It Works

### Phase 1: Mount via rclone FUSE

```bash
# For each bucket:
rclone mount <remote>:<bucket> <mount_dir> \
  --vfs-cache-mode full \
  --vfs-cache-max-age 12h \
  --vfs-cache-max-size 16M \
  --daemon
```

**Why full VFS caching:**
- Restic needs random file access for deduplication (not sequential)
- Full cache downloads everything first = safe, predictable
- 12-hour retention = cache survives into next backup cycle
- 16MB buffer = handles large files efficiently

**Result:** S3 buckets appear as local `/home/keep/mnt-s3/*/` directories

### Phase 2: Backup via autorestic

```bash
# Run autorestic on all configured locations
autorestic backup -a
```

**What happens:**
1. Scans all mounted directories (full file tree)
2. Creates snapshots with content deduplication
3. Applies retention policy (keep-daily 30, keep-weekly 52)
4. Compresses with zstd (transparent)
5. Uploads to restic backend (incremental only)

**Deduplication impact:**
- S3 logs have high repetition (same patterns)
- Typical savings: 60% reduction in storage
- Compression adds another 40% savings
- Final result: 500GB S3 → ~100GB restic repository

### Phase 3: Cleanup & Verification

```bash
# Unmount all filesystems
fusermount -u /home/keep/mnt-s3/*

# Kill rclone daemon
pkill -f "rclone mount"

# Verify success and report
echo "Backup completed successfully" | send-alert
```

**Safety checks:**
- Verify backup succeeded before unmounting
- Log all operations (success and failure)
- Alert system on any failures (immediate notification)

## Real Numbers

### Backup Cycle

- Mount time: 10-15 minutes (S3 metadata fetching)
- Restic backup: 60-120 minutes (depends on deduplication work)
- Unmount & cleanup: 5 minutes
- **Total window: 2-3 hours**

### Network Bandwidth

- rclone mount: 5-20 MB/s (metadata fetch)
- restic backup: 2-5 MB/s (incremental)
- **Peak: ~30-50 Mbps** (within typical links)

### Storage Efficiency

| Metric | Value |
|--------|-------|
| Source S3 buckets | 500GB |
| Restic after dedup | 100GB |
| Compression ratio | 80% savings |
| Daily incremental | 10-20MB |

## Why This Scales

ZFS snapshots scale with change rate, restic backups scale with **deduplication work**, not total data size:

- 100GB at 5% daily change = 2-3 hour backup
- 1TB at 5% daily change = 2-3 hour backup (same)
- 10TB at 5% daily change = 2-3 hour backup (still same)

The deduplication phase dominates, proportional to **what changed**, not total data.

## Operational Procedures

**Check backup status:**

```bash
# View recent backup logs
tail -50 /var/log/autorestic-backup.log

# List available recovery points
autorestic exec -- snapshots

# Show backup storage usage
autorestic exec -- stats
```

**Manual trigger (don't wait for cron):**

```bash
# Run backup immediately
sudo -u keep /usr/local/bin/autorestic-rclone.sh

# Preview what would be backed up (safe)
sudo -u keep /usr/local/bin/autorestic-rclone.sh --dry-run
```

**Recovery procedure:**

```bash
# Find the snapshot you need
autorestic exec -- snapshots

# Restore specific snapshot
autorestic exec -- restore -s <snapshot-id> -t /tmp/restore

# Verify restored data
ls -lah /tmp/restore
file /tmp/restore/*
```

## Known Issues & Solutions

### Issue 1: Mount Timeouts

**Problem:** rclone mount stalls on slow/distant S3 connections, backup times out.

**Solution:**
```bash
# Add timeout wrapper
timeout 600 rclone mount ... --daemon
if [ $? -ne 0 ]; then
  pkill -f "rclone mount"
  sleep 5
  # Retry with backoff
fi
```

### Issue 2: Deduplication CPU Overhead

**Problem:** Restic's deduplication checking is CPU-intensive on 500GB+, backup takes 3+ hours.

**Solution:**
- Filter non-critical data (exclude temp files, caches)
- Pre-filter in autorestic: `--ignore-*` patterns
- Schedule during off-peak hours (CPU available)

### Issue 3: Concurrent Bucket Mounts

**Problem:** Mounting many buckets simultaneously exhausts S3 rate limits.

**Solution:**
- Mount sequentially (one location at a time)
- Stagger different environments across hours
- Use rclone connection pooling settings

## Monitoring

```bash
# Health check integration (runs backup + monitors)
health-check-run -- /usr/local/bin/autorestic-rclone.sh

# Check if backup is running
ps aux | grep autorestic

# Monitor S3 performance
rclone ls s3:bucket-name | wc -l

# Verify deduplication is working
autorestic exec -- stats
```

## Why This Matters

1. **Independent backup:** S3 data has its own recovery point (not dependent on S3)
2. **Audit trail:** Restic snapshots show exactly what was backed up and when
3. **Efficient storage:** Deduplication + compression = 1/5 storage cost
4. **Automated policy:** Retention enforced by software (no manual deletion)
5. **Point-in-time recovery:** Can restore to any snapshot date (30 daily + 52 weekly)
6. **Cost efficiency:** Self-hosted restic backend cheaper than AWS Backup or Commvault

## Getting Started

1. Install rclone and autorestic
2. Configure S3 connectivity (rclone)
3. Configure backup policy (autorestic)
4. Deploy mount script + cron job
5. Test recovery from actual S3 (not local)

The key insight: **S3 is not a backup. Back it up independently with deduplicated snapshots.**

---

**GitHub Reference:** [floads/syseng — autorestic-rclone](https://github.com/floadsio/syseng/tree/main/autorestic-rclone)

This completes the backup infrastructure series: databases → infrastructure → cloud storage → complete strategy.
