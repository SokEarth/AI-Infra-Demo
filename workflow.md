# Infrastructure Assignment Workflow

## Phase 1 — Architecture
- Review requirements
- Present infrastructure diagram
- Review HA/security boundaries

## Phase 2 — Repository Structure
- Generate repo layout
- Create folders [kubernetes, helm, scripts, terraform]
- Define ownership boundaries

## Phase 3 — Infrastructure
- Generate S3 backend (bootstrap) Terraform
- **After bootstrap apply: migrate state to S3.** The bootstrap root starts with `backend "local" {}`. After the first `terraform apply` creates the bucket + DynamoDB table, swap to `backend "s3" { ... }` populated with the real names (the bootstrap module's outputs print them), then run `terraform init -migrate-state` — answer `yes` to copy local state to S3. Subsequent roots reference the same bucket + table.
- Generate VPC Terraform
- Generate EKS Terraform
- **After EKS apply: update kubeconfig** so `kubectl` can reach the cluster. Run per env:
  ```bash
  aws eks update-kubeconfig --region <region> --name <org>-<env>-<region_short>-eks --alias <org>-<env>
  ```
  For this project (prod): `aws eks update-kubeconfig --region eu-west-1 --name somto-prod-euw1-eks --alias somto-prod`. Verify with `kubectl --context somto-prod get nodes` — Unauthorized means the IAM identity isn't in `access_entries` yet (re-apply EKS module to add).
- Generate RDS Terraform
- Generate IAM Terraform
- Generate EKS addons Terraform (Helm: external-secrets everywhere; ArgoCD prod-only — hub)
- After eks-addons apply: run [`scripts/utils/get-argocd-credentials.sh`](scripts/utils/get-argocd-credentials.sh) to fetch the initial ArgoCD admin password from the `argocd-initial-admin-secret` Secret. Login at the ArgoCD URL the script prints, rotate the password, then delete the initial secret.
- Review infrastructure

## Phase 4 — Kubernetes
- Generate Helm/Kustomize packages
- Generate manifests
- Apply kubernetes.md standards
- After Ingress is generated: ask if testing with local DNS; if yes, ask which hostname(s) to map to the ALB per env, then switch to HTTP-only with HTTPS kept as commented production reference
- Reminder: before working with the cluster in this phase, run [`scripts/utils/get-argocd-credentials.sh`](scripts/utils/get-argocd-credentials.sh) to log into the ArgoCD UI. Useful for inspecting manifest syncs and rollouts as you scaffold the apps. Rotate the password on first login and delete the initial secret.

## Phase 5 — GitOps
- Generate ArgoCD ApplicationSets
- Configure environments
- Hub-and-spoke: ArgoCD installed only in prod (eks-addons). Dev/stg are spoke clusters registered with the hub via `scripts/bootstrap/argocd-register-cluster.sh`.

## Phase 6 — CI/CD
- Generate GitHub Actions
  - `.github/workflows/ci.yaml` — PR validation (terraform fmt/validate/tfsec, kustomize build + kubeconform, ApplicationSet validation, shellcheck/shfmt)
  - `.github/workflows/terraform-apply.yaml` — manual workflow_dispatch with env+module inputs, OIDC auth, GitHub environment protection rules gate prod approval
- Add validation/linting

## Phase 7 — Validation
- helm lint
- kubectl dry-run

## Phase 8 — Deployment
- Deploy infrastructure
- Bootstrap ArgoCD
- Deploy applications
- If local DNS was chosen in Phase 4: once ALBs are up, run [`scripts/utils/setup-local-dns.sh`](scripts/utils/setup-local-dns.sh) — an executable RFC-PLAT-001-compliant script that discovers each env's ALB via `kubectl`, resolves to an IPv4 with `dig`, and writes an idempotent `# >>> somto-demo` ... `# <<< somto-demo` block to `/etc/hosts` (requires sudo). Supports `--dry-run`. Reminds you to mirror the entries into the Windows hosts file since WSL `/etc/hosts` is not visible to Windows browsers.
