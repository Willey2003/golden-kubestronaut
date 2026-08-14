# Module 6 — Mesh & GitOps (ICA · CAPA · CGOA)

Module 6 connects services and then ships them declaratively. It targets
**ICA** (Istio service mesh), **CAPA** (Argo CD, Workflows, Rollouts,
Events), and **CGOA** (GitOps principles and practice). These three pair
naturally: a mesh controls east–west traffic, Argo CD makes the cluster the
reconciled output of a Git repository, and CGOA supplies the theory that
turns "it works on my laptop" into an auditable pipeline. The ICA and CAPA
banks are mixed; CGOA is knowledge.

## Learning objectives

1. Explain the service mesh: data plane vs. control plane, sidecar proxies,
   and what a mesh adds over plain Services.
2. Route, split, and fail over traffic with Istio: Gateways,
   VirtualServices, DestinationRules.
3. Secure mesh traffic: mTLS, PeerAuthentication, AuthorizationPolicy.
4. Operate Argo CD: apps, sync, health, `app of apps`, and RBAC.
5. Author Argo Workflows, CronWorkflows, and Events; drive rollouts with
   Argo Rollouts.
6. Apply GitOps principles: declarative state, pull-based delivery, drift
   detection, and maturity.

## Key concepts

- **Data plane vs. control plane**: Envoy sidecar proxies carry all
  pod-to-pod traffic (data plane); `istiod` distributes config and
  certificates (control plane). Traffic to a sidecared Pod is TLS-encrypted,
  routed, and metered even if the app knows nothing about it.
- **Mesh value**: mTLS between every workload, canary/split traffic,
  per-request retries and timeouts, and uniform telemetry (traces + metrics)
  — the ICA answers for "why a mesh".
- **Traffic management**: an `IstioGateway` exposes ingress; a
  `VirtualService` says which Service receives what (by host, path, or
  header) and routes to subsets; a `DestinationRule` defines subsets
  (`version: v1`) and traffic policies (connection pool, TLS). Weighted
  routing (`weight: 90`) is the canary primitive.
- **Security**: `PeerAuthentication` sets mTLS mode per mesh/namespace;
  `AuthorizationPolicy` is allow/deny on principals, methods, and paths —
  this is where ICA's security domain lives. Mutual TLS identity comes from
  per-pod certificates issued by istiod.
- **Observability in the mesh**: Envoy emits HTTP metrics and access logs
  for every request; Kiali renders topology; traces are propagated across
  sidecars, which is why meshes and OpenTelemetry (Module 5) pair well.
- **GitOps (CGOA)**: the desired state of a system is versioned in Git; a
  controller watches Git and reconciles the cluster to it. Everything is
  a pull — the cluster pulls the desired state, nothing pushes to it.
  Benefits: auditability, rollback is a revert, and no imperative drift.
- **Declarative vs. imperative**: `kubectl apply -f` from a repo is
  declarative; ad-hoc `kubectl scale` is imperative and the drift CGOA
  exists to prevent. The four GitOps principles: declarative, desired state
  in Git, approved change, and automated remediation (drift detection).
- **Argo CD core**: an application is `kind: Application` pointing at a Git
  source (`repoURL`, `path`, `targetRevision`) with a destination cluster.
  `SyncPolicy: automated` + `selfHeal` keeps the cluster reconciled; health
  and sync status (`Healthy`, `Synced`) drive the UI and alerts.
- **App of apps**: one meta-Application whose `source` is a directory of
  other `Application` manifests — how you ship a whole platform from one
  repo.
- **Argo Workflows**: DAG or steps of containers on the cluster with
  artifacts and parameters; `CronWorkflow` schedules them; Argo Events
  (webhook, S3) trigger them. A workflow is a first-class k8s object
  (`kind: Workflow`).
- **Argo Rollouts**: `Rollout` replaces `Deployment` for canary/blue-green;
  `AnalysisTemplate`/`AnalysisRun` gate promotion on metrics — the
  GitOps-native answer to "deploy when the canary is healthy".
- **Tooling contrast (CGOA)**: Argo CD (app-level, pull model, rich UI) vs.
  Flux (gitops toolkit, kustomize/helm-native, controllers per concern).
  The CGOA exam asks which tool fits which constraint — not which is better.

## Hands-on exercises

