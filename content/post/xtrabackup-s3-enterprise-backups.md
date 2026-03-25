+++
title = "XtraBackup + S3: Enterprise Database Backup Strategy"
date = "2026-04-01T00:00:00-00:00"
draft = false
slug = "xtrabackup-s3-enterprise-backups"
tags = ["database", "backups", "mysql", "mariadb", "s3", "operations", "infrastructure"]
summary = "Database backups at TB scale require incremental backups with intelligent chain management. Learn how XtraBackup + S3 storage with chain-aware retention solves point-in-time recovery without the cost of managed backup solutions. Includes automation patterns used in production."
comments = true
image = "img/xtrabackup-header.jpg"
+++

In the previous post, we moved MySQL off Kubernetes when backups became the blocker at TB scale. But moving to VMs was just the first step.

Now came the real problem: how do you actually back up a TB-scale database efficiently?

## The Backup Problem at Scale

Full backups don't work when your database is large:

- A 500GB MySQL backup takes hours
- Running that every day is wasteful (most data hasn't changed)
- Running it weekly leaves a 6-day recovery window (unacceptable)
- Storage costs explode (full backup every day = 500GB × 7 = 3.5TB/week)

You need incremental backups. But incremental backups are complicated:

- Every incremental depends on its base backup
- Delete the base = all incrementals become worthless
- Most backup tools don't understand these dependencies
- People accidentally delete base backups while incrementals still reference them

The second problem: Point-in-time recovery (PITR) requires the complete chain.

If you want to restore to "2:45 PM on March 15", you need:
1. The full backup from whenever it was taken
2. Every incremental backup up to 2:45 PM
3. Binary logs to replay until the exact timestamp

Miss one incremental? Recovery fails.

## Why XtraBackup + S3

We chose this approach for three reasons:

**1. Chain-Aware Retention**

Most backup tools only look at backup age: "this backup is 8 days old, delete it." They don't check if other backups depend on it.

XtraBackup includes all metadata in filenames and tracking. We built retention logic that says: "only delete a backup if nothing depends on it AND it's older than 7 days."

**2. Local-First Recovery**

Keeping the most recent backups locally on SSD means recovery is seconds, not minutes. S3 is the archive copy for compliance and offsite resilience.

**3. Cost**

A managed backup solution (AWS Backup, Commvault) costs $150+/month for 150GB of backups. Our solution:

- S3 storage: ~$3/month
- Infrastructure: already have the backup host
- Automation: scripts handle retention and verification

**4. Multi-Database Support**

The same scripts work for MySQL 8.0, Percona, MariaDB standalone, and MariaDB Galera clusters. The tool auto-detects which to use.

## How It Works

### Three Scripts

We use three scripts from the [floads/syseng repository](https://github.com/floadsio/syseng/tree/main/mysql/xtrabackup-s3):

**xtrabackup-s3.sh** — Takes backups
```bash
# Full backup (with cleanup of old chains)
xtrabackup-s3.sh full --cleanup

# Incremental backup (automatically chains to latest full)
xtrabackup-s3.sh inc

# Sync all backups to S3
xtrabackup-s3.sh sync-all
```

**xtrabackup-s3-restore.sh** — Restores backups
```bash
# Restore single backup
restore-backup 2026-03-15_08-57-49_full_1750928269

# Restore entire chain (full + all incrementals)
restore-chain 2026-03-15_08-57-49_full_1750928269

# Point-in-time recovery
restore-chain full_name incremental_name_at_target_time
```

**xtrabackup-s3-check.sh** — Analyzes chain health
```bash
# Show backup storage breakdown
analyze-chains

# List all backups with sizes
list

# Verify entire chain integrity
check backup-name
```

### Backup Naming & Chain Management

Backups are named to encode their relationships:

- Full backup: `YYYY-MM-DD_HH-MM-SS_full_TIMESTAMP`
- Incremental: `inc_base-BASE_TIMESTAMP_inc_TIMESTAMP`

Example chain:
```
2026-03-15_02-00-00_full_1741000000         (full)
  └── inc_base-1741000000_inc_1741018000    (inc 5 hours later)
  └── inc_base-1741000000_inc_1741036000    (inc 10 hours later)
  └── inc_base-1741000000_inc_1741054000    (inc 15 hours later)
```

To restore to 2:45 PM, you apply the full backup + the third incremental + replay binary logs for 45 minutes.

All backups are encrypted (AES-256) and compressed (zstd).

### Automation Pattern

**Weekly schedule:**
- Sunday 2 AM: Full backup, delete chains older than 7 days
- Monday-Saturday every 6 hours: Incremental backup
- Weekly: Run chain analysis, verify storage breakdown

**Local-first approach:**
1. Write backup to `/mnt/backup` (SSD, fast)
2. Verify backup integrity locally
3. Sync to S3 asynchronously (happens in background)
4. Recovery checks local first, falls back to S3

**Chain retention logic:**
```
Only delete a backup if:
  AND all downstream backups are also >7 days old
  AND no incremental backups reference it
```

This prevents orphaned incrementals (incrementals without a base) while still cleaning up old data.

## Real Numbers

### Storage Example

A production setup backing up MariaDB Galera cluster:

| Metric | Value |
|--------|-------|
| Total full backups | 5 |
| Total incremental backups | 23 |
| Local storage | 15GB |
| S3 storage | 142GB |
| Total backup space | 157GB |
| Compressed + encrypted | Yes |

### Recovery Time

| Scenario | Time |
|----------|------|
| Full restore from local SSD | <5 minutes |
| Full restore from S3 | 15-20 minutes |
| Point-in-time recovery | Add <5 minutes for replay |

### Cost Comparison

| Solution | Cost | Storage | PITR | Chains |
|----------|------|---------|------|--------|
| AWS Backup | ~$150/month | 150GB | Yes | No |
| This solution | ~$3/month | 150GB | Yes | Yes |

The human cost is minimal because:
- Backups run via cron (automated)
- Retention is automatic (chain-aware logic)
- Recovery procedures are tested monthly

## Operations in Production

### Monitoring Backups

```bash
# Check if backups are running on schedule
tail -50 /var/log/xtrabackup-backup.log

# Show chain breakdown and storage usage
/usr/local/bin/xtrabackup-s3-check.sh analyze-chains

# List all available backups
/usr/local/bin/xtrabackup-s3-check.sh list
```

### Manual Backup Trigger

```bash
# Force full backup immediately (instead of waiting for Sunday)
/usr/local/bin/xtrabackup-s3.sh full --cleanup

# Force incremental backup
/usr/local/bin/xtrabackup-s3.sh inc

# Manually sync to S3
/usr/local/bin/xtrabackup-s3.sh sync-all
```

### Recovery Procedure

```bash
# List available backups
/usr/local/bin/xtrabackup-s3-check.sh list

# Restore entire chain to /mnt/restore
/usr/local/bin/xtrabackup-s3-restore.sh restore-chain \
  2026-03-15_02-00-00_full_1741000000

# Verify restored data
ls -lah /mnt/restore/
file /mnt/restore/*

# Start MySQL from restored data
mysql -u root -p --socket=/mnt/restore/mysql.sock
```

### Chain Integrity Checks

```bash
# Verify entire chain is present and uncorrupted
/usr/local/bin/xtrabackup-s3-check.sh check \
  2026-03-15_02-00-00_full_1741000000

# This verifies:
# - Base full backup exists and is readable
# - All incremental backups reference valid base
# - All files have valid checksums
# - Compression and encryption integrity
```

## What We Learned

**1. Incremental backups require discipline.**

You can't just delete backups arbitrarily. One mistake (deleting a base full backup) orphans all downstream incrementals. We built chain-aware retention to make this automatic.

**2. Naming matters.**

By encoding the backup relationships in the filename (`inc_base-TIMESTAMP_inc_TIMESTAMP`), we can reconstruct the chain automatically. This is simpler than external database tracking.

**3. Test recovery procedures.**

We restore a random backup monthly from S3 (not local) to verify:
- S3 uploads worked correctly
- Restore procedures actually work
- Recovery time estimates are realistic

Finding recovery failures in production is unacceptable.

**4. Local-first + S3 archive is the sweet spot.**

- Local: Fast recovery when you need it (minutes)
- S3: Immutable archive for compliance (can't be accidentally deleted)
- Costs: 1/3 of managed solutions
- Automation: Fully scripted, no manual intervention

**5. Automate retention or humans will skip backups.**

When retention is manual ("someone remember to delete old backups"), it doesn't happen. Automated chain-aware retention is reliable.

## Next: The Complete Backup Strategy

This post shows the database-specific solution. In the next post, we'll show the complete three-layer backup strategy:

1. **ZFS snapshots** for infrastructure (fast, incremental)
2. **Restic backups** for immutable offsite copies
3. **S3 archive** for long-term compliance

Together, these three layers give you defense-in-depth: fast local recovery, offsite immutable backups, and compliance-ready archives.

---

**GitHub Implementation:** [floads/syseng — mysql/xtrabackup-s3](https://github.com/floadsio/syseng/tree/main/mysql/xtrabackup-s3)

Includes:
- `xtrabackup-s3.sh` — Backup script with chain-aware retention
- `xtrabackup-s3-restore.sh` — Restore entire chains or PITR
- `xtrabackup-s3-check.sh` — Analyze chains and verify integrity
- Cron job examples and automation patterns
