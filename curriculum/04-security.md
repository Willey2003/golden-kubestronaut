# Module 4 — Security (CKS)

Module 4 is the **CKS** track: hardening the cluster, securing the supply
chain, and detecting compromise at runtime. It builds directly on Module 2 —
the CKS bank assumes you can administer a cluster — and on the security
fundamentals from Module 1's KCSA. CKS is the most defensive certification
in the program and the one whose exercises touch the control plane
directly: many of them require a kubeadm cluster where you can edit static
Pod manifests and API-server flags.

## Learning objectives

1. Harden cluster setup: RBAC review, ServiceAccounts, kube-apiserver flags,
   and API access over secure channels.
2. Harden the cluster: CIS benchmarks with `kube-bench`, NetworkPolicies,
   Secrets, and Pod Security admission.
3. Harden the host and workload: AppArmor, seccomp, kernel hardening,
   SecurityContexts, and resource constraints.
4. Mitigate microservice vulnerabilities: image scanning, admission
   controllers, and Pod-level mitigations.
5. Secure the supply chain: signed images, provenance, SLSA, and admission
   of verified artifacts.
6. Detect and respond at runtime: Falco, OPA/Gatekeeper policy, and
   immutable workload behaviour.

## Key concepts

- **RBAC hardening**: audit `kubectl get clusterrolebindings` and trim
  `system:*` and `cluster-admin` bindings to a minimum; use namespaced Roles
  where possible; test with `kubectl auth can-i`. Automation has an
  identity — a dedicated ServiceAccount per app, never `default` with
  elevated Roles.
- **kube-apiserver flags**: `--disable-anonymous`, `--enable-admission-plugins`,
  `--authorization-mode=RBAC`, TLS on every listener. On kubeadm clusters
  these live in `/etc/kubernetes/manifests/kube-apiserver.yaml`; `kube-bench`
  audits them against the CIS benchmark.
- **Pod Security admission**: replaces PSPs. `pod-security.kubernetes.io/`
  namespaces are `privileged`, `baseline`, or `restricted`; start with
  `enforce` on your critical namespaces and watch the audit mode before
  enforcing.
- **NetworkPolicy**: default-deny everything, then allow what the app needs.
  Labels select peers; the policy is evaluated by the CNI at packet time.
  Missing default-deny is the most common CKS miss.
- **Secrets**: encrypt at rest (`--encryption-provider-config` with AES-CBC
  or KMS), rotate keys, and never put secrets in env vars a `ps` could read
  — mount them.
- **SecurityContext**: the workload-side controls: `runAsNonRoot: true`,
  `runAsUser: 1000`, `readOnlyRootFilesystem: true`,
  `allowPrivilegeEscalation: false`, and drop all capabilities
  (`capabilities: {drop: [ALL]}`).
- **seccomp and AppArmor**: restrict syscalls a container may make. Profile
  types `RuntimeDefault`/`Localhost` on `seccompProfile`; AppArmor via
  `container.apparmor.security.beta.kubernetes.io/<name>` annotations on
  supporting nodes.
- **Audit logging**: `--audit-log-path` records who called the API server
  with what, at `Metadata`/`Request`/`RequestResponse` verbosity. A
  policy that says *admit, but log, then alert* is how you catch the
  first sign of abuse before the attack reaches a workload.
- **Immutable containers and images**: `readOnlyRootFilesystem`, `tmpfs` for
  writable dirs, `imagePullPolicy: Always` only where tags are mutable, and
  pinned digests for production.
- **Admission controllers**: the gate between the API server and etcd —
  `NodeRestriction`, `PodSecurity`, `LimitRanger`, `ResourceQuota`, and
  webhooks run in order and can mutate or reject. A webhook that requires
  an immutable tag is the difference between "policy written" and "policy
  enforced".
- **Supply chain**: sign images with `cosign sign`, verify with `cosign
  verify`, attach attestations (SBOM, provenance) with `--attestation`, and
  gate admission with an `ImagePolicyWebhook` or policy engine so only signed
  images deploy. SLSA describes the build provenance trail that makes that
  gating meaningful.
- **Scanning**: `trivy image` and `trivy repo` find CVEs in images and
  manifests; scanning is admission-time, not build-time. A known-vulnerable
  image should be re-tagged and the gate re-run.
- **Runtime detection (Falco)**: Falco watches syscalls and fires on
  behavioural rules — shell in a container, `ptrace`, unexpected binds,
  `kubectl exec` into production. Alerts surface, you investigate; detection
  is the last line of a defence-in-depth stack.
- **Policy as code (OPA/Gatekeeper)**: admission-time enforcement — "no
  `latest` tags", "must set requests", "privileged containers denied" — as
  ConstraintTemplates and Constraints. This is where CKS and the KCSA
  `workload-security` domain converge.

