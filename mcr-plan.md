# Plan: extract config/workload-split reconcilers into `pkg/`

Goal: let a downstream multicluster-runtime operator reuse the existing per-type reconcile
logic with two clients — reading `operator.kcp.io` resources from the local manager's control
plane and writing workloads into a remote cluster — without rewriting the in-tree controllers.

The in-tree controllers keep working unchanged by passing the same client for both roles.

## 1. The move (mechanical, one commit)

| from                       | to                  |
| -------------------------- | ------------------- |
| `internal/resources/**`    | `pkg/resources/**`  |
| `internal/reconciling/**`  | `pkg/reconciling/**`|
| `internal/kubernetes`      | `pkg/kubernetes`    |
| `internal/controller/util` | `pkg/util`          |

`internal/` cannot be imported from another Go module, so anything downstream needs has to
move. The dependency graph is already clean: these packages depend only on each other, the
sdk, and third-party code. No pull on `internal/config`, `internal/client` or
`internal/metrics`, which all stay internal along with `internal/controller/*`.

~11k LOC, ~3.7k of it tests. Pure `git mv` plus import rewrite.

Touch-ups: `hack/update-codegen.sh:39` output path, `hack/reconciling.yaml` package name,
`make imports`. Add `pkg/doc.go` stating the stability level — this becomes public surface the
moment it lands, and `sdk` is currently the only thing in the repo with a compat story.

Package names stay identical so the diff is rename-only. `pkg/resources` reads oddly as public
API, but renaming to `pkg/components` would bury the real changes in churn.

## 2. Per-type API

Each of `pkg/resources/{rootshard,shard,frontproxy,virtualworkspace,cacheserver}` gets a
`reconciler.go` in the shape `frontproxy` already has on the `mode-split` branch:

```go
func NewX(x *operatorv1alpha1.X, deps...) *reconciler
func ResolveX(ctx, config client.Client, x *operatorv1alpha1.X) (*reconciler, []metav1.Condition, error)

func (r *reconciler) ReconcileConfig(ctx, config client.Client, ns string, mods ...ObjectModifier) error
func (r *reconciler) ReconcileWorkload(ctx, config, workload client.Client, ns string, owner Owner) (requeue bool, err error)
func (r *reconciler) DeleteWorkload(ctx, workload client.Client, ns string, owner Owner) error
```

`Resolve*` returns conditions as well as the reconciler because the existing lookups already
produce them — `util.FetchRootShard` (`internal/controller/util/util.go:94`) returns
`(metav1.Condition, *RootShard)`, and the RootShard→VirtualWorkspace lookup sets `vwConfigValid`
(`internal/controller/rootshard/controller.go:242-251`). Keeping that contract means downstream
inherits the same failure reporting.

In-tree controllers become: fetch CR → `Resolve*` → `ReconcileConfig(r.Client, …)` →
`ReconcileWorkload(r.Client, r.Client, …)` → existing `reconcileStatus`. Same client twice, no
behaviour change.

`ReconcileWorkload` takes both clients on purpose: the mount set is only known after the
Deployment is rendered, so mirroring cannot be a step the caller runs beforehand.

## 3. `Owner`

```go
type Owner struct {
    Ref    *metav1.OwnerReference // nil when the CR lives in another cluster
    Labels map[string]string      // always set
}
```

`Ref != nil` → today's `OwnerRefWrapper`. `Ref == nil` → labels only.

This is not cosmetic: a dangling cross-cluster ownerRef makes the target cluster's garbage
collector treat the owner as deleted and **remove the dependents**. The label sets already exist
and are already applied (`resources.GetRootShardResourceLabels`,
`internal/resources/resources.go:131`).

## 4. Mount mirroring

`modifier.RelatedRevisionsLabels` (`internal/reconciling/modifier/revision_labels.go:38`) already
walks pod-spec volumes and containers and `Get`s every referenced Secret/ConfigMap, purely to hash
a ResourceVersion into a pod template label. That walk *is* the mirror set — exact, complete, and
derived from the Deployment being rendered. It becomes stateful:

```go
m := modifier.NewMounts(config, workload, owner)
… ReconcileDeployments(ctx, factories, ns, workload, ownerMod, m.Modifier())
err := m.Prune(ctx)
```

Inside the modifier: `Get` from `config`; if `config == workload`, no write at all (identical to
today); otherwise reconcile into `workload` and record the key. A missing source object keeps
producing `ErrMountNotFound` and the existing requeue path.

Only pass it to `ReconcileDeployments` — it panics on other kinds
(`internal/reconciling/modifier/revision_labels.go:69`).

One `Mounts` instance per `ReconcileWorkload` call, not per Deployment, so RootShard's own
Deployment and its embedded proxy Deployment (`frontproxy.NewRootShardProxy(rootShard)`,
`internal/controller/rootshard/controller.go:271`) share one prune pass.

This replaces the separate syncer, and it closes real gaps. `test/syncer/main.go` on `mode-split`
copies only objects labelled `operator.kcp.io/<component>`, so it silently drops every mounted
Secret without one:

| mount                            | defined at                              |
| -------------------------------- | --------------------------------------- |
| `spec.etcd.tlsConfig.secretRef`  | `internal/resources/utils/shard.go:170` |
| audit webhook config             | `internal/resources/utils/audit.go:130` |
| authorization webhook config     | `internal/resources/utils/authorization.go:58` |
| OIDC CA file                     | `internal/resources/utils/authentication.go:59` |
| token auth file                  | `internal/resources/utils/authentication.go:141` |
| `<cs>-kubeconfig`                | `internal/resources/cacheserver/kubeconfigs.go:36` |

It also never deletes anything. Mount-driven mirroring covers all of the above and prunes.

