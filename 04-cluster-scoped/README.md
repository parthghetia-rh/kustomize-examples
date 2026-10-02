# 04 - Cluster-Scoped Resources with Kustomize

## Why Cluster-Scoped Configs Need Special Handling

Cluster-scoped resources differ from namespaced resources in several important ways:

- **No namespace field**: Resources like ClusterRoles, ClusterRoleBindings, and
  SecurityContextConstraints do not belong to any namespace. Applying them affects
  the entire cluster, so mistakes propagate everywhere at once.
- **ClusterRoleBindings affect all namespaces**: A ClusterRoleBinding grants
  permissions across every namespace. Accidentally broadening a binding can give
  users access to secrets or workloads they should never see.
- **SCCs affect pod security cluster-wide**: On OpenShift, SecurityContextConstraints
  determine what any pod on the cluster is allowed to do (privileged containers,
  host networking, volume types). A misconfigured SCC can either lock out
  legitimate workloads or open security holes across all projects.
- **OAuth configuration is a singleton**: The `OAuth` CR named `cluster` is the
  single source of truth for authentication. An invalid change can lock every
  user out of the cluster.

## Structure of This Example

```
base/          ClusterRole and ClusterRoleBinding for read-only platform access
components/    Optional cluster-scoped addons composed via Kustomize Components
  scc-restricted/    Hardened SecurityContextConstraints
  oauth-ldap/        Corporate LDAP identity provider
  audit-policy/      API server audit policy ConfigMap
overlays/
  production/   All components enabled
  staging/      SCC and OAuth only (no audit policy)
  lab/          Base RBAC only, no extra components
```

## Enforcing with RHACM (Red Hat Advanced Cluster Management)

RHACM Governance policies can enforce that these resources exist and match the
desired state on every managed cluster:

- **Policy CRs** wrap one or more `object-templates` with a `complianceType` of
  `musthave` (resource must exist with at least these fields) or
  `mustonlyhave` (resource must match exactly, extra fields are removed).
- **PolicySets** group related policies. For example, a `cluster-hardening` set
  could bundle the SCC policy, audit-policy enforcement, and RBAC checks into
  a single unit that is placed on all production clusters.
- **Placement rules** control which clusters receive which policies. Production
  clusters get the full set; lab clusters may only get the base RBAC policy.

### Practical workflow

1. Store these Kustomize manifests in Git.
2. Use an RHACM `Policy` with `musthave` templates pointing at each resource.
3. Group the policies into a `PolicySet` named `platform-baseline`.
4. Bind the `PolicySet` to cluster selectors (e.g., `env=production`).
5. RHACM reports compliance per cluster; set `remediationAction: enforce` to
   auto-remediate drift.

This pattern gives you declarative, auditable, per-environment control over
resources that would otherwise require manual `oc apply` on every cluster.
