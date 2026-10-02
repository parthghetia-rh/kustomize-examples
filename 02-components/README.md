# Strategy 2: Component-Based Pattern

## Overview

Kustomize **Components** (`kind: Component`) are reusable, composable units of configuration.
Unlike overlays (which are full Kustomizations that reference a base), components are mixed into
an overlay to add a specific capability. This lets you assemble environments from independent
building blocks rather than duplicating configuration across overlays.

This example deploys a simple nginx application. Each environment selects which components to
include: network policies, resource limits, monitoring, and pod disruption budgets.

## When to Use

- Environment differences are **multi-dimensional** (security, monitoring, HA are independent axes).
- You want to avoid the "overlay explosion" problem where every combination needs its own overlay.
- Multiple teams or policies contribute configuration independently.

## Why Components Over Overlays

| Concern           | Base/Overlay                          | Components                                      |
|-------------------|---------------------------------------|--------------------------------------------------|
| Composability     | Each overlay is a flat fork of base   | Components compose independently                 |
| Adding a feature  | Touch every overlay                   | Create one component, reference where needed     |
| Combinatorics     | N overlays for N environments         | M components, combined as needed per environment |
| Reusability       | Overlays are specific to one base     | Components can apply to any compatible base      |

## Directory Layout

```
02-components/
├── base/                              # Core application
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── deployment.yaml
├── components/                        # Independent, composable features
│   ├── network-policies/
│   │   ├── kustomization.yaml         # kind: Component
│   │   └── deny-all.yaml
│   ├── resource-limits/
│   │   ├── kustomization.yaml         # kind: Component (adds LimitRange + patches Deployment)
│   │   └── limitrange.yaml
│   ├── monitoring/
│   │   ├── kustomization.yaml         # kind: Component
│   │   └── servicemonitor.yaml
│   └── pod-disruption-budget/
│       ├── kustomization.yaml         # kind: Component
│       └── pdb.yaml
└── overlays/
    ├── production/                     # All components
    ├── staging/                        # Network policies + resource limits
    └── development/                    # Resource limits only
```

## Usage

```bash
# Build production (all components included)
kustomize build overlays/production

# Build development (only resource-limits component)
kustomize build overlays/development
```

## How Components Enable Composability

The `resource-limits` component both adds a LimitRange resource AND patches the existing
Deployment to include explicit resource requests and limits. This demonstrates how a single
component can contribute new resources and modify existing ones -- something that would
require duplication in a pure overlay model.

To add a new cross-cutting concern (e.g., a SecurityContextConstraint), create a new
component directory and reference it in the overlays that need it. No existing files change.
