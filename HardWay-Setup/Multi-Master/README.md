# ☸️ Kubernetes The Hard Way — High Availability Multi-Master Production Cluster

<div align="center">

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.0-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![etcd](https://img.shields.io/badge/etcd-v3.5.21-419EDA?style=for-the-badge&logo=etcd&logoColor=white)
![Calico](https://img.shields.io/badge/Calico-v3.31.0-FB8C00?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![HAProxy](https://img.shields.io/badge/HAProxy-Load_Balancer-1572B6?style=for-the-badge)

**A production-grade, fully manual, High Availability Kubernetes cluster — 3 control plane masters, 5 worker nodes, no kubeadm, no magic.**

[Setup Guide](#-setup-guide) • [What is a Deployment?](#-what-is-a-deployment) • [Security](#-security-model) • [Prerequisites](#-prerequisites)

</div>

---

## 📁 Repository Contents

| File | Description |
|------|-------------|
| [`kubernetes-ha-multimaster-hardway.md`](./Multi-master-setup.md) | Complete step-by-step production setup guide (7 Phases, 115 steps) |
| [`README.md`](./README.md) | This file — full project overview, architecture, and conceptual deep-dives |

> **Also check out my single-master version** — a simpler starting point if you're new to KTHW:
> [`Single-Master Setup (README)`](https://github.com/Jash-459/kubernetes-k8s/tree/main/HardWay-Setup/Single-Master/README.md)

---

## 🧭 Table of Contents

1. [What is Kubernetes?](#-what-is-kubernetes)
2. [What is "The Hard Way"?](#-what-is-kubernetes-the-hard-way)
3. [Why Learn It The Hard Way?](#-why-learn-it-the-hard-way)
4. [What is a Deployment?](#-what-is-a-deployment)
5. [How a Deployment Works — Internally](#-how-a-deployment-works--internally)
6. [Deployment vs Other Workload Types](#-deployment-vs-other-workload-types)
7. [Cluster Topology — HA Multi-Master](#-cluster-topology--ha-multi-master)
8. [Traffic & Request Flow](#-traffic--request-flow)
9. [Security Model](#-security-model)
10. [Component Versions](#-component-versions)
11. [Prerequisites](#-prerequisites)
12. [Setup Guide](#-setup-guide)
13. [Key Concepts Glossary](#-key-concepts-glossary)
14. [Further Learning](#-further-learning)

---

## 🐳 What is Kubernetes?

Kubernetes (also written as **K8s**) is an open-source **container orchestration platform** originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).

When you run applications inside **containers** (Docker, containerd), Kubernetes is the system that:

- **Deploys** your containers across a fleet of machines
- **Scales** them up or down based on demand
- **Self-heals** — restarts crashed containers, replaces failed nodes
- **Load balances** traffic between replicas automatically
- **Manages secrets, configs, storage, and networking**
- **Rolls out updates** with zero downtime

Think of Kubernetes as the **operating system for your cloud infrastructure** — it manages where and how your applications run so you don't have to do it manually.

### Why Not Just Use Docker Alone?

Docker runs a container on *one machine*. But real-world applications need to run across *many machines*, survive failures, serve millions of users simultaneously, and scale on demand. Kubernetes orchestrates containers **across a cluster of machines** — and does so reliably, at scale.

---

## 🔧 What is "Kubernetes The Hard Way"?

**"Kubernetes The Hard Way"** (KTHW) is a famous hands-on guide originally created by **Kelsey Hightower** (Google) that teaches you to set up a Kubernetes cluster **completely from scratch** — no `kubeadm`, no `minikube`, no managed cloud services (EKS, GKE, AKS).

Every single component — TLS certificates, etcd, the API server, the scheduler, the controller manager, kubelet, kube-proxy — is installed, configured, and wired together **by hand**.

### The Hard Way vs. The Easy Way

| Method | Tool | Who It's For | What's Hidden |
|--------|------|-------------|---------------|
| **Hard Way** | Raw binaries + manual config | Engineers wanting deep understanding | Nothing — you do everything |
| **kubeadm** | `kubeadm init` | Teams wanting quick cluster setup | Certificate generation, component configs |
| **Minikube** | `minikube start` | Local development/testing | Everything — single-node only |
| **Managed (EKS/GKE/AKS)** | Cloud console/CLI | Production teams | The entire control plane |

This project goes even further than the original KTHW guide — it implements a **3-master HA cluster** designed with production security hardening, not just a learning environment.

---

## 🎯 Why Learn It The Hard Way?

- **Deep understanding** — You know exactly what every component does because you built it yourself
- **Better troubleshooting** — When something breaks in production, you know where to look
- **CKA exam prep** — The Certified Kubernetes Administrator exam rewards this kind of deep knowledge
- **Career advantage** — Engineers who understand Kubernetes internals are highly valued
- **Cloud-agnostic skills** — Not tied to any vendor; skills transfer everywhere

> *"If you want to truly understand how Kubernetes works under the hood, do it the hard way at least once."*

---

## 🚀 What is a Deployment?

A **Deployment** is one of the most important objects in Kubernetes. It is a declarative specification that tells Kubernetes:

> *"I want N copies of this container to be running at all times. Keep them running. If one dies, replace it. If I want to update the image, roll it out safely."*

You describe the **desired state** in a YAML file, and Kubernetes continuously works to make the actual cluster state match that desired state. This is the **declarative model** at the heart of Kubernetes.

### A Simple Deployment — What it Looks Like

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  namespace: production
spec:
  replicas: 3                          # "I want 3 copies running"
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25            # The container image to run
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

When you `kubectl apply` this file, Kubernetes creates 3 pods running nginx and keeps them alive — forever, automatically.

---

## ⚙️ How a Deployment Works — Internally

Understanding what happens under the hood when you apply a Deployment is one of the key insights KTHW gives you. Here is the complete internal flow:

```
You run: kubectl apply -f deployment.yaml
          │
          ▼
┌─────────────────────────┐
│     kube-apiserver      │  ← Receives the request over HTTPS (port 6443)
│                         │    Authenticates (your cert/token)
│                         │    Authorizes (RBAC check)
│                         │    Validates the YAML spec
└──────────┬──────────────┘
           │ Writes Deployment object
           ▼
┌─────────────────────────┐
│          etcd           │  ← The cluster's database — stores the desired state
│  (distributed k/v store)│    "I want 3 replicas of nginx:1.25"
└──────────┬──────────────┘
           │ Deployment controller watches for changes
           ▼
┌─────────────────────────┐
│  kube-controller-manager│  ← The Deployment controller sees the new object
│  (Deployment Controller)│    Compares desired replicas (3) vs actual (0)
│                         │    Creates 3 ReplicaSet → 3 Pod specs
└──────────┬──────────────┘
           │ Pods exist in etcd but have no node assigned
           ▼
┌─────────────────────────┐
│     kube-scheduler      │  ← Watches for unscheduled pods
│                         │    Scores all worker nodes (CPU, memory, taints,
│                         │    affinity rules, existing load)
│                         │    Assigns each pod to the best available node
└──────────┬──────────────┘
           │ Node assignment written to etcd
           ▼
┌─────────────────────────┐
│   kubelet (on worker)   │  ← Watches the API for pods assigned to its node
│                         │    Pulls the container image from the registry
│                         │    Calls containerd to start the container
│                         │    Reports pod status back to the API server
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   Pod is Running ✅     │  ← Container is live, serving traffic
└─────────────────────────┘
```

### The Reconciliation Loop — Why Deployments Are Self-Healing

The Deployment controller doesn't just create pods once. It continuously runs a **reconciliation loop**:

```
Every few seconds:
  actual_state  = "how many pods are actually running right now?"
  desired_state = "how many replicas does the Deployment spec say?"

  if actual < desired:
    create more pods   (a node crashed, a pod OOM-killed, etc.)

  if actual > desired:
    delete excess pods (you scaled down the replicas field)

  if image has changed:
    perform a RollingUpdate (drain old pods, bring up new pods)
```

This is why Kubernetes is said to be **declarative** and **self-healing** — you declare what you want, and the system continuously drives toward that state without any manual intervention.

### Rolling Updates — Zero-Downtime Deploys

When you update a Deployment (e.g., changing the container image version), Kubernetes performs a **RollingUpdate** by default:

```
Initial state:   [Pod v1] [Pod v1] [Pod v1]   ← 3 old pods running

Step 1:          [Pod v1] [Pod v1] [Pod v2]   ← 1 new pod created, 1 old terminated
Step 2:          [Pod v1] [Pod v2] [Pod v2]   ← 1 more new pod, 1 more old terminated
Step 3:          [Pod v2] [Pod v2] [Pod v2]   ← fully updated, zero downtime
```

You can control the rollout behaviour with:
- `maxUnavailable` — how many pods can be down at once during the update
- `maxSurge` — how many extra pods can be created above the desired count

### Rollback — Going Back When Something Goes Wrong

Kubernetes keeps a history of your Deployment revisions. If a bad image is deployed:

```bash
kubectl rollout undo deployment/my-web-app              # Roll back to previous version
kubectl rollout undo deployment/my-web-app --to-revision=3  # Roll back to a specific revision
kubectl rollout history deployment/my-web-app           # View revision history
```

---

## 📦 Deployment vs Other Workload Types

A Deployment is not the only way to run workloads in Kubernetes. Here is when to use each:

| Workload Type | Use When | Examples |
|---------------|----------|----------|
| **Deployment** | Stateless apps — any pod is identical and replaceable | Web servers, APIs, microservices |
| **StatefulSet** | Stateful apps — each pod has a unique identity and stable storage | Databases (Postgres, MySQL), Kafka, Elasticsearch |
| **DaemonSet** | One pod on every node | Log collectors (Fluentd), monitoring agents (Prometheus node exporter), CNI plugins |
| **Job** | Run a task to completion once | Database migrations, batch processing, one-off scripts |
| **CronJob** | Run a task on a schedule | Nightly backups, hourly reports, cleanup jobs |

---

## 🏗️ Architecture Overview

```
                          Internet
                              │
                              ▼
                     ┌─────────────────┐
                     │    HAProxy      │  ← Only public-facing node
                     │  Port 6443      │    Kube-api workflow
                     │                 │    
                     │  Port 443       │    Application Traffic 
                     └────────┬────────┘
                              │ Round-robin to healthy masters
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼──┐   ┌────────▼───┐   ┌───────▼────┐
    │  master-1  │   │  master-2  │   │  master-3  │
    │            │   │            │   │            │
    │ apiserver  │   │ apiserver  │   │ apiserver  │
    │ scheduler* │   │ scheduler* │   │ scheduler* │
    │ ctrl-mgr*  │   │ ctrl-mgr*  │   │ ctrl-mgr*  │
    │ etcd       │◄─►│ etcd       │◄─►│ etcd       │
    │ kubelet    │   │ kubelet    │   │ kubelet    │
    │ kube-proxy │   │ kube-proxy │   │ kube-proxy │
    └────────────┘   └────────────┘   └────────────┘
         │                  │                │
         └──────────────────┼────────────────┘
                            │ Calico CNI (BGP mesh)
              ┌─────────────┼─────────────────────────┐
              │         │         │         │          │
    ┌─────────▼──┐ ┌────▼───┐ ┌──▼─────┐ ┌─▼──────┐ ┌▼───────┐
    │  worker-1  │ │worker-2│ │worker-3│ │worker-4│ │worker-5│
    │  kubelet   │ │kubelet │ │kubelet │ │kubelet │ │kubelet │
    │ kube-proxy │ │k-proxy │ │k-proxy │ │k-proxy │ │k-proxy │
    │ containerd │ │conterd │ │conterd │ │conterd │ │conterd │
    │ [Pods...]  │ │[Pods..]│ │[Pods..]│ │[Pods..]│ │[Pods..]│
    └────────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

`*` Leader election — only one instance is active at a time across the 3 masters.

---

## 🗺️ Cluster Topology — HA Multi-Master

This project implements the **Stacked etcd HA topology** — each master node runs both the Kubernetes control plane components and an etcd member.

### Why 3 Masters?

etcd uses the **Raft consensus algorithm**, which requires a quorum (majority) to function. With 3 nodes:

| etcd nodes | Quorum needed | Can survive this many failures |
|------------|--------------|-------------------------------|
| 1 | 1 | 0 |
| 3 | 2 | **1** |
| 5 | 3 | **2** |

3 masters gives you fault tolerance for 1 master failure with the minimum number of nodes. If master-2 goes down, master-1 and master-3 still hold quorum — the cluster keeps running.

### Topology Comparison

```
Single Master (this repo's earlier project):
  ┌──────────┐       ┌────────────────────────────┐
  │  Master  │──────►│  workers...                │
  └──────────┘       └────────────────────────────┘
  Simple but: single point of failure

HA Stacked etcd (THIS project):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ master-1 │◄►│ master-2 │◄►│ master-3 │
  │  + etcd  │  │  + etcd  │  │  + etcd  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └─────────────┼─────────────┘
                HAProxy LB
              ┌───────┴───────┐
          [workers...]    [workers...]

HA External etcd (beyond this project):
  Separate etcd cluster on dedicated nodes —
  even more resilient but requires more machines
```

---

## 🌐 Traffic & Request Flow

### 1. External HTTP/S Traffic → Your Application

```
[Browser / Client]
       │ HTTP/HTTPS
       ▼
[HAProxy :443]
       │ TCP pass-through — TLS untouched
       ▼
[NGINX Gateway Fabric]   ← Kubernetes Gateway API controller
       │ Matches HTTPRoute rules (hostname, path)
       ▼
[Kubernetes Service]     ← Stable ClusterIP endpoint
       │ kube-proxy iptables rules select a healthy pod
       ▼
[Pod / Container]        ← Your application processes the request
       │
       ▼
[Response returned]
```

### 2. kubectl Command Flow

```
[Developer: kubectl get pods]
       │ HTTPS + client certificate (TLS)
       ▼
[HAProxy :6443]
       │ Routes to any healthy master
       ▼
[kube-apiserver]
       ├── Authenticates  (validates your x509 cert)
       ├── Authorizes     (RBAC: can this user list pods?)
       └── Reads from etcd → returns pod list to kubectl
```

### 3. New Deployment Scheduling Flow

```
kubectl apply -f deployment.yaml
       │
       ▼
[kube-apiserver] ──writes──► [etcd]  ← desired state stored
       │
       ▼ (controller-manager watches etcd)
[Deployment Controller] creates ReplicaSet → Pod specs written to etcd
       │
       ▼ (scheduler watches for pods with no node assigned)
[kube-scheduler] scores nodes, assigns pod to worker-3
       │
       ▼ (kubelet on worker-3 watches its assigned pods)
[kubelet on worker-3] pulls image → calls containerd → starts container
       │
       ▼
[Pod Running ✅]  status written back to etcd via apiserver
```

---

## 🔐 Security Model

This cluster is built with a **defence-in-depth** security posture. Ten key security decisions are baked into the setup:

| # | Decision | Why |
|---|----------|-----|
| 1 | **No public IPs on masters/workers** | Only HAProxy is internet-reachable; masters are invisible to the internet |
| 2 | **Mutual TLS (mTLS) everywhere** | Every component authenticates via x509 certificates — no shared passwords |
| 3 | **Encryption at rest** | Secrets in etcd are AES-CBC encrypted — raw etcd access reveals no plaintext |
| 4 | **etcd peer authentication** | Separate peer certs prevent rogue nodes from joining the etcd cluster |
| 5 | **Least-privilege RBAC** | Every component uses a dedicated service account with minimal permissions |
| 6 | **Full audit logging** | Every API server request is logged with metadata and stored for 30 days |
| 7 | **Admission controllers** | NodeRestriction, ResourceQuota, PodSecurity enforced at the API layer |
| 8 | **UFW firewalls on all nodes** | Ports open only to the specific IPs that need them |
| 9 | **`ca.key` never leaves master-1** | Worker certs are signed on master-1; the CA private key is never copied to workers |
| 10 | **Leader election for controllers** | kube-scheduler and controller-manager use built-in HA — only 1 active at a time |

---

## 📦 Component Versions

| Component | Version | Purpose |
|-----------|---------|---------|
| Kubernetes | v1.34.0 | Orchestration platform |
| etcd | v3.5.21 | Distributed cluster data store |
| Calico | v3.31.0 | CNI plugin — pod networking + NetworkPolicy |
| CoreDNS | v1.12.0 | In-cluster DNS resolution |
| HAProxy | Latest stable | Layer 4 load balancer for the API server |
| Gateway API CRDs | v1.2.0 | Modern successor to Ingress |
| NGINX Gateway Fabric | v1.5.1 | Ingress / Gateway controller |
| Ubuntu | 22.04 LTS | Node operating system |

---

## ✅ Prerequisites

Before following the setup guide, ensure you have:

- 9 Linux VMs: 1 HAProxy + 3 Masters + 5 Workers (Ubuntu 22.04 LTS recommended)
- Master nodes: minimum **2 GB RAM**, **2 vCPUs** each
- Worker nodes: minimum **2 GB RAM**, **2 vCPUs** each
- HAProxy node: minimum **1 GB RAM**, **1 vCPU**
- All nodes on the same private network (same VPC / subnet)
- `sudo` / root access on all nodes
- SSH key-based access between master-1 and all other nodes
- Basic Linux familiarity (`ssh`, `vim`, `systemctl`, `openssl`)

### Tools Configured During Setup

| Tool | Purpose |
|------|---------|
| `openssl` | Generate all x509 TLS certificates |
| `etcd` / `etcdctl` | Distributed key-value store for cluster state |
| `kubectl` | Kubernetes CLI |
| `kubelet` | Node agent that runs and monitors pods |
| `kube-proxy` | Manages iptables rules for Service routing |
| `containerd` | Container runtime (runs the actual containers) |
| `haproxy` | TCP load balancer in front of API servers |
| `calico` | CNI plugin — assigns pod IPs and routes traffic |
| `systemd` | Manages all Kubernetes components as services |
| `ufw` | Firewall — restricts ports per node role |

---

## 📖 Setup Guide

The full step-by-step instructions are in **[`Multi-master-setup.md`](./Multi-master-setup.md)**.

The setup is divided into 7 phases:

| Phase | What You Build | Key Steps |
|-------|---------------|-----------|
| **Phase 0** | HAProxy Load Balancer | Install and configure TCP pass-through LB for API server HA |
| **Phase 1** | PKI & Certificate Authority | Generate CA, etcd peer certs, API server cert, all component certs |
| **Phase 2** | etcd Cluster | Bootstrap 3-node etcd cluster with mTLS peer auth |
| **Phase 3** | Control Plane | kube-apiserver, controller-manager, scheduler, kubelet on all masters |
| **Phase 4** | Worker Nodes | containerd, kubelet, kube-proxy on all 5 workers |
| **Phase 5** | Networking | Calico CNI, CoreDNS, Gateway API, NGINX Gateway Fabric |
| **Phase 6** | Security Hardening | RBAC policies, NetworkPolicies, Pod Security Admission, UFW |
| **Phase 7** | Verification | Health checks, DNS tests, HA failover test |

> **Important:** Replace all `<PLACEHOLDER>` values (e.g. `<MASTER1_IP>`, `<HAPROXY_IP>`) with your actual server IPs throughout the guide before running any commands. A full placeholder reference table is at the bottom of the setup guide.

---

## 📖 Key Concepts Glossary

| Term | Meaning |
|------|---------|
| **Pod** | Smallest deployable unit in Kubernetes. Wraps one or more containers that share a network and storage. |
| **Deployment** | Manages a set of identical, stateless pods. Handles scaling, rolling updates, and self-healing. |
| **ReplicaSet** | Created by a Deployment — ensures N copies of a pod are running at all times. |
| **Service** | A stable network endpoint (ClusterIP or NodePort) to access a set of pods, even as pods come and go. |
| **Node** | A physical or virtual machine in the cluster. Can be a master (control plane) or worker (data plane). |
| **Cluster** | The set of all nodes managed together by Kubernetes. |
| **etcd** | The distributed database that stores all cluster state. Think of it as Kubernetes's brain. |
| **kubelet** | The agent on every node. Receives pod specs from the API server and manages containers on that node. |
| **kube-proxy** | Manages iptables rules on each node. Implements Service-to-pod traffic routing. |
| **Namespace** | A logical partition inside a cluster to group and isolate resources. |
| **RBAC** | Role-Based Access Control — controls who (user/service account) can do what (verbs) on which resources. |
| **CNI** | Container Network Interface — the plugin that assigns IPs to pods and routes traffic between them. |
| **TLS/mTLS** | Encrypted + mutually authenticated communication. Every Kubernetes component uses this. |
| **kubeconfig** | A config file used by `kubectl` to know which cluster to connect to and which credentials to use. |
| **Ingress / Gateway** | API objects that manage external HTTP/S access to services inside the cluster. |
| **DaemonSet** | Ensures exactly one pod runs on every node. Used for log collectors, monitoring agents, CNI plugins. |
| **StatefulSet** | Like a Deployment but for stateful apps — each pod has a stable identity and persistent storage. |
| **ConfigMap / Secret** | Kubernetes objects for storing non-sensitive config data and sensitive values respectively. |
| **PVC / PV** | PersistentVolumeClaim / PersistentVolume — how pods request and use persistent storage. |
| **Taint / Toleration** | Taints on nodes repel pods; tolerations on pods override taints. Used here to keep workloads off masters. |
| **Leader Election** | Mechanism where multiple instances of a component race to be active — only 1 wins at a time. |
| **HAProxy** | High Availability Proxy — here used as a Layer 4 TCP load balancer in front of the API servers. |
| **Raft** | The consensus algorithm etcd uses to agree on cluster state across multiple nodes. |
| **Pod CIDR** | The IP range allocated for pod IPs. This setup uses `192.168.0.0/16`. |
| **Service CIDR** | The IP range allocated for Service ClusterIPs. This setup uses `10.96.0.0/12`. |

---

## 📚 Further Learning

### Hands-On Labs
- [KillerCoda Kubernetes Labs](https://killercoda.com/playgrounds/scenario/kubernetes) — Free browser-based cluster
- [Play with Kubernetes](https://labs.play-with-k8s.com/) — Free playground, no install needed


### Going Deeper
- [Kubernetes Official Docs](https://kubernetes.io/docs/home/) — The definitive reference


### Certifications to Target
- **CKA** — Certified Kubernetes Administrator (this project is ideal preparation)
- **CKAD** — Certified Kubernetes Application Developer
- **CKS** — Certified Kubernetes Security Specialist (advanced — the security hardening in this project applies here)

---

## 🤝 Connect & Contribute

If you found this project helpful, consider ⭐ starring the repo and sharing it with others learning DevOps and Cloud.

LinkedIn: [View the original post](https://www.linkedin.com/posts/activity-7469260583268220928-tmR7?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEV-FWgBBTo0pqSie4CPFSR_jgWhfh8bNss)

> **Built entirely from scratch. Broken many times. Rebuilt with understanding.**
> That's Kubernetes The Hard Way. ☸️

---

*Happy Clustering!*