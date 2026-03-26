+++
title = "ZFS + NFS as Kubernetes Storage: Point-in-Time Recovery Without the Cost"
date = "2026-04-15T00:00:00-00:00"
draft = true
slug = "zfs-nfs-kubernetes-storage"
tags = ["kubernetes", "storage", "infrastructure", "zfs", "nfs", "freebsd", "backup-series", "point-in-time-recovery", "reliability", "cost-efficiency"]
summary = "Kubernetes storage is either ephemeral or expensive. We run FreeBSD + ZFS + NFS as our persistent storage backend for K8s. Hourly snapshots protect your data. Fast recovery. Zero vendor lock-in. Here's how we do it."
comments = true
image = "img/zfs-nfs-header.jpg"
+++

Kubernetes storage is a problem with no good solution.

Your options: store data in containers (lost on crash), use cloud storage (vendor lock-in, expensive), or manage your own NFS server (who wants that?).

We built the fourth option: **FreeBSD + ZFS + NFS as our K8s storage backend**.

It's not flashy. It's not cloud-native. But it works.

## The K8s Storage Problem

When you run stateful applications in Kubernetes, you hit a wall:

**Option 1: Ephemeral Storage (emptyDir)**
- Data lost when pod crashes
- No recovery
- Use case: scratch space, caching

