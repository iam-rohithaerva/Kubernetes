# How to Build a Self-Managed Kubernetes Cluster on Bare Metal with kubeadm

This guide builds a Kubernetes cluster on physical servers with kubeadm: one control plane node and two worker nodes, containerd as the container runtime, and Calico as the pod network. For every step it explains why the step exists, what breaks if you skip it, and the commands to run.

---

## Table of Contents

1. [Environment](#1-environment)
2. [Why kubeadm on Bare Metal](#2-why-kubeadm-on-bare-metal)
3. [Overview of the Steps](#3-overview-of-the-steps)
4. [Step 1: Hostnames and Name Resolution](#4-step-1-hostnames-and-name-resolution)
5. [Step 2: Disable Swap](#5-step-2-disable-swap)
6. [Step 3: Kernel Modules and Network Settings](#6-step-3-kernel-modules-and-network-settings)
7. [Step 4: Install and Configure containerd](#7-step-4-install-and-configure-containerd)
8. [Step 5: Install kubeadm, kubelet, and kubectl](#8-step-5-install-kubeadm-kubelet-and-kubectl)
9. [Step 6: Initialize the Control Plane](#9-step-6-initialize-the-control-plane)
10. [Step 7: Install the Pod Network (Calico)](#10-step-7-install-the-pod-network-calico)
11. [Step 8: Join the Worker Nodes](#11-step-8-join-the-worker-nodes)
12. [Step 9: Verify the Cluster](#12-step-9-verify-the-cluster)
13. [Production Checklist](#13-production-checklist)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Environment

| Item | Value |
|---|---|
| Servers | 3 physical servers, Ubuntu 24.04 LTS |
| Control plane node | `cp-01` (10.0.1.10) |
| Worker nodes | `wk-01` (10.0.1.21), `wk-02` (10.0.1.22) |
| Kubernetes version | v1.31 |
| Container runtime | containerd, systemd cgroup driver |
| Pod network (CNI) | Calico v3.28 |
| Pod CIDR | `192.168.0.0/16` (must not overlap with the server LAN) |
| Service CIDR | `10.96.0.0/12` (kubeadm default) |

Minimum hardware:

| Role | CPU | RAM | Disk |
|---|---|---|---|
| Control plane | 2 cores | 2 GB | 50 GB (SSD recommended for etcd) |
| Worker | 1 core | 2 GB | 50 GB |

Ports that must be open between the servers:

| Node | Ports | Used by |
|---|---|---|
| Control plane | 6443 | Kubernetes API server |
| Control plane | 2379-2380 | etcd |
| Control plane | 10250, 10257, 10259 | kubelet, controller-manager, scheduler |
| Workers | 10250 | kubelet |
| Workers | 30000-32767 | NodePort Services |

---

## 2. Why kubeadm on Bare Metal

With a managed service such as Amazon EKS, the cloud provider runs the control plane for you. On physical servers there is no provider, so you build and operate the control plane yourself. kubeadm is the official Kubernetes tool for this: it generates certificates, starts etcd and the control plane components, and joins nodes securely.

| | Managed (EKS) | Self-managed (kubeadm) |
|---|---|---|
| Control plane | Run by AWS | Run by you |
| Upgrades | A few clicks | You upgrade each node in order |
| etcd backups | Handled by AWS | Your responsibility |
| Certificates | Handled by AWS | Expire after 1 year unless renewed |
| LoadBalancer Services | AWS load balancers | You add MetalLB |
| Typical reasons to choose | Speed, less operations work | Data residency, existing hardware, air-gapped sites, cost |

---

## 3. Overview of the Steps

| Step | What | Where |
|---|---|---|
| 1 | Hostnames and name resolution | All nodes |
| 2 | Disable swap | All nodes |
| 3 | Kernel modules and network settings | All nodes |
| 4 | Install and configure containerd | All nodes |
| 5 | Install kubeadm, kubelet, kubectl | All nodes (kubectl is optional on workers) |
| 6 | Initialize the control plane | `cp-01` only |
| 7 | Install the pod network (Calico) | `cp-01` only |
| 8 | Join the workers | `wk-01`, `wk-02` |
| 9 | Verify | `cp-01` |

Run every command as a user with `sudo`.

---

## 4. Step 1: Hostnames and Name Resolution

**Why:** Kubernetes identifies each node by its hostname. The kubelet registers the node under that name, and it also resolves its own hostname to decide which IP address to report. Nodes still talk to each other over IP addresses; the hostname is the node's identity.

**Without it:**
- Servers installed from the same image often share a hostname such as `ubuntu`. The second node then fails to join (`a Node with name "ubuntu" already exists`) or overwrites the first node's Node object.
- On servers with more than one network card, the kubelet can report the wrong IP. `kubectl logs` and `kubectl exec` then hang because the API server cannot reach that IP.
- kubeadm preflight prints `hostname "cp-01" could not be reached`.

**With it:** every node appears under a clear, unique name, and the kubelet reports the correct LAN IP.

Set the hostname on each server (use its own name):

```bash
sudo hostnamectl set-hostname cp-01     # wk-01 and wk-02 on the workers
```

Add all nodes to `/etc/hosts` on every server. If your company DNS already has these records, you can skip this.

```bash
cat <<EOF | sudo tee -a /etc/hosts
10.0.1.10 cp-01
10.0.1.21 wk-01
10.0.1.22 wk-02
EOF
```

> Set hostnames **before** `kubeadm init` or `kubeadm join`. Changing a hostname afterwards means resetting and rejoining that node.

---

## 5. Step 2: Disable Swap

**Why:** The scheduler places pods based on real RAM, and pod memory limits are enforced against real RAM. Swap lets a process spill onto disk, which breaks that accounting.

**Without it:**
- kubeadm preflight fails with `[ERROR Swap]: running with swap on is not supported`, and the kubelet will not start.
- If you only run `swapoff` and do not edit `/etc/fstab`, swap comes back after the next reboot (for example after OS patching), the kubelet fails, and the node goes `NotReady`.

**With it:** a pod that exceeds its memory limit is OOMKilled and restarted, which is predictable and visible.

```bash
sudo swapoff -a                              # off now
sudo sed -i '/ swap / s/^/#/' /etc/fstab     # stays off after reboot
```

`/etc/fstab` is the file Linux reads at boot to decide which disks to mount and whether to enable swap. Commenting out the swap line keeps swap disabled permanently.

---

## 6. Step 3: Kernel Modules and Network Settings

Kernel modules are plugins that add features to the Linux kernel. Kubernetes needs two of them and three network settings.

| Setting | Why | Without it |
|---|---|---|
| `overlay` module | containerd stacks image layers (OS, runtime, app) into a single filesystem for each container, and containers share the read-only layers | Pods stay in `ContainerCreating` with `failed to mount overlay` |
| `br_netfilter` module + `net.bridge.bridge-nf-call-iptables = 1` | Pods on the same node talk through a Linux bridge, and bridge traffic skips iptables by default. Services are iptables NAT rules written by kube-proxy, so the reply must pass through iptables to be translated back | When a Service picks a pod on the **same** node, the reply is not translated and the caller drops it. Result: intermittent timeouts and random DNS failures |
| `net.bridge.bridge-nf-call-ip6tables = 1` | Same as above for IPv6 | IPv6 Service calls fail the same way on dual-stack clusters |
| `net.ipv4.ip_forward = 1` | The node must act as a router and forward pod traffic to other nodes and to the outside | Cross-node pod traffic and NodePort traffic is dropped; preflight fails |

Load the modules now and on every boot:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

Apply the network settings now and on every boot:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

Check:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

---

## 7. Step 4: Install and Configure containerd

**Why:** Kubernetes does not run containers itself. The kubelet asks a container runtime to pull images and start containers. Docker Engine support was removed in v1.24, so containerd is the standard choice.

**The cgroup driver:** cgroups are the Linux feature that enforces CPU and memory limits for each pod. A cgroup driver is the component that creates and manages them. There are two options that do the same job, `cgroupfs` and `systemd`, and you use only one. systemd already manages cgroups for every service on the server, and kubeadm configures the kubelet with `systemd`, so containerd must use `systemd` too.

**Without it:**
- No runtime: preflight fails and the cluster cannot be created.
- Driver mismatch (containerd's default is `cgroupfs`): two managers keep separate views of the same resources. Control plane pods such as etcd and kube-apiserver restart every few minutes, and `kubectl` intermittently returns `connection refused`.

**With it:** one cgroup manager, accurate resource limits, stable control plane pods.

```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Check:

```bash
grep SystemdCgroup /etc/containerd/config.toml     # SystemdCgroup = true
systemctl is-active containerd                     # active
```

---

## 8. Step 5: Install kubeadm, kubelet, and kubectl

| Tool | What it does | Needed on |
|---|---|---|
| kubeadm | Creates, joins, and upgrades the cluster | All nodes |
| kubelet | Agent on every node that runs the pods assigned to it | All nodes |
| kubectl | Command-line client for the API server | Only where you administer the cluster (control plane, laptop, bastion). Not needed on workers |

**Why hold the versions:** upgrades must follow an order (control plane first, then workers, one minor version at a time). If the OS package manager upgrades the kubelet during routine patching, it can become newer than the control plane and the node goes `NotReady`.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

> The kubelet restarts in a loop until `kubeadm init` or `kubeadm join` gives it a configuration. This is expected.

---

## 9. Step 6: Initialize the Control Plane

Run on `cp-01` only.

**Why:** this turns `cp-01` into the brain of the cluster.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.10 \
  --pod-network-cidr=192.168.0.0/16
```

What `kubeadm init` does:

| Phase | Result |
|---|---|
| Preflight | Checks swap, ports, runtime, kernel settings |
| Certificates | Creates the cluster CA and certificates in `/etc/kubernetes/pki` |
| kubeconfig files | Creates `admin.conf`, `kubelet.conf`, and others in `/etc/kubernetes` |
| Static pods | Writes manifests for etcd, kube-apiserver, kube-controller-manager, and kube-scheduler to `/etc/kubernetes/manifests`; the kubelet starts them |
| Bootstrap token | Creates the token that workers use to join |
| Add-ons | Installs CoreDNS and kube-proxy |

Set up `kubectl` for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Save the `kubeadm join ...` command printed at the end of the output.

**Expected state now:** `cp-01` is `NotReady` and CoreDNS is `Pending`. That is normal until the pod network is installed.

**Common mistake:** a pod CIDR that overlaps with the server LAN, or that differs from the CNI's range. Pods then get no IP or traffic is misrouted. Fixing it requires `kubeadm reset` and a new `init`.

---

## 10. Step 7: Install the Pod Network (Calico)

Run on `cp-01`.

**Why:** Kubernetes does not implement pod networking itself. A CNI plugin gives every pod an IP address, routes pod traffic between nodes, and (with Calico) enforces NetworkPolicy.

**Without it:** every node stays `NotReady`, CoreDNS stays `Pending`, and no application pod can start.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Calico's default pool is `192.168.0.0/16`, which matches the `--pod-network-cidr` above. If you chose a different CIDR, set `CALICO_IPV4POOL_CIDR` in the manifest to the same value before applying.

Wait until the node is `Ready`:

```bash
kubectl get nodes -w
```

---

## 11. Step 8: Join the Worker Nodes

Run on `wk-01` and `wk-02` the join command saved from Step 6:

```bash
sudo kubeadm join 10.0.1.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

| Part | Purpose |
|---|---|
| `10.0.1.10:6443` | Address of the API server |
| `--token` | Proves the worker is allowed to join |
| `--discovery-token-ca-cert-hash` | Lets the worker verify it is talking to the real control plane, not an impostor |

The token expires after 24 hours. Create a new join command on `cp-01` when needed:

```bash
kubeadm token create --print-join-command
```

**Without workers:** the control plane node has a `NoSchedule` taint by default, so application pods stay `Pending`.

---

## 12. Step 9: Verify the Cluster

On `cp-01`:

```bash
kubectl get nodes -o wide
```

```
NAME    STATUS   ROLES           VERSION   INTERNAL-IP
cp-01   Ready    control-plane   v1.31.x   10.0.1.10
wk-01   Ready    <none>          v1.31.x   10.0.1.21
wk-02   Ready    <none>          v1.31.x   10.0.1.22
```

```bash
kubectl get pods -n kube-system     # calico, coredns, etcd, kube-* all Running
```

Smoke test pod scheduling, cross-node networking, and NodePort:

```bash
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide            # pods spread across wk-01 and wk-02
kubectl get svc nginx               # note the NodePort, for example 31234
curl http://10.0.1.21:31234         # nginx welcome page
```

Test cluster DNS:

```bash
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default
```

Clean up:

```bash
kubectl delete svc,deployment nginx
```

---

## 13. Production Checklist

| Need | Solution | Risk if missing |
|---|---|---|
| Highly available control plane | 3 control plane nodes behind a virtual IP (HAProxy + Keepalived), created with `kubeadm init --control-plane-endpoint=<VIP>:6443 --upload-certs` | One server failure means you cannot manage the cluster |
| LoadBalancer Services | MetalLB | `EXTERNAL-IP` stays `Pending` forever |
| Persistent storage | Longhorn, Rook-Ceph, or an NFS CSI driver | Database pods stay `Pending` on their PVCs |
| Ingress | ingress-nginx | One NodePort per application |
| etcd backups | See [How to Back Up etcd](etcd-backup-kubeadm-baremetal.md) | Losing etcd loses the whole cluster state |
| Certificate renewal | `kubeadm certs check-expiration`; certificates expire after 1 year (upgrades renew them) | The API server stops working the day they expire |
| Upgrades | `kubeadm upgrade plan`, then `kubeadm upgrade apply`; one minor version at a time, control plane first | Version skew and `NotReady` nodes |
| Admin access | kubectl and `admin.conf` only on a bastion, not on workers | A compromised worker gives full cluster control |

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `[ERROR Swap]` during preflight | Swap is on | Step 2 |
| Node `NotReady` after a reboot | Swap re-enabled because `/etc/fstab` was not edited | Comment out the swap line in `/etc/fstab` |
| `[ERROR FileContent--proc-sys-net-ipv4-ip_forward]` | `ip_forward` not set | Step 3 |
| etcd or kube-apiserver in `CrashLoopBackOff`, frequent restarts | containerd uses `cgroupfs` while the kubelet uses `systemd` | Set `SystemdCgroup = true`, restart containerd and kubelet |
| Pods stuck in `ContainerCreating` with `failed to mount overlay` | `overlay` module not loaded | Step 3 |
| Service calls and DNS time out about half the time | `br_netfilter` not loaded or `bridge-nf-call-iptables` is 0 | Step 3 |
| All nodes `NotReady`, CoreDNS `Pending` | No CNI installed | Step 7 |
| Join fails with `a Node with name ... already exists` | Duplicate hostnames | Step 1, then `kubeadm reset` on the worker and rejoin |
| Join fails with an invalid or expired token | Token is older than 24 hours | `kubeadm token create --print-join-command` |
| Join times out | Port 6443 blocked between the servers | Open the firewall ports listed in [Environment](#1-environment) |
| `kubectl logs` or `exec` hangs on one node | Kubelet reported the wrong IP (multiple network cards) | Fix `/etc/hosts`, or set `--node-ip` for the kubelet |

To start a node over:

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
```
