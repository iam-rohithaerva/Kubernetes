# How to Restore an etcd Backup on a Bare Metal kubeadm Kubernetes Cluster

This guide covers restoring etcd from an encrypted backup created with [the backup guide](etcd-backup-kubeadm-baremetal.md). It includes single control plane restore, rebuilding a lost node, and restoring a multi-member cluster after quorum loss.

---

## Table of Contents

1. [Environment](#1-environment)
2. [When to Restore](#2-when-to-restore)
3. [Before You Start](#3-before-you-start)
4. [Prepare the Backup](#4-prepare-the-backup)
5. [Restore on a Single Control Plane](#5-restore-on-a-single-control-plane)
6. [Restore When the Node Was Lost](#6-restore-when-the-node-was-lost)
7. [Restore a Multi-Member Cluster](#7-restore-a-multi-member-cluster)
8. [Post-Restore Validation](#8-post-restore-validation)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Environment

| Item | Value |
|---|---|
| Cluster type | kubeadm, bare metal |
| Control plane nodes | `cp-01` (10.0.1.10), `cp-02` (10.0.1.11), `cp-03` (10.0.1.12) |
| API endpoint | `k8s-api.acme.local:6443` |
| etcd data directory | `/var/lib/etcd` |
| Static pod manifests | `/etc/kubernetes/manifests` |
| Backup location | `minio/etcd-backups/cp-01/` |
| Example backup | `cp-01-2026-10-04-0600.tgz.enc` |
| Passphrase | Vault path `secret/k8s/etcd-backup` |

---

## 2. When to Restore

| Situation | Restore needed? | Action |
|---|---|---|
| One member down, quorum intact | No | Remove and re-add the member |
| etcd pod crashlooping, data intact | No | Fix the cause (disk, certs, manifest) |
| Data corrupted on a single control plane | **Yes** | [Section 5](#5-restore-on-a-single-control-plane) |
| Control plane VM or disk lost | **Yes** | [Section 6](#6-restore-when-the-node-was-lost) |
| Quorum permanently lost in a multi-member cluster | **Yes** | [Section 7](#7-restore-a-multi-member-cluster) |
| Bad change (mass delete, broken upgrade) | **Yes** | Section 5 or 7 |

> Restoring rolls the whole cluster back to the snapshot time. Every change made after the snapshot is lost and must be re-applied, ideally from GitOps.

---

## 3. Before You Start

- Announce a maintenance window. The API server will be unavailable during the restore.
- Pick the latest healthy backup taken **before** the incident.
- Make sure the passphrase is available outside the cluster.
- Make sure the Kubernetes and etcd versions match the ones in use when the backup was taken.
- Work as root on the control plane node:

```bash
ssh admin@cp-01
sudo -i
```

---

## 4. Prepare the Backup

### Step 1: List available backups

```bash
mc ls minio/etcd-backups/cp-01/
```

### Step 2: Download the backup

```bash
mkdir -p /restore && chmod 700 /restore
mc cp minio/etcd-backups/cp-01/cp-01-2026-10-04-0600.tgz.enc /restore/
```

For AWS S3:

```bash
aws s3 cp s3://acme-etcd-backups/cp-01/cp-01-2026-10-04-0600.tgz.enc /restore/
```

### Step 3: Fetch the passphrase

```bash
vault kv get -field=passphrase secret/k8s/etcd-backup > /root/.etcd-backup-pass
chmod 600 /root/.etcd-backup-pass
```

### Step 4: Decrypt

Use the same cipher options as the backup, plus `-d`.

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in  /restore/cp-01-2026-10-04-0600.tgz.enc \
  -out /restore/cp-01-2026-10-04-0600.tgz \
  -pass file:/root/.etcd-backup-pass
```

A `bad decrypt` error means the passphrase is wrong.

### Step 5: Extract

```bash
mkdir -p /restore/extracted
tar xzf /restore/cp-01-2026-10-04-0600.tgz -C /restore/extracted
ls /restore/extracted/backup/
ls /restore/extracted/etc/kubernetes/pki/
```

Expected:

```text
etcd-2026-10-04-0600.db  kubeadm-config-2026-10-04-0600.yaml
apiserver.crt  apiserver.key  ca.crt  ca.key  etcd  front-proxy-ca.crt  sa.key  sa.pub ...
```

### Step 6: Verify the snapshot

```bash
SNAP=/restore/extracted/backup/etcd-2026-10-04-0600.db
etcdutl snapshot status $SNAP -w table
```

---

## 5. Restore on a Single Control Plane

Use this when the node is alive but etcd data is corrupted or needs to be rolled back.

### Step 1: Stop the API server and etcd

Static pods stop when their manifest leaves `/etc/kubernetes/manifests`. Stopping the API server first prevents writes during the restore.

```bash
mkdir -p /root/manifests-bak
mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/manifests-bak/
mv /etc/kubernetes/manifests/etcd.yaml /root/manifests-bak/
```

Wait until both containers are gone:

```bash
watch "crictl ps | grep -E 'etcd|kube-apiserver'"
```

### Step 2: Move the old data aside

Keep it until the restore is confirmed, do not delete it.

```bash
mv /var/lib/etcd /var/lib/etcd-old-$(date +%F-%H%M)
```

### Step 3: Restore the snapshot

This rebuilds the etcd data directory (`member/snap/db` and `member/wal`) from the snapshot.

```bash
etcdutl snapshot restore $SNAP --data-dir /var/lib/etcd
ls /var/lib/etcd/member
```

Expected:

```text
snap  wal
```

> If you restore into a different directory, update both `--data-dir` and the `etcd-data` `hostPath` in `etcd.yaml`.

### Step 4: Start etcd

```bash
mv /root/manifests-bak/etcd.yaml /etc/kubernetes/manifests/
watch "crictl ps | grep etcd"
```

Check health:

```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Step 5: Start the API server

```bash
mv /root/manifests-bak/kube-apiserver.yaml /etc/kubernetes/manifests/
watch "crictl ps | grep kube-apiserver"
```

### Step 6: Refresh the other components

The scheduler, controller-manager, and kubelet hold caches from before the restore.

```bash
systemctl restart kubelet
kubectl -n kube-system delete pod -l component=kube-scheduler
kubectl -n kube-system delete pod -l component=kube-controller-manager
```

Restart the kubelet on worker nodes as well:

```bash
for node in worker-01 worker-02 worker-03; do
  ssh root@$node "systemctl restart kubelet"
done
```

Continue with [Post-Restore Validation](#8-post-restore-validation).

---

## 6. Restore When the Node Was Lost

Use this when the control plane VM or disk is gone and a new machine replaces it.

### Step 1: Prepare the new machine

- Same hostname (`cp-01`) and IP (10.0.1.10), or point `k8s-api.acme.local` at the new IP.
- Install the OS and prepare it exactly like the original: swap off, kernel modules, sysctl, containerd with `SystemdCgroup = true`.
- Install the **same** versions of kubeadm, kubelet, kubectl, etcdctl, and etcdutl.

### Step 2: Prepare the backup

Follow [Section 4](#4-prepare-the-backup) on the new machine.

### Step 3: Restore the PKI

The original CA and service account keys let existing workers and tokens keep working without rejoining.

```bash
mkdir -p /etc/kubernetes
cp -a /restore/extracted/etc/kubernetes/pki /etc/kubernetes/
ls -l /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/sa.key
```

### Step 4: Restore the etcd data

```bash
etcdutl snapshot restore $SNAP --data-dir /var/lib/etcd
```

### Step 5: Rebuild the control plane with kubeadm

Extract the `ClusterConfiguration` from the saved ConfigMap:

```bash
yq '.data.ClusterConfiguration' \
  /restore/extracted/backup/kubeadm-config-2026-10-04-0600.yaml > /root/kubeadm-config.yaml
```

Initialize, reusing the existing PKI and etcd data:

```bash
kubeadm init --config /root/kubeadm-config.yaml \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

kubeadm detects the existing certificates and keeps them.

### Step 6: Configure kubectl

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

### Step 7: Refresh components

```bash
kubectl -n kube-system delete pod -l component=kube-scheduler
kubectl -n kube-system delete pod -l component=kube-controller-manager
```

Workers reconnect automatically because the CA and endpoint are unchanged. Continue with [Post-Restore Validation](#8-post-restore-validation).

---

## 7. Restore a Multi-Member Cluster

Use this when quorum is permanently lost. Every member must be restored from the **same** snapshot.

> If only one member is lost and quorum is intact, do not restore. Replace the member instead:
>
> ```bash
> etcdctl member list -w table
> etcdctl member remove <member-id>
> kubeadm join k8s-api.acme.local:6443 --control-plane ...
> ```

### Step 1: Copy the snapshot to all control plane nodes

```bash
for node in cp-02 cp-03; do
  scp $SNAP root@$node:/restore/
done
```

### Step 2: Stop the API server and etcd on all nodes

Run on `cp-01`, `cp-02`, and `cp-03`:

```bash
mkdir -p /root/manifests-bak
mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/manifests-bak/
mv /etc/kubernetes/manifests/etcd.yaml /root/manifests-bak/
mv /var/lib/etcd /var/lib/etcd-old-$(date +%F-%H%M)
```

### Step 3: Restore on each node with its own identity

Each member needs its own `--name` and `--initial-advertise-peer-urls`, and the same `--initial-cluster` list. The names must match the `--name` value in each node's `etcd.yaml`.

**cp-01:**

```bash
etcdutl snapshot restore /restore/etcd-2026-10-04-0600.db \
  --name cp-01 \
  --initial-cluster cp-01=https://10.0.1.10:2380,cp-02=https://10.0.1.11:2380,cp-03=https://10.0.1.12:2380 \
  --initial-cluster-token etcd-cluster-restore-1 \
  --initial-advertise-peer-urls https://10.0.1.10:2380 \
  --data-dir /var/lib/etcd
```

**cp-02:**

```bash
etcdutl snapshot restore /restore/etcd-2026-10-04-0600.db \
  --name cp-02 \
  --initial-cluster cp-01=https://10.0.1.10:2380,cp-02=https://10.0.1.11:2380,cp-03=https://10.0.1.12:2380 \
  --initial-cluster-token etcd-cluster-restore-1 \
  --initial-advertise-peer-urls https://10.0.1.11:2380 \
  --data-dir /var/lib/etcd
```

**cp-03:**

```bash
etcdutl snapshot restore /restore/etcd-2026-10-04-0600.db \
  --name cp-03 \
  --initial-cluster cp-01=https://10.0.1.10:2380,cp-02=https://10.0.1.11:2380,cp-03=https://10.0.1.12:2380 \
  --initial-cluster-token etcd-cluster-restore-1 \
  --initial-advertise-peer-urls https://10.0.1.12:2380 \
  --data-dir /var/lib/etcd
```

### Step 4: Start etcd on all nodes

Run on all three nodes, close together in time so the members can form quorum:

```bash
mv /root/manifests-bak/etcd.yaml /etc/kubernetes/manifests/
```

Check the cluster:

```bash
ETCDCTL_API=3 etcdctl endpoint status --cluster -w table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

One member should show `IS LEADER: true`.

### Step 5: Start the API server on all nodes

```bash
mv /root/manifests-bak/kube-apiserver.yaml /etc/kubernetes/manifests/
```

### Step 6: Refresh components

```bash
kubectl -n kube-system delete pod -l component=kube-scheduler
kubectl -n kube-system delete pod -l component=kube-controller-manager
for node in cp-01 cp-02 cp-03 worker-01 worker-02 worker-03; do
  ssh root@$node "systemctl restart kubelet"
done
```

---

## 8. Post-Restore Validation

### Cluster health

```bash
kubectl get --raw='/readyz?verbose'
kubectl get nodes -o wide
kubectl get pods -A | grep -vE 'Running|Completed'
```

### Workloads

```bash
kubectl get deploy,sts,ds -A
kubectl get svc,ingress -A
kubectl get pvc -A
```

### Authentication and tokens

```bash
kubectl auth can-i --list
kubectl -n argocd get pods
```

Pods that call the API, such as ArgoCD or operators, should not show `401 Unauthorized` in their logs. If they do, the PKI was not restored correctly.

### Re-sync changes made after the snapshot

```bash
argocd app list
argocd app sync --all
```

### Clean up

Once everything is confirmed healthy:

```bash
rm -rf /restore /var/lib/etcd-old-*
rm -f /root/.etcd-backup-pass   # only if it was fetched just for this restore
```

### Restore checklist

| Check | Expected |
|---|---|
| `etcdctl endpoint health` | All members healthy |
| `kubectl get nodes` | All nodes `Ready` |
| System pods in `kube-system` | Running |
| Application pods | Running, matching Git state after sync |
| Service account tokens | No `401` errors |
| Backup job | Next scheduled run succeeds |

---

## 9. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `bad decrypt` | Wrong passphrase | Fetch the correct passphrase from Vault |
| `data-dir "/var/lib/etcd" exists` | Old data not moved | Move `/var/lib/etcd` aside first |
| etcd starts but API server cannot connect | Wrong certs or etcd not healthy | Check `crictl logs` for etcd and `endpoint health` |
| `member ... has already been bootstrapped` | Mixed old and new data across members | Restore every member from the same snapshot with the same `--initial-cluster-token` |
| Nodes `NotReady` with `x509: unknown authority` | New CA generated instead of restored | Restore the original `/etc/kubernetes/pki` |
| Pods get `401 Unauthorized` | Service account keys changed | Restore original `sa.key` and `sa.pub`, restart affected pods |
| Deleted objects reappear or new ones missing | Expected, cluster is at snapshot time | Re-sync from GitOps |

Debug static pods when `kubectl` is unavailable:

```bash
crictl ps -a | grep -E 'etcd|kube-apiserver'
crictl logs <container-id>
journalctl -u kubelet -n 100 --no-pager
```

---

Previous: [How to Back Up etcd](etcd-backup-kubeadm-baremetal.md)
