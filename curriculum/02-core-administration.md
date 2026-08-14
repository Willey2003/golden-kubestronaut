# Module 2 — Core administration (CKA · LFCS)

Module 2 is where the path turns hands-on. It targets the **CKA** track —
cluster architecture, installation, workloads, storage, and troubleshooting
— and the **LFCS** track, the Linux sysadmin skills that are the ground
under every command you type. CKA's bank is the biggest in the program
because the exam is a live cluster: you will be graded on behaviour, not on
recalling facts. LFCS gives you the terminal speed the timer demands.

## Learning objectives

1. Explain and repair cluster architecture: control-plane components, etcd,
   kubelet, `kubeadm`, and static Pods.
2. Install a cluster with `kubeadm` and join nodes; back up and restore etcd.
3. Schedule and manage workloads: taints, tolerations, node selectors,
   `nodeName`, DaemonSets, and maintenance drains.
4. Provide storage: `emptyDir`, hostPath, PVCs, and StorageClasses.
5. Troubleshoot the cluster and workloads: `kubectl` diagnostics, pod logs,
   events, and etcd health.
6. Operate Linux systems: filesystem layout, users/groups/permissions,
   processes, `systemd` services, packages, and shell basics.

## Key concepts

- **Control plane**: `kube-apiserver` (front door, authn/authz),
  `kube-scheduler` (placement), `kube-controller-manager` (reconciliation
  loops), `etcd` (source of truth). On kubeadm clusters these run as static
  Pods in `/etc/kubernetes/manifests/` — edit the manifest, the API server
  flags change.
- **Nodes**: `kubelet` registers the node and reports status; `kube-proxy`
  implements Services (iptables/IPVS); the container runtime (`containerd`)
  runs containers via CRI. `kubectl get nodes -o wide` and
  `kubectl describe node` are your first diagnostics.
- **kubeadm**: `kubeadm init --pod-network-cidr=10.244.0.0/16` then
  `kubeadm join` with the token printed at the end. An existing cluster's
  certificate is at `/etc/kubernetes/pki/`.
- **etcd**: the only stateful component. Backup with `etcdctl snapshot save`
  (`ETCDCTL_API=3`), restore to a **new** data directory and point
  `--data-dir` at it — restoring in place corrupts the live store.
- **Scheduling**: taints repel (`node-role.kubernetes.io/control-plane:NoSchedule`),
  tolerations allow; `nodeSelector` and `nodeAffinity` pull; `kubectl drain
  --ignore-daemonsets` for maintenance; `kubectl cordon` stops new work.
- **DaemonSet**: one Pod per node — the right primitive for log shippers
  and node agents, which a Deployment (best-effort spread) cannot promise.
- **Storage**: `emptyDir` (scratch, dies with the Pod), `hostPath` (node
  directory), `PersistentVolumeClaim` → `PersistentVolume` (provisioned by a
  StorageClass). A PVC is the only one that survives node loss.
- **Troubleshooting**: events (`kubectl describe pod`, `kubectl get events
  --sort-by=.metadata.creationTimestamp`), logs (`kubectl logs -f`), image
  pull errors, `CrashLoopBackOff` vs `ImagePullBackOff`, and API-server
  failures (cert expiry, etcd down, scheduler not scheduling). Work top-down:
  API server → nodes → workloads.
- **Linux foundations (LFCS)**: the filesystem hierarchy (`/etc`, `/var`,
  `/tmp`, `/proc`, `/sys`), `fdisk`/`lsblk`/`mount`, and `/etc/fstab`.
- **Users and permissions**: `useradd`, `usermod -aG`, `chown`, `chmod`
  (octal: `750`), `setfacl`, `umask`, and `sudo` configuration — the LFCS
  bank's biggest domain.
- **Processes and services**: `ps`, `top`, `kill`/`kill -9`, `nohup`, and
  `systemctl enable --now unit`; units live in `/etc/systemd/system/` and
  `/usr/lib/systemd/system/`. `journalctl -u unit -f` is your log.
- **Packages**: `apt`/`dnf`, `rpm -q`, and local archives (`tar`, `zip`) —
  install, upgrade, verify, and remove without breaking dependencies.
- **Shells and scripting**: `bash`, variables, `if`/`for`, `grep`/`sed`/
  `awk` pipelines, exit codes, and `chmod +x` scripts. CKA questions about
  `kubectl` output almost always involve a pipe through one of these.

## Hands-on exercises

All exercises need a live cluster except where noted; the LFCS exercises run
on the same Linux host, cluster or not.

### Exercise 1 — Install and join (needs two nodes or a lab)

- Task: `kubeadm init` a control-plane node, then join a worker.
- Expected outcome: `kubectl get nodes` shows both `Ready` after a CNI is
  applied. This is the single highest-value CKA exercise; do it until it is
  boring.

```sh
kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube && cp /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl apply -f <cni-manifest>
kubeadm token create --print-join-command   # on the worker:
kubeadm join <control-plane-host>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

- Verification:

```sh
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Exercise 2 — etcd backup and restore

