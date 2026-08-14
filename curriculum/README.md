# Golden Kubestronaut 2026 — Learning Curriculum

This directory is the guided study path for the **Golden Kubestronaut 2026**
platform — the complete CNCF badge set plus LFCS, sixteen certifications in
one simulator. It is not a topic dump: it tells you **what** to study, **in
what order**, and **when** to sit which practice attempt against the
simulator and your real cluster. Work the modules top to bottom — later
modules assume the skills of earlier ones.

The sixteen certifications are **CKA** · **CKAD** · **CKS** · **KCNA** ·
**KCSA** · **PCA** · **ICA** · **CCA** · **CAPA** · **CGOA** · **CBA** ·
**OTCA** · **KCA** · **CNPA** · **CNPE** · **LFCS**. They are not studied as
sixteen separate sprints: they cluster into **eight stages** and seven
modules, so a single study session feeds several banks at once.

| Module | Stage | Certs | Bank(s) | Engine |
|---|---|---|---|---|
| [01](01-foundations.md) | Foundations | KCNA · KCSA · CCA | `banks/kcna`, `banks/kcsa`, `banks/cca` | knowledge · mixed |
| [02](02-core-administration.md) | Core administration | CKA · LFCS | `banks/cka`, `banks/lfcs` | hands-on |
| [03](03-application-development.md) | Application development | CKAD | `banks/ckad` | hands-on |
| [04](04-security.md) | Security | CKS | `banks/cks` | hands-on |
| [05](05-observability.md) | Observability | PCA · OTCA | `banks/pca`, `banks/otca` | mixed |
| [06](06-mesh-and-gitops.md) | Mesh & GitOps | ICA · CAPA · CGOA | `banks/ica`, `banks/capa`, `banks/cgoa` | mixed |
| [07](07-networking-and-cost.md) | Networking & cost | CBA · CNPA · CNPE · KCA | `banks/cba`, `banks/cnpa`, `banks/cnpe`, `banks/kca` | mixed |

## The eight stages

1. **Foundations** (Module 1) — KCNA, KCSA, CCA. Cloud-native concepts,
   Kubernetes fundamentals, security fundamentals, and the cloud-networking
   vocabulary every later stage reuses. This stage is mostly knowledge
   questions; the CCA piece introduces CNI and eBPF at a conceptual level.
2. **Core administration** (Module 2) — CKA and LFCS. Cluster architecture,
   installation, workloads, storage, and troubleshooting, plus the Linux
   sysadmin fundamentals that underpin every command you will type.
3. **Developer** (Module 3) — CKAD. Designing, building, configuring, and
   exposing applications on Kubernetes, from the developer's side of the API.
4. **Security** (Module 4) — CKS. Cluster hardening, supply-chain security,
   and runtime security on top of the CKA cluster skills.
5. **Observability** (Module 5) — PCA (Prometheus, PromQL, exporters,
   alerting) and OTCA (OpenTelemetry traces, metrics, logs, the collector).
6. **Mesh & GitOps** (Module 6) — ICA (Istio service mesh), CAPA (Argo CD,
   Workflows, Rollouts, Events), CGOA (GitOps principles and practice).
7. **Networking** (Module 7) — CBA (Cilium and eBPF), CNPA and CNPE (the
   professional track: multi-cluster networking, Cluster Mesh, platform
   engineering).
8. **Cost** (Module 7) — KCA cost and FinOps: allocating, right-sizing, and
   optimising spend on cloud-native infrastructure.

The last two stages share a module because their material is small; the
module's "Test yourself" block splits them by `focus_domain`.

## How to use this path

1. **Read the module.** Start with the learning objectives, then the key
   concepts. Concepts are ordered so each one builds on the last.
2. **Do the exercises on a real cluster.** Every exercise has a concrete
   task, an expected outcome, and the exact `kubectl` command to verify it.
   Do not skip verification — if the check fails, the cluster state is not
   what you think it is.
3. **Finish each module with its "Test yourself" block.** Sit a simulator
   attempt on the module's bank, narrowed to that module's `focus_domain`.
   Start in **Training** mode (solutions shown), then redo it in **Mastery**
   mode (timed, no hints).
