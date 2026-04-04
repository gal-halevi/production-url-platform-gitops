# production-url-platform-gitops

This repository contains the **GitOps configuration** for the `production-url-platform` project.

It defines **what is deployed** to each environment and is consumed by **ArgoCD** running in the Azure AKS cluster. Application source code, CI pipelines, image builds, and the Helm chart live in the [application repository](https://github.com/gal-halevi/production-url-platform).

---

## Purpose

- Act as the **single source of truth** for environment deployments
- Decouple application code from deployment state
- Enable safe, auditable **promotions between environments**
- Support GitOps-driven continuous delivery via ArgoCD

---

## Repositories overview

| Repo | Responsibility |
|---|---|
| `production-url-platform` | Application code, Helm chart, CI (build/test/publish images) |
| `production-url-platform-gitops` | ArgoCD Applications, per-environment values, observability stack config, Grafana dashboards, smoke tests |

---

## Repository structure

```
.
├── argocd/
│   ├── projects/
│   │   ├── platform.yaml         # ArgoCD AppProject for platform tooling
│   │   └── url-platform.yaml     # ArgoCD AppProject for the URL platform
│   └── applications/
│       ├── monitoring.yaml       # kube-prometheus-stack Application
│       ├── alloy.yaml            # Grafana Alloy log collection agent
│       ├── loki.yaml             # Grafana Loki log aggregation
│       ├── tempo.yaml            # Grafana Tempo distributed tracing
│       ├── otel-collector.yaml   # OpenTelemetry Collector (trace ingestion)
│       ├── url-platform-dev.yaml
│       ├── url-platform-stg.yaml
│       └── url-platform-prod.yaml
│
├── envs/
│   ├── dev/values.yaml           # Dev image tags + host names + feature flags
│   ├── stg/values.yaml           # Stg image tags + host names + feature flags
│   ├── prod/values.yaml          # Prod image tags + host names + feature flags
│   └── shared/                   # Shared values for cluster-wide platform tooling
│       ├── alloy-values.yaml
│       ├── loki-values.yaml
│       ├── otel-collector-values.yaml
│       └── tempo-values.yaml
│
├── monitoring/
│   ├── dashboards/
│   │   ├── kustomization.yaml
│   │   └── assets/
│   │       └── url-platform-all-services.json   # Grafana dashboard
│   ├── loki-datasource.yaml      # Grafana Loki datasource (with Tempo trace links)
│   └── tempo-datasource.yaml     # Grafana Tempo datasource (with Loki log links)
│
└── .github/workflows/
    ├── _smoke-env.yml            # Reusable smoke test workflow
    ├── smoke-dev.yml
    ├── smoke-stg.yml
    └── smoke-prod.yml
```

---

## Environments

| Environment | Update method | Auto-sync |
|---|---|---|
| dev | Auto-updated by CI on every merge to `main` | Yes, with prune |
| stg | PR-based promotion from dev (via `promote-stg` workflow) | Yes, with prune |
| prod | PR-based promotion from stg (via `promote-prod` workflow) | Yes, with prune |

Each environment is deployed into a dedicated Kubernetes namespace and uses its own values file. ArgoCD uses multi-source Applications, combining the Helm chart from the application repo with the values file from this repo.

---

## Image versioning strategy

Each service has an independent image tag, allowing per-service independent promotion. All tags are immutable `sha-XXXXXXX` values (first 7 chars of the commit SHA). No mutable tags (e.g. `latest` or `main`) are used for deployments.

```yaml
images:
  urlService:
    tag: sha-8eefcfe
  redirectService:
    tag: sha-cb47b20
  analyticsService:
    tag: sha-3ec2b0a
```

---

## Promotion flow

```
merge to main
    │
    ▼
CI builds & publishes images (GHCR)
    │
    ▼
CI auto-updates envs/dev/values.yaml
    │
    ▼
ArgoCD syncs dev (auto)
    │
    ▼
smoke-dev.yml runs e2e verification
    │
    ▼ (manual: promote-stg workflow_dispatch)
PR opens against envs/stg/values.yaml
    │
    ▼ (PR merged)
ArgoCD syncs stg (auto)
    │
    ▼
smoke-stg.yml runs e2e verification
    │
    ▼ (manual: promote-prod workflow_dispatch)
PR opens against envs/prod/values.yaml
    │
    ▼ (PR merged)
ArgoCD syncs prod (auto)
    │
    ▼
smoke-prod.yml runs e2e verification
```

---

## Smoke tests

Each environment has a triggered smoke workflow (`smoke-<env>.yml`) that runs after ArgoCD syncs. The reusable `_smoke-env.yml` workflow:

1. Reads host names and expected image tags from the environment's values file
2. Waits for url-service to report the correct deployed version at `/health`
3. Creates a short URL via url-service
4. Verifies the redirect resolves correctly via redirect-service
5. Verifies the analytics event was recorded via analytics-service

The version check ensures the smoke test only passes once the expected version is actually running, not on a stale deployment.

---

## ArgoCD integration

ArgoCD is installed and managed via Terraform in the application repository (`infra/aks/02-bootstrap`). This repo defines:

- `AppProject` resources for access control and source restrictions
- `Application` resources per environment, using multi-source to combine chart + values
- Automated sync with self-healing and pruning enabled

---

## Observability stack

In addition to the URL platform applications, this repo manages the full observability stack deployed to the `monitoring` namespace:

| Component | Chart | Purpose |
|---|---|---|
| kube-prometheus-stack | `prometheus-community/kube-prometheus-stack` | Prometheus, Alertmanager, Grafana |
| Grafana Alloy | `grafana/alloy` | DaemonSet log collector — scrapes pod logs, parses JSON, ships to Loki |
| Grafana Loki | `grafana/loki` | Log aggregation backend |
| Grafana Tempo | `grafana/tempo` | Distributed tracing backend (Azure Blob Storage, Workload Identity) |
| OpenTelemetry Collector | `open-telemetry/opentelemetry-collector` | OTLP trace ingestion → Tempo |

Shared values for these components live in `envs/shared/`. Grafana datasource ConfigMaps (`monitoring/loki-datasource.yaml`, `monitoring/tempo-datasource.yaml`) wire up cross-signal correlation: traces link to logs, logs link to traces, both link to Prometheus metrics.

---

## Out of scope

- CI pipelines and image builds (live in the application repo)
- Infrastructure provisioning (Terraform in the application repo)
- Secret management (Kubernetes Secrets managed by Terraform)

---

## Notes

- This repository intentionally contains **no secrets**
- All values files are safe to commit — sensitive configuration is injected via Kubernetes Secrets managed by Terraform