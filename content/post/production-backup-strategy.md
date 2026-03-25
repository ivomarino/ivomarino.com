+++
title = "Production Backup Strategy: Three Layers of Defense"
date = "2026-05-05T00:00:00-00:00"
draft = false
slug = "production-backup-strategy"
tags = ["backup-series", "infrastructure", "backups", "operations", "zfs", "restic", "s3", "disaster-recovery", "compliance", "cost-efficiency"]
summary = "Single backup tool doesn't work at production scale. Learn the three-layer backup architecture combining ZFS snapshots (local fast recovery), Restic backups (offsite immutable), and S3 archives (compliance). Achieves defense-in-depth with 1TB data, point-in-time recovery, and only $50/month cost."
comments = true
image = "img/production-backup-header.jpg"
+++

You're managing a production platform with TB-scale data. Database backups, filesystem data, configuration files—everything needs to be protected.

Single-tool backup approaches fail:

- **ZFS snapshots alone:** Fast local recovery, but no offsite copy if the storage fails
- **Restic without snapshots:** Everything gets scanned daily (expensive at 1TB)
- **S3 backups without local snapshots:** Slow recovery (must download and decompress)

The answer is all three, working together.

## The Three-Layer Architecture

```
Layer 1: LOCAL SNAPSHOTS          Layer 2: OFFSITE STORAGE         Layer 3: COMPLIANCE
════════════════════════════════════════════════════════════════════════════════════

Production NAS (FreeBSD)           Offsite rsync.net                S3 Object Storage
─────────────────────────────────────────────────────────────────────────────────────
ZFS snapshots (hourly)      →      Receive ZFS datasets      →      Restic snapshots
  • Automatic retention     →      (with full retention)     →      • 8-11 month retention
  • Point-in-time recovery →                                 →      • Keep policy
  • Local to NAS            →                                 →      • Immutable archive

Restic backups (2x daily)
  • Stage + prod sets
  • Tag-based exclusions
  • Retention policy


Recovery Path:
─────────────
Fast (minutes):    ZFS clone → Mount → Done
Medium (hours):    Restic restore → Download → Verify
Slow (compliance): S3 download → Verify → Archive
```

## Layer 1: ZFS Snapshots (Local & Offsite)

**Responsibility:** Fast recovery, hourly point-in-time capability

Snapshots are created automatically on production NAS systems. zfs-autobackup replicates them hourly to an offsite rsync.net target.

**Real numbers:**
- Snapshots created hourly (automatic, background)
- 101GB dataset: 47-hour initial sync, then incremental minutes
- Storage: ~500GB offsite (retention of 10+ snapshots)
- Recovery: Clone offsite snapshot, mount on target, done in <5 minutes

**When to use:** "I need data from 3 hours ago right now."

## Layer 2: Restic Backups (Offsite Immutable)

**Responsibility:** Offsite immutable copy, deduplication, compliance trail

Restic runs 2x daily on production systems, creating snapshots of:
- `/etc` (system configuration)
- `/mnt/data` (application data)
- `/mnt/data1` (customer databases, sensitive)

Backups use tag-based exclusions (`.nobackup`, `.resticignore` files) to skip non-critical data.

**Retention policy:**

```yaml
keep-last: 3             # Last 3 snapshots always
keep-daily: 5            # 1 per day for 5 days
keep-weekly: 3           # 1 per week for 3 weeks
keep-monthly: 6          # 1 per month for 6 months
keep-yearly: 1           # 1 per year forever
```

**Real numbers:**
- Primary data: ~1TB
- Restic repository (deduplicated): ~100GB
- 2x daily backups (stage + prod on different schedules)
- Backup windows: 02:00 UTC (stage), 04:00 UTC (prod)
- CPU: Minimal (deduplication in background)

**When to use:** "I need to verify data integrity from last week" or "restore customer database from yesterday."

## Layer 3: S3 Archive (Long-Term Compliance)

**Responsibility:** Geo-distributed archive, immutable, compliance-ready

Restic snapshots are pushed to S3-compatible object storage via autorestic-rclone:

1. Mount S3 buckets locally via rclone FUSE
2. Run restic forget/prune (applies retention)
3. Automatically unmount

This runs post-backup once Restic completes locally.

**Buckets:**
- 8 stage buckets (one per environment)
- 3 prod buckets (production, sensitive data, archive)
- Total: ~150GB compressed snapshots

