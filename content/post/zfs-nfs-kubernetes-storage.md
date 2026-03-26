+++
title = "ZFS + NFS as Kubernetes Storage: Point-in-Time Recovery Without the Cost"
date = "2026-04-15T00:00:00-00:00"
draft = true
slug = "zfs-nfs-kubernetes-storage"
tags = ["kubernetes", "storage", "infrastructure", "zfs", "nfs", "freebsd", "backup-series", "point-in-time-recovery", "reliability", "cost-efficiency"]
summary = "Kubernetes storage is either ephemeral or expensive. We run a FreeBSD VM with ZFS + NFS in the same VPC as our K8s cluster. Hourly snapshots protect your data. Fast recovery. No vendor lock-in. Here's how we do it."
comments = true
image = "img/zfs-nfs-header.jpg"
+++

Kubernetes storage is a problem with no good solution.

Your options: store data in containers (lost on crash), use cloud storage (vendor lock-in, expensive), or manage your own NFS server (who wants that?).

We built the fourth option: **A FreeBSD VM running ZFS + NFS in the same VPC as our K8s cluster**.

It's not flashy. It's not a managed service. But it works.

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

Best ZFS support (it was invented for Solaris, FreeBSD is the standard). Minimal overhead. NFS performance is excellent. Runs on standard cloud VMs without bloat.

Together: **automated data protection without the cost**.

## Real Architecture

Everything runs in the same VPC. Here's the three-layer setup:

**Layer 1: Kubernetes Clusters (Stage + Prod) - Cloud VMs**

Apps request storage via PersistentVolumeClaim. Kubernetes mounts standard NFS storage class. Pods connect to the FreeBSD storage VM via internal VPC network.

**Layer 2: Storage Node (Single FreeBSD VM in same VPC)**

- VM specs: 2 vCPU, 4GB RAM (minimal, but sufficient for NFS)
- Export: `/export/k8s-data` via NFS
- Filesystem: `zroot/k8s-data` (ZFS with snapshots)
- Snapshots: Hourly (automatically created, pruned after 3 days)
- Protection: Point-in-time recovery ready
- Access: Low-latency internal NFS connection (same VPC)

**Layer 3: Offsite Replication (External Storage)**

- Target: rsync.net or similar external offsite storage
- Replication: Nightly incremental sends over WireGuard/SSH
- Retention: Full dataset + 30 days of snapshots
- Recovery: Clone offsite snapshot to new volume, mount as new PV

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
Too late for rm -r recovery
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

We use [zfs-autobackup](https://github.com/jimklimov/zfs-autobackup), a Python tool that handles the entire workflow automatically.

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

zfs-autobackup supports two modes:

**Push mode** (run on source, push to destination):
```bash
zfs-autobackup \
  --clear-mountpoint \
  --destroy-incompatible \
  --ssh-source storage-node.internal \
  k8s-offsite \
  zroot/remote/k8s-data
```

**Pull mode** (run on destination, pull from source):
```bash
zfs-autobackup \
  --clear-mountpoint \
  --destroy-incompatible \
  storage-node.internal \
  zroot/remote/k8s-data
```

Use push mode if the storage node initiates backups. Use pull mode if the offsite target initiates backups (more common in air-gapped environments).

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

## Maintenance & High Availability

**Keeping FreeBSD Updated**

FreeBSD is trivial to maintain. Security updates are seamless:

```bash
freebsd-update fetch install
```

No restarts required for most updates. Reboot only when kernel patches land. A single storage node is simpler to manage than a managed service—no vendor updates to wait for, no API changes to track.

**High Availability (Optional)**

For production HA, replicate to a second storage node in the same VPC:

```bash
# On second storage node, pull from primary via zfs-autobackup
zfs-autobackup \
  --destroy-incompatible \
  primary-storage.internal \
  zroot/k8s-data-replica
```

Then use **CARP** (Common Address Redundancy Protocol) on both FreeBSD storage nodes to provide a virtual IP (VIP) for NFS access. K8s storage class points to the VIP instead of a single node. Sub-second failover is automatic—if the primary storage node fails, CARP promotes the replica and NFS requests resume on the secondary without pod restarts.

This gives you true HA storage with automatic failover, all managed with open-source tools.

## Real Numbers

**Our setup:**

- **Data:** 1TB active K8s storage
- **Snapshot storage:** 50GB additional (snapshots use copy-on-write)
- **Hourly snapshots:** 72 retained (3 days)
- **Daily replication:** <10 minutes (incremental only)
- **Recovery time:** <5 min (clone snapshot)

**Cost:**

- FreeBSD VM (2 vCPU, 4GB RAM, 2TB storage): ~$30-40/month
- Offsite storage: ~$50/month (rsync.net)
- Kubernetes CSI driver: Free (open source)
- **Total: ~$80-90/month**

**vs. AWS EBS + backup solution:**

- EBS: $300+/month for 1TB
- Backup service: $200+/month
- **Total: $500+/month**

**Savings: ~$410-420/month vs managed solutions**

## Why This Matters

You're not betting on a vendor's backup strategy. You control recovery.

You're not paying per-gigabyte for enterprise backup software. You're using filesystem-level snapshots.

You're not hoping the cloud provider backs up your data correctly. You're replicating to your own offsite target.

## Trade-offs

This isn't perfect for everyone:

**Good for:**
- Self-managed Kubernetes (cloud VMs, on-premises, or hybrid)
- Private clusters where you control the infrastructure
- Applications that tolerate occasional brief NFS hiccups
- Teams comfortable managing their own storage layer

**Not ideal for:**
- Managed Kubernetes services (GKE, EKS) where you can't provision your own VMs
- Multi-region requirements (NFS latency across regions)
- Applications requiring sub-millisecond latency (use local storage)
- Completely serverless workloads (you need to manage the storage VM)

## The Lesson

Don't assume managed services or enterprise backup tools are the only way. A single FreeBSD VM with ZFS + NFS gives you better data protection and costs 3-4x less than managed cloud storage.

The key: understand what you're protecting (point-in-time data), choose tools that do that well (ZFS snapshots), and automate everything. Run it in the cloud, on-premises, or hybrid—the architecture works anywhere as long as your K8s cluster and storage node can talk on low-latency networks.

That's how you survive.