## 5. Shared mounts need refcounting

`<rs>-ca`, `<rs>-server-ca`, `<rs>-client-ca` and `<cs>-kubeconfig` are mounted by RootShard,
Shard, FrontProxy **and** VirtualWorkspace Deployments — four separate CRs mirroring into the
same target namespace. A naive "delete everything carrying my owner label" prune would rip a
Secret out from under the others.

So each mirrored object carries one marker label per owner:

```
mirror.operator.kcp.io/<kind>-<hash(name)>: ""
```

hashed the way `getLabelName` already does
(`internal/reconciling/modifier/revision_labels.go:106`) to stay under the 63 character limit.

`Prune` and `DeleteWorkload` both drop the caller's marker and delete the object only when no
`mirror.operator.kcp.io/` marker remains. One mechanism serves both, so it costs little beyond
getting it right once — but it needs dedicated unit tests for the two-owner case.

## 6. Content-hash revision labels

Switch `getRelatedRevisionLabels` from hashing `ResourceVersion` to hashing the object's data.
ResourceVersion is meaningless across clusters, and hashing content gives one code path for both
the local and the mirrored case.

Consequence: pod template labels change once on upgrade, so every kcp, front-proxy and
virtual-workspace pod rolls once. Needs a release note.

## 7. Explicitly out of scope

- **cert-manager stays config-side.** Trust is centralized: RootShard owns one root Issuer plus
  four intermediate CA Certificate/Issuer pairs
  (`internal/controller/rootshard/controller.go:177-205`) and every other component's leaf certs
  `issuerRef` them. `Issuer` is namespaced, so splitting them across clusters means each target
  mints divergent intermediates and cross-component mTLS breaks.
- **Kubeconfig, KubeconfigRBAC and Bundle controllers** are config-side only, untouched.
- **Status.** `ReconcileWorkload` returns `(requeue, err)`; `util.GetDeploymentAvailableCondition`
  already takes a client, so downstream aims it at the workload client and composes its own
  status. No API change here, and multi-cluster status aggregation stays downstream's problem.
- **kcp reachability.** `internal/client/clients.go:36,47,58` hardcode
  `https://<svc>.<ns>.svc.cluster.local:6443`. Once kcp runs in a different cluster than the
  operator, the Shard cleanup finalizer (`internal/controller/shard/controller.go:342`) and
  KubeconfigRBAC (`internal/controller/kubeconfig-rbac/controller.go:135,210`) cannot reach it.
  Not blocking this refactor; follow-up to parameterise the base URL.

## 8. Commit sequence

1. `Move reusable reconcile packages to pkg/` — rename, import rewrite, codegen paths. No logic
   change.
2. `Hash mounted object contents for revision labels` — self-contained, release-noted.
3. `Add Owner to describe workload ownership` — type plus modifier selection; all call sites pass
   `Ref`, no behaviour change.
4. `Mirror mounted Secrets and ConfigMaps to the workload client` — `Mounts` modifier, marker
   labels, `Prune`. No-op when the clients are equal.
5. `Split the RootShard reconciler into config and workload halves` — `ResolveRootShard`,
   `ReconcileConfig`, `ReconcileWorkload`, `DeleteWorkload`; controller passes the same client
   twice.
6. Repeat step 5 per type. FrontProxy is already done on `mode-split` (`87a7b03`); the
   RootShard/Shard/CacheServer/VirtualWorkspace splits exist there too (`0030336`, `ffd63bc`,
   `6322a9c`, `a8957b7`) and can be reworked with the `Compiled*` half dropped.
7. Two-client harness proving the split before downstream depends on it. Reuse
   `test/utils/topology.go` from `mode-split`; delete `test/syncer`.

Steps 1–4 land independently with no functional change, so the risk concentrates in 5–6.

## 9. Tests

- unit, `Mounts` modifier: local no-op; cross-cluster mirror; missing source → `ErrMountNotFound`;
  prune with one owner; prune with two owners sharing a Secret; `DeleteWorkload`.
- unit: content-hash labels stable across two clusters holding identical data.
- e2e: existing suites pass unchanged after steps 1–6 (single-cluster path), plus one two-cluster
  topology exercising a split RootShard.

## 10. Open questions

- Does `pkg/` get a stability disclaimer or a real compat promise?
- Is the step 7 harness in-tree, or is validation done from the downstream repo?

## Appendix: why not an mcr controller in this repo

Considered and rejected: making the workload halves multicluster-runtime controllers here.
`For()` would watch the CRD in every engaged cluster, which is wrong — the CRs only live on the
local manager. Watching locally while tagging requests with a target cluster name requires a
custom `mcsource.TypedSource` fanning out over engaged clusters, which means one CR reconciles
into *every* engaged cluster. Placement policy belongs downstream, not here. The library split
above gives downstream both clients and lets it decide.

For reference, if downstream does build that controller:

- `mcbuilder.WithEngageWithLocalCluster` / `WithEngageWithProviderClusters`
  (`pkg/builder/multicluster_options.go:47-60`) control which clusters a watch engages.
- `Owns()` is unusable across clusters — ownerRefs are cluster-local. Use `Watches()` with a
  label-based mapper against the target instead.
- A fan-out source is ~40 LOC: wrap `mcsource.TypedKind` and have `ForCluster(name, _)` delegate
  to the inner source with the *local* cluster substituted, so the informer is local while
  requests carry the remote cluster name. Register with `Build()` plus
  `ctrl.MultiClusterWatch` (`pkg/controller/controller.go:179`).
- `providers/kubeconfig` in multicluster-runtime turns labelled kubeconfig Secrets into clusters.
- multicluster-runtime v0.24.1 matches the controller-runtime v0.24.1 already in `go.mod`.
