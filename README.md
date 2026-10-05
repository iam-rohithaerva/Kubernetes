# Kubernetes

Operational runbooks for Kubernetes clusters.

## Cluster setup

| Guide | Description |
|---|---|
| [How to Build a Self-Managed Cluster with kubeadm](kubeadm-cluster-setup-baremetal.md) | Building a kubeadm cluster on physical servers, with the reason for each step and what breaks if you skip it |

## etcd

| Guide | Description |
|---|---|
| [How to Back Up etcd](etcd-backup-kubeadm-baremetal.md) | Manual and automated etcd backups on a bare metal kubeadm cluster, with encryption, off-cluster storage, and monitoring |
| [How to Restore an etcd Backup](etcd-restore-kubeadm-baremetal.md) | Restoring etcd on a single control plane, after losing a node, and after quorum loss in a multi-member cluster |

## Add-ons

| Guide | Description |
|---|---|
| [How to Install Istio and Secure, Route, and Observe Service Traffic](addons/istio/README.md) | Istio service mesh on Kubernetes: mTLS, zero trust AuthorizationPolicy, canary releases, ingress gateway, and tracing, with manifests and verification commands |
