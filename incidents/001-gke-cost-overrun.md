# Incident 001: GKE Cost Overrun — 58% Reduction

**Date:** January 2024  
**Severity:** P2 (Cost)  
**Duration:** Ongoing → Resolved over 2 weeks  
**Impact:** $2,400/month → $1,008/month infrastructure cost  

---

## Summary

Monthly GKE infrastructure bill grew to $2,400 despite running only 15 microservices with moderate traffic (~200 RPS peak). Investigation revealed over-provisioned node pools, always-on non-production workloads, and missing resource requests/limits on most pods.

## Detection

- Monthly cloud billing alert triggered at $2,200 (threshold: $2,000)
- GCP Cost Breakdown dashboard showed 73% of spend on GKE Compute Engine nodes
- Prometheus metrics confirmed average cluster CPU utilization was 18% and memory at 34%

## Root Cause Analysis

```
Issue 1: Fixed node pool (3 × e2-standard-4) — no autoscaling
  → Nodes running 24/7 even when load was near zero
  → Each node: 4 vCPU, 16GB RAM = massively over-provisioned

Issue 2: No resource requests/limits on 10 of 15 deployments
  → Kubernetes scheduler couldn't bin-pack pods efficiently
  → Nodes appeared "full" due to default resource allocation

Issue 3: Dev/staging workloads running in production cluster
  → 5 staging pods consuming resources 24/7 with zero traffic

Issue 4: 100GB PD-SSD attached to PostgreSQL but only 8GB used
```

## Resolution

### Step 1: Right-size nodes (Week 1)
```yaml
# Before
machineType: e2-standard-4    # 4 vCPU, 16GB RAM
minNodeCount: 3
maxNodeCount: 3

# After
machineType: e2-custom-2-8192  # 2 vCPU, 8GB RAM (custom)
minNodeCount: 0
maxNodeCount: 3
```

### Step 2: Add resource requests/limits to ALL deployments
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```
Applied via Helm values across all 15 apps.

### Step 3: Enable cluster autoscaler
```hcl
autoscaling {
  min_node_count = 0
  max_node_count = 3
}
```
Nodes scale to zero during low-traffic periods (midnight–6 AM).

### Step 4: Move staging to separate namespace with Spot VMs
```yaml
nodeSelector:
  cloud.google.com/gke-spot: "true"
```

### Step 5: Resize PVC
Migrated PostgreSQL to 20GB PD-Balanced (cheaper than PD-SSD by 40%).

## Cost Breakdown

| Component | Before | After | Savings |
|-----------|--------|-------|---------|
| GKE Nodes | $1,680 | $560 | 67% |
| Persistent Disks | $320 | $85 | 73% |
| Network/LB | $280 | $260 | 7% |
| Other | $120 | $103 | 14% |
| **Total** | **$2,400** | **$1,008** | **58%** |

## Preventive Measures

1. **Budgets with alerts** at 80% and 100% of $1,200/month target
2. **Resource requests mandatory** — added OPA Gatekeeper policy:
   ```yaml
   kind: ConstraintTemplate
   spec:
     targets:
       - target: admission.k8s.gatekeeper.sh
         rego: |
           violation[{"msg": msg}] {
             not input.review.object.spec.containers[_].resources.requests
             msg := "Resource requests are required"
           }
   ```
3. **Weekly cost review** dashboard in Grafana with GCP billing export
4. **Autoscaler monitoring** — alert if node count stays at max for >4 hours

## Lessons Learned

- Default GKE node sizes are optimized for Google's revenue, not your workload. Always right-size.
- Resource requests are the single most impactful cost optimization — they enable bin-packing and autoscaling.
- Spot/preemptible VMs are ideal for staging — 60-91% savings with acceptable disruption risk.
