# 06 - GitOps Multi-Cluster with RHACM and ArgoCD

This example demonstrates a production-grade GitOps approach for managing
configuration across multiple OpenShift clusters using Red Hat Advanced
Cluster Management (RHACM) and ArgoCD ApplicationSets.

## Architecture Overview

The repository is organized into three layers:

### 1. ACM Resources (Hub Cluster)

Resources deployed on the RHACM hub cluster to orchestrate multi-cluster delivery:

- **Channels** define the Git repository connection. The `git-channel.yaml`
  points to this repository and references credentials stored in a Secret.
- **Placements** select target clusters using label selectors. Three placement
  rules exist: `all-clusters` (any OpenShift cluster), `production-only`
  (clusters labeled `env=production`), and `non-production` (all others).
- **Subscriptions** tie a channel path to a placement. Each subscription uses
  the `apps.open-cluster-management.io/github-path` annotation to point at a
  specific Kustomize overlay or base directory, so only the relevant config
  is delivered to each cluster group.

### 2. Config Directory (Kustomize Bases and Overlays)

All cluster configuration lives under `config/`, organized by concern:

- **compliance/** -- Installs the Compliance Operator via OLM Subscription.
  The production overlay adds a `ScanSetting` for daily compliance scans.
- **namespaces/** -- Declares standard namespaces (`team-alpha`, `team-beta`,
  `shared-services`). Production adds extra namespaces for extended monitoring
  and security scanning. Non-production inherits the base with an environment
  label applied via `commonLabels`.
- **monitoring/** -- Creates the `custom-monitoring` namespace and deploys
  PrometheusRule alert definitions. The production overlay patches in
  additional critical-severity alerts (disk usage, CPU critical) and tightens
  thresholds. Non-production uses the base rules with relaxed defaults.

Each concern follows the standard Kustomize base/overlay pattern, giving
teams independent release cycles while sharing common definitions.

### 3. ApplicationSets (ArgoCD)

The `applicationsets/appset.yaml` uses the ArgoCD Git directory generator to
scan `config/*/overlays/*`. For every overlay directory discovered, it
automatically creates an ArgoCD Application with:

- Automated sync (prune + self-heal enabled)
- Namespace creation allowed
- Application naming derived from the config area and environment

## How the Flow Works

1. A developer pushes a change to `config/monitoring/base/prometheus-rule.yaml`.
2. RHACM Subscriptions detect the commit on the `main` branch.
3. Based on placement rules, the subscription delivers the updated Kustomize
   path to matching managed clusters.
4. ArgoCD ApplicationSets independently detect the directory change and sync
   the corresponding Application on each target cluster.
5. Both RHACM and ArgoCD reconcile the desired state, ensuring drift is
   corrected automatically.

## Adding a New Cluster

1. Import the cluster into RHACM.
2. Apply the appropriate labels (`vendor: OpenShift`, `env: production` or
   `env: staging`, etc.).
3. Placement rules automatically include the cluster in the correct group.
4. RHACM Subscriptions begin delivering configuration immediately.

## Adding a New Environment

1. Create a new overlay directory under the relevant config area, e.g.,
   `config/monitoring/overlays/staging/`.
2. Add a `kustomization.yaml` referencing `../../base` with any
   environment-specific patches.
3. The ApplicationSet generator picks up the new directory on the next sync
   cycle and creates an ArgoCD Application for it automatically.
4. Optionally, add a new Placement and Subscription in `acm/` if RHACM-based
   delivery is also desired for the new environment.

## Directory Structure

Each config area under `config/` follows the standard Kustomize layout:

```
config/<area>/
  base/              # Shared resources and kustomization.yaml
  overlays/
    production/       # Production-specific patches and resources
    non-production/   # Non-production overrides (labels, relaxed settings)
```

This separation ensures that base definitions are never duplicated and that
environment-specific changes are isolated, auditable, and easy to review.
