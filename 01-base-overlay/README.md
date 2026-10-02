# Strategy 1: Base/Overlay Pattern

## Overview

The base/overlay pattern is the foundational Kustomize pattern. A shared **base** defines the
common resources, and per-environment **overlays** patch, extend, or restrict what gets deployed.

This example deploys the OpenShift Compliance Operator. The base installs the operator (Namespace,
OperatorGroup, Subscription), and each overlay decides what scanning configuration to apply.

## When to Use

- You have a single workload that must be deployed across multiple environments.
- Environment differences are small and well-understood (channel, replica count, schedules).
- The set of environments is stable (you rarely add new ones).

## Pros

- Simple and well-documented; most Kustomize users already know this pattern.
- Clear separation between shared config and environment-specific overrides.
- Easy to diff between environments (`kustomize build overlays/production` vs `overlays/staging`).

## Cons

- Can lead to duplication if overlays start diverging significantly.
- Does not compose well when environments differ along multiple independent axes (use Components for that).
- Adding a new cross-cutting concern (e.g., monitoring) means touching every overlay.

## Directory Layout

```
01-base-overlay/
├── base/                          # Shared resources
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── subscription.yaml
│   └── operatorgroup.yaml
└── overlays/
    ├── production/                 # Full scanning, stable channel
    │   ├── kustomization.yaml
    │   └── scan-setting.yaml
    ├── staging/                    # Weekly scanning
    │   ├── kustomization.yaml
    │   └── scan-setting.yaml
    └── development/                # Operator only, no scanning
        └── kustomization.yaml
```

## Usage

```bash
# Build and review the production manifests
kustomize build overlays/production

# Apply to a cluster
kustomize build overlays/production | oc apply -f -
```

## Mapping to RHACM

Each overlay maps to a separate RHACM **Application** or **ApplicationSet** entry:

| Overlay       | PlacementRule / Placement                        |
|---------------|--------------------------------------------------|
| `production`  | `matchLabels: { environment: production }`       |
| `staging`     | `matchLabels: { environment: staging }`          |
| `development` | `matchLabels: { environment: development }`      |

In an RHACM Channel + Subscription model, create one Subscription per overlay pointing at the
overlay's path in the Git repository. With ApplicationSets (Argo CD), use the Git generator's
`directories` field to discover overlays automatically.
