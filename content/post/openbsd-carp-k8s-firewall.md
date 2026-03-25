+++
title = "OpenBSD CARP Firewalls in Front of a Private Kubernetes Cluster"
date = "2026-03-25T00:00:00-00:00"
draft = false
slug = "openbsd-carp-k8s-firewall"
tags = ["openbsd", "kubernetes", "networking", "firewall", "carp", "pf", "operations"]
summary = "We run two OpenBSD firewalls in CARP HA mode in front of a private Kubernetes cluster. Sub-second failover, full state sync, and pf rules that load-balance across 18+ worker nodes."
comments = true
+++

## The Problem

We had a Kubernetes cluster running in the cloud, but in a **completely private network** (VPC). Only the firewall nodes expose public IPs to the internet. The cluster needed a stateful front-end that could:

1. **Handle HA** — a single firewall failing would kill everything
2. **Forward traffic** to Kubernetes NodePorts without cloud magic
3. **Survive failover** without dropping established connections
4. **Load-balance** across multiple worker nodes

Cloud load balancers (AWS ALB, NLB, etc.) felt like overkill — we'd be paying for managed services when we wanted full control over traffic rules and failover behavior. We looked at HAProxy in HA mode (Keepalived), but Keepalived is stateless. Every connection would drop on failover.

We settled on **OpenBSD running CARP (Common Address Redundancy Protocol)** with `pfsync` state synchronization. It's been in production for **years** and remains the pragmatic choice for stateful, sub-second failover.

**Status:** Still running, proven stable, no plans to change.

## Why OpenBSD + CARP

OpenBSD's CARP is not a new protocol, but it solves a specific problem well: **stateful, synchronized failover between two firewalls**.

Here's what makes it different from Keepalived:

- **CARP** — handles virtual IP failover (master/backup), with the backup taking over automatically
- **pfsync** — synchronizes firewall state (all connections, NAT tables, etc.) in real-time
- Together — when master fails, backup inherits all active connections without dropping a single packet

The alternative (HAProxy + Keepalived) would require application-level reconnect logic or connection pooling. With stateful firewall sync, TCP connections just keep working.

Other options we considered and rejected:
- **Cisco/Juniper** — overkill, expensive, requires vendor support
- **pfSense** — built on OpenBSD, but pricey; OpenBSD itself was free
- **Keepalived + HAProxy** — stateless failover, requires app-level reconnect handling
- **Cloud load balancers** — not an option for on-prem

OpenBSD was the only thing that gave us stateful sync without paying enterprise fees.

## The Architecture

Two OpenBSD VMs in separate availability zones (for fault isolation).

```
Internet
   ↓
[Firewall-Primary] ← CARP Master (72.X.X.1) - public IP
[Firewall-Backup]  ← CARP Backup (72.X.X.2) - public IP
   ↓ (both have CARP VIP: 72.X.X.100 - public)
Private Network (VPC)
   ↓
[Kubernetes Cluster - 18+ worker nodes]
   ↓
[Traefik Ingress Controller]
```

**Network Design:**

1. **External interface** — public IP, handles internet traffic
2. **Internal interface** — private network to K8s cluster
3. **Dedicated sync network** — high-speed link between firewalls for pfsync (critical for low-latency state sync)
4. **CARP VIP** (external) — 72.X.X.100, load balancing incoming traffic
5. **CARP VIP** (internal) — 10.X.X.1, cluster access

The sync network was crucial. If pfsync packets got queued behind regular traffic, state sync would lag, and failover wouldn't be clean.

**pf Configuration (simplified):**

```
# CARP VIP for external HTTP/HTTPS traffic
pass in on egress proto tcp to 72.X.X.100 port { 80, 443 } \
  rdr-to { 10.Y.Y.1, 10.Y.Y.2, ..., 10.Y.Y.18 } port { 80, 443 }

# SSH tunnel through firewall
pass in on egress proto tcp to 72.X.X.100 port 2222 \
  rdr-to 10.Y.Y.5 port 22

# NAT: all outbound traffic from cluster to public IP
pass out on egress from 10.Y.Y.0/24 nat-to 72.X.X.100
```

The firewall does **simple port forwarding**: traffic on public IPs (72.X.X.100) gets redirected to private IPs in the cluster. Inside the cluster, **Traefik (Kubernetes ingress controller)** handles routing. This separation of concerns is clean:

- **pf (firewall):** Public ↔ Private IP translation, HA failover, NAT
- **Traefik (cluster):** HTTP routing, TLS termination, rate limiting, path-based routing

No load balancing logic at the firewall level. The `rdr-to` pool distributes new connections, but Traefik is what actually routes requests to backend services.

## Real-Time Failover Demo

Here's what happens when the master firewall is shut down mid-connection. Notice: **zero packet loss**.

<video controls width="100%" style="border-radius:4px">
  <source src="/video/pf-failover-demo.mp4" type="video/mp4">
</video>

The demo shows:
- **Before failover:** pings going through master firewall (RTT ~5ms)
- **Failover happens:** master goes down, backup takes over in <100ms
- **After failover:** pings continue, new RTT shows traffic now goes through backup
- **No failures:** not a single ping was dropped

This works because pfsync keeps the backup's state table synchronized. When the backup becomes master, all existing connections are already in its state table.

## Port Forwarding Strategy

Instead of traditional port ranges, we used **semantic port mapping**:

- **80** → NodePort **30080** (HTTP)
- **443** → NodePort **30443** (HTTPS)
- **2222** → Pod on master node port **22** (SSH to cluster)
- **3389** → NodePort **30389** (RDP for management)

