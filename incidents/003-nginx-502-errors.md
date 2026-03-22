# Incident 003: Nginx Ingress 502 Errors Under Load

**Date:** March 2024  
**Severity:** P1 (Partial Outage)  
**Duration:** 12 minutes  
**Impact:** ~15% of API requests returning 502; frontend displaying error state  

---

## Summary

During a trading volume spike (Sui token listing event), the Nginx Ingress Controller started returning 502 Bad Gateway errors for ~15% of requests to the `api-backend` service. Backend pods were healthy; the issue was between Nginx and the upstream pods.

## Detection

- **14:32 UTC** — Grafana alert: `5xx error rate > 5%` for `api.suidex.io`
- **14:33 UTC** — User reports on Discord: "Getting error page on swaps"
- **14:34 UTC** — Nginx ingress access logs showing 502s with `upstream connect() failed`

## Timeline

```
14:30  Sui token listing event → 3x traffic spike (200 → 600 RPS)
14:31  HPA scales api-backend from 2 → 5 pods
14:32  502 errors begin appearing (new pods not yet ready)
14:32  Grafana alert fires
14:34  Engineer identifies issue: upstream connection failures in nginx logs
14:36  Root cause identified: keep-alive mismatch + readiness probe timing
14:38  Applied config changes: upstream keep-alive + readiness probe fix
14:44  Error rate drops to 0%; all clear
```

## Root Cause

Three compounding issues:

**1. Keep-alive connection mismatch**
```
Nginx default: keepalive_timeout 75s
Node.js default: server.keepAliveTimeout 5s (too low!)

Result: Nginx sends request on a connection that Node.js already closed
        → 502 Bad Gateway
```

**2. Readiness probe too aggressive**
```yaml
readinessProbe:
  initialDelaySeconds: 2   # Too fast — app not fully initialized
  periodSeconds: 1          # Too frequent
```
New pods marked "Ready" before the HTTP server was accepting connections.

**3. No `proxy_next_upstream` retry**
Nginx didn't retry failed requests on healthy upstream pods.

## Resolution

### Fix 1: Align keep-alive timeouts
```javascript
// api-backend server.js
const server = app.listen(PORT);
server.keepAliveTimeout = 90000;    // 90s (> nginx 75s)
server.headersTimeout = 95000;      // Must be > keepAliveTimeout
```

### Fix 2: Add Nginx upstream keep-alive
```yaml
# ingress-nginx values.yaml
controller:
  config:
    upstream-keepalive-connections: "100"
    upstream-keepalive-timeout: "60"
    proxy-next-upstream: "error timeout http_502 http_503"
    proxy-next-upstream-tries: "3"
```

### Fix 3: Fix readiness probe
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10    # Wait for app to fully initialize
  periodSeconds: 5           # Less aggressive
  failureThreshold: 3
startupProbe:                  # Added startup probe for slow starts
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 2
  failureThreshold: 15
```

### Fix 4: Pre-stop lifecycle hook
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```
Gives Nginx time to stop sending requests before pod termination.

## Monitoring Added

```yaml
- alert: IngressHighErrorRate
  expr: |
    sum(rate(nginx_ingress_controller_requests{status=~"5.."}[2m]))
    / sum(rate(nginx_ingress_controller_requests[2m])) > 0.02
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Ingress 5xx rate: {{ $value | humanizePercentage }}"
```

## Preventive Measures

1. **Keep-alive alignment** documented as standard for all Node.js services
2. **Startup probes** added to all deployments (not just readiness)
3. **`proxy_next_upstream`** configured globally in ingress-nginx values
4. **Load testing** with `k6` before major events:
   ```bash
   k6 run --vus 100 --duration 5m load-test.js
   ```
5. **Pre-scaling** before known traffic events: manually set HPA min replicas

## Lessons Learned

- 502s between Nginx and upstream are almost always keep-alive timeout mismatches or premature readiness.
- Readiness probes should verify the app is actually serving, not just that the process started.
- `proxy_next_upstream` is the difference between 1 retry stopping the error cascade vs. a full outage.
- Pre-stop hooks + sleep prevent in-flight request drops during pod termination.