4. **Escalate to Exam mode only after several clean Mastery passes.** The
   rhythm is *Training → Mastery → Exam*. Jumping straight to Exam wastes the
   most informative mode you have.
5. **Let the attempt report drive review.** The report ranks your domains
   weakest-first. Re-run the matching exercises, then drill that domain with
   a focused Mastery attempt.
6. **Repeat until 0.70.** The banks' `pass_threshold` is 0.70. Sit full-length
   Exam attempts until you score at or above it on three consecutive attempts
   per bank.

## Prerequisite skills

You should be comfortable with the following **before** starting Week 1:

- **Linux command line** — navigating the filesystem, running commands,
  redirecting and piping output, editing files with `vim`/`nano`.
- **`kubectl` basics** — you have run `kubectl get nodes` at least once and
  understand contexts and `-n` namespace flags.
- **YAML** — indentation, mappings, lists. Almost every exercise is writing
  or reading YAML.
- **Container basics** — what an image is, images vs. containers, registries
  and tags, and a rough idea of what a `Dockerfile` does.
- **Networking fundamentals** — IP addresses, ports, TCP, HTTP, and DNS at a
  conceptual level.
- **git** — clone, commit, push; GitOps and Argo exercises in Module 6 assume
  it.

No Kubernetes administration experience is required to start Module 1. The
faster you can type a `kubectl` command, the further each module's exercises
take you.

## Tooling setup

Run `./ga doctor` in the repo root for a preflight check, then install:

- **`kubectl` CLI** — within ±1 of your cluster's server version. The
  curriculum uses `kubectl` throughout.
- **A cluster** — `kind` (fastest, ideal for Modules 1–3), `k3s` (low-RAM),
  or a kubeadm cluster (recommended for Module 4's security exercises, which
  touch `kube-apiserver` static-pod flags).
- **Python 3 + Docker** — the platform stack (`./ga doctor`, `./ga up`).
- **Per-cert CLIs, installed when a module needs them**: `helm`, `jq`,
  `promtool`, `otelcol`, `istioctl`, `argocd`, `cilium`, `falco`, `trivy`,
  `cosign`, `kubecost`.

Verify the plumbing, then start the platform:

```bash
./ga doctor
./ga up                    # http://127.0.0.1:8902
kubectl cluster-info
kubectl get nodes
```

## Simulator modes

Every bank can be attempted in three modes. They exist to be used **in that
order**:

| Mode | Timer | Answer reveal | Use it for |
|---|---|---|---|
| **Training** | Off | Solutions + explanations shown immediately | Learning each domain right after a module |
| **Mastery** | On | Hidden until graded | Practicing under time pressure without hints |
| **Exam** | On | Hidden until graded | Full dress rehearsal at real exam duration |

Attempts are drawn stratified by **domain**. Restricting a Training or
Mastery attempt to one domain is a `focus_domain` drill — that is how you
target exactly what a module just taught. Domain names live in each bank's
`exam.yaml` and match the vocabulary at the end of every module.

## The focus_domain drill

The single most effective use of the simulator is the focused drill:

1. Open the bank for the cert you just studied (`/banks/<cert>` or
   `./ga exam <cert>`).
2. Set `focus_domain` to the domains the module covered.
3. Run it in **Training** first and read every explanation — right or wrong.
4. Re-run the same `focus_domain` in **Mastery**, timed. 80%+ means move on;
   below that, redo the module exercises and drill again.

Every module ends with the exact drill to run. Example from Module 2: *Open
the `cka` bank → `focus_domain = troubleshooting`*, then re-run the
troubleshooting exercises if the report flags them.

## Study rhythm

The pattern repeats for every module, and it is the whole point of the
platform:

1. **After a module** → a **Training** attempt, focused on that module's
   domains. Read every explanation, even for questions you answered correctly.
2. **After two modules** → a **Mastery** attempt spanning those domains.
   Time-box it; no hints.
3. **After all seven modules** → one full-bank **Training** attempt, then one
   full-bank **Mastery** attempt, on every bank.
4. **The week before the exam** → repeated full-length **Exam** attempts until
   you hit the bank's `pass_threshold` (0.70) on three in a row.

A weak domain on the attempt report is a command to **redo exercises**, not to
re-read the module — the grader checks the same real cluster the exercises
used, so muscle memory is what scores.