Exercise 1 needs a mesh-enabled cluster (install Istio's `istioctl
install`); 3–4 need an Argo CD instance (`kubectl create namespace argocd &&
kubectl apply -k https://github.com/argoproj/argo-cd/manifests/...`). CGOA
exercises need only git.

### Exercise 1 — Deploy a sidecar mesh (needs istioctl)

- Task: install Istio with `istioctl`, label a namespace `istio-injection=enabled`,
  deploy an app, and confirm the sidecar.
- Expected outcome: `kubectl get pods -n mesh -l app=web -o jsonpath='{.items[0].spec.containers[*].name}'`
  shows both the app container and `istio-proxy`.

```sh
istioctl install --set profile=demo -y
kubectl label ns mesh istio-injection=enabled
kubectl create deployment web --image=nginx:stable-alpine -n mesh
kubectl get pods -n mesh -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

- Verification:

```sh
kubectl get pods -n mesh
istioctl proxy-status
```

### Exercise 2 — Split traffic with a VirtualService

- Task: define two subsets of a Service (`v1`, `v2`) and route 90% / 10% by
  weight; verify with `istioctl` proxy config or request counting.
- Expected outcome: `istioctl analyze` is clean, and requests land mostly on
  `v1`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: {name: web, namespace: mesh}
spec:
  hosts: [web]
  http:
    - route:
        - destination: {host: web, subset: v1}
          weight: 90
        - destination: {host: web, subset: v2}
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: {name: web, namespace: mesh}
spec:
  host: web
  subsets:
    - {name: v1, labels: {version: v1}}
    - {name: v2, labels: {version: v2}}
```

- Verification:

```sh
kubectl apply -n mesh -f routing.yaml
istioctl analyze -n mesh
```

### Exercise 3 — mTLS + AuthorizationPolicy

- Task: set mesh-wide `PeerAuthentication` to `STRICT` and allow only the
  frontend principal to call the backend.
- Expected outcome: a non-permitted client gets HTTP 403 from the sidecar;
  the permitted one succeeds.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: backend-only, namespace: mesh}
spec:
  selector: {matchLabels: {app: backend}}
  action: ALLOW
  rules:
    - from:
        - source: {principals: ["cluster.local/ns/mesh/sa/frontend"]}
```

- Verification:

```sh
kubectl exec -n mesh deploy/frontend -- curl -s http://backend/
kubectl exec -n mesh deploy/other -- curl -s http://backend/   # 403
```

### Exercise 4 — First Argo CD application

- Task: register a git repo and deploy an `Application` from it with
  automated sync + self-heal; introduce drift and watch it be corrected.
- Expected outcome: the app is `Synced`/`Healthy`; scaling to 5 manually is
  reverted to the repo value within a minute.

```sh
argocd app create web --repo <git-url> --path manifests --dest-server https://kubernetes.default.svc \
  --sync-policy automated --auto-prune --self-heal
kubectl scale deployment web --replicas=5   # drift
```

- Verification:

```sh
argocd app get web
kubectl get deployment web -o jsonpath='{.spec.replicas}{"\n"}'   # back to repo value
```

### Exercise 5 — GitOps reasoning (no cluster needed)

- Task: sketch the drift-detection loop for one cluster and one repo.
- Expected outcome: you can name each piece — git as source of truth, the
  controller polling/reconciling, `kubectl apply` only as bootstrap, and
  health checks as the release gate — and explain why a `kubectl exec`-style
  change is the enemy of the loop.

## Test yourself

- **Bank**: `banks/ica` — **Training**, `focus_domain = traffic-management`,
  then `istio-security` as **Mastery**.
- **Bank**: `banks/capa` — **Training**, `focus_domain = argocd-core-concepts`,
  then `argo-workflows` as **Mastery**.
- **Bank**: `banks/cgoa` — **Training**, `focus_domain = gitops-principles`,
  then **Mastery** on `gitops-tooling`.
- Weak `traffic-management` → Exercise 2; weak `argocd-core-concepts` →
  Exercise 4.

## Self-check quiz

1. **Data plane vs. control plane in Istio?** — data plane: the Envoy
   sidecars carrying real traffic; control plane: `istiod` issuing config
   and certificates to those sidecars.
2. **What does a `VirtualService` need from a `DestinationRule` to route by
   weight?** — named subsets; the rule defines `v1`/`v2` by labels, the
   VirtualService assigns them weights.
3. **GitOps' defining loop?** — desired state in Git, controller pulls and
   reconciles, drift is detected and corrected automatically — no push, no
   imperative changes.
4. **Argo CD `selfHeal` does what to an out-of-band `kubectl scale`?** —
   reverts it, because the cluster's actual state must match the declared
   state in Git.
5. **CronWorkflow vs. Workflow?** — a `CronWorkflow` schedules `Workflow`
   runs on a cron expression; the `Workflow` itself is one execution.

## See also

- ICA, CAPA, and CGOA pages on linuxfoundation.org / cncf.io and the Istio
  and Argo docs (referenced by name; objectives change between releases).
- [Module 7 — Networking & cost (CBA · CNPA · CNPE · KCA)](07-networking-and-cost.md) —
  next: the datapath under the mesh and the money under the cluster.
