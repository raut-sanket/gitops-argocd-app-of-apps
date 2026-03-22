# Incident 002: OOMKilled Pods — Indexer Service

**Date:** February 2024  
**Severity:** P1 (Service Outage)  
**Duration:** 23 minutes  
**Impact:** Blockchain event indexing stopped; 1,200 transactions missed from on-chain data  

---

## Summary

The `suidex-indexer-backend` pod entered a CrashLoopBackOff state with OOMKilled termination reason. The indexer processes Sui blockchain events in batches, and a sudden spike in on-chain activity (NFT mint event) caused memory consumption to exceed the 512Mi limit.

## Detection

- **00:14 UTC** — PagerDuty alert: `PodCrashLooping` fired for `suidex-indexer-backend`
- **00:15 UTC** — Grafana dashboard showed pod restart count increasing rapidly
- **00:16 UTC** — `kubectl describe pod` showed `Last State: Terminated - Reason: OOMKilled`

## Timeline

```
00:12  Large NFT collection mint on Sui Network (~3,000 events in 2 minutes)
00:14  Indexer tries to process batch; memory spike to 1.2GB
00:14  Container killed by OOM killer (limit: 512Mi)
00:14  Pod restarts; immediately tries to re-process same batch
00:15  OOMKilled again → CrashLoopBackOff
00:16  On-call engineer paged
00:22  Temporary fix: `kubectl set resources` to increase limit to 2Gi
00:25  Pod stabilizes; begins processing backlog
00:37  All missed events indexed; service fully recovered
```

## Root Cause

```
1. Memory limit (512Mi) was set based on average usage, not peak batch processing
2. Indexer loads entire batch into memory before processing
3. No circuit breaker or memory-aware batch sizing
4. CrashLoopBackOff exponential backoff delayed recovery
```

## Resolution

### Immediate (during incident)
```bash
kubectl set resources deployment/suidex-indexer-backend \
  --limits=memory=2Gi --requests=memory=512Mi -n production
```

### Permanent fixes

**1. Increased memory limits with buffer:**
```yaml
# values.yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 2Gi  # 4x headroom for batch spikes
```

**2. Added memory-aware batch sizing in application code:**
```typescript
// Before: Fixed batch size
const BATCH_SIZE = 1000;

// After: Dynamic batch sizing based on available memory
const memUsage = process.memoryUsage();
const availableMB = (2048 - memUsage.heapUsed / 1024 / 1024);
const BATCH_SIZE = Math.max(50, Math.min(1000, Math.floor(availableMB / 2)));
```

**3. Added VPA (Vertical Pod Autoscaler) recommendation:**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: suidex-indexer-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: suidex-indexer-backend
  updatePolicy:
    updateMode: "Off"  # Recommendation only, don't auto-update
```

## Monitoring Added

```yaml
# PrometheusRule
- alert: ContainerMemoryNearLimit
  expr: |
    container_memory_working_set_bytes{namespace="production"}
    / container_spec_memory_limit_bytes > 0.85
  for: 3m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.pod }} memory at {{ $value | humanizePercentage }}"
```

## Preventive Measures

1. **Memory limits set at 3-4x average** for batch processing workloads
2. **VPA recommendations** reviewed weekly for all stateful services
3. **Application-level memory monitoring** — backpressure when heap exceeds 70%
4. **Runbook created** for OOMKill incidents with step-by-step recovery

## Lessons Learned

- Setting memory limits based on average usage is wrong for batch workloads. Use peak + 50% buffer.
- OOMKill → CrashLoopBackOff is a cascading failure: the pod retries the same failing work.
- Always add memory-aware batch sizing for data processing services.
- VPA in recommendation mode is free insight into actual resource needs.