## Hands-on exercises

These need a **kubeadm cluster** (Exercises 1 and 4 edit static Pod
manifests). On `kind`/`k3s` you can still do 2, 3, and 5.

### Exercise 1 — Harden the API server (kubeadm)

- Task: add `--disable-anonymous=true` and `--audit-log-path` to the
  kube-apiserver manifest, then verify the API server restarts with them.
- Expected outcome: `kubectl -n kube-system get pod kube-apiserver -o yaml |
  grep -A1 '--disable-anonymous'` shows the flag, and audit logs appear.

```sh
vim /etc/kubernetes/manifests/kube-apiserver.yaml   # add the two flags
kubectl -n kube-system wait --for=condition=Ready pod/kube-apiserver --timeout=120s
```

- Verification:

```sh
kubectl -n kube-system get pod kube-apiserver -o yaml | grep -E 'disable-anonymous|audit-log-path'
sudo tail -5 /var/log/kubernetes/audit.log
```

### Exercise 2 — Default-deny NetworkPolicy

- Task: write a NetworkPolicy that denies all ingress to a namespace, then
  add an allow rule for a specific client label.
- Expected outcome: a probe from a non-permitted Pod fails; from the
  permitted one it succeeds. Do this in a throwaway namespace.

```sh
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all, namespace: secure}
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
kubectl exec probe -- wget -qO- http://target-svc/ | head -1   # times out
```

- Verification:

```sh
kubectl describe networkpolicy -n secure deny-all
kubectl get networkpolicies -n secure
```

### Exercise 3 — Enforce Pod Security admission

- Task: mark the `secure` namespace `restricted` (enforce), then try to
  create a privileged Pod.
- Expected outcome: the privileged Pod is rejected with a PodSecurity
  admission error.

```sh
kubectl label ns secure pod-security.kubernetes.io/enforce=restricted
kubectl run bad --image=nginx:stable-alpine --privileged --dry-run=client -o yaml | kubectl apply -f -
```

- Verification:

```sh
kubectl get pods -n secure
```

### Exercise 4 — Image policy admission for signed images (kubeadm)

- Task: configure an `ImagePolicyWebhook` (or a Gatekeeper constraint) that
  rejects images not carrying a `cosign` signature.
- Expected outcome: an unsigned image is refused at admission; `cosign
  verify` succeeds on the allowed one.

```sh
cosign sign --key cosign.key ghcr.io/example/app:1.0
cosign verify --key cosign.pub ghcr.io/example/app:1.0
```

- Verification:

```sh
kubectl run unsigned --image=ghcr.io/example/unsigned:1.0 --dry-run=server
```

### Exercise 5 — Runtime detection with Falco

- Task: install Falco (or run the driver container), trigger a rule by
  running a shell inside a container, and confirm the event.
- Expected outcome: Falco's output shows a "shell in container" event for
  your exec.

```sh
helm install falco falcosecurity/falco --namespace falco --create-namespace
kubectl exec -n secure app -- sh -c 'echo pwned'   # triggers the rule
```

- Verification:

```sh
kubectl logs -l app.kubernetes.io/name=falco -n falco | grep -i shell
```

## Test yourself

- **Bank**: `banks/cks` — **Training**, `focus_domain = cluster-setup` and
  `cluster-hardening` first; then `supply-chain` and `runtime-security`.
- **Mastery**: `focus_domain = microservice-vulns`, then a full-bank Mastery.
  CKS's threshold is 0.67 but its hands-on questions grade live cluster
  state — treat the exercises as exam rehearsal.
- Weak `system-hardening` → Exercise 3; weak `supply-chain` → Exercise 4.

## Self-check quiz

1. **Why `--disable-anonymous`?** — anonymous requests bypass identity, so
   they must not reach API operations; disabling forces authentication on
   every call.
2. **restricted vs baseline Pod Security standards — one concrete
   difference?** — restricted requires `runAsNonRoot` and drops all
   capabilities; baseline only rejects privileged escalation patterns.
3. **Where does a NetworkPolicy get enforced, and why does default-deny
   matter?** — in the CNI data path, per packet; without a default-deny rule
   the implicit allow-all makes policy additions meaningless.
4. **What does `cosign verify` actually prove?** — the image's digest matches
   what the keyholder signed; pair it with admission so verification is
   mandatory, not optional.
5. **Falco observes what that RBAC cannot?** — runtime behaviour: syscalls
   from inside a running container, which authorisation never sees.

## See also

- CKS page on linuxfoundation.org (referenced by name; objectives change
  between releases).
- [Module 5 — Observability (PCA · OTCA)](05-observability.md) — next: making
  the hardened cluster measurable.
