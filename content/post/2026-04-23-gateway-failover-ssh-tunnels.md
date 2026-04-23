---
title: "Out-of-Band SSH Tunnel Failover: Why Stability Beats Speed"
date: 2026-04-23
draft: false
summary: "Building reliable automatic gateway failover for critical infrastructure recovery access. How naive health checks broke everything, and why a 30-second stable recovery is better than a 5-second flapping failover."
image: "img/network-switch-failover-header.jpg"
image_credit: "Photo by [Taylor Vick](https://unsplash.com/@tvick) on [Unsplash](https://unsplash.com)"
tags: [freebsd-series, infrastructure, operations, networking, freebsd, ssh, autossh, failover, automation, reliability]
---

We deployed a FreeBSD Bastille jail to remote infrastructure with two internet uplinks: LTE (primary, unreliable) and WiFi (secondary, stable).

This jail serves a critical role: **out-of-band management access** for the entire remote site. When the primary uplink failed, manual intervention was required to restore access. For infrastructure without remote hands-on-deck, this is unacceptable.

The solution: automatic gateway failover with `autossh` + `ifstated` to maintain persistent SSH tunnel access regardless of which uplink is operational. But naive health checking made the system worse than useless—constant flapping, tunnel drops, and lost access.

**Note:** This implementation uses FreeBSD's `ifstated` for the state machine, but the approach applies equally to OpenBSD (which also has `ifstated`) and Linux systems using alternatives like `keepalived` or custom health check scripts. The core lesson—stability over speed for recovery access—is universal.

## The Naive Approach

Single ping health check. Every 10 seconds, `ifstated` would:

```bash
ping -c 1 8.8.8.8 > /dev/null 2>&1 && echo "up" || echo "down"
```

One failed ping = gateway down = switch routes. Simple. It failed.

**What happened:** Any transient packet loss triggered immediate failover. Failover cascaded: every 30-60 seconds, the route switched, the SSH tunnel broke, `autossh` tried to reconnect, but by then the gateway had switched again.

Single transient failure = full system flap.

The root cause was architectural: we needed `autossh` to establish the tunnel, but the health check was too trigger-happy to let it succeed.

## Why Timeout Tuning Alone Wasn't Enough

We tried aggressive SSH timeouts:

```
ServerAliveInterval 10
ServerAliveCountMax 2
ExitOnForwardFailure yes
```

This meant: detect failure in 20 seconds. But the gateway was switching every 30 seconds. By the time `autossh` restarted, the next health check had already triggered failover.

The problem wasn't tunnel detection. The problem was health check sensitivity.

## The Fix: Two-Tier Health Checks

Require 2 consecutive successful pings before declaring a gateway down. Here's the health check script:

```bash
#!/bin/sh
# Universal gateway health check script

ENDPOINT="${1:-8.8.8.8}"
GATEWAY_NAME="${2:-Unknown}"
PINGS_REQUIRED="${3:-2}"
PING_TIMEOUT=2
PINGS_SUCCESS=0

# Run consecutive pings and count successes
for i in $(seq 1 $PINGS_REQUIRED); do
  ping -c 1 -W $PING_TIMEOUT "$ENDPOINT" > /dev/null 2>&1 && PINGS_SUCCESS=$((PINGS_SUCCESS + 1))
done

# Pass if majority succeeded
THRESHOLD=$(((PINGS_REQUIRED / 2) + 1))
if [ $PINGS_SUCCESS -ge $THRESHOLD ]; then
  exit 0
fi

logger -t ifstated "$GATEWAY_NAME: unreachable ($ENDPOINT, $PINGS_SUCCESS/$PINGS_REQUIRED)"
exit 1
```

**Usage:**
```bash
fallback-check-connectivity 8.8.8.8 "LTE Gateway" 2
fallback-check-connectivity 1.1.1.1 "WiFi Gateway" 2
```

One transient packet loss = no action. Two consecutive failures = genuine outage. This gives `autossh` a stable 20-second window to establish the tunnel. Single change, eliminated flapping.

## FreeBSD/OpenBSD ifstated Configuration

Both FreeBSD and OpenBSD provide `ifstated` as the state machine daemon. Here's the configuration (identical on both systems):

