# Kustomize Multi-Cluster Strategy Examples

A collection of Kustomize folder structure patterns for managing multi-cluster Kubernetes/OpenShift deployments. Each strategy is self-contained with realistic YAML and its own README explaining when and why to use it.

Designed for use with **Red Hat Advanced Cluster Management (RHACM)**, **OpenShift GitOps (ArgoCD)**, and **Open Cluster Management (OCM)**.

---

## Strategies

| # | Pattern | Use Case | Key Kustomize Feature |
|---|---------|----------|----------------------|
| [01](01-base-overlay/) | **Base/Overlay** | Compliance Operator on prod only | `resources`, `patches` |
| [02](02-components/) | **Component-Based** | Mix-and-match capabilities per environment | `components` (v1alpha1) |
| [03](03-multi-tenant/) | **Multi-Tenant / Per-Team** | Onboard teams with shared platform guardrails | `patches`, multi-base composition |
| [04](04-cluster-scoped/) | **Cluster-Scoped Configs** | ClusterRoles, SCCs, OAuth per cluster tier | `components` + cluster-scoped resources |
| [05](05-helm-kustomize/) | **Helm + Kustomize Hybrid** | Patch Helm chart output per environment | `helmCharts` generator |
| [06](06-gitops-multi-cluster/) | **GitOps-Ready Multi-Cluster** | Full RHACM + ArgoCD repo structure | Channels, Subscriptions, ApplicationSets |

---

## Quick Start

Preview what any overlay produces:

```bash
# Example: see what production compliance looks like
kustomize build 01-base-overlay/overlays/production/

# Example: see the component composition for staging
kustomize build 02-components/overlays/staging/

# Example: see all teams deployed to production
kustomize build 03-multi-tenant/clusters/production/
```

---

## How These Map to RHACM

Each strategy directory can be consumed by RHACM in one of two ways:

### 1. RHACM Subscriptions (Application Lifecycle)

RHACM Subscriptions watch a Git repo via a **Channel** and deploy resources to managed clusters selected by a **Placement**. Point the Subscription's `apps.open-cluster-management.io/github-path` annotation at the appropriate overlay directory:

```yaml
annotations:
  apps.open-cluster-management.io/github-path: 01-base-overlay/overlays/production
```

Each overlay = a different Subscription + Placement combination targeting clusters by label (e.g., `env=production`).

### 2. ArgoCD ApplicationSets (GitOps)

An ApplicationSet with a **git directory generator** scans the repo and auto-creates an ArgoCD Application per overlay:

```yaml
generators:
  - git:
      repoURL: https://github.com/your-org/kustomize-examples.git
      directories:
        - path: "*/overlays/*"
```

See [06-gitops-multi-cluster/applicationsets/](06-gitops-multi-cluster/applicationsets/) for a complete working example.

### 3. PolicyGenerator Plugin

For policy-driven enforcement (rather than direct resource deployment), use the RHACM **PolicyGenerator** Kustomize plugin to wrap any of these manifests into RHACM Policies with Placements and PlacementBindings. This is the recommended approach for compliance and governance use cases.

---

## Choosing a Strategy

```
Do you need to deploy the same thing with minor tweaks per environment?
  --> 01-base-overlay

Do you need to compose features a la carte (env A gets X+Y, env B gets X+Z)?
  --> 02-components

Do you onboard multiple teams with shared platform guardrails?
  --> 03-multi-tenant

Are you managing cluster-scoped resources (RBAC, SCCs, OAuth)?
  --> 04-cluster-scoped

Do you want to customize a Helm chart with Kustomize patches?
  --> 05-helm-kustomize

Do you need the full RHACM + ArgoCD repo structure with Channels/Subscriptions/ApplicationSets?
  --> 06-gitops-multi-cluster
```

Strategies can be combined. For example, use `02-components` inside a `06-gitops-multi-cluster` config directory.

---

## RHACM + Kustomize + GitOps References

### Official Red Hat Documentation

- [RHACM 2.15 GitOps Documentation](https://docs.redhat.com/es/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/gitops/index) -- Latest RHACM GitOps guide covering ArgoCD integration, ApplicationSets, and Subscription model
- [RHACM 2.13 GitOps Documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html-single/gitops/index) -- Covers PolicyGenerator init container setup with OpenShift GitOps
- [RHACM 2.11 PolicyGenerator Integration](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html/governance/integrate-policy-generator) -- Integrating the PolicyGenerator Kustomize plugin with ArgoCD
- [OCP 4.18 Managing Cluster Policies with PolicyGenerator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/edge_computing/managing-cluster-policies-with-policygenerator-resources) -- ZTP and edge computing policy management (replaces deprecated PolicyGenTemplate)
- [OCP 4.16 PolicyGenerator for Edge Computing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/edge_computing/managing-cluster-polices-with-policygenerator-resources) -- Common/group/site hierarchy pattern for fleet management

### GitHub -- Community Examples and Tools

- [open-cluster-management-io/policy-collection](https://github.com/open-cluster-management-io/policy-collection) -- Official OCM policy examples with PolicyGenerator plugin usage, Subscription/Channel/Placement samples, and kustomize integration
- [open-cluster-management-io/policy-generator-plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) -- The PolicyGenerator Kustomize exec plugin source code and documentation
- [stolostron/application-samples](https://github.com/open-cluster-management/application-samples) -- Official OCM application samples using Subscription, Channel, and Placement APIs
- [stolostron/demo-subscription-gitops](https://github.com/open-cluster-management/demo-subscription-gitops) -- End-to-end demo of RHACM Subscriptions with GitOps
- [open-cluster-management-io/multicloud-operators-subscription](https://github.com/open-cluster-management-io/multicloud-operators-subscription) -- Subscription operator source with Kustomize override examples (`spec.packageOverrides`)
- [redhat-cop/acm-policies](https://github.com/redhat-cop/acm-policies) -- Red Hat Consulting's curated policy layout for GitOps-managed RHACM policies (HIPAA, best practices)
- [bry-tam/acm-policy-samples](https://github.com/bry-tam/acm-policy-samples) -- Advanced patterns: dev/qa/prod promotion, ManagedClusterSets per environment, maintenance windows
- [noseka1/rhacm-kustomization](https://github.com/noseka1/rhacm-kustomization) -- Kustomize-based RHACM Hub deployment (deploying RHACM itself via Kustomize)
- [giannisalinetti/rhacm-gitops-example](https://github.com/giannisalinetti/rhacm-gitops-example) -- RHACM + OpenShift GitOps integration for multi-cluster infrastructure policies
- [josgonza-rh/rhacm-gitops](https://github.com/josgonza-rh/rhacm-gitops) -- Demo showing both Subscription-based and ApplicationSet-based GitOps deployment via RHACM

### Key Concepts Quick Reference

| RHACM Resource | Purpose |
|----------------|---------|
| **Channel** | Points to a source repository (Git, Helm, ObjectBucket) |
| **Subscription** | Subscribes to a Channel, deploys resources to selected clusters |
| **Placement** | Dynamically selects managed clusters by labels/clustersets |
| **PlacementBinding** | Binds a Placement to a Policy or PolicySet |
| **PolicyGenerator** | Kustomize plugin that wraps manifests into RHACM Policies |
| **ApplicationSet** | ArgoCD resource that auto-generates Applications from templates |
| **ManagedClusterSet** | Groups managed clusters for access control and placement scoping |

> **Note:** `PlacementRule` is deprecated. Use `Placement` (cluster.open-cluster-management.io/v1beta1) instead.
