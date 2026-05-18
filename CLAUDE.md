# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an AI-assisted infrastructure demo that generates IaC (Terraform/Terragrunt), Kubernetes manifests, and Bash scripts following RFC platform standards. The workflow progresses through 8 phases: Architecture → Repo Structure → Infrastructure → Kubernetes → GitOps → CI/CD → Validation → Deployment.

## Validation Commands

```bash
# Terraform
terraform fmt -check -recursive
terraform validate
terraform plan          # never apply directly to prod

# Helm / Kubernetes
helm lint ./charts/<chart>
kubectl apply --dry-run=client -f <manifest>

# Bash scripts
shellcheck <script>.sh
shfmt -d <script>.sh
```

CI/CD pipeline order: `fmt` → `validate` → security scan → policy eval → `plan` → approval (prod) → `apply`

## Standards

### Terraform — RFC-PLAT-002

- Module structure: `modules/{vpc,eks,rds,...}/` and `environments/{dev,staging,prod}/`
- Naming convention: `<org>-<env>-<region>-<service>-<resource>` (e.g. `awesome-prod-euw1-eks-cluster`)
- Remote state only (S3 + DynamoDB, GCS, or Terraform Cloud) — local state is forbidden in shared envs
- All module refs must be version-pinned (`ref=v1.2.3`) — no floating branches
- No wildcard IAM (`"Action": "*"`); no public databases; no open SSH (`0.0.0.0/0`)
- Encryption mandatory: S3, EBS, RDS, secrets storage
- Secrets via AWS Secrets Manager or Vault — never in `.tf` files
- Critical resources: `lifecycle { prevent_destroy = true }`
- All resources require tags: `Environment`, `Owner`, `CostCenter`, `ManagedBy = "terraform"`
- Variable validation required for `env` (allowed: `dev`, `stg`, `prod`)

### Kubernetes — RFC-PLAT-003

- All containers require a security context: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`
- Resource limits are mandatory (requests: `cpu: 100m`, `memory: 128Mi`; limit: `memory: 512Mi`)
- Required probes: `readinessProbe`, `livenessProbe`, and `startupProbe` for slow-start services
- Production HA: ≥ 2 replicas (prefer 3+), `PodDisruptionBudget` with `minAvailable: 2`, pod anti-affinity on `kubernetes.io/hostname`, topology spread across zones
- Image tags must be immutable — `latest` is forbidden; images must be scanned (Trivy) and signed (Cosign)
- Default-deny `NetworkPolicy` required in all namespaces
- Secrets via External Secrets Operator or Vault — never plaintext in Git
- No wildcard RBAC verbs; no `cluster-admin` bindings for applications
- All deployments via GitOps (Argo CD) — direct `kubectl apply` in production is forbidden
- Workloads must expose Prometheus metrics and emit structured JSON logs

### Bash Scripts — RFC-PLAT-001

Every script must start with:
```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- Structured log format: `YYYY-MM-DDTHH:MM:SSZ [LEVEL] [SCRIPT] message`
- Exit codes: `0`=success, `1`=generic, `2`=validation, `3`=dependency, `4`=timeout, `5`=permission
- `eval`, hardcoded secrets, and `|| true` silent failures are forbidden
- All inputs validated before use: `: "${ENV:?ENV is required}"`
- Scripts must be idempotent and support `--dry-run`
- Debug mode: `[[ "$DEBUG" == "true" ]] && set -x`
- Lint with `shellcheck`, format with `shfmt` before committing
