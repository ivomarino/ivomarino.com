+++
title = "Kubernetes Node Autoscaling: When Pod Scaling Isn't the Answer"
date = "2026-05-15T00:00:00-00:00"
draft = false
slug = "node-autoscaling-cluster-autoscaler"
tags = ["kubernetes", "autoscaling", "cluster-autoscaler", "node-scaling", "infrastructure", "operations"]
summary = "HPA scales pods, but pods can't run if there's no space on the cluster. Learn why cluster autoscaling is the answer for bursty, unpredictable workloads. Includes pod disruption budgets, graceful shutdown, and real patterns for scaling nodes alongside pod autoscaling."
comments = true
image = "img/cluster-autoscaler-header.jpg"
+++

You've scaled pods with HPA. Works great.

Then you hit a traffic spike. HPA spins up 50 new pods to handle the load.

But something's wrong: they're all showing "Pending."

The scheduler can't place them. No nodes have capacity. HPA created the pods, but they're sitting idle consuming API quota while your customers see timeouts.

This is when pod autoscaling isn't enough. You need cluster autoscaling.

## The Problem

Pod-level autoscaling assumes you have nodes with available resources. For bursty, unpredictable workloads, that assumption breaks.

Real scenarios where this fails:

**Spike 1: News breaks**
- Requests jump from normal to 10x in seconds
- HPA detects high CPU, spins up 100 new pods
- Nodes are full. Pods sit pending.
- Cluster has no capacity to add more pods
- Customers timeout while idle pods wait for scheduling

**Spike 2: Time-sensitive event**
- Election day, breaking news, product launch
- Unpredictable traffic, no prior warning
- Can't pre-provision nodes for "unknown unknowns"
- HPA is useless if there's nowhere to run pods

**Spike 3: Mixed workload sizes**
- Some pods need 2GB memory, others need 16GB
- Can't pack 16GB pod on a node with only 4GB free
- Cluster needs different node sizes, not just more replicas

Pod autoscaling alone answers the wrong question: "How many copies of my app?" It ignores the real question: "Can my cluster hold all these pods?"

## The Solution: Cluster Autoscaler

Cluster Autoscaler watches cluster capacity and adds/removes nodes based on actual demand.

**How it works:**

1. Traffic spike hits
2. HPA (or manual scaling) creates new pods
3. Pods can't fit → they show "Pending"
4. Cluster Autoscaler detects pending pods
5. Requests new node from cloud provider
6. Scheduler places pending pods on new node
7. Pods start serving traffic

And for scale-down:

