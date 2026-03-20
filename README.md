# GitOps ArgoCD App-of-Apps — Production DeFi Platform

Production-grade GitOps deployment managing **15 microservices** and **6 infrastructure addons** on **Google Kubernetes Engine (GKE)** using the **ArgoCD App-of-Apps pattern**. Powers a decentralized exchange (DEX) platform on the Sui blockchain handling real-time on-chain data indexing, API serving, and multi-frontend delivery.

---

## Problem Statement

Managing 15+ microservices across multiple namespaces with manual `kubectl apply` or ad-hoc Helm installs creates configuration drift, deployment inconsistencies, and operational overhead. Need a single source of truth that can declaratively manage the entire platform — from infrastructure addons (ingress, cert-manager, monitoring) to application workloads (APIs, frontends, databases) — with automated sync, rollback capabilities, and environment separation.

## Solution

Implemented a **GitOps-driven deployment architecture** using ArgoCD's App-of-Apps pattern where a single root Application manages all child applications. Every deployment change goes through Git — providing full audit trail, peer review via MRs, and automated reconciliation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GitLab Repository                       │
│                    (Single Source of Truth)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  addons/                          apps/                     │
│  ├── argo-cd/                     ├── suidex-api-backend/   │
│  ├── argo-cd-apps/ (Root App)     ├── suidex-frontend/      │
│  ├── cert-manager/                ├── suidex-indexer/        │
│  ├── external-secrets/            ├── suidex-worker/         │
│  ├── ingress-nginx/               ├── suidex-postgres/       │
│  ├── kube-prometheus-stack/       ├── redis/                 │
│  └── loki-stack/                  ├── dex-frontend/          │
│                                   ├── farm-suidex-frontend/  │
│                                   ├── presale-backend/       │
│                                   └── presale-frontend/      │
└──────────────────────┬──────────────────────────────────────┘
                       │ Git Poll / Webhook
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    ArgoCD Server (GKE)                       │
│              App-of-Apps Root Application                    │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Auto-Sync   │  │ Self-Heal    │  │ Prune Orphans     │  │
│  │ (argo-cd-   │  │ (drift       │  │ (remove deleted   │  │
│  │  apps only) │  │  correction) │  │  resources)       │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │ Deploy
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   GKE Production Cluster                    │
│            e2-custom-2-8192 · Autoscaling 0–3 nodes         │
│            3 Availability Zones · 50+ Pods                  │
│                                                             │
│  Namespaces:                                                │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐    │
│  │ argocd       │ │ postgres   │ │ suidex-api-backend │    │
│  │ ingress-nginx│ │ redis      │ │ suidex-frontend    │    │
│  │ cert-manager │ │ monitoring │ │ dex-frontend-prod  │    │
│  │ ext-secrets  │ │ loki       │ │ presale-prod       │    │
│  └──────────────┘ └────────────┘ └────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Features

- **App-of-Apps Pattern** — Single root application manages all child apps and addons declaratively
- **Environment Separation** — Mainnet and testnet configurations via value overrides (`values.mainnet.yaml`, `values.testnet.yaml`)
- **Standardized Helm Charts** — All 12 application charts follow identical template patterns (deployment, service, ingress, HPA, configmap, external secrets)
- **External Secrets Integration** — GCP Secret Manager via ExternalSecrets operator (60s refresh)
- **Automated TLS** — cert-manager with Let's Encrypt ClusterIssuer for all ingress endpoints
- **Horizontal Pod Autoscaling** — CPU-target HPAs (70-80%) with min/max replica bounds
- **Resource Governance** — Explicit requests/limits on every workload preventing unbounded consumption
- **Retry & Self-Healing** — Exponential backoff sync retries (5s→20s→3m) with automated drift correction
- **Monitoring Stack** — Prometheus + Grafana + Loki for metrics, dashboards, and log aggregation

## Tech Stack

