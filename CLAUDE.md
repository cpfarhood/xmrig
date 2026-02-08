# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a Kubernetes deployment for XMRig cryptocurrency miner (Monero/XMR). The deployment runs as a DaemonSet on worker nodes with minimal resource priority to avoid impacting cluster workloads.

**IMPORTANT**: This repository contains cryptocurrency mining software. While you can analyze the code and help with Kubernetes configuration, security, and best practices, be aware that the core functionality involves mining operations.

## Architecture

### Core Components

- **daemonset.yaml**: Main XMRig DaemonSet that runs on worker nodes (non-control-plane)
  - Runs with `priorityClassName: low-priority` and uses `nice -n 19` and `ionice -c3` for minimal system impact
  - Requires privileged security context for MSR (Model-Specific Register) access via `/dev/cpu`
  - Memory limits set to 4Gi (no CPU requests/limits to avoid resource contention)
  - Avoids control-plane nodes using node affinity rules

- **networkpolicy.yaml**: NetworkPolicy restricting pod traffic
  - Allows DNS resolution to kube-system
  - Permits namespace-local communication
  - Allows egress to public internet only (blocks private IP ranges)

- **kustomization.yaml**: Kustomize build configuration that bundles both manifests

### Gitea CI/CD Workflows

The repository uses Gitea Actions (compatible with GitHub Actions) with three standardized workflows:

1. **validate.yaml**: Manifest validation
   - YAML linting with yamllint
   - Kustomize build tests
   - kubeconform Kubernetes schema validation
   - Flux build validation

2. **security.yaml**: Security scanning with automated PR reviews
   - Trivy config scanning with automated PR reviews (CRITICAL findings block, HIGH warns)
   - Checkov IaC scanning with automated PR reviews (CRITICAL findings block, HIGH warns)
   - Both post review comments directly to PRs using Gitea API

3. **best-practices.yaml**: Best practices and standards
   - kube-score Kubernetes best practices analysis
   - Polaris security/reliability audit with automated PR reviews
   - Resource usage analysis reporting

### PR Review Workflow

Security and best practices scans automatically post reviews to PRs:
- **CRITICAL** findings → `REQUEST_CHANGES` state (blocks merge)
- **HIGH** findings → `COMMENT` state (warns but doesn't block)
- **No issues** → `APPROVED` state

Review tokens are configured via secrets: `TRIVY_GITEA_TOKEN`, `CHECKOV_GITEA_TOKEN`, `POLARIS_GITEA_TOKEN`.

## Common Commands

### Validation and Testing

```bash
# Lint YAML files
yamllint -c .yamllint.yaml .

# Build with kustomize
kubectl kustomize .

# Validate with kubeconform
kubectl kustomize . | kubeconform \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary -ignore-missing-schemas

# Flux build validation
flux build kustomization homeassistant --path . --dry-run
```

### Security Scanning

```bash
# Trivy config scan
trivy config --severity CRITICAL,HIGH --ignorefile .trivyignore .

# Checkov IaC scan
checkov -d . --config-file .checkov.yaml --output cli
```

### Best Practices Analysis

```bash
# kube-score analysis
kubectl kustomize . | kube-score score - \
  --ignore-test pod-networkpolicy \
  --ignore-test deployment-has-poddisruptionbudget \
  --ignore-test container-security-context-user-group-id \
  --ignore-test container-security-context-readonlyrootfilesystem \
  --output-format ci

# Polaris audit
kubectl kustomize . > manifests.yaml
polaris audit --audit-path manifests.yaml --format pretty \
  --set-exit-code-on-danger --set-exit-code-below-score 70
```

### Deployment

```bash
# Apply to cluster
kubectl apply -k .

# Check DaemonSet status
kubectl get daemonset xmrig -n <namespace>
kubectl describe daemonset xmrig -n <namespace>

# View pod logs
kubectl logs -l app.kubernetes.io/name=xmrig -n <namespace>
```

## Configuration Files

- **.checkov.yaml**: Checkov security scanner configuration
  - Skips: CKV_K8S_21 (default namespace), CKV_K8S_43 (image tag validation)

- **.trivyignore**: Trivy vulnerability exceptions
  - CVE-2021-26720 (Avahi in base images)
  - CVE-2023-52425 (accepted risk)

- **.yamllint.yaml**: YAML linting rules
  - Disables: line-length, document-start, truthy checks

## Key Architectural Decisions

1. **DaemonSet over Deployment**: Ensures mining runs on all worker nodes for distributed resource utilization

2. **No CPU requests/limits**: Allows the Linux scheduler to manage CPU allocation dynamically without Kubernetes interference, combined with `nice`/`ionice` for true background priority

3. **Privileged security context**: Required for MSR access (`/dev/cpu`) which XMRig uses for RandomX algorithm optimization

4. **Network policy isolation**: Restricts communication to essential services (DNS, public mining pool) while blocking private network access

5. **Node affinity**: Explicitly avoids control-plane nodes to prevent mining from impacting cluster management

## Workflow Integration

All three workflows run on:
- Push to `main` branch
- Pull requests targeting `main`

Workflows use `catthehacker/ubuntu:act-latest` container image for consistency with local testing using [act](https://github.com/nektos/act).

When modifying manifests, expect automated reviews from Trivy, Checkov, and Polaris. Address CRITICAL findings before merge; HIGH findings are advisory.