## Recommended weekly plan

Roughly 10–12 hours per week. Weeks 1–12 introduce material; weeks 13–14 are
consolidation and rehearsal. Compress to 10 weeks if you are already
comfortable with Linux or cloud-native fundamentals; stretch the security or
networking weeks if not — the schedule is a floor, not a trap.

| Week | Study | Practice attempts to sit |
|---|---|---|
| 1 | Module 1 — cloud-native + Kubernetes fundamentals | Training: `kcna` → `cloud-native-architecture`, `kubernetes-resources` |
| 2 | Module 1 — KCSA security + CCA concepts | Training: `kcsa` → `threat-modeling`, `cluster-security`; **Mastery**: `kcna` → `kubernetes-resources` |
| 3 | Module 2 — cluster architecture, install, maintenance | Training: `cka` → `cluster-architecture`; Training: `lfcs` → `filesystem-storage` |
| 4 | Module 2 — workloads, storage, troubleshooting | Training: `cka` → `workloads-scheduling`, `troubleshooting`; **Mastery**: `cka` → `cluster-architecture` |
| 5 | Module 2 — LFCS + Module 3 — CKAD design/deploy | Training: `lfcs` → `users-groups-permissions`, `processes-services`; Training: `ckad` → `application-design-build` |
| 6 | Module 3 — config, services, networking | Training: `ckad` → `configuration-security`, `services-networking`; **Mastery**: `ckad` → `application-deployment` |
| 7 | Module 4 — CKS cluster + system hardening | Training: `cks` → `cluster-setup`, `cluster-hardening`, `system-hardening` |
| 8 | Module 4 — CKS supply chain + runtime | Training: `cks` → `supply-chain`, `runtime-security`; **Mastery**: `cks` → `cluster-hardening` |
| 9 | Module 5 — PCA Prometheus + PromQL | Training: `pca` → `promql`, `scraping-targets`; **Mastery**: `pca` → `alerting-recording-rules` |
| 10 | Module 5 — OTCA + Module 6 — ICA | Training: `otca` → `instrumentation`, `collector-configuration`; Training: `ica` → `traffic-management` |
| 11 | Module 6 — CAPA + CGOA | Training: `capa` → `argocd-core-concepts`, `argo-workflows`; Training: `cgoa` → `gitops-principles` |
| 12 | Module 7 — Cilium, CNPA/CNPE, KCA cost | Training: `cba` → `network-policies-security`, `cluster-mesh`; Training: `kca` → `finops-fundamentals` |
| 13 | Weak-domain drill + re-run failing exercises | Full-bank **Mastery** on your weakest banks; focused Mastery per weak domain |
| 14 | Rehearsal | Full-length **Exam** dry runs on every bank until ≥ 0.70 three times in a row |

## Reading your attempt report

After every Mastery or Exam attempt the score screen shows per-domain
performance. Treat domains below 70% as a backlog: do one focused Mastery
attempt per weak domain, and re-run the module exercises named in that
domain's row above.

## Relationship to the official certifications

The banks in `banks/` are original practice questions written to the
published exam objectives of the official certifications, which are
referenced here **by name only** — always consult the vendor site for the
current objectives, since they change between exam releases. The Kubestronaut
badge set is defined by CNCF; the list of qualifying certifications for the
current program year lives on the Kubestronaut site (kubestronaut.io), and
each exam's objectives page is on linuxfoundation.org or cncf.io.

Golden Kubestronaut 2026 is an independent simulator, not affiliated with the
Linux Foundation or CNCF. Certification names are trademarks of their owners.

## Contents

- [Module 1 — Foundations (KCNA · KCSA · CCA)](01-foundations.md)
- [Module 2 — Core administration (CKA · LFCS)](02-core-administration.md)
- [Module 3 — Application development (CKAD)](03-application-development.md)
- [Module 4 — Security (CKS)](04-security.md)
- [Module 5 — Observability (PCA · OTCA)](05-observability.md)
- [Module 6 — Mesh & GitOps (ICA · CAPA · CGOA)](06-mesh-and-gitops.md)
- [Module 7 — Networking & cost (CBA · CNPA · CNPE · KCA)](07-networking-and-cost.md)