| Component | Technology |
|---|---|
| **Orchestration** | Kubernetes (GKE) |
| **GitOps** | ArgoCD v2.13.1 (App-of-Apps) |
| **Package Management** | Helm v3 |
| **Ingress** | NGINX Ingress Controller v1.13.3 |
| **TLS** | cert-manager v1.16.2 + Let's Encrypt |
| **Secrets** | External Secrets Operator v0.11.0 + GCP Secret Manager |
| **Monitoring** | kube-prometheus-stack v77.12.0, Grafana, Google Managed Prometheus |
| **Logging** | Loki v6.24.0 + Promtail (GCS backend) |
| **Database** | TimescaleDB (PostgreSQL 16) with StatefulSet + PVC |
| **Cache** | Redis |
| **CI/CD** | GitLab CI/CD |

## Repository Structure

```
.
├── addons/                          # Infrastructure components
│   ├── argo-cd/                     # ArgoCD server installation
│   ├── argo-cd-apps/                # Root App-of-Apps (manages everything)
│   │   ├── templates/
│   │   │   ├── application.yaml     # Dynamic Application CRD generator
│   │   │   └── projects.yaml        # ArgoCD project definitions
│   │   ├── values.yaml              # Base configuration
│   │   ├── values.mainnet.yaml      # Production app/addon registry
│   │   └── values.testnet.yaml      # Staging configuration
│   ├── cert-manager/                # TLS certificate automation
│   ├── cert-clusterissuer/          # Let's Encrypt ClusterIssuer
│   ├── external-secrets/            # GCP Secret Manager integration
│   ├── ingress-nginx/               # NGINX Ingress Controller
│   ├── kube-prometheus-stack/       # Prometheus + Grafana monitoring
│   └── loki-stack/                  # Log aggregation (Loki + Promtail)
│
├── apps/                            # Application workloads
│   ├── suidex-api-backend/          # Main REST API (HPA: 1-3 replicas)
│   ├── suidex-frontend/             # Main DEX UI (suidex.org)
│   ├── suidex-indexer-backend/      # Blockchain event indexer
│   ├── suidex-worker-backend/       # Background job processor (HPA: 2-4)
│   ├── suidex-postgres/             # TimescaleDB StatefulSet (10Gi PVC)
│   ├── dex-frontend/                # DEX trading interface
│   ├── farm-suidex-frontend/        # Yield farming UI
│   ├── presale-backend/             # Token presale API
│   ├── presale-frontend/            # Presale UI
│   ├── redis/                       # Redis cache/queue
│   ├── api-backend/                 # Legacy API service
│   └── event-backend/               # Event processing service
│
└── README.md
```

## How It Works

### Adding a New Application

1. Create Helm chart under `apps/<app-name>/` following the standard template
2. Register the app in `addons/argo-cd-apps/values.mainnet.yaml`:
   ```yaml
   apps:
     - name: my-new-app
       namespace: my-new-app
       releaseName: my-new-app
       autosync: false
   ```
3. Commit and push — ArgoCD detects the change and creates the Application CR
4. Sync manually or enable `autosync: true` for automated deployment

### Deploying Infrastructure Addons

Addons are registered separately and can use either Helm charts (with dependencies) or raw manifests:
```yaml
addons:
  - name: cert-manager
    namespace: cert-manager
    autosync: false
  - name: cert-clusterissuer    # Raw manifest example
    source:
      directory: true
    autosync: false
```

## Production Metrics

| Metric | Value |
|---|---|
| Applications managed | 15 (10 apps + 5 addons) |
| Total pods | 50+ |
| Monthly cloud cost | $106 (reduced from $250) |
| Node pool | e2-custom-2-8192, autoscaling 0–3 |
| Availability zones | 3 |
| Database size migrated | 694MB (25 tables, 252K+ rows) |
| Downtime during migrations | Zero |

---

## Author

**Sanket Raut** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/sanket-raut) · [Email](mailto:sanketraut.cloud@gmail.com) · [GitHub](https://github.com/raut-sanket)