1. Traffic returns to normal
2. Pods finish or scale down
3. Nodes become empty
4. Cluster Autoscaler detects empty nodes
5. Respects pod disruption budgets (doesn't evict critical pods)
6. Drains remaining pods gracefully (30+ minute timeout)
7. Removes empty node

## Configuration

**Basic setup:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
          name: cluster-autoscaler
          args:
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws              # or gcp, azure, custom
            - --nodes=4:10:my-asg               # Min 4, Max 10, ASG name
            - --skip-nodes-with-local-storage=false
            - --skip-nodes-with-system-pods=false
          env:
            - name: AWS_REGION
              value: "us-west-2"
          resources:
            requests:
              cpu: 100m
              memory: 300Mi
            limits:
              cpu: 100m
              memory: 300Mi
```

**Key parameters:**

- **`--cloud-provider`** — AWS, GCP, Azure, or custom
- **`--nodes=min:max:pool-name`** — Min/max node counts and node pool
- **`--skip-nodes-with-local-storage`** — Can we drain nodes with local data?
- **`--skip-nodes-with-system-pods`** — Can we drain system pods?

Different cloud providers use different node group naming:
- AWS: Auto Scaling Group (ASG) name
- GCP: Node pool name
- Azure: VM scale set name
- Custom: Whatever your provisioning system uses

## Critical: Pod Disruption Budgets

Without Pod Disruption Budgets (PDBs), cluster autoscaler can evict critical pods mid-processing, causing data loss and cascade failures.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-pipeline-pdb
  namespace: prod
spec:
  minAvailable: 1           # Always keep at least 1 replica
  selector:
    matchLabels:
      app: critical-pipeline
```

**How it works during scale-down:**

1. Node marked for removal
2. Kubelet sends SIGTERM to pods
3. Pod has `terminationGracePeriodSeconds: 1800` (30 minutes)
4. Pod's graceful shutdown code: Finish current batch, flush queues, close connections
5. Pod exits cleanly
6. Node is removed

If a pod doesn't exit in 30 minutes, it's forcefully killed. With proper graceful shutdown code, this rarely happens.

**Without PDB:** Cluster autoscaler can't respect pod constraints. It might evict a critical pod that's processing data, causing loss.

**With PDB:** Cluster autoscaler knows to leave at least 1 replica running at all times.

## Real Numbers

**Before (manual node management):**
- Ops team monitors queue depth, manually adds nodes when capacity looks low
- Overprovision to avoid being caught off-guard (80-90% utilization)
- Scale-down happens once per week in scheduled maintenance
- Cost: Always running at high capacity "just in case"
- Latency during spikes: 15-30 minutes (waiting for manual node addition)

**After (cluster autoscaler):**
- Cluster starts at 4 nodes (baseline)
- Spike hits → pods pending → node added automatically (2-3 minutes)
- New pods scheduled, traffic served (latency ~5 minutes total)
- Spike ends → nodes empty → removed automatically
- Cost: Only pay for what you use
- Latency during spikes: 5-10 minutes (automatic response)

## HPA + Cluster Autoscaler Together

Use both for best results:

```
┌─────────────┐
│ Traffic     │
│ Spike       │
└──────┬──────┘
       │
       ├→ HPA: Scale pods from 10 to 50
       │
       ├→ Scheduler: Can't fit 40 new pods
       │
       ├→ Cluster Autoscaler: Adds 3 new nodes
       │
       └→ Scheduler: Places pending pods
              Pods run, traffic served
```

**HPA responsibility:** "How many replicas of my app?"
**Cluster Autoscaler responsibility:** "Do I have nodes to run them?"

They're answering different questions.

## When to Use What

**Use HPA (pod autoscaling) for:**
- Always-on services (APIs, web servers)
- Predictable or gradual load changes
- Stable pod resource requests
- You know the max replica count

**Use Cluster Autoscaler (node autoscaling) for:**
- Bursty or unpredictable workloads
- Diverse pod sizes (some need 2GB, others 16GB)
- You can't predict pod count
- Scaling pods without scaling nodes is pointless

**Use both:**
- Mixed workloads (always-on services + bursty processing)
- HPA handles pod optimization
- Cluster autoscaler handles cluster capacity

## Common Mistakes

❌ **Using HPA alone for bursty workloads**
- Pods sit pending forever
- Wasted API calls
- Frustrated users

❌ **Not setting resource requests**
- Scheduler can't pack pods
- Cluster autoscaler adds wrong-sized nodes
- Cascade failures when pods exceed requests

❌ **No pod disruption budgets**
- Pods evicted mid-processing
- Data loss, transaction rollbacks
- Cascade failures

❌ **Max nodes too high**
- Runaway scaling during spike
- Shocking bills at month end
- No spending ceiling

❌ **Min nodes too low**
- Baseline services can't run
- First scale-up takes forever (node provisioning overhead)

## Operations

**Monitor pending pods:**

```bash
# Watch for pods that can't be scheduled
kubectl get pods -A --field-selector=status.phase=Pending

# Check node capacity
kubectl top nodes
kubectl describe node <node-name>

# Check cluster autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler
```

**During a spike:**

```bash
# Monitor autoscaler response
watch kubectl get nodes

# Check pod placement
kubectl get pods -w

# Verify graceful shutdown (30-minute default)
# Adjust if needed
kubectl patch pod <name> -p \
  '{"spec":{"terminationGracePeriodSeconds":1800}}'
```

## Getting Started

1. **Choose your cloud provider:**
   - AWS: Use Auto Scaling Groups
   - GCP: Use Node Pools
   - Azure: Use VM Scale Sets
   - Custom: Implement custom cloud provider driver

2. **Deploy cluster autoscaler:**
   - Use official Helm chart or YAML manifests
   - Configure min/max nodes and node pool names
   - Set resource requests/limits

3. **Add pod disruption budgets:**
   - Critical pods: `minAvailable: 1`
   - Less critical: `minAvailable: 0` with budget rules
   - Always set `terminationGracePeriodSeconds`

4. **Test scale-up and scale-down:**
   - Trigger traffic spike, verify nodes are added
   - Let spike subside, verify nodes are removed
   - Check pod disruption budgets are respected

## The Key Insight

Pod autoscaling asks: "How many copies do I need?"

Cluster autoscaling asks: "Do I have capacity to run them?"

For unpredictable, bursty workloads, the second question matters more. Cluster Autoscaler gives you reactive capacity that grows and shrinks with actual demand.

---

**This concludes the autoscaling series:** Pod autoscaling with metrics → Queue-based scaling with KEDA → Cluster autoscaling for unpredictable demand. Together, they handle any workload pattern Kubernetes can throw at you.
