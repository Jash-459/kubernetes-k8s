# ☸️ Kubernetes The Hard Way — Single Master Cluster Setup

> **A complete, from-scratch Kubernetes cluster built manually — no kubeadm, no magic, just raw commands and real understanding.**

---

## 📁 Repository Contents

| File | Description |
|------|-------------|
| `Single-master-setup.md` | Full step-by-step guide with all commands to bootstrap the cluster manually |
| `Architecture.png` | Cluster topology diagram showing node layout and request/traffic flow |
| `screenshots/` | Visual walkthrough of key deployment stages and verification steps |

---

## 🧭 Table of Contents

1. [What is Kubernetes?](#-what-is-kubernetes)
2. [What is "The Hard Way"?](#-what-is-kubernetes-the-hard-way)
3. [Why Learn It The Hard Way?](#-why-learn-it-the-hard-way)
4. [Kubernetes Architecture — Core Components](#-kubernetes-architecture--core-components)
5. [Types of Cluster Topologies](#-types-of-cluster-topologies)
6. [This Setup — Single Master Topology](#-this-setup--single-master-topology)
7. [Traffic & Request Flow](#-traffic--request-flow)
8. [Prerequisites](#-prerequisites)
9. [How to Use This Repo](#-how-to-use-this-repo)
10. [Key Concepts Glossary](#-key-concepts-glossary)
11. [Further Learning](#-further-learning)

---

## 🐳 What is Kubernetes?

Kubernetes (also written as **K8s**) is an open-source **container orchestration platform** originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).

In simple terms: when you run applications inside **containers** (think Docker), Kubernetes is the system that:

- **Deploys** your containers across multiple machines
- **Scales** them up or down based on demand
- **Heals** them automatically when they crash
- **Load balances** traffic between them
- **Manages secrets, configs, storage**, and networking

Think of Kubernetes as the **operating system for your cloud infrastructure** — it manages where and how your apps run, so you don't have to do it manually.

### 🤔 Why Not Just Use Docker Alone?

Docker runs a container on *one machine*. But real-world apps need to run on *many machines*, handle failures, serve millions of users, and scale on demand. That's where Kubernetes comes in — it orchestrates containers **across a cluster of machines**.

---

## 🔧 What is "Kubernetes The Hard Way"?

**"Kubernetes The Hard Way"** (KTHW) is a famous hands-on guide originally created by **Kelsey Hightower** (Google) that teaches you to set up a Kubernetes cluster **completely from scratch**, manually, without any automation tools like `kubeadm`, `minikube`, or managed cloud services (EKS, GKE, AKS).

Every single component — TLS certificates, etcd, the API server, the scheduler, kubelet, kube-proxy — is installed, configured, and wired together **by hand**.

### ⚡ The Hard Way vs. The Easy Way

| Method | Tool | Who It's For | What's Hidden |
|--------|------|-------------|---------------|
| **Hard Way** | Raw binaries + manual config | Learners, engineers wanting deep understanding | Nothing — you do everything |
| **kubeadm** | `kubeadm init` | Teams wanting quick cluster setup | Certificate generation, component configs |
| **Minikube** | `minikube start` | Local development/testing | Everything — single-node only |
| **Managed (EKS/GKE/AKS)** | Cloud console/CLI | Production teams | The entire control plane |

---

## 🎯 Why Learn It The Hard Way?

If you're a student or someone exploring DevOps and Cloud, here's why KTHW is worth the effort:

- **Deep understanding** — You'll know *exactly* what every component does because you set it up yourself
- **Better troubleshooting** — When something breaks in production, you'll know where to look
- **CKA/CKAD exam prep** — The Certified Kubernetes Administrator exam rewards this kind of deep knowledge
- **Career advantage** — Engineers who understand Kubernetes internals are highly valued
- **Cloud-agnostic skills** — You're not tied to any vendor; you can adapt anywhere

> *"If you want to truly understand how Kubernetes works under the hood, do it the hard way at least once."*

---

## 🏗️ Kubernetes Architecture — Core Components

A Kubernetes cluster has two main layers: the **Control Plane (Master)** and the **Worker Nodes**.

```
┌─────────────────────────────────────────────┐
│              CONTROL PLANE (Master)         │
│                                             │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │  API Server  │  │  Controller Manager  │ │
│  │  (kube-      │  │  (kube-controller-   │ │
│  │  apiserver)  │  │   manager)           │ │
│  └──────────────┘  └──────────────────────┘ │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Scheduler   │  │        etcd          │ │
│  │  (kube-      │  │  (cluster data store)│ │
│  │  scheduler)  │  │                      │ │
│  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
┌────────▼───┐  ┌──────▼────┐  ┌─────▼─────┐
│  Worker 1  │  │  Worker 2 │  │  Worker N │
│            │  │           │  │           │
│  kubelet   │  │  kubelet  │  │  kubelet  │
│  kube-proxy│  │ kube-proxy│  │ kube-proxy│
│  Pods      │  │  Pods     │  │  Pods     │
└────────────┘  └───────────┘  └───────────┘
```

### Control Plane Components

| Component | Role |
|-----------|------|
| **kube-apiserver** | The front door of the cluster. All communication (kubectl, internal) goes through here. Validates and processes REST requests. |
| **etcd** | A distributed key-value store. The **single source of truth** for all cluster state — nodes, pods, configs, secrets. |
| **kube-scheduler** | Watches for new pods with no assigned node and picks the best worker node for them based on resources. |
| **kube-controller-manager** | Runs control loops — ensures the desired state matches the actual state (e.g., if a pod dies, it creates a new one). |

### Worker Node Components

| Component | Role |
|-----------|------|
| **kubelet** | The agent on every worker node. Receives instructions from the API server and manages pods on that node. |
| **kube-proxy** | Manages network rules on each node. Enables pods to communicate with each other and with the outside world. |
| **Container Runtime** | The software that actually runs containers (e.g., containerd, Docker, CRI-O). |

---

## 🗺️ Types of Cluster Topologies

Kubernetes clusters can be arranged in several different topologies depending on your needs for **availability**, **scalability**, and **fault tolerance**.

### 1. 🔵 Single Master (Single Control Plane)

```
     [kubectl / User]
           │
    ┌──────▼──────┐
    │   Master    │  ← One control plane node
    │  (API, etcd,│
    │  scheduler) │
    └──────┬──────┘
           │
   ┌───────┼───────┐
   │       │       │
[Worker1][Worker2][Worker3]
```

- **Simple** setup, easiest to learn
- **Single point of failure** — if the master goes down, you can't manage the cluster (but running apps continue)
- **Best for:** Learning, dev/test environments, small non-critical workloads
- ✅ **This is what this repo implements**

---

### 2. 🟡 Stacked etcd (High Availability)

```
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ Master1 │   │ Master2 │   │ Master3 │
   │+etcd    │───│+etcd    │───│+etcd    │
   └────┬────┘   └────┬────┘   └────┬────┘
        └──────────────┼─────────────┘
                       │ Load Balancer
              ┌────────┼────────┐
           [Worker1][Worker2][Worker3]
```

- Multiple control plane nodes with **etcd co-located** on each master
- A **load balancer** sits in front of the API servers
- Failure of one master doesn't bring down the cluster
- **Best for:** Production environments, moderate HA requirements

---

### 3. 🔴 External etcd (High Availability)

```
   ┌─────────┐   ┌─────────┐
   │ Master1 │   │ Master2 │    ┌───────────────┐
   │(no etcd)│   │(no etcd)│    │  etcd Cluster │
   └────┬────┘   └────┬────┘    │ [e1][e2][e3]  │
        └──────┬───────┘        └───────────────┘
          Load Balancer
        ┌──────┴──────┐
     [Worker1]     [Worker2]
```

- etcd runs on **separate dedicated nodes**
- Better isolation; etcd failures don't impact master nodes
- More machines required but **highest resilience**
- **Best for:** Large-scale production clusters where data reliability is critical

---

### 4. 🟢 Managed Kubernetes (Cloud Provider)

```
   [You]──► kubectl ──► Cloud Provider API
                              │
                    ┌─────────▼──────────┐
                    │   Managed Masters  │  ← Invisible to you
                    │   (AWS/GCP/Azure   │
                    │   manages these)   │
                    └─────────┬──────────┘
                              │
                   ┌──────────┼──────────┐
                [Worker1]  [Worker2]  [Worker3]
```

- AWS EKS, Google GKE, Azure AKS
- You only manage worker nodes; cloud handles the control plane
- **Best for:** Teams that want to focus on apps, not infrastructure

---

## 🖥️ This Setup — Single Master Topology

This repository documents a **Single Master Kubernetes cluster** built entirely from scratch — the hard way.

### Cluster Overview

```
                          ┌─────────────────────────────────┐
                          │         MASTER NODE             │
                          │                                 │
                          │  ● kube-apiserver  (port 6443)  │
                          │  ● kube-scheduler               │
                          │  ● kube-controller-manager      │
                          │  ● etcd            (port 2379)  │
                          │  ● kubectl (admin access)       │
                          └────────────┬────────────────────┘
                                       │ TLS Secured
                         ┌─────────────┼─────────────┐
                         │             │             │
              ┌──────────▼──┐   ┌──────▼──────┐   (more...)
              │  Worker  1  │   │  Worker  2  │
              │             │   │             │
              │  kubelet    │   │  kubelet    │
              │  kube-proxy │   │  kube-proxy │
              │  containerd │   │  containerd │
              │  [Pod][Pod] │   │  [Pod][Pod] │
              └─────────────┘   └─────────────┘
```

### Key Highlights of This Setup

- All TLS certificates generated manually (CA, API server, kubelet, etc.)
- etcd bootstrapped from binary with custom systemd service
- CNI (Container Network Interface) configured for pod-to-pod networking
- RBAC (Role-Based Access Control) enabled
- Secrets encrypted at rest in etcd

---

## 🌐 Traffic & Request Flow

Understanding how a request travels through the cluster is fundamental. Here's the full journey:

### 1. External User → Cluster (Ingress Flow)

```
[Browser / Client]
       │
       │ HTTP/HTTPS Request
       ▼
[Load Balancer / NodePort]
       │
       │ Routes to NodeIP:Port
       ▼
[kube-proxy on Worker Node]
       │
       │ iptables rules forward traffic
       ▼
[Service (ClusterIP)]
       │
       │ Selects a healthy Pod (via label selector)
       ▼
[Pod / Container]
       │
       │ Application processes request
       ▼
[Response returned to client]
```

### 2. kubectl Command Flow (Admin → Cluster)

```
[Developer runs: kubectl get pods]
       │
       │ HTTPS (TLS) to port 6443
       ▼
[kube-apiserver]
       │
       ├── Authenticates (client cert / token)
       ├── Authorizes (RBAC check)
       ▼
[etcd] ──► Returns cluster state
       │
       ▼
[kubectl] displays result to user
```

### 3. Pod Scheduling Flow (New Deployment)

```
[kubectl apply -f deployment.yaml]
       │
       ▼
[kube-apiserver] ──► writes to etcd
       │
       ▼
[kube-scheduler] watches for unscheduled pods
       │ Picks best node based on resources
       ▼
[kube-apiserver] ──► updates pod spec in etcd
       │
       ▼
[kubelet on Worker Node] sees the new pod
       │ Pulls container image
       ▼
[containerd] starts the container
       │
       ▼
[Pod is Running] ✅
```

---

##  Prerequisites

Before following the setup guide, ensure you have:

- [ ] 3+ Linux VMs (1 Master + 2 Workers) — Ubuntu 20.04/22.04 recommended
- [ ] Minimum **2 GB RAM** and **2 vCPUs** for the Master node
- [ ] Minimum **1 GB RAM** for each Worker node
- [ ] All nodes can communicate with each other (same network/VPC)
- [ ] `sudo` / root access on all nodes
- [ ] Basic Linux command-line familiarity (`ssh`, `vim`, `systemctl`)
- [ ] Understanding of what containers and Docker are (helpful but not mandatory)

### Tools You'll Work With

| Tool | Purpose |
|------|---------|
| `openssl` / `cfssl` | Generate TLS certificates for all components |
| `etcd` / `etcdctl` | Deploy and manage the cluster data store |
| `kubectl` | Kubernetes CLI to interact with the cluster |
| `kubelet` | Node agent that manages pods |
| `kube-proxy` | Manages network rules on nodes |
| `containerd` | Container runtime |
| `systemd` | Manages all Kubernetes components as services |

---

##  How to Use This Repo

1. **Start here** — Read this README fully to understand what you're building
2. **View the architecture** — Open `Architecture.png` to visualize the cluster topology and traffic flow
3. **Follow the guide** — Open `Single-master-setup.md` and follow every step in order
4. **Reference screenshots** — Check the `screenshots/` folder if you want to compare your output at any stage
5. **Experiment** — Once the cluster is up, try deploying apps, scaling pods, and breaking things to learn!

---

## 📖 Key Concepts Glossary

| Term | Meaning |
|------|---------|
| **Pod** | The smallest deployable unit in Kubernetes. A Pod wraps one or more containers. |
| **Node** | A physical or virtual machine in the cluster. Can be master or worker. |
| **Cluster** | A set of nodes managed together by Kubernetes. |
| **Deployment** | Tells Kubernetes to run N replicas of a Pod and keep them running. |
| **Service** | A stable network endpoint to access a set of Pods (even as Pods come and go). |
| **etcd** | The distributed database that stores all cluster state. |
| **kubelet** | The agent on each worker node that runs and monitors Pods. |
| **Namespace** | A logical partition inside a cluster (like a folder) to group resources. |
| **RBAC** | Role-Based Access Control — controls who can do what in the cluster. |
| **CNI** | Container Network Interface — the plugin that handles pod networking. |
| **TLS/mTLS** | Encrypted communication between cluster components. |
| **kubeconfig** | A config file used by `kubectl` to know which cluster to talk to. |
| **Ingress** | An API object that manages external HTTP/S access to services in the cluster. |
| **DaemonSet** | Ensures a copy of a Pod runs on every node (e.g., kube-proxy, log collectors). |
| **ConfigMap / Secret** | Kubernetes objects to store configuration data and sensitive values. |

---

##  Further Learning

### Beginner Starting Points
- [Kubernetes Official Docs](https://kubernetes.io/docs/home/) — The best reference
- [KillerCoda Kubernetes Labs](https://killercoda.com/playgrounds/scenario/kubernetes) — Free browser-based cluster
- [Play with Kubernetes](https://labs.play-with-k8s.com/) — Free playground in your browser

### For Going Deeper
- [Kelsey Hightower's KTHW (Original)](https://github.com/kelseyhightower/kubernetes-the-hard-way) — The guide that inspired this project
- [Kubernetes The Very Hard Way — iximiuz Labs](https://labs.iximiuz.com/) — Interactive deep-dive course
- [DevOpsCube KTHW Guide](https://devopscube.com/building-kubernetes-cluster-the-hard-way/) — Practical insights with personal notes

### Certifications to Target
- **CKA** — Certified Kubernetes Administrator (I hope this project would help for prep).
- **CKAD** — Certified Kubernetes Application Developer
- **CKS** — Certified Kubernetes Security Specialist (Advanced)

---

## 🤝 Connect & Contribute

If you found this project helpful, feel free to ⭐ star the repo and share it with others learning DevOps and Cloud.

LinkedIn Post: [View the original post](https://www.linkedin.com/posts/activity-7463832926468833280-sDLo?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEV-FWgBBTo0pqSie4CPFSR_jgWhfh8bNss)

> **Built with curiosity, broken many times, and rebuilt with understanding.**
> That's the Kubernetes Hard Way. 🚀

---

*Happy Clustering! ☸️*