**When to use:** "Need to audit what happened 6 months ago" or "annual compliance archive."

## Why All Three

### ZFS Snapshots
✅ Fastest recovery (local, cloning is instant)
✅ Point-in-time granularity (hourly)
✅ Minimal offsite bandwidth (incremental)
❌ No protection if storage fails (needs backup copy)
❌ Requires FreeBSD/ZFS knowledge

### Restic Backups
✅ Offsite immutable copy (can't be accidentally deleted)
✅ Deduplication saves space (60%+ savings)
✅ Point-in-time recovery
✅ Audit trail (retention policy documented)
❌ Slower restore (must decompress)
❌ CPU overhead for deduplication

### S3 Archive
✅ Geo-distributed (inherent resilience)
✅ Long-term retention (1+ years)
✅ Immutable (compliance requirement)
✅ Cheap per GB
❌ Slowest to restore
❌ No local advantage

**Together:** You have fast recovery (ZFS), offsite protection (Restic), and compliance archive (S3). Defend in depth.

## Real Production Deployment

### Storage Capacity

- Primary NAS: ~1TB active data (customer databases + files)
- Offsite storage (rsync.net): ~500GB ZFS datasets + Restic snapshots
- S3 object storage: ~150GB compressed snapshots

### Backup Windows

- Stage backups: 02:00 UTC (low traffic)
- Prod backups: 04:00 UTC (post-maintenance)
- ZFS snapshots: Hourly (background)

### Recovery Time Objectives (RTO)

| Scenario | Time | Trigger |
|----------|------|---------|
| ZFS clone | <5 min | Need local snapshot |
| Restic restore | 30-60 min | Need offsite data |
| S3 restore | 2-4 hours | Compliance/audit |

### Monitoring

```bash
# Check ZFS replication
zfs list -t snapshot -S creation | head -20

# Check Restic snapshots
restic snapshots --repo /mnt/restic-prod | tail -5

# Check S3 sync status
ls -lah /mnt/restic-prod/

# Check logs
tail -100 /var/log/autorestic-prod.log
```

## Known Issues & Solutions

### Issue 1: Stuck Backup Locks

**Problem:** autorestic-rclone.sh runs as unprivileged user, can't unmount S3 buckets. Lock file persists, subsequent backups blocked.

**Solution:**
```bash
# Add to sudoers
backup_user ALL=(root) NOPASSWD: /sbin/umount, /sbin/mount
```

**Lesson:** Automation scripts need escape hatches for edge cases.

### Issue 2: ZFS Hold Release

**Problem:** If transfer fails, ZFS holds prevent cleanup. Orphaned snapshots remain.

**Solution:** Use `--destroy-missing` in zfs-autobackup to prune orphaned snapshots automatically.

## Why This Matters

This isn't theoretical. These are patterns that work at scale:

1. **Defense-in-depth:** Each layer protects against different failure modes
2. **Cost efficiency:** $50/month total vs $500+ for managed backup solutions
3. **Operational simplicity:** ZFS and Restic are automated; S3 is immutable (can't accidentally delete)
4. **Compliance-ready:** Retention policies documented, audit trail built-in, geographic distribution
5. **Recovery tested:** Monthly test restores from S3 (not local) verify procedures work

## Operations Checklist

- [ ] ZFS snapshots created hourly (automated)
- [ ] Offsite replication running 2x daily (automated)
- [ ] Restic backups running on schedule (automated)
- [ ] S3 sync running post-backup (automated)
- [ ] Weekly: Review backup logs for errors
- [ ] Monthly: Test restore from S3 (not local)
- [ ] Quarterly: Full recovery procedure walkthrough
- [ ] Annually: Audit retention policies

## Getting Started

1. **Set up ZFS:**
   - Tag datasets: `zfs set autobackup:offsite=true zroot/data`
   - Deploy zfs-autobackup via cron

2. **Set up Restic:**
   - `restic init` to create repository
   - Configure retention policy in autorestic config
   - Deploy backup runner via cron

3. **Set up S3 Archive:**
   - Mount S3 buckets via rclone
   - Push Restic snapshots to S3
   - Deploy sync script post-backup

Each layer is independent and can be deployed separately, but together they provide complete protection.

---

**Next:** We'll show how to apply this same pattern to S3 bucket backups (backing up your backups).
