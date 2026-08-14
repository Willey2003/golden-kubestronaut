# Module 7 — Networking & cost (CBA · CNPA · CNPE · KCA)

Module 7 closes the path at both ends of the production conversation: the
**networking** underneath everything, and the **cost** on top of it. It
targets **CBA** (Cilium and eBPF), **CNPA** and **CNPE** (the professional
track: multi-cluster networking, Cluster Mesh, platform engineering), and
**KCA** (cost and FinOps). Everything from Modules 1–6 shows up here as a
trade-off: the mesh you chose, the policies you wrote, and the money the
whole platform spends.

## Learning objectives

1. Explain eBPF and what it lets a CNI do that iptables cannot.
2. Operate Cilium: installation, identity-based policies, Hubble
   observability, and Cluster Mesh for multi-cluster networking.
3. Apply professional-track networking: Gateway API, Ingress, DNS,
   multi-cluster and east–west at scale (CNPA/CNPE).
4. Reason about platform engineering: Service Meshes in production,
   network security boundaries, and reliability trade-offs (CNPE).
5. Account for cloud-native spend: FinOps fundamentals, allocation,
   right-sizing, autoscaling economics, and optimisation levers (KCA).

## Key concepts

- **eBPF**: a sandboxed in-kernel virtual machine. Programs attach to kernel
  hooks (packet arrival, syscalls) and run at line rate without copying to
  userspace. That is why a Cilium datapath out-performs iptables chains and
  sees every connection instead of rules iterating over the chain.
- **CNI role**: the CNI plugin assigns IPs and wires the pod network. Cilium
  is a CNI that also does load-balancing, network policy, and
  observability from the same eBPF datapath — one engine, many jobs.
- **Identity-based policy**: Cilium replaces pod-IP matching with a security
  *identity* derived from labels. `CiliumNetworkPolicy`/`CiliumClusterwideNetworkPolicy`
  select by label and can enforce L7 (HTTP methods and paths, Kafka topics,
  DNS names) — the CBA bank's core skill.
- **Cilium service mesh**: sidecarless load balancing and L7 policy using
  the same eBPF engine, plus mTLS; where Istio (Module 6) is the classic
  sidecar mesh, Cilium is the high-throughput alternative — a favourite
  CNPA/CNPE comparison.
- **Hubble**: the observability half — service maps, flow logs
  (`hubble observe`), and policy verdicts, all from the datapath. Your first
  answer to "is that NetworkPolicy what I think it is".
- **Cluster Mesh**: Cilium's multi-cluster networking — clusters share
  identities and service discovery over a mesh of tunnels or direct routes,
  so a pod in one cluster can reach a service in another with the same
  policy model. This is the CBA "Cluster Mesh" domain and the spine of
  CNPE's multi-cluster questions.
- **Gateway API**: the successor to Ingress — `GatewayClass`, `Gateway`,
  `HTTPRoute` separate routing from infrastructure and are implemented by
  nginx, Istio, and Cilium alike. CNPA expects you to map classic Ingress
  concepts onto it.
- **Ingress and DNS at scale**: Ingress → Gateway, node ports → LB, plus
  external DNS and TLS termination decisions; east–west (pod-to-pod) stays
  the mesh's job, north–south (ingress) is the gateway's.
- **Professional track (CNPA/CNPE)**: CNPA is the breadth exam — containers,
  orchestration, networking, storage, security, observability, and GitOps —
  at working-professional depth; CNPE is the engineering exam — multi-cluster
  topology, platform abstractions, reliability budgets, and the trade-offs
  you defend in an architecture review.
- **Multi-cluster topologies**: hub-and-spoke, federated, and cluster-mesh —
  when services span clusters, you choose where discovery and policy live.
  CNPE asks you to reason about failure domains and blast radius per
  topology.
- **FinOps (KCA)**: the discipline of optimising cloud spend —
  *inform* (visibility), *optimize* (right-size, autoscale, spot/savings),
  *operate* (continuous). It is an accounting of what Kubernetes makes
  cheap and what it hides: idle CPU, orphaned volumes, and over-provisioned
  requests.
- **Allocation and showback**: costs follow labels, not teams —
  `kubecost`, `opencost`, or a custom `kube-billing` report attributes spend
  to namespaces and workloads; showback tells teams their numbers.
- **Right-sizing**: `requests` are a budget — 3 replicas requesting 1 CPU
  reserve 3 CPUs whether or not they run. Metrics-driven sizing
  (`kubectl top`, Prometheus) beats guessing; the KCA questions turn on this
  accounting.
- **Autoscaling economics**: HPA/VPA smooth utilisation but you pay for
  peaks; spot instances and node autoscaling shift the trade-off to
  interruption risk — the KCA optimisation levers are requests, replicas,
  node pool shape, and discount instruments.

## Hands-on exercises

