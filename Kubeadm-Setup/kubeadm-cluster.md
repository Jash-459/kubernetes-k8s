# ☸️ Kubernetes Cluster Setup on On-Premise Servers

> **On-premise Kubernetes v1.33 installation guide using containerd as CRI and Calico/Cilium as CNI — tested on Ubuntu 24.04.**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1 — System Preparation (All Nodes)](#step-1--system-preparation-all-nodes)
- [Step 2 — Configure Kernel Parameters (All Nodes)](#step-2--configure-kernel-parameters-all-nodes)
- [Step 3 — Install containerd (All Nodes)](#step-3--install-containerd-all-nodes)
- [Step 4 — Install Kubernetes Components (All Nodes)](#step-4--install-kubernetes-components-all-nodes)
- [Step 5 — Initialize the Cluster (Master Node)](#step-5--initialize-the-cluster-master-node)
- [Step 6 — Join Worker Nodes](#step-6--join-worker-nodes)
- [Step 7 — Install Network Plugin (Master Node)](#step-7--install-network-plugin-master-node)
- [Step 8 — Verify the Cluster](#step-8--verify-the-cluster)
- [Step 9 — Hubble UI (Cilium only)](#step-9--hubble-ui-cilium-only)

---

## Overview

| Field | Value |
|---|---|
| **Author** | Jash Hamirani |
| **Date** | 22/08/2025 |
| **OS** | Ubuntu 24.04 |
| **Kubernetes Version** | v1.33 |
| **CRI** | containerd |
| **CNI** | Calico / Cilium |

This guide walks through setting up a multi-node Kubernetes cluster on bare-metal / on-premise Ubuntu servers — one **master (control plane)** node and one or more **worker** nodes.

---

## Prerequisites

- Minimum **2 Ubuntu 24.04** instances (1 master + 1 or more workers)
- SSH access with `sudo` privileges on all nodes
- Each node must meet:
  - ≥ 2 GB RAM
  - ≥ 2 vCPUs
  - ≥ 20 GB free disk space
- Network connectivity between all nodes

---

## Step 1 — System Preparation (All Nodes)

### 1.1 Update system packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Disable swap

Kubernetes requires swap to be off for stable scheduling.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 1.3 Configure hostnames

Run the appropriate command on each node:

```bash
# On master node
sudo hostnamectl set-hostname master

# On worker node
sudo hostnamectl set-hostname server1
```

Update `/etc/hosts` on **all nodes** with the private IPs and hostnames of every node in the cluster (for hostname resolution):

```
<master-private-ip>   master
<worker-private-ip>   server1
```

### 1.4 Verify hostname resolution

```bash
hostname

# Or test ping from worker to master and vice versa
ping -c 3 master
```

### 1.5 Install dependencies

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

---

## Step 2 — Configure Kernel Parameters (All Nodes)

### 2.1 Load required kernel modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Make them persistent across reboots:

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

### 2.2 Configure IPv4 networking for Kubernetes

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the settings:

```bash
sudo sysctl --system
```

---

## Step 3 — Install containerd (All Nodes)

### 3.1 Add the Docker repository

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg

sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### 3.2 Install containerd

```bash
sudo apt update
sudo apt install -y containerd.io
```

### 3.3 Configure containerd to use systemd as cgroup driver

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

### 3.4 Enable and restart containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 4 — Install Kubernetes Components (All Nodes)

### 4.1 Add the Kubernetes GPG key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 4.2 Add the Kubernetes repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 4.3 Install kubeadm, kubelet, and kubectl

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
```

---

## Step 5 — Initialize the Cluster (Master Node)

### 5.1 Initialize the control plane

```bash
sudo kubeadm init --pod-network-cidr=10.10.0.0/16
```

> The `--pod-network-cidr` defines the internal pod network range. Adjust if it conflicts with your existing network.

### 5.2 Set up kubeconfig

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 6 — Join Worker Nodes

After `kubeadm init` completes on the master, it will output a `kubeadm join` command. Run it on each worker node:

```bash
kubeadm join <master-ip>:6443 \
  --token <your-token> \
  --discovery-token-ca-cert-hash <your-hash>
```

### Verify nodes from master

```bash
kubectl get nodes
```

---

## Step 7 — Install Network Plugin (Master Node)

Choose **one** of the following CNI plugins:

### Option A — Calico

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
```

### Option B — Cilium (via Helm)

```bash
helm upgrade cilium cilium/cilium --version 1.18.1 \
  --namespace kube-system \
  --reuse-values \
  --set hubble.relay.enabled=true
```

---

## Step 8 — Verify the Cluster

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

All nodes should show `Ready` and all system pods should be `Running`.

---

## Step 9 — Hubble UI (Cilium only)

Hubble provides a visual network observability dashboard for Cilium.

### 9.1 Enable Hubble

```bash
# If installed via CLI
cilium hubble enable

# If installed via Helm
helm upgrade cilium cilium/cilium --version 1.18.1 \
  --namespace kube-system \
  --reuse-values \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

### 9.2 Check Cilium status

```bash
cilium status
```

### 9.3 Expose Hubble UI via NodePort

```bash
kubectl patch svc hubble-ui -n kube-system \
  -p '{"spec": {"type": "NodePort"}}'
```

### 9.4 Get the NodePort

```bash
kubectl get svc -n kube-system hubble-ui
```

### 9.5 Access the UI

Open in your browser:

```
http://<node-public-ip>:<nodeport>
```

---

## 📎 Quick Reference

| Command | Purpose |
|---|---|
| `kubectl get nodes` | List all cluster nodes |
| `kubectl get pods -n kube-system` | Check system pods |
| `cilium status` | Check Cilium health |
| `kubectl get svc -n kube-system hubble-ui` | Get Hubble UI NodePort |

---

## 📄 License

This documentation is provided for educational and internal use.