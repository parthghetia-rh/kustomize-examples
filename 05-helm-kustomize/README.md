# 05 - Helm + Kustomize Hybrid

## What This Pattern Does

Kustomize has a built-in `helmCharts` generator that renders a Helm chart at
build time using `helm template`, then applies standard Kustomize
patches and transformations on top of the rendered output. This gives you
Helm's powerful templating for the initial resource generation while keeping
Kustomize's overlay model for per-environment customization.

Build with:

```bash
kustomize build --enable-helm overlays/production/
```

The `--enable-helm` flag is required. Without it, Kustomize ignores the
`helmCharts` field entirely.

## Directory Layout

- **base/** - Declares the Helm chart source, version, and default values.
- **overlays/production/** - Adds an HPA resource via patch. The
  `values-override.yaml` documents prod-specific Helm values for reference.
- **overlays/staging/** - Minimal overlay referencing base with no extra patches.
- **overlays/development/** - Same structure; dev-specific values documented
  in its own `values-override.yaml`.

## When to Use Helm + Kustomize

Use this hybrid approach when:

- You depend on a complex upstream Helm chart (ingress-nginx, cert-manager,
  Prometheus) and don't want to rewrite its templates as plain manifests.
- You need Helm's template logic for the heavy lifting but want Kustomize's
  overlay model to manage environment-specific patches and additional resources.
- You want to add resources (like an HPA or NetworkPolicy) that the chart
  doesn't provide without forking the chart.

Use pure Helm instead when:

- All you need is values overrides per environment with no structural patches.
- Your team already has a Helmfile or ArgoCD ApplicationSet workflow.

Use pure Kustomize instead when:

- The manifests are simple enough that Helm templating adds no value.
- You want to avoid the Helm dependency entirely.

## RHACM Integration

Red Hat Advanced Cluster Management (RHACM) Subscriptions can deploy
Kustomize overlays that wrap Helm charts. Set the Subscription's
`apps.open-cluster-management.io/kustomize-path` annotation to point at the
overlay directory (e.g., `overlays/production`). RHACM runs
`kustomize build --enable-helm` internally, so the helmCharts generator
works without additional configuration. This lets you manage multi-cluster
deployments using the same overlay structure shown here.
