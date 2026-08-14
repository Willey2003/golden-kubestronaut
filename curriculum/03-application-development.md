# Module 3 — Application development (CKAD)

Module 3 looks at Kubernetes from the application developer's side of the
API. It targets **CKAD**: designing and building applications, deploying
them safely, wiring configuration and security, and exposing them to
traffic. Where Module 2 fixed the cluster, this module fixes the workload —
and because CKAD is a live-cluster exam, every exercise here is a command
you must type fast. The CKAD bank is `hands-on`; the grader runs the same
`kubectl` commands you practise here.

## Learning objectives

1. Design multi-container Pods and choose the right primitive: Pod,
   Deployment, StatefulSet, Job, CronJob.
2. Build, tag, and reference container images; handle registry auth and
   pull policies.
3. Deploy and release safely: rollouts, strategies, rollbacks, and
   self-healing.
4. Configure applications with ConfigMaps, Secrets, env vars, and
   SecurityContext; grant identity with ServiceAccounts.
5. Keep applications healthy and observable: probes, resource limits,
   logs, and ephemeral debugging.
6. Expose applications: Services, Ingress, NetworkPolicy, and cluster DNS.

## Key concepts

- **Multi-container Pods**: containers in one Pod share a network namespace
  and optionally volumes — the pattern for sidecars (log shippers, proxies,
  watchers). `initContainers` run to completion before the main container
  starts, for setup that must happen first.
- **Workload primitives**: Deployment (stateless, rolling), StatefulSet
  (stable identity + ordinal DNS, for stateful apps), Job (run-to-
  completion), CronJob (schedule). Pick by lifecycle, not habit.
- **Images**: `image: registry/repo:tag`. `imagePullPolicy` — `IfNotPresent`
  for most workloads, `Always` for mutable tags in dev; images without a tag
  default to `:latest` with `Always`. `kubectl set image deployment/web
  nginx=nginx:stable-alpine` is the fastest release change.
- **Rollouts**: Deployments roll via ReplicaSets. `maxSurge` and
  `maxUnavailable` shape the rolling update; `kubectl rollout status` and
  `kubectl rollout undo deployment/web` manage it; `kubectl rollout history`
  shows revisions. `kubectl rollout restart` forces a new revision without
  changing the image.
- **ConfigMaps and Secrets**: both are `data` (Secret values must be
  base64). Inject as env vars or volume mounts; mounting a ConfigMap as a
  volume updates live as the ConfigMap changes; env vars do not update until
  restart. Never put a Secret in an image or a Deployment annotation.
- **SecurityContext**: per-Pod or per-container `runAsUser`,
  `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation`,
  `capabilities`. The minimum viable hardened container is non-root with a
  read-only root filesystem.
- **ServiceAccount**: a Pod's identity — `spec.serviceAccountName`. Used for
  API access and in-cluster auth; image-pull secrets attach to it.
- **Probes**: `livenessProbe` (restart if dead), `readinessProbe` (send
  traffic only when ready), `startupProbe` (protect slow starters from the
  liveness probe). All are `httpGet`/`exec`/`tcpSocket` on a port + path.
- **Resources**: `requests` (scheduling floor), `limits` (cap; above CPU
  limit throttles, above memory limit kills). A Pod without requests in a
  quota'd namespace will not schedule.
- **Observability and maintenance**: `kubectl logs`, `kubectl exec -it`,
  `kubectl cp`, and ephemeral containers (`kubectl debug`) for inspecting a
  running Pod without changing it.
- **Services**: ClusterIP (stable VIP), NodePort (node:port), LoadBalancer
  (cloud LB), Headless (`clusterIP: None`, for StatefulSet ordinals).
  Selectors must match Pod labels exactly or you get empty Endpoints.
- **Ingress**: `Ingress` objects route HTTP to Services by host and path;
  an IngressClass + controller (nginx, Cilium) implements it. Path and host
  rules are CKAD favourites.
- **NetworkPolicy**: namespace-scoped, label-selected rules on ingress and
  egress. Default-deny first, then allow — the CKAD test of whether you
  understand selectors.
- **DNS**: `service.namespace.svc.cluster.local`; same-namespace Services
  resolve by short name (`web-svc`). A Service with no selector but a custom
  `endpoints`/`ExternalName` can point outside the cluster.

## Hands-on exercises

All exercises need a live cluster.

### Exercise 1 — Design a multi-container Pod with an init container

- Task: write a Pod with an `initContainers` step that writes a config file,
  and a main container that reads it; verify order of execution.
- Expected outcome: `kubectl get pod -o jsonpath='{.status.initContainerStatuses}'`
  shows the init step succeeded, and the main container logs the file.

