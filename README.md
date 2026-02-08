# XMRig Kubernetes Deployment

Kubernetes deployment for XMRig cryptocurrency miner (Monero/XMR) that runs as a low-priority background workload on worker nodes.

## Overview

This deployment runs XMRig as a DaemonSet on all worker nodes (excluding control-plane nodes) with minimal system impact using Linux process priority controls (`nice -n 19` and `ionice -c3`).

### Key Features

- **DaemonSet deployment** - Runs on all worker nodes for distributed mining
- **Low priority** - Uses Kubernetes `priorityClassName: low-priority` and Linux scheduler controls
- **Node targeting** - Avoids control-plane nodes using node affinity
- **Network isolation** - NetworkPolicy restricts traffic to DNS and public internet only
- **MSR access** - Privileged mode for RandomX algorithm optimization via `/dev/cpu`
- **Memory limits** - Fixed 4Gi memory allocation (no CPU limits for flexible scheduling)

## Architecture

```
├── daemonset.yaml       # XMRig DaemonSet with mining pool configuration
├── networkpolicy.yaml   # Network isolation rules
├── kustomization.yaml   # Kustomize build configuration
└── .gitea/workflows/    # CI/CD validation and security scanning
    ├── validate.yaml       # YAML lint, kustomize build, kubeconform
    ├── security.yaml       # Trivy and Checkov security scans with PR reviews
    └── best-practices.yaml # kube-score and Polaris audits with PR reviews
```

## Deployment

### Prerequisites

- Kubernetes cluster with worker nodes
- `low-priority` PriorityClass defined in cluster
- Nodes with `/dev/cpu` available for MSR access

### Deploy

```bash
# Apply manifests using kustomize
kubectl apply -k .

# Or apply directly
kubectl apply -f networkpolicy.yaml
kubectl apply -f daemonset.yaml
```

### Verify

```bash
# Check DaemonSet status
kubectl get daemonset xmrig

# View running pods
kubectl get pods -l app.kubernetes.io/name=xmrig

# Check mining logs
kubectl logs -l app.kubernetes.io/name=xmrig -f
```

## Configuration

### Mining Pool

Edit the pool configuration in `daemonset.yaml`:

```yaml
- -o "pool.supportxmr.com:443"  # Pool address
- -u "YOUR_WALLET_ADDRESS"       # Wallet address
- -p "${POD_NAME}"               # Worker name (uses pod name)
```

### Resource Limits

Memory is fixed at 4Gi (request = limit) to prevent eviction:

```yaml
resources:
  requests:
    memory: "4Gi"
  limits:
    memory: "4Gi"
```

**Note:** CPU limits are intentionally not set to allow the Linux scheduler (`nice`/`ionice`) to dynamically allocate idle CPU cycles without Kubernetes interference.

## Security Considerations

⚠️ **This deployment requires privileged containers** for MSR access (`/dev/cpu`) needed by the RandomX algorithm. Security scanning tools will flag this as intentional:

- Privileged container mode
- Root user access
- Privilege escalation allowed
- Read-write root filesystem

All security exemptions are documented in `.checkov.yaml` and as Polaris annotations in `daemonset.yaml`.

## CI/CD Workflows

Three automated workflows run on PRs and main branch pushes:

1. **Validation** - YAML linting, kustomize build tests, Kubernetes schema validation
2. **Security** - Trivy and Checkov scans with automated PR reviews
3. **Best Practices** - kube-score and Polaris audits with automated PR reviews

### Automated PR Reviews

Security and best practices scans automatically post reviews:
- **CRITICAL** findings → Blocks merge (REQUEST_CHANGES)
- **HIGH** findings → Warning only (COMMENT)
- **Passing** → Approves (APPROVED)

## Development

### Local Validation

```bash
# Lint YAML
yamllint -c .yamllint.yaml .

# Build with kustomize
kubectl kustomize .

# Validate Kubernetes schemas
kubectl kustomize . | kubeconform \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary -ignore-missing-schemas

# Security scan
trivy config --severity CRITICAL,HIGH --ignorefile .trivyignore .

# Best practices analysis
kubectl kustomize . | kube-score score - \
  --ignore-test pod-networkpolicy \
  --ignore-test container-security-context-privileged \
  --output-format ci
```

### Testing with act

Test Gitea Actions workflows locally using [act](https://github.com/nektos/act):

```bash
# Run validation workflow
act pull_request -W .gitea/workflows/validate.yaml

# Run security scan
act pull_request -W .gitea/workflows/security.yaml

# Run best practices audit
act pull_request -W .gitea/workflows/best-practices.yaml
```

## Monitoring

Monitor mining performance through logs:

```bash
# Follow logs for all XMRig pods
kubectl logs -l app.kubernetes.io/name=xmrig -f --all-containers=true

# Get resource usage
kubectl top pods -l app.kubernetes.io/name=xmrig
```

## License

Unlicense - See [LICENSE](LICENSE) file.

## Disclaimer

This repository is for educational and personal use. Ensure you have permission to use compute resources for cryptocurrency mining. Mining may violate terms of service for cloud providers or organizational policies.