- Task: snapshot etcd, delete a namespace, restore the snapshot.
- Expected outcome: the deleted namespace comes back. Practice the "restore
  to a new directory + edit the manifest" sequence, not the shortcut.

```sh
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/server.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd.snap
```

- Verification (restore then check the object exists):

```sh
kubectl get ns lost-ns
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd.snap --data-dir=/var/lib/etcd-restore
kubectl get ns lost-ns
```

### Exercise 3 — Taints, tolerations, and node assignment

- Task: taint a node, schedule a Pod only onto it with a toleration, and
  keep everything else off it with a nodeSelector.
- Expected outcome: `kubectl get pods -o wide` shows the Pod on the tainted
  node; Pods without the toleration stay `Pending`.

```sh
kubectl taint nodes node-1 disktype=ssd:NoSchedule
kubectl run pin --image=nginx:stable-alpine --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"disktype","operator":"Equal","value":"ssd","effect":"NoSchedule"}],"nodeSelector":{"disktype":"ssd"}}}'
```

- Verification:

```sh
kubectl get pod pin -o wide
kubectl describe pod pin | grep -A2 Tolerations
```

### Exercise 4 — Storage via PVC

- Task: create a StorageClass-backed PVC, mount it into a Pod, write a file,
  delete the Pod, and confirm the data survives in a new Pod.
- Expected outcome: the file persists across Pod recreation — the point of
  PVCs.

```sh
kubectl create namespace storage
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
kubectl run writer --image=busybox --restart=Never -n storage \
  --overrides='{"spec":{"containers":[{"name":"w","image":"busybox","command":["sh","-c","echo data > /mnt/file && sleep 3600"],"volumeMounts":[{"mountPath":"/mnt","name":"v"}]}],"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"data"}}]}}'
```

- Verification:

```sh
kubectl get pvc -n storage
kubectl exec writer -n storage -- cat /mnt/file
```

### Exercise 5 — Troubleshoot a broken workload

- Task: given a Deployment stuck `Pending`, find the cause with `describe`
  and `events`, fix it, and confirm `Running`.
- Expected outcome: you can name the exact cause (unschedulable, image
  misspelling, quota) from the events and fix it in under a minute.

```sh
kubectl describe deployment broken
kubectl get events --sort-by=.lastTimestamp | tail -10
```

- Verification:

```sh
kubectl get pods -l app=broken
```

### Exercise 6 — LFCS: user, unit, and script (Linux host)

- Task: create user `ops` in group `ops`, make `/srv/data` owned by the
  group with `rwxr-x---`, write a `systemd` unit that runs a script on boot,
  and enable it.
- Expected outcome: `id ops` shows both groups, `ls -ld /srv/data` shows
  `770 ops:ops`, and `systemctl is-enabled srv-job` prints `enabled`.

```sh
groupadd ops && useradd -m -G ops ops
mkdir /srv/data && chgrp ops /srv/data && chmod 770 /srv/data
cat > /etc/systemd/system/srv-job.service <<'EOF'
[Unit]
Description=Run ops job
[Service]
Type=oneshot
ExecStart=/usr/local/bin/ops-job.sh
EOF
systemctl enable --now srv-job
```

- Verification:

```sh
systemctl is-enabled srv-job
journalctl -u srv-job --no-pager | tail -5
```

## Test yourself

- **Bank**: `banks/cka` — **Training**, `focus_domain = cluster-architecture`
  first, then `troubleshooting`. Re-sit as **Mastery** after Exercises 1–5.
  CKA's threshold is 0.66 but aim for 80% on Mastery; the bank's hands-on
  questions grade the same cluster you just practised on.
- **Bank**: `banks/lfcs` — **Training**, `focus_domain =
  users-groups-permissions`, then **Mastery** on `processes-services`.
- Weak `troubleshooting` → redo Exercise 5; weak Linux domains → Exercise 6.

## Self-check quiz

1. **Why restore etcd to a fresh data directory?** — the running etcd keeps
   the old directory open; restoring in place corrupts the live store, so you
   restore to a new path and point `--data-dir` at it.
2. **A DaemonSet or a Deployment for a node agent, and why?** — DaemonSet; it
   guarantees one Pod per node, which a Deployment cannot promise.
3. **What do `kubectl cordon` and `kubectl drain` each do?** — cordon stops
   new scheduling; drain evicts existing Pods (use `--ignore-daemonsets`).
4. **`chmod 750` on a directory grants what to group members?** —
   read, execute, and traverse (`r-x`), but no writes.
5. **Which three checks explain an unreachable API server?** — etcd health
   (`etcdctl endpoint health`), apiserver cert expiry, and the static Pod
   manifest; then scheduler/kubelet health on nodes.

## See also

- CKA and LFCS pages on linuxfoundation.org (referenced by name; objectives
  change between releases).
- [Module 3 — Application development (CKAD)](03-application-development.md) —
  next: the developer side of the same API.
