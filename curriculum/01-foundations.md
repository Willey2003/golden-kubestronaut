# Module 1 — Foundations (KCNA · KCSA · CCA)

Module 1 builds the vocabulary every later module reuses: what *cloud
native* means, how Kubernetes is put together, and the security and
cloud-networking fundamentals you must hold before you touch the control
plane. It targets the **KCNA**, **KCSA**, and **CCA** tracks. These are
associate-level, mostly-knowledge certifications, so this module reads
denser than it types — but the concepts here are load-bearing for every
hands-on module that follows.

## Learning objectives

1. Explain cloud-native concepts: the CNCF landscape, containers,
   orchestration, and why immutable infrastructure matters.
2. Describe Kubernetes architecture — control plane, nodes, etcd — and its
   core objects: Pod, Deployment, Service, Namespace, ConfigMap, Secret.
3. Drive `kubectl` at associate level: `get`, `describe`, `logs`, `exec`,
   labels, selectors, and namespaces.
4. Explain cloud-native security fundamentals: defense in depth, least
   privilege, RBAC, ServiceAccounts, secrets, and container isolation.
5. Identify threat models, supply-chain risks, and compliance frameworks
   (CIS, NIST, GDPR) for cloud-native environments.
6. Define cloud-computing models (IaaS/PaaS/SaaS, virtualization,
   elasticity) and the cloud-networking concepts — CNI, eBPF — that the CCA
   track introduces.

## Key concepts

- **Cloud native**: applications built from containers, orchestrated as
  services, managed declaratively, and designed for horizontal scale. The
  CNCF landscape organises the ecosystem into layers — runtime, orchestration,
  service mesh, observability, security.
- **Container**: an isolated process with its own filesystem, cgroup limits,
  and namespaces, built from an image. Images are read-only layer stacks;
  tags are mutable, digests (`@sha256:...`) are not.
- **Orchestration**: the layer that schedules, heals, and scales containers.
  Kubernetes is the dominant orchestrator; the scheduler places Pods on nodes
  that have capacity and match constraints.
- **Kubernetes architecture**: a control plane (`kube-apiserver`,
  `kube-scheduler`, `kube-controller-manager`, `etcd`) and worker nodes
  running `kubelet` and `kube-proxy`. Everything goes through the API server;
  the API server is the only component others talk to.
- **Object model**: you declare desired state (`kubectl apply`) and
  controllers reconcile it. Pods are the scheduling unit; Deployments own
  ReplicaSets which own Pods; Services give Pods a stable virtual IP; a
  Namespace is a virtual cluster boundary.
- **Security fundamentals (KCSA)**: defense in depth — no single control is
  trusted; least privilege — grant only what a workload needs; the CIA triad
  (confidentiality, integrity, availability) as the lens for every decision.
- **RBAC**: Kubernetes authorisation is Role/ClusterRole bound to a
  Subject via RoleBinding/ClusterRoleBinding. `kubectl auth can-i get pods`
  tests it. Namespaces limit the blast radius of Roles.
- **ServiceAccount**: the identity a Pod runs as. Containers use their
  namespace's `default` ServiceAccount unless told otherwise; tokens from the
  projected service account are how in-cluster clients authenticate.
- **Secrets and ConfigMaps**: configuration injected into Pods — never bake
  credentials into images. `kubectl create secret generic db-creds
  --from-literal=password=...` then mount or env them.
- **Threat modeling**: KCSA expects you to reason about an attacker's view —
  STRIDE (spoofing, tampering, repudiation, information disclosure, denial of
  service, elevation of privilege) as a checklist, and attack trees over the
  cluster's components.
- **Supply chain**: every image you pull is a supply-chain decision —
  pinned digests, signed images, scanned registries. (Module 4 does the full
  CKS treatment.)
- **Compliance**: CIS benchmarks (baseline hardening), NIST 800-53 (controls
  and monitoring), and GDPR (data residency and retention) are the frameworks
  you will be asked to map findings to.
- **Cloud computing models**: IaaS (VM, storage), PaaS (managed runtime),
  SaaS (managed application); the value proposition is elasticity — paying
  for what you use and scaling with demand.
- **Cloud networking (CCA, conceptual)**: a CNI plugin (Flannel, Calico,
  Cilium) implements the cluster's pod network and policy. eBPF lets kernel
  programs run in a sandboxed VM at packet-arrival speed — that is what makes
  Cilium's identity-based policies and per-pod observability possible. The
  full datapath is Module 7.

## Hands-on exercises

The KCNA/KCSA banks are knowledge engines, so these exercises exist to make
the concepts concrete. Exercises 1–3 need a live cluster; 4–6 do not.

### Exercise 1 — Inventory your cluster

- Task: list nodes, namespaces, and the control-plane components that are
  running.
