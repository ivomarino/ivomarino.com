+++
title = "Kubernetes Pod Autoscaling with KEDA: Queue Depth Beats CPU Every Time"
date = "2026-04-05T00:00:00-00:00"
draft = false
slug = "keda-queue-autoscaling"
tags = ["autoscaling-series", "kubernetes", "operations", "keda", "rabbitmq", "event-driven", "queue-based", "autoscaling", "cost-efficiency"]
summary = "Kubernetes HPA with CPU metrics fails for queue-based workloads. Learn why queue depth is the right signal, how KEDA makes event-driven autoscaling trivial, and the configuration that reduced processing latency from 45 minutes to 5-15 minutes with zero manual scaling."
comments = true
image = "img/keda-header.jpg"
+++

We had a batch processing system running on Kubernetes that consumed messages from a queue. The platform handled vendor data transformations and product list updates—unpredictable, bursty traffic that spiked without warning.

We tried Kubernetes HPA with CPU metrics.

It failed immediately.

## The Problem with CPU-Based Autoscaling for Queues

Here's what we did:

**Test 1: Set HPA target to 70% CPU**

- Workers sit idle while the queue builds up (low CPU usage)
- Queue hits 50,000 messages waiting to be processed
- CPU finally spikes to 90%
- HPA triggers and starts new pods
- But 3+ minutes have passed
- Workers were never actually busy—they were just waiting for work

The metric was wrong. CPU doesn't measure queue congestion.

**Test 2: More aggressive—target CPU at 50%**

- HPA triggers faster, but CPU is still a lagging indicator
- Traffic spikes arrive faster than CPU metrics can detect them
- By the time HPA scales up, traffic has already dropped
- Pods start spinning up, then traffic stops
- HPA immediately scales down (thrashing)
- Pod startup/shutdown cycle becomes more expensive than actual work
- Queue keeps growing

**Test 3: Switched to memory-based HPA**

- Memory grows gradually as workers process
- HPA almost never triggered
- We just hit out-of-memory limits (OOM kills)
- Gave up and managed replicas manually
- This is unacceptable for a production system

## The Insight

For queue workloads, **the queue depth IS the signal of overload.**

If you have 50,000 messages sitting in RabbitMQ, you are overloaded *right now*. It doesn't matter if your CPUs show 20% usage. Your workers could be processing 10x faster if you gave them capacity.

Queue depth = real demand. CPU = lagging indicator. Memory = wrong signal entirely.

## The Solution: KEDA

KEDA (Kubernetes Event-driven Autoscaling) extends Kubernetes autoscaling beyond CPU and memory. It works with 50+ event sources:

- Message queues: RabbitMQ, AWS SQS, Apache Kafka
- Databases: PostgreSQL, MySQL, Redis
- Cloud services: AWS CloudWatch, Azure Service Bus
- Custom: HTTP endpoints, webhooks, cron schedules
- Metrics: Prometheus, Datadog, Grafana Loki

Installation takes 5 minutes. Configuration is straightforward Kubernetes YAML.

### Configuration Pattern

Deploy KEDA to your cluster:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

Then define a `ScaledObject` that watches your queue:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: data-processor-worker-scaler
  namespace: prod
spec:
  scaleTargetRef:
    kind: Deployment
    name: data-processor-worker
  minReplicaCount: 16        # Baseline capacity
  maxReplicaCount: 40        # Safety ceiling
  cooldownPeriod: 60         # Wait 60s between scale events
  pollingInterval: 30        # Check metrics every 30s
  triggers:
  - type: prometheus
    metadata:
      serverAddress: "https://<prometheus-endpoint>/api/prom"
      metricName: rabbitmq_queue_messages_ready
      threshold: "5000"      # Scale when queue > 5k messages
      query: "avg_over_time(rabbitmq_queue_messages_ready[5m])"
    authenticationRef:
      name: keda-prometheus-creds