```bash
# /etc/ifstated.conf

init-state primary

lte_up  = '( "/usr/local/bin/fallback-check-connectivity 8.8.8.8 LTE 2" every 10 )'
wifi_up = '( "/usr/local/bin/fallback-check-connectivity 1.1.1.1 WiFi 2" every 10 )'

state primary {
    init {
        run "route change default 192.168.160.254"
        run "logger -t ifstated 'PRIMARY: Using LTE gateway'"
        run "/usr/local/bin/fallback-autossh-restart"
    }
    if ! $lte_up {
        set-state failover
    }
}

state failover {
    init {
        run "route change default 192.168.160.253"
        run "logger -t ifstated 'FAILOVER: Using WiFi gateway'"
        run "/usr/local/bin/fallback-autossh-restart"
    }
    if $lte_up {
        set-state primary
    }
}
```

Every 10 seconds, `ifstated` tests the primary gateway. On failure, it switches to the secondary gateway and restarts the SSH tunnel.

## SSH Timeout Tuning & Exponential Backoff

SSH keep-alive detection catches tunnel failures in ~20 seconds:

```
ServerAliveInterval 10
ServerAliveCountMax 2
ExitOnForwardFailure yes
```

When `autossh` restarts, exponential backoff prevents hammering the remote server:

```bash
attempt=$(($(cat /var/run/autossh-restarts 2>/dev/null || echo 0) + 1))
if [ $attempt -gt 1 ]; then
  delay=$((2 ** (attempt - 1)))
  [ $delay -gt 16 ] && delay=16
  sleep $delay
fi
echo $attempt > /var/run/autossh-restarts
```

Without backoff: 50+ restart attempts per minute. With it: gradually backs off to 16 seconds max.

## What Changed Operationally

**Before (Naive failover):**
- Gateway switches every 30-60 seconds
- SSH tunnel unreliable (constant drops)
- Users can't access the fallback infrastructure
- Manual intervention required

**After (Stable health checks + proper timeouts):**
- Gateway switches only when LTE genuinely fails (detectable after 20 seconds)
- SSH tunnel survives failover (reconnects within 20-30 seconds)
- Automatic recovery without intervention
- 10-minute sustained test: 100% uptime

## The Real Lesson

**Out-of-band management requires different trade-offs.**

This system exists to provide SSH access when the primary infrastructure link is down. Unlike user-facing services where millisecond failover matters, recovery access is about **never losing connectivity**, even briefly.

The temptation is aggressive: fast detection, immediate action, retry without delay. But for a recovery mechanism:

- One false positive (transient packet loss) triggers cascading failures
- Restart hammering turns a temporary network glitch into permanent loss of access
- Stability matters more than speed

For critical out-of-band access, **steady recovery beats fast failover.**

A 20-30 second failover window is acceptable if it means the tunnel never flaps. A 2-second failover that oscillates every 30 seconds removes your last way to access the infrastructure.

## What We Learned

1. **Out-of-band access cannot afford to flap**
   - A stable 30-second recovery beats a 5-second failover that oscillates
   - Losing access twice per minute is worse than losing it once for 30 seconds
   - When this is your recovery path, availability > speed

2. **Single-packet sensitivity is a reliability killer**
   - Require multiple pings before declaring failure; one transient loss cascades
   - Prevents rapid state oscillation and gives downstream systems recovery time
   - False positives turn temporary glitches into permanent access loss

3. **Timeout values are empirical, not theoretical**
   - 90-second SSH timeout was too slow; 5-second was too aggressive
   - 20-second timeout (2 keep-alives × 10s) proved optimal
   - Test under real failure conditions, not on paper

4. **Orphaned processes silently break recovery**
   - When autossh parent dies, SSH children persist and hold the port
   - New restarts fail with "address already in use"
   - Must explicitly kill both parent and children

5. **Exponential backoff is non-negotiable**
   - Failed restarts without delay hammer the remote server (50+ attempts/minute)
   - Turns network glitches into denial of service
   - Backoff with 16-second maximum stabilizes recovery

6. **Integration testing is mandatory for recovery systems**
   - Individual scripts work in isolation; combined system oscillates under stress
   - Sustained load test (10-minute, 100% uptime) proves viability
   - Lab testing insufficient; real-world failover is the only validation

## Trade-offs

**What we gave up:**
- Faster failover speed (20-30 seconds instead of instant)
- Simpler restart logic (now requires backoff state tracking)

**What we got:**
- Reliable failover without flapping
- Automatic recovery from transient network glitches
- Stable SSH tunnel that survives gateway switches
- System that self-heals without manual intervention

---
