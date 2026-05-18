# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A from-scratch AI-assisted platform engineering demo. The repo is the *scaffold target* — Claude generates the Terraform, Kubernetes manifests, Helm charts, Bash utilities, and CI workflows by walking the user through an 8-phase process. Treat each session as continuing that build, not maintaining an existing system.

**Current state:** entering Phase 3. No `terraform/` tree on disk yet; `.github/workflows/` is empty. The repo contains only [workflow.md](workflow.md), the RFC docs under [docs/](docs/), and this guidance file. Always re-check the actual directory contents before assuming where the build is — the "current state" line in CLAUDE.md is the most common thing to fall stale and should be updated as phases progress.

## Layout

- `terraform/modules/<name>/` — wrapper module (one per concern). Pins upstream community modules.
- `terraform/environments/<env>/<module>/` — root that instantiates the wrapper, sets provider + backend, reads sibling-root outputs via `terraform_remote_state`.
- Each root has the standard split: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `backend.tf`. The bootstrap root's `backend.tf` ships pre-populated with the S3 bucket/table names it will create — first apply uses local state, then `terraform init -migrate-state` moves it.
- `.tfvars` and `*.lock.hcl` are gitignored. Lock files are intentionally not committed for this demo.

## Canonical references — read these before generating anything

- [workflow.md](workflow.md) — the 8-phase script (Architecture → Repo → Infrastructure → Kubernetes → GitOps → CI/CD → Validation → Deployment). Includes the hand-off steps between phases (state migration after bootstrap, kubeconfig update after EKS, ArgoCD credentials fetch after eks-addons, local-DNS setup after ALBs).
- [docs/terraform.md](docs/terraform.md) — RFC-PLAT-002 (Terraform)
- [docs/kubernetes.md](docs/kubernetes.md) — RFC-PLAT-003 (Kubernetes)
- [docs/bashscript.md](docs/bashscript.md) — RFC-PLAT-001 (Bash)
- [docs/infrastructure.png](docs/infrastructure.png) — target architecture diagram

The RFC docs are the source of truth for standards; the summary below only flags points that recur as gotchas in this project.

## Project conventions

- **Org prefix:** `somto`. Resource names follow `somto-<env>-<region_short>-<service>` (e.g. `somto-prod-euw1-eks`).
- **Envs:** `dev`, `stg`, `prod` (note `stg`, not `staging`, in variable validation).
- **Region:** primary is `eu-west-1` / `euw1`.
- **Hub-and-spoke ArgoCD:** ArgoCD is installed in **prod only** via `eks-addons`. Dev and stg are spoke clusters registered with the prod hub via `scripts/bootstrap/argocd-register-cluster.sh`. `external-secrets` is installed in every env.
- **Demo scope:** this project name contains "demo" — CI omits shellcheck/shfmt, EKS `access_entries` skips a `ci_deployer` entry, and Ingresses default to HTTP-only with HTTPS kept as commented production reference.

## Phase 3 gating (most common point of failure)

After generating **each** Terraform module in Phase 3 (bootstrap → vpc → eks → rds → iam → eks-addons), stop and ask the user before scaffolding the next one. The user validates one module end-to-end (`fmt` → `validate` → `init` → `plan` → `apply`) before moving on. Chaining module generation produces compounding errors that are painful to unwind.

## Validation commands

```bash
terraform fmt -check -recursive
terraform validate
terraform plan                              # never apply directly to prod

helm lint ./charts/<chart>
kubectl apply --dry-run=client -f <manifest>
```

CI/CD pipeline order: `fmt` → `validate` → security scan → policy eval → `plan` → approval (prod) → `apply`.

## Standards quick-reference

These are the rules most likely to bite during generation. For the full lists see the `docs/` files.

**Terraform**
- Remote state only. Bootstrap root starts with `backend "local" {}`, then migrates to S3 after first apply (`terraform init -migrate-state`). Bootstrap also owns the GitHub Actions OIDC provider + per-env CI roles.
- All module sources version-pinned (`ref=v1.2.3`).
- Mandatory tags: `Environment`, `Owner`, `CostCenter`, `ManagedBy = "terraform"`.
- `variable "env"` must validate against `["dev", "stg", "prod"]`.
- Critical resources need `lifecycle { prevent_destroy = true }`.
- Encryption mandatory on S3, EBS, RDS, secrets. No `Action = "*"`, no public DB, no `0.0.0.0/0` SSH.

**Kubernetes**
- Every container: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`.
- Resource requests/limits mandatory (defaults: `cpu: 100m`, `memory: 128Mi` / `memory: 512Mi`).
- `readinessProbe` + `livenessProbe` always; `startupProbe` for slow-start services.
- Prod HA: ≥2 replicas, `PodDisruptionBudget minAvailable: 2`, pod anti-affinity on hostname, topology spread across zones.
- No `latest` tags. Default-deny `NetworkPolicy` per namespace. Secrets only via External Secrets Operator. All deploys via ArgoCD (no direct `kubectl apply` to prod).
- Kustomize: use `labels: [{ pairs: ..., includeSelectors: true }]`, not deprecated `commonLabels`.

**Bash**
- Header: `#!/usr/bin/env bash` + `set -euo pipefail` + `IFS=$'\n\t'`.
- Log format: `YYYY-MM-DDTHH:MM:SSZ [LEVEL] [SCRIPT] message`.
- Exit codes: 0 success, 1 generic, 2 validation, 3 dependency, 4 timeout, 5 permission.
- Must be idempotent and support `--dry-run`. No `eval`, no `|| true`, no hardcoded secrets.

## Available skills

This project ships skills the user expects Claude to invoke instead of hand-writing the equivalent files: `terraform-modules`, `terragrunt-generator`, `terragrunt-layout`, `app-scaffold`, `bash-generator`. When a phase task matches one of these, call the skill rather than producing the output from scratch.
