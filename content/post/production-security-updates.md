+++
title = "Production Security Updates Without Downtime: Debian, FreeBSD, and OpenBSD"
date = "2026-03-28T00:00:00-00:00"
draft = true
slug = "production-security-updates"
tags = ["infrastructure", "operations", "security", "debian", "freebsd", "openbsd", "automation", "platform-engineering"]
summary = "Security patches can't wait, but unplanned reboots break production. The update runbook for Debian, FreeBSD, and OpenBSD — with real scripts, hold patterns, and a zero-downtime cluster update sequence."
comments = true
image = "img/security-updates-header.jpg"
+++

## The Problem

CVEs can't wait. A zero-day lands on Monday morning—you have hours to patch before exploits hit. But unplanned reboots at midnight kill production.

What actually breaks:

- **Kubernetes nodes upgrading Docker mid-cluster:** Container runtime gets yanked, all pods evicted
- **FreeBSD jails losing network:** Reboot without coordination means NFS disconnects, databases go down
- **DRBD torn down:** Storage replication stops, failover doesn't work
- **Storage nodes offline simultaneously:** Everything on the cluster loses disks at once

The pattern is the same: emergency security update → unplanned reboot → multiple cascading failures → 2am pages.

The answer isn't to avoid updates. It's to control *when* you reboot.

## How Often Do Kernel Reboots Actually Happen?

Before we had a process: every time a critical CVE dropped, someone would check exploitability, escalate, and an emergency maintenance window would open 48-72 hours later.

What that actually looked like:
- Monday: CVE published, CVSS 9.2
- Wednesday: emergency change request approved
- Thursday night: late-night maintenance window
- 1am: storage node reboots, one jail doesn't come back
- 2am: pages

After a couple of incidents like that, we stopped reacting per CVE.

**The key insight:** patching the kernel image doesn't affect the running kernel. The new kernel loads on next boot. That gap—between "patch applied" and "reboot happened"—is your breathing room. Use it.

What we actually run now:

| OS | Reboot trigger | Frequency | Notes |
|----|---------------|-----------|-------|
| Debian | Kernel patches via security.debian.org | Monthly maintenance window | Patch lands immediately, reboot batched |
| FreeBSD | freebsd-update base system | 2–4x per year | Packages update anytime, base reboots are infrequent |
| OpenBSD | syspatch between 6-month releases | Per release cycle | Cleanest model—patches arrive infrequently by design |

**Packages update continuously. Kernel reboots batch into windows.**

The one exception: a CVE actively being exploited in the wild. That changes the calculus—immediate patch and fast-track the reboot. But check before you decide. Most critical-looking CVEs aren't exploitable in your environment. Check the exploitability score, check your exposure, then schedule accordingly.

Unplanned reboots at 2am because of CVE anxiety cause more incidents than the CVEs themselves.

## The Three Systems

Most production platforms run a heterogeneous mix:

- **Debian:** Compute nodes, Kubernetes, app servers, load balancers
- **FreeBSD:** Storage, jails (containerization), stateful services, database servers
- **OpenBSD:** Edge, firewalls, proxies, security-critical gateways

Each OS has a different security update model, different patch mechanisms, and different reboot triggers. A one-size-fits-all approach fails.

## Debian: Non-Interactive Updates

Debian is the most flexible. You can update packages without rebooting immediately—the kernel patch stays pending until you're ready.

### The Pattern

```bash
#!/bin/bash
set -e

export DEBIAN_FRONTEND=noninteractive

# Update package lists
apt-get update

# Upgrade packages (non-interactive)
# --force-confold: Keep current config files
# --force-confdef: Accept default on new config files
apt-get upgrade -y \
  -o Dpkg::Pre-Install-Pkgs::=/usr/sbin/etckeeper \
  -o Dpkg::Post-Install-Pkgs::=/usr/sbin/etckeeper \
  --allow-change-held-packages \
  -o Apt::AutoRemove::SuggestsImportant=false

# Enable unattended-upgrades for security channel only
apt-get install -y unattended-upgrades

# Configure for security updates
cat > /etc/apt/apt.conf.d/50unattended-upgrades <<'EOF'
Unattended-Upgrade::Allowed-Origins {
  "${distro_id}:${distro_codename}-security";
};
EOF

echo "Updates complete. Reboot when ready."
```

### Key Insight: Hold Critical Packages

On compute nodes, Docker/Containerd updates are dangerous mid-cluster:

```bash
# Hold packages that shouldn't upgrade without coordination
apt-mark hold docker-ce containerd.io
apt-mark hold kubeadm kubectl kubelet

# Run the standard upgrade
apt-get update && apt-get upgrade -y
```

Hold is critical. It lets you:

1. Update security packages on all nodes (fast, safe)
2. Schedule Docker/Containerd upgrades separately
3. Drain the node, upgrade, rejoin the cluster
4. Repeat per node in a controlled sequence

### The Cluster Window

On Kubernetes:

```bash
# Before upgrade
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Update (packages won't trigger reboot yet)
apt-get update && apt-get upgrade -y

# Unhold if coordinated cluster-wide upgrade
apt-mark unhold docker-ce containerd.io
# ... coordinate with other nodes

# Reboot when all nodes ready
reboot

# Rejoin cluster
kubectl uncordon <node>
```

## OpenBSD: syspatch + pkgupdate

OpenBSD has the cleanest security model: `syspatch` for kernel/base patches, `pkgupdate` for packages.

### The Pattern

```sh
#!/bin/sh
set -e

echo "OpenBSD Security Updates"

# Step 1: Check for patches
echo "Checking for available patches..."
syspatch -l

# Step 2: Apply patches (dry-run first on critical systems)
if [ "$1" = "--dry-run" ]; then
  echo "Dry-run mode: would apply patches"
  exit 0
fi

echo "Applying patches..."
syspatch -a

# Check if reboot is needed
if [ -f /var/run/reboot-required ]; then
  echo "Reboot required. Run 'reboot' when ready."
  exit 0
fi

# Step 3: Update packages
echo "Updating packages..."
pkg_update -a
pkg_add -u

# Check again for reboot requirement
if [ -f /var/run/reboot-required ]; then
  echo "Reboot required after package update."
  exit 0
fi

echo "All updates applied. No reboot required."
```

### Key Insight: Reboot Detection

OpenBSD creates `/var/run/reboot-required` when the kernel is patched. You can:

1. **Check before rebooting:** `[ -f /var/run/reboot-required ] && echo "Reboot needed"`
2. **Run the script again post-reboot:** It detects completion and moves to packages
3. **Handle gracefully:** Never surprise-reboots mid-cluster

This is the best model of the three OSes.

## FreeBSD: Packages, Jails, and Base Updates

FreeBSD is the most complex: packages can be updated like Debian, but jails are isolated containers, and base system (kernel) updates require careful coordination.

### The Pattern

```bash
#!/bin/bash
set -e

echo "FreeBSD Security Updates"

# Dry-run first on critical systems
if [ "$1" = "--dry-run" ]; then
  echo "Dry-run: would update packages"
  pkg audit -r
  pkg update -y -n
  exit 0
fi

# Step 1: Audit for known vulnerabilities
echo "Checking for vulnerabilities..."
pkg audit -r || true

# Step 2: Update packages (non-interactive)
echo "Updating packages..."
pkg update -y
pkg upgrade -y

# Step 3: Update jails independently
# Reduces blast radius—each jail upgraded separately
if command -v bastille &>/dev/null; then
  echo "Updating jails..."

  # List all jails
  for jail in $(bastille list | tail -n +2 | awk '{print $1}'); do
    echo "  Updating jail: $jail"
    bastille pkg "$jail" update -y
    bastille pkg "$jail" upgrade -y
  done
fi

# Step 4: freebsd-update (base system)
# Only run if kernel patches available
echo "Checking for base system updates..."
if freebsd-update cron | grep -q "No updates are available"; then
  echo "  No base updates needed"
else
  echo "  WARNING: freebsd-update requires manual review"
  echo "  Run: freebsd-update fetch && freebsd-update install"
fi

echo "Updates complete."
```

### Why Jails Matter

Jails are FreeBSD's containerization. Updating host + jails simultaneously is dangerous:

```bash
# Update host packages only
pkg update -y && pkg upgrade -y

# Update each jail independently
# Reduces blast radius—if jail fails, host is fine
for jail in app-01 app-02 app-03; do
  bastille pkg "$jail" update -y
  bastille pkg "$jail" upgrade -y
done
```

### Boot Environments for Major Upgrades

For major releases (13.0 → 13.1), use ZFS boot environments:

```bash
# Create a new boot environment (atomic rollback)
bectl create upgrade-candidate

# Activate it (next reboot)
bectl activate upgrade-candidate

# Boot into it
reboot

# If it fails, revert atomically
bectl activate previous-environment
reboot
```

