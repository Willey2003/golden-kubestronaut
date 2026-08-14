# Golden Kubestronaut 2026

The **complete Kubestronaut badge program** + **LFCS** — every CNCF
certification on the 2026 badge list in one simulator, with recent-objective
learning, original practice tests, behaviour-graded hands-on scenarios, and a
guided curriculum.

> Certifications covered (2026):
> **CKA** · **CKAD** · **CKS** · **KCNA** · **KCSA** · **PCA** · **ICA** ·
> **CCA** · **CAPA** · **CGOA** · **CBA** · **OTCA** · **KCA** · **CNPA** ·
> **CNPE** · **LFCS**

Question banks are **original practice questions written to the published
exam objectives** — they are not reproductions of real exam dumps.

## The stack

- **Engine** — facilitator (UI/API), conductor (grader), bank loader +
  validator, `ga` CLI. Pure Python stdlib.
- **Exams** — one bank per certification, Training / Mastery / Exam modes,
  stratified draws, pass thresholds, weakest-domain reporting.
- **Curriculum** — a guided path with a module per certification track.
- **Labs** — high-level scenario labs with verification commands.

## Quick start

```bash
./ga doctor        # preflight
./ga up            # start platform (default http://127.0.0.1:8902)
```

Open <http://127.0.0.1:8902> (or the LAN address after `./ga expose`).

## Certification banks

| Bank | Certification | Focus |
|---|---|---|
| `cka` | Certified Kubernetes Administrator | cluster ops, hands-on |
| `ckad` | Certified Kubernetes Application Developer | app dev, hands-on |
| `cks` | Certified Kubernetes Security Specialist | security, hands-on |
| `kcna` | Kubernetes and Cloud Native Associate | fundamentals, knowledge |
| `kcsa` | Cloud Native Security Associate | security concepts |
| `pca` | Prometheus Certified Associate | metrics/PromQL |
| `ica` | Istio Certified Associate | service mesh |
| `cca` | Cloud Certified Associate | cloud-native platform concepts |
| `capa` | Certified Argo Project Associate | GitOps / Argo |
| `cgoa` | Certified GitOps Associate | GitOps practices |
| `cba` | Cilium Certified Associate | CNI / eBPF networking |
| `otca` | OpenTelemetry Certified Associate | observability |
| `kca` | Kubernetes Cost Associate | cloud costs / FinOps |
| `cnpa` | Cloud Native Professional Associate | professional track |
| `cnpe` | Cloud Native Professional Engineer | engineering track |
| `lfcs` | Linux Foundation Certified System Administrator | Linux sysadmin |

See `docs/` for architecture, bank spec, install, and security.

## License

Apache-2.0. Independent project — not affiliated with the Linux Foundation or
CNCF. Certification names are trademarks of their owners.
