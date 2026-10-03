# How to Back Up etcd on a Bare Metal kubeadm Kubernetes Cluster

This guide covers taking a manual etcd backup on a kubeadm cluster running on bare metal, then automating it with a cron job, off-cluster storage, encryption, and monitoring.

---

## Table of Contents

1. [Environment](#1-environment)
2. [What Gets Backed Up and Why](#2-what-gets-backed-up-and-why)
3. [Prerequisites](#3-prerequisites)
4. [Manual Backup](#4-manual-backup)
5. [Automated Backup](#5-automated-backup)
6. [Monitoring and Alerting](#6-monitoring-and-alerting)
7. [Best Practices](#7-best-practices)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Environment

| Item | Value |
|---|---|
| Cluster type | kubeadm, bare metal |
| Control plane nodes | `cp-01` (10.0.1.10), `cp-02` (10.0.1.11), `cp-03` (10.0.1.12) |
| etcd topology | Stacked (static pod on each control plane node) |
| etcd data directory | `/var/lib/etcd` |
| etcd certificates | `/etc/kubernetes/pki/etcd/` |
| Local backup directory | `/backup` |
| Off-cluster storage | MinIO bucket `etcd-backups` at `https://minio.acme.local:9000` |
| Schedule | Every 6 hours |
| Retention | 14 days in MinIO, last 3 copies locally |

> In a multi-member cluster, a snapshot from **one healthy member** is enough. All members hold the same replicated data.

---

## 2. What Gets Backed Up and Why

| Item | Path | Why it is needed |
|---|---|---|
| etcd snapshot | `etcd-<timestamp>.db` | Full cluster state: Deployments, Pods, Secrets, ConfigMaps, RBAC, CRDs |
| PKI directory | `/etc/kubernetes/pki` | Cluster CA, etcd CA, service account signing keys. Without the same PKI, worker kubelets stop trusting the API server and all service account tokens become invalid |
| kubeadm config | `kubeadm-config` ConfigMap | Cluster settings needed to rebuild a control plane node |

---

## 3. Prerequisites

Log in to a control plane node as root:

```bash
ssh admin@cp-01
sudo -i
```

Confirm `etcdctl` and `etcdutl` are installed:

```bash
etcdctl version
etcdutl version
```

If they are missing, install the binaries matching your etcd version:

```bash
ETCD_VER=v3.5.15
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
tar xzf /tmp/etcd.tar.gz -C /tmp
mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /tmp/etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/
```

Install the MinIO client and `jq`:

```bash
curl -L https://dl.min.io/client/mc/release/linux-amd64/mc -o /usr/local/bin/mc
chmod +x /usr/local/bin/mc
apt install -y jq
```

---

## 4. Manual Backup

### Step 1: Find the etcd endpoint and certificate paths

etcd uses mutual TLS, so the client must present the correct CA, certificate, and key.

```bash
grep -E "listen-client-urls|cert-file|key-file|trusted-ca-file" /etc/kubernetes/manifests/etcd.yaml
```

Expected output:

```text
--listen-client-urls=https://127.0.0.1:2379,https://10.0.1.10:2379
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

### Step 2: Check etcd health

Never back up an unhealthy member.

```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Expected output:

```text
https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 3.1ms
```

### Step 3: Create the backup directory

```bash
mkdir -p /backup
chmod 700 /backup
TS=$(date +%F-%H%M)
```

### Step 4: Take the snapshot

`etcdctl` connects to the running etcd server over gRPC and streams a consistent, point-in-time copy of the database into a local file. It does not copy raw files from disk.

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$TS.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Expected output:

```text
Snapshot saved at /backup/etcd-2026-10-04-1030.db
```

| Flag | Purpose |
|---|---|
| `ETCDCTL_API=3` | Use the v3 API, which Kubernetes uses |
| `--endpoints` | etcd client address |
| `--cacert` | CA used to verify the etcd server |
| `--cert`, `--key` | Client certificate and key used to authenticate |

### Step 5: Verify the snapshot

```bash
etcdutl snapshot status /backup/etcd-$TS.db -w table
```

Expected output:

```text
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 7ef84a1c |   458231 |       1342 |     5.1 MB |
+----------+----------+------------+------------+
```

A very low key count or an error here means the snapshot is not usable.

### Step 6: Export the kubeadm config

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system \
  get cm kubeadm-config -o yaml > /backup/kubeadm-config-$TS.yaml
```

### Step 7: Bundle the snapshot, PKI, and kubeadm config

`tar` packs everything into one compressed file and preserves file permissions, which matters for private keys.

```bash
tar czf /backup/cp-01-$TS.tgz \
  /backup/etcd-$TS.db \
  /backup/kubeadm-config-$TS.yaml \
  /etc/kubernetes/pki
```

### Step 8: Encrypt the bundle

The archive contains every Secret and the cluster private keys, so encrypt it before it leaves the node.

Create the passphrase file once and store a copy in Vault or your password manager. Never store it in the same bucket as the backups.

```bash
openssl rand -base64 32 > /root/.etcd-backup-pass
chmod 600 /root/.etcd-backup-pass
```

Encrypt:

```bash
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in  /backup/cp-01-$TS.tgz \
  -out /backup/cp-01-$TS.tgz.enc \
  -pass file:/root/.etcd-backup-pass
```

### Step 9: Upload to off-cluster storage

Configure the MinIO alias once:

```bash
mc alias set minio https://minio.acme.local:9000 etcd-backup-sa '<secret-key>'
```

Upload:

```bash
mc cp /backup/cp-01-$TS.tgz.enc minio/etcd-backups/cp-01/
mc ls minio/etcd-backups/cp-01/
```

For AWS S3 instead of MinIO:

```bash
aws s3 cp /backup/cp-01-$TS.tgz.enc s3://acme-etcd-backups/cp-01/ --sse aws:kms
```

### Step 10: Clean up plain-text files

```bash
rm -f /backup/etcd-$TS.db /backup/kubeadm-config-$TS.yaml /backup/cp-01-$TS.tgz
```

---

## 5. Automated Backup

### Step 1: Create the backup script

```bash
cat > /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

TS=$(date +%F-%H%M)
HOST=$(hostname -s)
DIR=/backup
PKI=/etc/kubernetes/pki/etcd
SNAP=$DIR/etcd-$TS.db
CONFIG=$DIR/kubeadm-config-$TS.yaml
ARCHIVE=$DIR/$HOST-$TS.tgz
METRIC_DIR=/var/lib/node_exporter/textfile
METRIC=$METRIC_DIR/etcd_backup.prom

ETCD_FLAGS="--endpoints=https://127.0.0.1:2379 \
  --cacert=$PKI/ca.crt --cert=$PKI/server.crt --key=$PKI/server.key"

mkdir -p "$DIR" "$METRIC_DIR"

# Run only on the current etcd leader so exactly one node backs up
IS_LEADER=$(ETCDCTL_API=3 etcdctl $ETCD_FLAGS endpoint status -w json | jq -r '.[0].Status.leader == .[0].Status.header.member_id')
if [ "$IS_LEADER" != "true" ]; then
  echo "$(date) Not the etcd leader, skipping."
  exit 0
fi

# 1. Snapshot
ETCDCTL_API=3 etcdctl $ETCD_FLAGS snapshot save "$SNAP"

# 2. Verify
KEYS=$(etcdutl snapshot status "$SNAP" -w json | jq '.totalKey')
if [ "$KEYS" -lt 100 ]; then
  echo "$(date) ERROR: snapshot looks invalid, keys=$KEYS" >&2
  exit 1
fi

# 3. kubeadm config
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system \
  get cm kubeadm-config -o yaml > "$CONFIG"

# 4. Bundle
tar czf "$ARCHIVE" "$SNAP" "$CONFIG" /etc/kubernetes/pki

# 5. Encrypt
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in "$ARCHIVE" -out "$ARCHIVE.enc" \
  -pass file:/root/.etcd-backup-pass

# 6. Upload
mc cp "$ARCHIVE.enc" "minio/etcd-backups/$HOST/"

# 7. Local cleanup: remove plain files, keep last 3 encrypted copies
rm -f "$SNAP" "$CONFIG" "$ARCHIVE"
ls -1t "$DIR"/*.enc | tail -n +4 | xargs -r rm -f

# 8. Success metric for Prometheus
echo "etcd_backup_last_success_timestamp_seconds $(date +%s)" > "$METRIC.tmp"
mv "$METRIC.tmp" "$METRIC"

echo "$(date) Backup OK: $ARCHIVE.enc keys=$KEYS"
EOF

chmod 700 /usr/local/bin/etcd-backup.sh
```

> `set -euo pipefail` makes the script stop on any failure, so the success metric is never written for a broken backup.

### Step 2: Deploy to all control plane nodes

Install the script, passphrase file, and MinIO alias on `cp-01`, `cp-02`, and `cp-03`. The leader check ensures only one node uploads each run, and backups keep working if any single node is down.

```bash
for node in cp-02 cp-03; do
  scp /usr/local/bin/etcd-backup.sh root@$node:/usr/local/bin/
  scp /root/.etcd-backup-pass root@$node:/root/
  ssh root@$node "chmod 700 /usr/local/bin/etcd-backup.sh && chmod 600 /root/.etcd-backup-pass"
done
```

### Step 3: Schedule with cron

```bash
cat > /etc/cron.d/etcd-backup <<'EOF'
0 */6 * * * root /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1
EOF
```

### Step 4: Configure retention in MinIO

```bash
mc ilm rule add minio/etcd-backups --expire-days 14
mc ilm rule ls minio/etcd-backups
```

### Step 5: Restrict bucket access

```bash
cat > /tmp/etcd-backup-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::etcd-backups", "arn:aws:s3:::etcd-backups/*"]
  }]
}
EOF

mc admin policy create minio etcd-backup-policy /tmp/etcd-backup-policy.json
mc admin policy attach minio etcd-backup-policy --user etcd-backup-sa
```

### Step 6: Test the job

```bash
/usr/local/bin/etcd-backup.sh
tail -n 20 /var/log/etcd-backup.log
mc ls minio/etcd-backups/cp-01/
cat /var/lib/node_exporter/textfile/etcd_backup.prom
```

---

## 6. Monitoring and Alerting

Run node_exporter with the textfile collector enabled:

```bash
--collector.textfile.directory=/var/lib/node_exporter/textfile
```

Prometheus alert rule. Fires if no successful backup in 8 hours (one missed run plus buffer):

```yaml
groups:
- name: etcd-backup
  rules:
  - alert: EtcdBackupStale
    expr: time() - max(etcd_backup_last_success_timestamp_seconds) > 8 * 3600
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "No successful etcd backup in the last 8 hours"
```

Using `max()` across all control plane nodes means the alert only fires when no node has produced a backup.

---

## 7. Best Practices

| Practice | Reason |
|---|---|
| Back up the PKI with every snapshot | A snapshot without the matching CA and service account keys cannot be cleanly restored |
| Encrypt before upload | Snapshots contain every Secret and the PKI contains private keys |
| Store the passphrase outside the cluster | If the control plane node is lost, a passphrase stored only on that node makes every backup useless |
| Keep backups off the node | A disk failure or VM loss would destroy local copies |
| Enable etcd encryption at rest for Secrets | Adds a second layer of protection inside the snapshot |
| Enable bucket versioning | Protects against accidental overwrite or deletion |
| Alert on stale backups | Silent cron failures are the most common backup failure |
| Test restores quarterly | A backup that has never been restored is not a backup |

---

## 8. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `context deadline exceeded` | Wrong endpoint or etcd unhealthy | Check `--endpoints`, run `endpoint health` |
| `x509: certificate signed by unknown authority` | Wrong CA file | Use `/etc/kubernetes/pki/etcd/ca.crt` |
| `tls: bad certificate` | Wrong client cert or key | Use the etcd `server.crt` / `server.key` or a dedicated client cert |
| `etcdctl: command not found` | Binary not installed | Install from the etcd release matching your version |
| Upload fails | MinIO alias or credentials wrong | `mc alias list`, recheck the access key |
| Alert fires but script exits 0 | Every node skipped the leader check | Check `endpoint status` output and cluster leader |

---

Next: [How to Restore an etcd Backup](etcd-restore-kubeadm-baremetal.md)