## The Cluster Pattern: Drain → Update → Reboot → Verify

For production clusters (Kubernetes, FreeBSD storage nodes, OpenBSD gateways), never update all nodes simultaneously.

### Real Cluster Orchestration

1. **Backup** — Snapshot storage, trigger off-cluster backup

2. **Hold critical packages** (Debian only) — `apt-mark hold docker-ce containerd.io kubelet`

3. **Update all nodes** (packages, no reboot) — Run apt/pkg/syspatch on every node at once

4. **Reboot each node individually:**
   - Drain workloads (kubectl drain, stop services)
   - Trigger reboot
   - Wait for SSH to respond (verify boot succeeded)
   - Restore workloads (uncordon, restart services)
   - Monitor 5 minutes, check logs

5. **Unhold packages** (Debian only) — `apt-mark unhold <package>` once all nodes are healthy

### Real Sequence Example

```bash
#!/bin/bash
NODES=("compute-01" "compute-02" "compute-03" "storage-01" "storage-02")
DRAIN_TIMEOUT=120
SSH_TIMEOUT=300

for node in "${NODES[@]}"; do
  echo "=== Updating $node ==="

  # Drain workloads
  echo "  Draining..."
  kubectl drain "$node" --ignore-daemonsets --delete-emptydir-data \
    --timeout=${DRAIN_TIMEOUT}s

  # Trigger reboot
  echo "  Rebooting..."
  ssh "$node" "reboot" || true

  # Wait for boot
  echo "  Waiting for boot..."
  start_time=$(date +%s)
  while true; do
    if ssh "$node" "echo ok" 2>/dev/null; then
      echo "  ✓ SSH available"
      break
    fi
    elapsed=$(($(date +%s) - start_time))
    if [ $elapsed -gt $SSH_TIMEOUT ]; then
      echo "  ✗ Timeout waiting for SSH"
      exit 1
    fi
    sleep 5
  done

  # Restore
  echo "  Uncordoning..."
  kubectl uncordon "$node"

  # Wait for stability
  echo "  Stabilizing (5 min)..."
  sleep 300

  echo "  ✓ $node complete"
done

echo "All nodes updated successfully"
```

## Key Lessons

### 1. Hold critical packages

On Debian, packages like Docker/Containerd are dangerous to upgrade mid-cluster. Use `apt-mark hold` to prevent surprise upgrades.

### 2. Dry-run first

All three OSes support dry-run:

- Debian: `apt-get upgrade -s` (simulates)
- FreeBSD: `pkg upgrade -n` (no-op)
- OpenBSD: Run script with `--dry-run` flag

Always test on a non-critical system first.

### 3. Never reboot all nodes at once

The cascade fails. Reboot one node at a time, verify it's back, then move on.

### 4. OpenBSD's syspatch is the simplest model

Kernel patches are handled automatically (no manual kernel upgrades). Base system stays simple. Packages are separate. This is the cleanest approach—other OSes should copy it.

### 5. FreeBSD jails reduce blast radius

Update jails independently from the host. If one fails, others aren't affected.

### 6. Storage nodes are special

Don't reboot storage nodes while they're handling writes. Drain connections, snapshot, reboot, wait for recovery before next node.

### 7. Verify before declaring success

After reboot:

- Check SSH (node is booting)
- Check services (node is usable)
- Wait 5 minutes (let services stabilize)
- Check logs (errors from upgrade)
- Then move to next node

## Getting Started

1. **Audit current security patches:**

   ```bash
   # Debian
   apt update && apt list --upgradable

   # FreeBSD
   pkg audit -r

   # OpenBSD
   syspatch -l
   ```

2. **Test on non-critical system first**
3. **Document your hold list** (what packages need coordination)
4. **Schedule a maintenance window** (nights/weekends when cluster is quiet)
5. **Have a rollback plan** (boot environment, snapshots, backup)

Security patches are non-optional. But unplanned reboots are optional. Plan the reboot sequence, execute it methodically, and you keep both security and availability.

## References

All scripts referenced are available in the [floads syseng toolchain](https://github.com/floadsio/syseng):

- [Debian updates](https://github.com/floadsio/syseng/tree/main/linux/debian)
- [FreeBSD updates](https://github.com/floadsio/syseng/tree/main/bsd/freebsd)
- [OpenBSD updates](https://github.com/floadsio/syseng/tree/main/bsd/openbsd)
