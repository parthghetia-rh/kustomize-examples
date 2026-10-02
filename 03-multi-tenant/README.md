# Strategy 3: Multi-Tenant / Per-Team Pattern

## Overview

This pattern separates **platform concerns** from **team workloads**. The platform team maintains
a shared base with guardrails (ResourceQuota, NetworkPolicy, RBAC), and each tenant team overlays
their own applications on top. A cluster-level aggregation layer controls which teams deploy to
which clusters.

## When to Use

- Multiple teams share cluster infrastructure and need consistent guardrails.
- The platform team wants to enforce quotas, network isolation, and RBAC without each team
  having to define them.
- Teams should be able to add their own workloads without modifying platform config.
- Different clusters host different subsets of teams.

## How It Works

### Platform Base (`platform-base/`)

Contains the "tenant template" -- a Namespace, ResourceQuota, default-deny NetworkPolicy, and
RoleBinding that grants a team group the `edit` ClusterRole. All resources use the placeholder
name `placeholder`, which each team patches to their own namespace.

### Team Directories (`teams/`)

Each team's `kustomization.yaml`:
1. References `../../platform-base` to inherit the guardrails.
2. Uses `namespace:` to set the namespace for all resources.
3. Applies patches to rename the Namespace from `placeholder` to the team name, update labels,
   and set the correct RBAC group in the RoleBinding.
4. Adds team-specific resources (Deployments, Services, etc.).

### Cluster Aggregation (`clusters/`)

Each cluster directory references the teams that should be deployed there. This is the entry
point for `kustomize build`.

## Directory Layout

```
03-multi-tenant/
├── platform-base/                 # Shared tenant template
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── resourcequota.yaml
│   ├── networkpolicy.yaml
│   └── rolebinding.yaml
├── teams/
│   ├── team-alpha/                # Each team patches the platform base
│   │   ├── kustomization.yaml
│   │   ├── namespace-patch.yaml
│   │   └── app-deployment.yaml
│   ├── team-beta/
│   │   └── ...
│   └── team-gamma/
│       └── ...
└── clusters/
    ├── production/                # All three teams
    ├── staging/                   # team-alpha + team-beta
    └── development/               # team-alpha only
```

## Usage

```bash
# Build everything for production (all teams)
kustomize build clusters/production

# Build only what goes to development (team-alpha)
kustomize build clusters/development
```

## Mapping to RHACM

This pattern maps naturally to RHACM in several ways:

### Option A: One ApplicationSet with Git Generator

Use an Argo CD ApplicationSet with a Git directory generator pointing at `clusters/`.
Each cluster's `kustomization.yaml` becomes the source for that cluster's Application.

### Option B: Separate RHACM Subscriptions per Team

Create one RHACM Channel pointing at the Git repo, then one Subscription per team directory.
Each Subscription uses a different PlacementRule to target the correct clusters:

| Team         | Subscription Path       | PlacementRule                                    |
|--------------|-------------------------|--------------------------------------------------|
| team-alpha   | `teams/team-alpha`      | All clusters (`environment in (dev, stg, prod)`) |
| team-beta    | `teams/team-beta`       | Staging + Production                             |
| team-gamma   | `teams/team-gamma`      | Production only                                  |

### Option C: Cluster-Level Subscriptions

One Subscription per cluster directory, with a PlacementRule that matches exactly one cluster:

| Path                    | Placement                                  |
|-------------------------|--------------------------------------------|
| `clusters/production`   | `matchLabels: { environment: production }` |
| `clusters/staging`      | `matchLabels: { environment: staging }`    |
| `clusters/development`  | `matchLabels: { environment: development }`|

This is the simplest approach and keeps the Git repo as the single source of truth for
which teams run where.

## Onboarding a New Team

1. Copy an existing team directory (e.g., `teams/team-alpha`) to `teams/team-new`.
2. Update `namespace-patch.yaml` with the new team's namespace name and labels.
3. Update `kustomization.yaml` to patch the RoleBinding with the new team's group name.
4. Replace `app-deployment.yaml` with the team's actual workload.
5. Add `../../teams/team-new` to the appropriate `clusters/*/kustomization.yaml` files.