**Option 2: Cloud Storage (EBS, Persistent Disks)**
- Works well at first
- Costs $500+ per month at scale
- Vendor lock-in (can't move easily)
- No snapshots for point-in-time recovery
- You're trusting someone else's backups

**Option 3: Self-Hosted Storage (NFS)**
- Requires setup, maintenance, monitoring
- No data protection (corruption destroys everything)
- Backups are separate problem
- Recovery is manual and slow

**Real scenario:** We have multiple Kubernetes clusters (stage + prod) that need:
- Persistent storage for databases, documents, caches
- Point-in-time recovery (crash/corruption/accidental delete)
- Offsite backups for compliance
- Monthly budget < $500

Cloud storage didn't fit. Manual NFS didn't fit. So we built something better.

## Why ZFS + NFS

We chose this combination because each layer does one thing well:

**NFS - Standard Kubernetes protocol**

Works across cluster boundaries. No fancy CSI drivers required. Kubernetes understands it natively. Can mount same storage from multiple clusters simultaneously.

**ZFS - Data protection layer**

Snapshots are point-in-time views (zero-copy, instant creation). Incremental replication for offsite backups. Copy-on-write means snapshots don't consume significant space. Transparent compression and deduplication. Automatic corruption detection.

**FreeBSD - Stable storage platform**

Best ZFS support (it was invented for Solaris, FreeBSD is the standard). Minimal overhead. NFS performance is excellent. Runs on cheap hardware (we use standard servers).

Together: **automated data protection without the cost**.

## Real Architecture

Here's the three-layer architecture:

**Layer 1: Kubernetes Clusters (Stage + Prod)**

Apps request storage via PersistentVolumeClaim. Kubernetes mounts standard NFS storage class.

**Layer 2: Storage Nodes (FreeBSD + ZFS)**

- Export: `/export/k8s-data` via NFS
- Filesystem: `zroot/k8s-data`
- Snapshots: Hourly (automatically created, pruned after 3 days)
- Protection: Point-in-time recovery ready

**Layer 3: Offsite Replication**

- Target: rsync.net or similar offsite storage
- Replication: Nightly incremental sends
- Retention: Full dataset + 30 days of snapshots
- Recovery: Clone offsite snapshot to new volume

## How Snapshots Protect K8s Data

A snapshot is a point-in-time view of the filesystem. It's not a copy.

**Key insight:** Creating a snapshot takes microseconds and costs almost nothing in storage space. You can have 365+ snapshots per dataset and use nearly the same space as the original.

**Real scenarios:**

### Scenario 1: Pod Crashes, Data Lost

```
Pod crashes → container restarts → mounts same NFS
Data is intact (stored on NFS, not in container)
Application sees its data, continues normally
```

**Recovery time:** Zero (automatic)

### Scenario 2: Application Writes Bad Data

```
App bug corrupts database files
You notice 30 minutes later
Run: zfs rollback zroot/k8s-data@2026-04-15-09-00
Database returns to clean state from 30 min ago
```

**Recovery time:** <5 minutes

### Scenario 3: Accidental Delete

```
Human error deletes important files
Too late for `rm -r` recovery
Clone offsite snapshot to recovery volume:
  zfs clone zroot/remote/k8s-data@2026-04-14 \
    zroot/recovery
Mount recovery dataset as new PV
App restores files from 24 hours ago
```

**Recovery time:** 5-10 minutes

### Scenario 4: Entire Cluster Loss

```
Cluster hardware fails
All pods gone, all PVs gone
Have offsite ZFS snapshot from this morning
Clone offsite snapshot:
  zfs clone remote/k8s-data@2026-04-15 \
    zroot/recovery
Create new PV pointing to recovery dataset
Redeploy pods pointing to new PV
Data restored, minimal loss
```

**Recovery time:** 15-30 minutes (depends on cluster redeploy speed)

## Implementation

### Storage Class Configuration

Standard Kubernetes NFS storage class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: storage-node.internal  # Your FreeBSD storage nodes
  share: /export/k8s-data
  nfsvers: 3
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

Apps request storage normally:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: nfs-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### Automatic Snapshots with zfs-autobackup

We use [zfs-autobackup](https://github.com/jimklimov/zfs-autobackup), a Python tool that handles the entire workflow automatically:

**Step 1: Tag datasets**

```bash
zfs set autobackup:k8s-offsite=true zroot/k8s-data
```

**Step 2: Run zfs-autobackup daily**

The tool:
- Creates snapshots automatically (tagged with the backup label)
- Sends full dataset on first run
- Sends only incremental changes on subsequent runs
- Destroys incompatible snapshots automatically
- Maintains retention policy (10 recent + daily + weekly + monthly)

```bash
zfs-autobackup \
  --clear-mountpoint \
  --destroy-incompatible \
  --ssh-source storage-node.internal \
  k8s-offsite \
  zroot/remote/k8s-data
```

Run this daily via cron (3:15 AM, for example):

```bash
15 3 * * * keep /usr/local/bin/zfs-autobackup-run.sh >> /var/log/zfs-autobackup.log 2>&1
```

**Real performance:**

- Initial full backup (100GB): ~47 hours
- Daily incremental: <10 minutes
- Bandwidth: Efficient (only changed blocks sent)
- Retention: Automatic (10 snapshots kept + time-based rotation)

### What zfs-autobackup Does

- **Automation:** No manual snapshot commands needed
- **Idempotent:** Safe to run multiple times
- **Incremental:** Only changed blocks transferred after first sync
- **Retention:** Automatic snapshot pruning (keeps N recent + age-based)
- **Resume-safe:** Tracks which snapshots were sent, safe to interrupt/resume

## Real Numbers

**Our setup:**

- **Data:** 1TB active K8s storage
- **Snapshot storage:** 50GB additional (snapshots use copy-on-write)
- **Hourly snapshots:** 72 retained (3 days)
- **Daily replication:** <10 minutes (incremental only)
- **Recovery time:** <5 min (clone snapshot)

**Cost:**

- FreeBSD storage nodes: $2-3k hardware
- Offsite storage: ~$50/month (rsync.net)
- Kubernetes CSI driver: Free (open source)

**vs. AWS EBS + backup solution:**

- EBS: $300+/month for 1TB
- Backup service: $200+/month
- **Total: $500+/month**

**Our monthly cost: ~$50**

## Why This Matters

You're not betting on a vendor's backup strategy. You control recovery.

You're not paying per-gigabyte for enterprise backup software. You're using filesystem-level snapshots.

You're not hoping the cloud provider backs up your data correctly. You're replicating to your own offsite target.

## Trade-offs

This isn't perfect for everyone:

**Good for:**
- On-premises Kubernetes
- Private clusters (not cloud-hosted)
- Applications that tolerate occasional brief NFS hiccups
- Teams comfortable running their own storage

**Not ideal for:**
- Managed Kubernetes (can't provision your own storage nodes)
- Multi-region requirements (NFS latency over WAN)
- Applications requiring sub-millisecond latency (use local storage)

## The Lesson

Don't assume cloud storage or enterprise backup tools are the only way. Sometimes the simplest infrastructure—FreeBSD, ZFS, NFS—gives you better data protection and costs less.

The key: understand what you're protecting (point-in-time data), choose tools that do that well (ZFS snapshots), and automate everything.

That's how you survive.
