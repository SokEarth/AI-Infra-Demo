# Kubernetes Standards (RFC-PLAT-003)

**Mandatory security context for all containers:**
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

**Mandatory resource limits:**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 512Mi
```

**Required probes:** `readinessProbe`, `livenessProbe`, and `startupProbe` (for slow-start services)

**Production HA requirements:**
- ≥ 2 replicas (prefer 3+)
- `PodDisruptionBudget` with `minAvailable: 2`
- Pod anti-affinity on `kubernetes.io/hostname`
- Topology spread across zones

Key rules:
- Image tags must be immutable — `latest` is forbidden
- Images must come from approved registries and be scanned (Trivy) and signed (Cosign)
- Default-deny NetworkPolicies required in all namespaces
- Secrets via External Secrets Operator or Vault — never plaintext in Git
- No wildcard RBAC verbs; no `cluster-admin` bindings for applications
- All deployments via GitOps (Argo CD) — direct `kubectl apply` in production is forbidden
- Workloads must expose metrics in Prometheus format and emit structured (JSON) logs