- Expected outcome: you can point at each control-plane component in the
  output and say what it does.

```sh
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

- Verification:

```sh
kubectl get ns
kubectl get pods -n kube-system | awk '{print $1}'
```

### Exercise 2 — Pods, labels, and describe

- Task: create a two-pod Deployment with a label, then inspect it with
  `describe` and a label selector.
- Expected outcome: `kubectl get pods -l app=web` returns exactly the
  Deployment's Pods; `describe` shows the events that created them.

```sh
kubectl create deployment web --image=nginx:stable-alpine --replicas=3
kubectl label deployment web app=web
kubectl get pods -l app=web
kubectl describe pod -l app=web | head -30
```

- Verification:

```sh
kubectl get deployment web -o jsonpath='{.spec.replicas}{"\n"}'
```

### Exercise 3 — Namespaces and quotas

- Task: create `dev`, set a ResourceQuota, and confirm an over-quota
  Deployment is rejected.
- Expected outcome: the quota is enforced — the second Deployment stays
  `Pending` or is refused, and `kubectl describe quota` shows the limit.

```sh
kubectl create namespace dev
kubectl create quota dev-quota --hard=requests.cpu=1 -n dev
kubectl create deployment a --image=nginx:stable-alpine --replicas=2 -n dev
kubectl create deployment b --image=nginx:stable-alpine --replicas=2 -n dev
kubectl describe quota dev-quota -n dev
```

- Verification:

```sh
kubectl get pods -n dev
```

### Exercise 4 — RBAC reasoning (no cluster needed)

- Task: given *"a ServiceAccount that may read Deployments in one
  namespace"*, write the RBAC objects on paper.
- Expected outcome: a `Role` with `get`/`list`/`watch` on `deployments`, a
  `RoleBinding` binding it to the ServiceAccount, both scoped to the
  namespace — not `ClusterRole`, which would leak read access cluster-wide.

### Exercise 5 — Threat-model one scenario (no cluster needed)

- Task: list the attack paths an unprivileged Pod has to the API server.
- Expected outcome: you can enumerate token exposure, RBAC over-granting,
  `hostNetwork`/privileged escape, and image tampering — and name the
  control (RBAC review, PSP/PSS, signed images) for each.

### Exercise 6 — Cloud model mapping (no cluster needed)

- Task: for IaaS, PaaS, and SaaS, name one Kubernetes-adjacent example and
  one control you hold in each.
- Expected outcome: you can argue why a managed Kubernetes (PaaS) still
  leaves you responsible for workloads, namespaces, and RBAC — the classic
  KCSA "shared responsibility" question.

## Test yourself

When you can do Exercises 1–6, sit simulator attempts:

- **Bank**: `banks/kcna` — **Training**, `focus_domain =
  cloud-native-architecture`, then **Mastery** on the same domain, then
  **Training** on `kubernetes-resources`. Aim for 80%+ on Mastery.
- **Bank**: `banks/kcsa` — **Training**, `focus_domain = threat-modeling`,
  then **Mastery**, `focus_domain = cluster-security`. KCSA's pass threshold
  is 0.75 — the steepest of the associate banks; treat 80% Mastery as the
  floor.
- **Bank**: `banks/cca` — **Training**, `focus_domain = ebpf-fundamentals`
  only, to fix the CNI/eBPF vocabulary before Module 7 goes deep.

A weak domain means redo the matching exercise — RBAC gaps point at Exercise
4, threat-model gaps at Exercise 5.

## Self-check quiz

1. **Which component is the only one other components talk to, and why does
   that matter for security?** — `kube-apiserver`. All authn/authz happens
   there, so RBAC and audit are enforced in one place.
2. **A Role vs a ClusterRole: when do you use which?** — Role for one
   namespace (scoped to `metadata.namespace`); ClusterRole for cluster-wide
   resources or reuse across namespaces via RoleBinding.
3. **Why are image digests safer than tags?** — tags are mutable pointers;
   a digest pins the exact content, so a re-pushed tag cannot change what
   you run.
4. **Name one control for each STRIDE category on a cluster.** — e.g. TLS
   for spoofing, signed images for tampering, RBAC for elevation of
   privilege, audit logs for repudiation, encryption for disclosure, quotas
   for DoS.
5. **In shared responsibility, who secures the Pod network policy on
   managed Kubernetes?** — you. The provider runs the control plane; your
   NetworkPolicies, RBAC, and image supply chain stay yours.

## See also

- KCNA, KCSA, and CCA pages on linuxfoundation.org / cncf.io (referenced by
  name; objectives change between releases).
- [Module 2 — Core administration (CKA · LFCS)](02-core-administration.md) —
  next: the control plane, kubeadm, etcd, and the first hands-on bank.