```

**What each setting does:**

**minReplicaCount: 16** — This is different from typical autoscaling (which uses min=1 or min=3). For always-on queue workloads, don't scale to zero. Keep baseline capacity ready. It's cheaper than constantly spinning up new pods.

**maxReplicaCount: 40** — Hard cap to prevent runaway scaling. Beyond this, let your cluster autoscaler add more nodes.

**pollingInterval: 30** — KEDA checks your queue depth every 30 seconds. Fast enough to catch bursts, slow enough to avoid thrashing.

**cooldownPeriod: 60** — After scaling, wait 60 seconds before the next scale decision. Prevents constant scaling up/down during traffic fluctuations.

**threshold: 5000** — Each worker processes 500-1000 messages per minute. With 16 baseline replicas (8k-16k msg/min throughput), a threshold of 5,000 messages keeps the queue from ever getting huge. It's a signal to add capacity proactively.

**5-minute metric window** — The Prometheus query averages queue depth over 5 minutes. Avoids reacting to 1-second spikes that might be temporary.

## The Results

**Before (manual + CPU HPA):**
- Queue would spike to 100,000+ messages during vendor pushes
- Processing delays, customer impact
- Manual scaling overhead
- Average processing latency: **45-90 minutes**

**After (KEDA):**
- Queue rarely exceeds 10,000 messages
- Automatic, reactive scaling
- Zero manual intervention
- Average processing latency: **5-15 minutes**
- Cost: Same or lower (fewer idle pods, better utilization)

A 6x improvement in latency with less operational overhead.

## When to Use KEDA

**Use KEDA when:**
- You have queue-based workloads (RabbitMQ, SQS, Kafka, etc.)
- Traffic is unpredictable or bursty
- You need responsive scaling without manual intervention
- Your metric is "queue depth" or event rate, not CPU/memory

**Don't use KEDA when:**
- Your workload is truly CPU-bound (machine learning, video processing)
- You need to scale based on custom application logic
- Your queue system doesn't expose metrics (implement that first)

## Common Mistakes

❌ **Using HPA with CPU for queue workloads** — CPU is a lagging indicator. Queue depth is the truth signal.

❌ **Setting minReplicaCount to 1 for always-on workloads** — Forces constant pod startup. Set min to handle baseline load comfortably.

❌ **Threshold too low (e.g., 500 messages)** — Causes constant scaling up/down. Pod startup cost exceeds any benefit.

❌ **Threshold too high (e.g., 100,000 messages)** — Queue builds up before scaling triggers. By the time pods are ready, messages are old.

❌ **No maxReplicaCount** — Runaway scaling exhausts cluster resources. Always set an upper bound.

## Operations

### Monitoring Queue Health

```bash
# Watch queue depth over time
kubectl port-forward -n prometheus prometheus-0 9090:9090
# Then in Prometheus UI: graph rabbitmq_queue_messages_ready

# Check current replica count
kubectl get deployment data-processor-worker -n prod

# Check KEDA status
kubectl get scaledobject -n prod
kubectl describe scaledobject data-processor-worker-scaler -n prod

# Check KEDA logs
kubectl logs -n keda deployment/keda-operator
```

### Manual Testing

Scale down to baseline:
```bash
kubectl patch deployment data-processor-worker -p '{"spec":{"replicas":16}}'
```

Verify KEDA scales back up when queue fills.

## The Pattern

Queue workloads need the right autoscaling signal:

1. **Event source** — Your queue exposing metrics (RabbitMQ to Prometheus, SQS to CloudWatch, etc.)
2. **Metrics collection** — Prometheus, CloudWatch, or KEDA's native queue scalers
3. **KEDA** — Watches metrics, scales deployments
4. **Right configuration** — Appropriate minReplicaCount, threshold, polling, and cooldown

This pattern works whether you have 100 messages per hour or 1 million messages per minute.

The key insight: **Stop scaling on CPU. Start scaling on queue depth.**

---

**Getting Started:** KEDA has [excellent documentation](https://keda.sh) and [50+ scalers](https://keda.sh/docs/scalers/) for different event sources. The GitHub repo is [kedacore/keda](https://github.com/kedacore/keda).