```sh
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: web
spec:
  initContainers:
    - name: setup
      image: busybox
      command: ["sh", "-c", "echo backend=db > /cfg/settings"]
      volumeMounts: [{name: cfg, mountPath: /cfg}]
  containers:
    - name: main
      image: busybox
      command: ["sh", "-c", "cat /cfg/settings && sleep 3600"]
      volumeMounts: [{name: cfg, mountPath: /cfg}]
  volumes:
    - name: cfg
      emptyDir: {}
EOF
kubectl get pod app -o jsonpath='{.status.initContainerStatuses[0].state}{"\n"}'
kubectl logs app
```

- Verification:

```sh
kubectl get pod app -o jsonpath='{.spec.initContainers[*].name}{"\n"}'
```

### Exercise 2 — Config from a ConfigMap and a Secret

- Task: create a ConfigMap and a Secret, inject both as env vars, and
  confirm the values in a Pod.
- Expected outcome: `kubectl exec app -- env | grep -E 'GREETING|PASSWORD'`
  shows `GREETING=hello` and the secret value.

```sh
kubectl create configmap app-config --from-literal=GREETING=hello
kubectl create secret generic app-secret --from-literal=PASSWORD=s3cret
kubectl create deployment app --image=nginx:stable-alpine \
  --dry-run=client -o yaml > app.yaml
# edit app.yaml to add envFrom for the ConfigMap and Secret, then:
kubectl apply -f app.yaml
```

- Verification:

```sh
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o jsonpath='{.data.PASSWORD}' | base64 -d
```

### Exercise 3 — Probes and resource limits

- Task: add a liveness and readiness probe plus CPU/memory requests and
  limits to a Deployment, then watch readiness gate traffic.
- Expected outcome: `kubectl describe pod -l app=web` shows both probes and
  the resource block; a Pod whose probe fails leaves Endpoints.

```sh
kubectl set resources deployment/web --requests=cpu=100m,memory=128Mi \
  --limits=cpu=250m,memory=256Mi
kubectl patch deployment web --type=merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/","port":80}},"livenessProbe":{"httpGet":{"path":"/","port":80}}}}]}}}}'
```

- Verification:

```sh
kubectl describe deployment web | grep -A4 Probes
kubectl get endpoints web-svc
```

### Exercise 4 — Rolling release, scale, rollback

- Task: scale a Deployment, push a bad image, watch it fail, then `undo`.
- Expected outcome: `kubectl rollout status` reports success for the good
  release and failure for the bad one; `undo` restores the working image and
  `rollout history` shows both revisions.

```sh
kubectl scale deployment web --replicas=4
kubectl set image deployment/web nginx=nginx:1.14.2-broken-tag
kubectl rollout status deployment/web   # wait for the failed rollout
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl rollout history deployment/web
```

- Verification:

```sh
kubectl get pods -l app=web
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### Exercise 5 — Expose with Service + Ingress + NetworkPolicy

- Task: expose the app via a ClusterIP Service, add an Ingress rule, then
  write a NetworkPolicy that default-denies and allows only the Ingress
  controller.
- Expected outcome: `kubectl get ingress` shows the rule; a curl from a Pod
  in the `default` namespace to the Service fails after the policy, while
  the policy-permitted client succeeds.

```sh
kubectl expose deployment web --name=web-svc --port=80 --target-port=80
kubectl create ingress web --rule="web.example.com/=web-svc:80"
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

- Verification:

```sh
kubectl get ingress web
kubectl exec <client-pod> -- wget -qO- http://web-svc/ | head -1
```

## Test yourself

- **Bank**: `banks/ckad` — **Training**, `focus_domain =
  application-design-build`, then `configuration-security`. Re-sit as
  **Mastery**, `focus_domain = application-deployment` and
  `services-networking`.
- CKAD's threshold is 0.67; the developer bank rewards speed, so time your
  Mastery passes. Weak `configuration-security` → redo Exercises 2–3; weak
  `services-networking` → Exercise 5.

## Self-check quiz

1. **Init containers vs. a sidecar — when does each fit?** — init: one-shot
   setup that must finish before start; sidecar: a long-running helper that
   lives with the app container.
2. **Why does a ConfigMap env var not update when the ConfigMap changes,
   but a mounted file does?** — env vars are copied into the container at
   start; volumes are read from the live ConfigMap, so they refresh.
3. **What is the default rollout strategy, and what do `maxSurge` and
   `maxUnavailable` control?** — RollingUpdate; how many Pods may be created
   above and removed below desired during the update.
4. **A readiness probe fails — what happens, and what happens if a liveness
   probe fails?** — readiness: traffic is removed from Endpoints but the Pod
   stays; liveness: the container is restarted.
5. **Your Service has no Endpoints. The two most likely causes?** —
   the selector labels don't match the Pod labels, or the Pods are not
   `Ready` (readiness probe failing).

## See also

- CKAD page on linuxfoundation.org (referenced by name; objectives change
  between releases).
- [Module 4 — Security (CKS)](04-security.md) — next: hardening the cluster
  and workloads you now know how to run.
