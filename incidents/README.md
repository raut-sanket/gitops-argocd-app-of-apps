# Real-World Incidents & Post-Mortems

This directory documents real production incidents encountered while managing GKE infrastructure for the Suidex DeFi platform. Each incident follows a structured format: detection, root cause, resolution, and preventive measures.

| # | Incident | Severity | Downtime | Root Cause |
|---|----------|----------|----------|------------|
| 1 | [GKE Cost Overrun — 58% Reduction](001-gke-cost-overrun.md) | P2 | None | Over-provisioned nodes + no autoscaling |
| 2 | [OOMKilled Pods — Indexer Service](002-oomkill-indexer.md) | P1 | 23 min | Memory limits too low for batch processing |
| 3 | [Nginx Ingress 502s Under Load](003-nginx-502-errors.md) | P1 | 12 min | Connection pool exhaustion + keep-alive mismatch |
| 4 | [CI/CD Pipeline Failure — Image Push Timeout](004-cicd-image-push-timeout.md) | P3 | None | Artifact Registry rate limiting + large image layers |
| 5 | [PostgreSQL Connection Exhaustion](005-postgres-connection-exhaustion.md) | P1 | 8 min | Connection leak in backend service + no pool limits |