NodePort range is typically 30000-32767. We stayed within it and forwarded inbound public ports to predictable NodePort destinations.

This was simpler than:
- Running ingress controllers on every node
- Managing hostname-based routing
- Dealing with SSL termination on the firewall

The firewall's job: **forward and NAT**. Kubernetes' job: **route and serve**.

## Load Balancing at the Firewall Level

The pf `rdr-to` rule includes a pool of backend worker IPs. pf distributes new connections using a hash of source/destination (not pure round-robin).

```pf
rdr-to { 10.Y.Y.1, 10.Y.Y.2, ..., 10.Y.Y.18 } port 80
```

This spreads traffic across workers, but **Traefik does the actual routing**. pf just ensures the connection gets to one worker; Traefik then handles:
- Layer-7 routing (by hostname, path, headers)
- TLS termination
- Session affinity (sticky cookies)
- Service discovery

So if one worker is down, pf doesn't know — but Traefik does. Traefik routes around the dead worker. pf's load balancing is just a coarse distribution mechanism; Traefik is where the intelligence lives.

## Updating OpenBSD VMs: No Big Deal

One of the best parts of this setup: **updating is straightforward**.

**Typical update flow:**

```bash
# SSH into primary firewall
ssh fw-primary

# Apply updates
sudo syspatch

# Reboot if needed
sudo shutdown -r now

# CARP failover happens automatically
# Backup takes over (stateful sync keeps connections alive)
# Users don't see it

# When primary comes back up, it rejoins as backup
# Manual failback when convenient: pfctl -i carp0 -Fr
```

There's no special orchestration needed. OpenBSD's `syspatch` is trivial—usually 30 seconds to download and apply. Reboot takes ~1 minute. During the reboot:
- Backup firewall becomes master (via CARP)
- pfsync keeps connection state in sync
- Existing TCP connections keep flowing
- New connections route through the backup

Then update the backup. No downtime, no exotic load balancer coordination. Just reboot and move on.

This simplicity is one reason we haven't replaced this setup. Upgrading a cloud load balancer often requires downtime or complex blue-green deployments.

## Remote Access: WireGuard on the Firewall

Managing firewalls from the internet is a security nightmare. We run **WireGuard** on the firewall itself.

```
WireGuard VPN (on firewall)
  ↓
Private firewall management network
  ↓
SSH to firewall, then to cluster
```

This meant:
- No SSH exposed to the internet on the firewall
- Secure tunnel to internal management
- A single source of truth for secrets (WireGuard config in Bitwarden)

pf rules allowed WireGuard traffic in, but nothing else on the management port.

## What We Learned

**1. State synchronization is non-negotiable for failover**

Stateless failover (Keepalived) is fine if your application handles reconnects. Most didn't. pfsync solved this by keeping state synchronized, so the backup could take over transparently.

**2. Dedicated sync networks matter**

If pfsync shares bandwidth with data traffic, state sync gets queued and lags. A fast, direct link between firewalls is worth the extra networking.

**3. OpenBSD/pf is simple but powerful**

pf syntax is clearer than iptables. Configuration lives in one file. Rules are easier to audit and modify. We didn't need a GUI — `pfctl` and a text editor were enough.

**4. CARP works, but requires planning**

CARP VIPs work great, but you need to think about:
- Which interface is the VIP on?
- What's the priority (master vs backup)?
- How fast do you want failover? (advskew tuning)
- Is your sync network fast enough?

Getting this wrong means either failover doesn't happen, or the master and backup both think they're the master (split-brain).

**5. Load balancing at the firewall has limits**

Round-robin distribution across NodePorts worked fine for HTTP/HTTPS, but it's not session-aware. Sticky sessions had to happen at the application layer (cookies, JSession, etc.) or via Kubernetes ingress rules.

## The Trade-off

**What we gave up:**
- Automatic scaling (firewall is fixed capacity)
- Cloud flexibility (bound to on-prem hardware)
- SLA guarantees (no vendor support — we own the code)
- Automatic updates (OpenBSD stable releases, manual patching)

**What we gained:**
- Full control over traffic rules
- No cloud egress charges
- Stateful failover without dropped connections
- Simplicity (two files: pf.conf and CARP config)
- Cost (free OS, standard hardware)

For a private cluster serving internal users, this trade-off made sense. For a public SaaS platform, cloud load balancers might be worth the cost.

## Would We Do It Again?

**Absolutely.** This setup has been running for years in production. Zero regrets.

CARP + pfsync is the right tool for **stateful HA failover when you control the infrastructure**. The setup is straightforward, the failover is sub-second, and the operational overhead is minimal. Updating firewalls is easier than updating most managed cloud services.

The trade-offs are favorable:
- ✅ Full control over traffic rules
- ✅ No cloud egress charges
- ✅ Sub-second stateful failover (unbeatable for connection continuity)
- ✅ Simple to operate and update
- ❌ You own it (no vendor support)
- ❌ Bound to your infrastructure provider's network

For a private cluster where you already own the infrastructure, OpenBSD CARP is the pragmatic choice. It works. It keeps working. Updates are trivial. Failover is transparent.

**If you're considering this:** You need comfort with:
- Network architecture (VIPs, multicast, ARP)
- Command-line firewall management (pf syntax is learnable)
- Monitoring for split-brain scenarios (rare but possible)
- Testing failover occasionally (we do this monthly)

If you have those skills and control your own infrastructure, stop paying for cloud load balancers. CARP is simpler and better for your use case.
