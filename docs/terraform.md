# Terraform Standards (RFC-PLAT-002)

**Structure:**
```
modules/
  vpc/
  eks/
  rds/
environments/
  dev/
  staging/
  prod/
```

**Naming convention:** `<org>-<env>-<region>-<service>-<resource>`
Example: `awesome-prod-euw1-eks-cluster`

**Mandatory tags on all resources:**
```hcl
tags = {
  Environment = "prod"
  Owner       = "platform"
  CostCenter  = "engineering"
  ManagedBy   = "terraform"
}
```

**Variable validation required:**
```hcl
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "stg", "prod"], var.env)
    error_message = "Invalid environment"
  }
}
```

Key rules:
- Remote state only (S3 + DynamoDB lock, GCS, or Terraform Cloud) — local state is forbidden in shared envs
- All modules must be version-pinned (`ref=v1.2.3`) — no floating branches
- No wildcard IAM permissions (`"Action": "*"`)
- Encryption mandatory: S3, EBS, RDS, secrets storage
- No public databases; no open SSH (0.0.0.0/0)
- Secrets via AWS Secrets Manager or Vault — never in `.tf` files
- Critical resources must have `lifecycle { prevent_destroy = true }`
- CI/CD pipeline order: `fmt` → `validate` → security scan → policy eval → `plan` → approval (prod) → `apply`
- `terraform apply` directly against production is forbidden
