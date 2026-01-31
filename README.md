# production-url-platform-gitops

This repository contains the **GitOps configuration** for the `production-url-platform` project.

It defines **what is deployed** to each environment and is consumed by **ArgoCD** running in the Kubernetes cluster.

Application source code, CI pipelines, and image builds live in a separate repository.

---

## Purpose

- Act as the **single source of truth** for environment deployments
- Decouple application code from deployment state
- Enable safe, auditable **promotions between environments**
- Support GitOps-driven continuous delivery via ArgoCD

---

## Repositories Overview

| Repo | Responsibility |
|----|----|
| `production-url-platform` | Application code, Helm chart, CI (build/test/publish images) |
| `production-url-platform-gitops` | Environment configuration and ArgoCD applications |

---

## Repository Structure

```
.
├── argocd/
│   ├── projects/
│   │   └── url-platform.yaml
│   └── applications/
│       └── url-platform-dev.yaml
│
├── envs/
│   ├── dev/
│   │   └── values.yaml
│   ├── stg/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
│
└── README.md
```

---

## Environments

- **dev**
  - Automatically updated
  - Used for fast feedback and validation
- **stg**
  - Promotion target from dev
  - Used for pre-production validation
- **prod**
  - Will deploy only versioned releases (e.g. `vX.Y.Z`)
  - Not active yet

Each environment is deployed into a dedicated Kubernetes namespace and uses its own values file.

---

## Image Versioning Strategy

- **Per-service image tags**
- Each service can be promoted independently
- Image tags are pinned explicitly in `envs/<env>/values.yaml`

Example:
```yaml
images:
  urlService:
    tag: sha-abcdef1
  redirectService:
    tag: sha-1234567
  analyticsService:
    tag: sha-89abcd0
```

No `:latest` or mutable tags are used for deployments.

---

## ArgoCD Integration

- ArgoCD is installed and managed via Terraform in the infrastructure repository
- This repo defines:
  - `AppProject` for access control
  - `Application` resources per environment
- Helm charts are sourced from the application repo
- Values files are sourced from this GitOps repo using ArgoCD multi-source support

Initial sync is **manual**; auto-sync will be enabled after validation.

---

## Promotion Flow (Planned)

1. CI builds and publishes images on `main`
2. A PR updates `envs/dev/values.yaml` with new image tags
3. Promotion to `stg` is done via PR copying tags from `dev`
4. Production deployments will be triggered only by versioned releases

All promotions are Git-based and reviewable.

---

## Out of Scope

- CI pipelines and image builds
- Infrastructure provisioning
- Secret management
- ArgoCD Image Updater automation

---

## Notes

- This repository intentionally contains **no secrets**
- All values files should be safe to commit
- Any sensitive configuration must be injected via Kubernetes Secrets managed elsewhere

---

## Status

- GitOps structure initialized
- Dev environment wired to ArgoCD
- Stg/prod placeholders present
- Promotion automation to be added in follow-up work