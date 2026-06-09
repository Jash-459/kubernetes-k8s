# ☸️ Kubernetes-k8s

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![kubeadm](https://img.shields.io/badge/kubeadm-Setup-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen?style=for-the-badge)

> A comprehensive, production-ready reference repository for setting up, configuring, and managing Kubernetes clusters — covering everything from bootstrapping with kubeadm to manual hard-way setups across single-master and multi-master topologies.


> 🤝 Let's Connect & Collaborate
>Whether you're an engineer building production-grade clusters, a student exploring Kubernetes for the first time, or simply someone who stumbled upon this repo and found it useful — you are most welcome here.

>This project is as much about community as it is about Kubernetes. If you:
     Want to contribute a guide, fix, or improvement
     Have a suggestion or a better way to do something
     Spotted an error or outdated information
     Are learning and want to discuss concepts or ask questions
     Just want to connect with someone who shares a passion for cloud-native tech

>...then don't hesitate to reach out. Open an issue, start a discussion, or just drop a message. Every idea, no matter how small, is valued.

> "Great infrastructure is never built alone — it's built by communities."

---

## 📌 Table of Contents

- [About](#about)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Setup Guides](#setup-guides)
  - [Cluster Setup (PDF Reference)](#cluster-setup-pdf-reference)
  - [Kubeadm Setup](#kubeadm-setup)
  - [Hard Way Setup](#hard-way-setup)
    - [Single-Master](#single-master)
    - [Multi-Master](#multi-master)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Useful References](#useful-references)
- [Contributing](#contributing)
- [License](#license)

---

## About

This repository serves as a single source of truth for all things Kubernetes — cluster bootstrapping, upgrade procedures, architecture references, and configuration files. Whether you're provisioning a quick dev cluster with kubeadm or building a highly available multi-master production setup the hard way, you'll find structured guides and supporting assets here.

---

## Repository Structure

```
Kubernetes-k8s/
│
├── README.md                          # You are here
├── Cluster Setup.pdf                  # Comprehensive cluster setup reference (PDF)
│
├── Kubeadm-Setup/
│   └── kubeadm-cluster.md             # Step-by-step kubeadm cluster setup guide
│
└── HardWay Setup/
    ├── Single-Master/
    │   ├── Single-master-setup.md     # Full single-master hard-way setup guide
    │   └── Architecture.png           # Architecture diagram for single-master topology
    │
    └── Multi-Master/
        └── (coming soon)              # Multi-master HA setup — work in progress
```

---

## Getting Started

Clone the repository and navigate to the setup guide relevant to your use case:

```bash
git clone https://github.com/Jash-459/Kubernetes-k8s.git
```

Choose your setup path:

| Setup Method | Complexity | Best For |
|---|---|---|
| [Kubeadm Setup](./Kubeadm-Setup/kubeadm-cluster.md) | ⭐⭐ Medium | Dev/staging clusters, quick provisioning |
| [Single-Master Hard Way](./HardWay%20Setup/Single-Master/Single-master-setup.md) | ⭐⭐⭐ High | Learning internals, custom PKI, deep control |
| Multi-Master Hard Way | ⭐⭐⭐⭐ Advanced | Production HA clusters *(coming soon)* |

---

## Setup Guides

### Cluster Setup (PDF Reference)

📄 [`Cluster Setup.pdf`](./Cluster-Setup.pdf)

A detailed PDF walkthrough covering end-to-end cluster provisioning — includes pre-requisites, network configuration, runtime setup, and post-install validation steps. Recommended as a companion to the markdown guides.

---

### Kubeadm Setup

📁 [`Kubeadm-Setup/kubeadm-cluster.md`](./Kubeadm-Setup/kubeadm-cluster.md)

The fastest way to get a working Kubernetes cluster up and running. This guide covers:

- System prerequisites and OS configuration
- Container runtime installation (containerd / CRI-O)
- Installing `kubeadm`, `kubelet`, and `kubectl`
- Initializing the control plane with `kubeadm init`
- Joining worker nodes with `kubeadm join`
- Deploying a CNI plugin (Flannel / Calico / Weave)
- Verifying cluster health

---

### Hard Way Setup

These guides were born out of pure curiosity — no vlogs, no blogs, no courses, no tutorials. Just raw exploration, trial and error, and a stubborn desire to understand how Kubernetes truly works under the hood. Every component here was figured out independently, the hard way, the real way.

#### Single-Master

📁 [`HardWay-Setup/Single-Master/Single-master-setup.md`](./HardWay-Setup/Single-Master/Single-master-setup.md)

Covers provisioning a fully functional single-master Kubernetes cluster from scratch:

- Certificate Authority (CA) setup and PKI generation
- etcd cluster configuration
- kube-apiserver, kube-controller-manager, kube-scheduler setup
- kubelet and kube-proxy configuration on worker nodes
- kubectl remote access configuration
- DNS add-on deployment

🖼️ See [`Architecture.png`](./HardWay-Setup/Single-Master/Architecture.png) for the node topology and component layout.

#### Multi-Master

📁 `HardWay Setup/Multi-Master/` — **Work in Progress**

Will cover High Availability (HA) control plane setup with:

- Multiple control plane nodes with load balancer fronting kube-apiserver
- Stacked etcd topology
- Failover and leader election behavior
- HAProxy / keepalived configuration

---

## Architecture

The diagram below (from [`HardWay-Setup/Multi-Master/Architecture.png`](./HardWay-Setup/Multi-Master/Architecture.png)) illustrates the single-master cluster topology, including the relationships between the control plane components, etcd, and worker nodes.

![Architecture](./HardWay-Setup/Multi-Master/Architecture.png)

---

## Prerequisites

Before using any setup guide in this repo, ensure your nodes meet the following baseline requirements:

| Requirement | Detail |
|---|---|
| OS | Ubuntu 20.04 / 22.04 LTS or CentOS/RHEL 8+ |
| CPU | 2+ vCPUs (control plane), 1+ vCPU (workers) |
| RAM | 2 GB+ (control plane), 1 GB+ (workers) |
| Disk | 20 GB+ per node |
| Network | Full network connectivity between all nodes |
| Swap | Disabled (`swapoff -a`) |
| Ports | See [Kubernetes Required Ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) |

---

## Useful References

| Resource | Link |
|---|---|
| Official Kubernetes Docs | [kubernetes.io/docs](https://kubernetes.io/docs/home/) |
| Kubernetes The Hard Way | [github.com/kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way) |
| kubeadm Install Guide | [kubernetes.io/docs/setup/production-environment/tools/kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) |
| Kubernetes Release Notes | [kubernetes.io/releases](https://kubernetes.io/releases/) |
| Kubernetes Networking (CNI) | [kubernetes.io/docs/concepts/cluster-administration/networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) |
| etcd Documentation | [etcd.io/docs](https://etcd.io/docs/) |
| containerd Runtime | [containerd.io/docs](https://containerd.io/docs/) |
| Kubernetes API Reference | [kubernetes.io/docs/reference/kubernetes-api](https://kubernetes.io/docs/reference/kubernetes-api/) |

---

## Contributing

Contributions, corrections, and additions are welcome! If you have a new setup guide, upgrade procedure, or configuration tip to add:

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/my-guide`
3. Commit your changes: `git commit -m "Add: multi-master kubeadm HA guide"`
4. Push to the branch: `git push origin feature/my-guide`
5. Open a Pull Request

Please follow the existing folder structure and markdown formatting conventions.

---

## License

This repository is open source project. Feel free to use, adapt, and share.

---

<div align="center">
  <sub>Maintained with ❤️ for the Kubernetes community</sub>
</div>