Exercises 1–3 need a cluster with Cilium installed (`cilium install`);
Exercise 5 needs only a shell.

### Exercise 1 — Install Cilium and check the datapath

- Task: `cilium install`, then confirm the CNI, agent, and operators are
  healthy.
- Expected outcome: `cilium status` shows the cluster mesh ready and
  `cilium connectivity test` passes (or at least the agent is `OK`).

```sh
cilium install
cilium status
cilium connectivity test --test 'pod-to-pod'
```

- Verification:

```sh
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status --brief
```

### Exercise 2 — Identity-based L3/L4 policy

- Task: write a `CiliumNetworkPolicy` that allows only pods labelled
  `role=frontend` to reach `app=backend` on TCP 80, then probe both sides.
- Expected outcome: `cilium policy get backend` lists the rule; a
  non-permitted client is dropped.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: {name: backend-ingress, namespace: default}
spec:
  endpointSelector: {matchLabels: {app: backend}}
  ingress:
    - fromEndpoints:
        - matchLabels: {role: frontend}
      toPorts:
        - ports: [{port: "80", protocol: TCP}]
```

- Verification:

```sh
cilium policy get backend-ingress
kubectl exec client-bad -- wget -qO- http://backend-svc/ | head -1   # fails
```

### Exercise 3 — Hubble flow logs

- Task: make some traffic, then inspect the datapath's view of it.
- Expected outcome: `hubble observe` shows the request flows with verdict
  and drop reasons.

```sh
hubble observe --namespace default
hubble observe --verdict DROPPED | head -5
```

- Verification:

```sh
hubble status
```

### Exercise 4 — Gateway API with Cilium (or your IngressClass)

- Task: create a `GatewayClass`, a `Gateway`, and an `HTTPRoute` that sends
  `api.example.com` to the backend Service.
- Expected outcome: the gateway reports `Ready`/`Programmed`, and curl to
  the host returns app HTML.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: {name: cilium}
spec: {controllerName: io.cilium/gateway-controller}
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: gw, namespace: default}
spec:
  gatewayClassName: cilium
  listeners:
    - {name: http, port: 80, protocol: HTTP, hostname: api.example.com}
```

- Verification:

```sh
kubectl get gateway gw
curl -s -H 'Host: api.example.com' http://<gateway-address>/ | head -1
```

### Exercise 5 — Cost allocation drill (no cluster needed)

- Task: given a workload with `requests: cpu=1` and 3 replicas idling at 5%
  CPU, compute the wasted reservation and propose the fix.
- Expected outcome: 3 CPUs reserved vs. ~0.15 used; the fix is right-sizing
  requests (HPA/VPA) and possibly fewer replicas — then reconcile the same
  maths in the KCA bank.

## Test yourself

- **Bank**: `banks/cba` — **Training**, `focus_domain =
  network-policies-security`, then **Mastery** on `hubble-observability` and
  `cluster-mesh`.
- **Bank**: `banks/cnpa` — **Training**, `focus_domain = networking-storage`;
  **Mastery** on `observability-security`. The CNPA bank samples the whole
  professional breadth, so review Modules 1–6 before a full pass.
- **Bank**: `banks/cnpe` — **Training**, `focus_domain = multi-cluster-networking`;
  **Mastery**, `focus_domain = platform-engineering`.
- **Bank**: `banks/kca` — **Training**, `focus_domain = finops-fundamentals`,
  then **Mastery** on `cost-allocation` and `rightsizing-autoscaling`.
- Weak `network-policies-security` → Exercise 2; weak `finops-fundamentals`
  → Exercise 5.

## Self-check quiz

1. **Why does identity beat IP-address matching in policy?** — pod IPs
   change on every restart; a Cilium identity is derived from labels, so
   policy survives churn and enforces intent, not addresses.
2. **What does Hubble give you that `kubectl describe` cannot?** — the
   datapath's view: actual flows, policy verdicts, and drops, per connection.
3. **Cluster Mesh lets a pod do what across clusters?** — reach services in
   a peer cluster with the same identity, policy, and load-balancing model,
   as if they were local.
4. **Gateway API vs. Ingress — the key shift?** — routing (HTTPRoute) and
   infrastructure (Gateway) are separated, and the API is portable across
   controllers.
5. **3 replicas × 1 CPU requested, idle: how much are you paying for?** —
   3 CPUs reserved but ~none consumed; right-sizing requests and autoscaling
   are the first FinOps lever.

## See also

- CBA, CNPA, CNPE, and KCA pages on linuxfoundation.org / cncf.io, and the
  Cilium, eBPF, and FinOps Foundations docs (referenced by name; objectives
  change between releases).
- Back to [README](README.md) — the consolidation weeks: full-bank Training →
  Mastery → Exam rehearsal across all sixteen banks until ≥ 0.70 on three
  consecutive attempts each.
