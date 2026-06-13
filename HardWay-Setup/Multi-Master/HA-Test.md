# 🧪 High Availability Testing — Multi-Master Kubernetes Cluster (The Hard Way)

> **Cluster:** 3 Masters · 1+ Workers · HAProxy Load Balancer · Calico CNI · etcd v3.5 · Kubernetes v1.34.0  
> **Environment:** AWS EC2 · Self-managed TLS PKI · Systemd-managed control plane  


---

## 📋 Table of Contents

1. [Purpose & Objectives](#-purpose--objectives)
2. [Cluster Architecture](#-cluster-architecture)
3. [Pre-Test State](#-pre-test-state)
4. [Test Summary](#-test-summary)
5. [Test 1 — Single API Server Down](#test-1--single-api-server-down)
6. [Test 2 — etcd Member Loss (Quorum Test)](#test-2--etcd-member-loss-quorum-test)
7. [Test 3 — Controller-Manager Leader Failover](#test-3--controller-manager-leader-failover)
8. [Test 4 — Scheduler Leader Failover](#test-4--scheduler-leader-failover)
9. [Test 5 — Full Master Node Failure (Process Level)](#test-5--full-master-node-failure-process-level)
10. [Test 6 — HAProxy Restart (Load Balancer Interruption)](#test-6--haproxy-restart-load-balancer-interruption)
11. [Quick Reference — Recovery Timelines](#-quick-reference--recovery-timelines)
12. [Key Learnings](#-key-learnings)
13. [Conclusion](#-conclusion)

---

## 🎯 Purpose & Objectives

This document records the formal **High Availability (HA) validation** performed on a production-grade, manually assembled multi-master Kubernetes cluster. The cluster was built "the hard way" — no kubeadm, no managed services — to develop deep operational knowledge of every control plane component.

### Goals

- **Validate failover behaviour** of all control plane components under real failure conditions
- **Confirm etcd quorum** boundaries with a 3-member cluster
- **Prove HAProxy** correctly reroutes traffic when backends go down
- **Demonstrate leader election** mechanics for `kube-controller-manager` and `kube-scheduler`
- **Verify workload continuity** — running pods must survive control plane disruptions
- **Document recovery timelines** for each failure class as operational runbook data

### Why This Matters

A multi-master cluster provides no HA value unless failover has been *observed and measured*. These tests move the cluster from "theoretically highly available" to **empirically validated**. Each test was designed to exercise a distinct failure domain, moving from component-level faults to full node-level failure.

---

## 🏗 Cluster Architecture

```
Internet
    │
    ▼
Route 53 (multimaster-k8s-hardway.entityhero.com)
    │
    ▼
HAProxy (TCP Load Balancer — port 6443 & 443)
    │
    ├──▶ master-1  (kube-apiserver · etcd · kube-controller-manager · kube-scheduler)
    ├──▶ master-2  (kube-apiserver · etcd · kube-controller-manager · kube-scheduler)
    └──▶ master-3  (kube-apiserver · etcd · kube-controller-manager · kube-scheduler)
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                worker-1  ...      worker-N
              (kubelet · kube-proxy · Calico)
```

**Key components:**

| Component | Technology | Notes |
|---|---|---|
| Load Balancer | HAProxy v3.2.x | TCP passthrough; health checks on port 6443 |
| CNI | Calico via Tigera Operator | Pod networking and network policy |
| Ingress | NGINX Gateway Fabric | `hostNetwork: true`; Gateway API HTTPRoutes |
| TLS PKI | Self-managed CA | Manual cert signing with SANs |
| DNS | CoreDNS | Patched for Ubuntu systemd-resolved loop |
| Service Mesh | None | Direct component-to-component TLS |

---

## 📸 Pre-Test State

Before testing began, the cluster was in a healthy, fully operational state:

```
NAME       STATUS   ROLES           AGE    VERSION
master-1   Ready    control-plane   7d9h   v1.34.0
master-2   Ready    control-plane   7d9h   v1.34.0
master-3   Ready    control-plane   7d9h   v1.34.0
worker-1   Ready    worker          7d8h   v1.34.0
```

All system pods (Calico, CoreDNS, CSI drivers, NGINX Gateway, Tigera Operator) were in `Running` state. The `valorant` namespace was hosting the `alpha-earth` deployment with 2 replicas as the test workload.

---

## 📊 Test Summary

| # | Test | Failure Injected | Result | Recovery Time |
|---|------|-----------------|--------|---------------|
| 1 | Single API Server Down | `systemctl stop kube-apiserver` on master-2 | ✅ PASS | ~10s (HAProxy reroute) |
| 2 | etcd Member Loss | `systemctl stop etcd` on master-3 | ✅ PASS | Immediate (quorum held) |
| 3 | Controller-Manager Failover | Stop leader controller-manager | ✅ PASS | ~15s (new lease acquired) |
| 4 | Scheduler Failover | Stop leader scheduler | ✅ PASS | ~15s (new lease acquired) |
| 5 | Full Master Node Failure | Stop all 4 services on master-3 | ✅ PASS | Immediate (2/3 etcd, 2 API servers remain) |
| 6 | HAProxy Restart | `systemctl restart haproxy` | ✅ PASS | <10s (kubectl reconnects) |

**Overall Result: 6/6 Tests Passed — Cluster HA validated across all failure domains.**

---

## Test 1 — Single API Server Down

### What It Validates
HAProxy detects a failed backend and fails over to the remaining two API servers. Cluster control plane operations continue without interruption.

### Procedure

```bash
# On master-2: stop the API server
systemctl stop kube-apiserver
systemctl status kube-apiserver   # Confirm inactive
```

### Observed Behaviour

- `master-2` transitioned to `NotReady` in `kubectl get nodes` as expected — its own API server was down, so kubelet heartbeats via that endpoint ceased
- HAProxy marked `master-2` backend as `DOWN` (Layer4 connection problem on port 6443)
- `master-1` and `master-3` API servers continued serving all control plane traffic without interruption
- HAProxy stats confirmed: `master-2 DOWN`, `master-1 UP`, `master-3 UP`

```bash
# HAProxy stat check (from HAProxy host)
echo "show stat" | socat stdio /run/haproxy/admin.sock | grep master-2
# Output: k8s-masters,master-2,...,DOWN,...,L4CON,...
```

### Outcome

✅ **PASS** — HAProxy rerouted within ~10 seconds. No kubectl commands failed from `master-1`. Restarting `kube-apiserver` on `master-2` restored full cluster state.

---

## Test 2 — etcd Member Loss (Quorum Test)

### What It Validates
With a 3-member etcd cluster, the Raft consensus protocol requires at least 2 members (quorum = `floor(3/2) + 1 = 2`). This test confirms the cluster tolerates exactly 1 member loss without entering read-only mode.

### Procedure

```bash
# On master-3: stop etcd
systemctl stop etcd
```

### Observed Behaviour

```bash
# From master-1: check etcd cluster health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://<master-1>:2379,https://<master-2>:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  endpoint status --write-out=table
```

- With master-3 stopped: **2 members remain**, quorum is maintained
- Cluster accepted writes — verified by creating and deleting a ConfigMap:

```bash
kubectl create configmap quorum-test --from-literal=test=pass
# configmap/quorum-test created

kubectl delete configmap quorum-test
# configmap "quorum-test" deleted
```

- After a 60-second sleep and restart of etcd on master-3, the member **re-joined automatically** and synced to the current Raft index (108494)
- The **etcd leader remained master-2** throughout — no re-election was triggered

### ⚠️ Critical Operational Note

> **Do NOT stop 2 etcd members simultaneously.** This breaks quorum (1 of 3 members cannot form majority), putting the API server into read-only mode — it will reject all write operations. Recovery requires restarting both members and waiting for leader re-election.

### Outcome

✅ **PASS** — etcd maintained quorum with 2/3 members. Writes continued uninterrupted. Third member auto-synced on restart with no manual intervention.

---

## Test 3 — Controller-Manager Leader Failover

### What It Validates
Only one `kube-controller-manager` instance holds the active leader lease at any time (leader election via Kubernetes `Lease` API). When the current leader stops, a standby instance acquires the lease within the `--leader-elect-retry-period` window (default: ~2s retry, 15s acquire timeout).

### Procedure

```bash
# Identify current leader
LEADER=$(kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='{.spec.holderIdentity}')
echo "Current leader: $LEADER"
# Output: master_1_2b2930ac-8dfa-440d-8bfa-0f421d18a43a

# Stop controller-manager on the leader node (master-1)
systemctl stop kube-controller-manager
```

### Observed Behaviour

```bash
# Watch lease holder change in real time
watch -n 2 kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='{.spec.holderIdentity}'

# New holder: master_2_ae962434-bb2b-4cbf-ae27-588066e18208
```

Lease object confirmed:
```yaml
spec:
  acquireTime: "2026-06-13T15:07:25.043970Z"
  holderIdentity: master-2_ae962434-bb2b-4cbf-ae27-588066e18208
  leaseDurationSeconds: 15
  leaseTransitions: 5
  renewTime: "2026-06-13T15:10:05.625507Z"
```

Workload continuity verified — scaled the `alpha-earth` deployment from 2 to 4 replicas *during* the leader transition:

```bash
kubectl scale deployment alpha-earth --replicas=4 -n valorant
# deployment.apps/alpha-earth scaled — all 4 pods Running within 32s
```

### Outcome

✅ **PASS** — New leader elected in ~15 seconds. Deployments scaled successfully during transition. `leaseTransitions: 5` confirms multiple previous elections, demonstrating stable, repeatable behaviour.

---

## Test 4 — Scheduler Leader Failover

### What It Validates
Same leader election pattern as Test 3, applied to `kube-scheduler`. The scheduler must recover and immediately be capable of placing new pods.

### Procedure

```bash
# Check current scheduler lease
kubectl get lease -n kube-system kube-scheduler \
  -o jsonpath='{.spec.holderIdentity}'
# Output: master-1_e8e79ffc-7c8f-41c4-a1a7-d075d4a9892b

# Stop scheduler on master-1 (current leader)
systemctl stop kube-scheduler
```

### Observed Behaviour

- Scheduler lease transitioned to **master-2** within ~15 seconds
- Pod scheduling was tested immediately by deleting a running pod and verifying rescheduling:

```bash
kubectl delete pod alpha-earth-59b7f9cdcf-fnzrh -n valorant
# pod deleted

kubectl get pod -n valorant -w
# New pod alpha-earth-59b7f9cdcf-hd7gl  →  Running  (18s)
```

New scheduler lease:
```
master-2_74df131b-aecb-4a08-8e66-ddb02fa63b16
```

### Outcome

✅ **PASS** — New scheduler leader elected in ~15 seconds. Pod rescheduled successfully by the new leader within 18 seconds of the delete. No scheduling backlog observed.

---

## Test 5 — Full Master Node Failure (Process Level)

### What It Validates
All four control plane services on a single master node fail simultaneously. This is the most severe single-node failure scenario. The cluster must maintain etcd quorum and continue serving API traffic through HAProxy.

### Procedure

```bash
# On master-3: stop all control plane services at once
systemctl stop kube-apiserver kube-controller-manager kube-scheduler etcd
```

### Observed Behaviour

**etcd health (2/3 members remaining):**
```
master-1:2379  →  IS LEADER: false  |  RAFT INDEX: 108494  — healthy
master-2:2379  →  IS LEADER: true   |  RAFT INDEX: 108494  — healthy
```
Both surviving etcd members committed proposals successfully within ~13–18ms.

**HAProxy stats:**
```
k8s-masters,master-3,...,DOWN,100,1,3,2,201,228,256,1,3,...,L4CON,Connection refused,...
```
HAProxy immediately marked master-3 as `DOWN` and stopped routing traffic to it.

**Cluster state from master-1:**
```
NAME       STATUS   ROLES           AGE    VERSION
master-1   Ready    control-plane   7d9h   v1.34.0
master-2   Ready    control-plane   7d9h   v1.34.0
master-3   Ready    control-plane   7d9h   v1.34.0   ← still shows Ready
worker-1   Ready    worker          7d9h   v1.34.0
```

> Note: `master-3` shows `Ready` because node status is reported via the `kubelet` heartbeat — kubelet was not stopped, only the control plane services. The API servers on master-1 and master-2 continued accepting all cluster traffic.

### ⚠️ Recovery Order for Full Node Restart

When restarting services on the failed node, **always start etcd first**, then the API server, then controller-manager and scheduler. Starting the API server before etcd causes it to fail with connection refused.

```bash
systemctl start etcd
sleep 5
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

### Outcome

✅ **PASS** — Total process-level failure of one master caused zero disruption to the running cluster. HAProxy traffic rerouted immediately. etcd quorum held at 2/3. API traffic served by remaining two masters.

---

## Test 6 — HAProxy Restart (Load Balancer Interruption)

### What It Validates
A brief HAProxy outage (the sole external entry point) does not lose cluster state or disrupt running workloads. In-cluster traffic (pod-to-pod, service DNS) must be completely unaffected. `kubectl` must reconnect within seconds of HAProxy coming back up.

### Procedure

```bash
# Simulate continuous traffic to a ClusterIP service
kubectl run traffic-gen --image=busybox:1.28 --restart=Never -- \
  sh -c "while true; do wget -qO- http://ha-test.default.svc.cluster.local && sleep 1; done"

# Restart HAProxy
systemctl restart haproxy
```

### Observed Behaviour

- `kubectl get nodes --request-timeout=30s` returned in **0.079 seconds real time** after HAProxy was back up
- HAProxy stats immediately restored: all 3 masters showing `UP 100`, backend `UP 300`

```
k8s-masters  master-1  UP  100
k8s-masters  master-2  UP  100
k8s-masters  master-3  UP  100
k8s-masters  BACKEND   UP  300
```

- In-cluster traffic (pod-to-ClusterIP) was **never interrupted** — HAProxy only handles external/kubectl traffic; the in-cluster `kube-proxy` + Calico data plane remains unaffected
- The `traffic-gen` pod continued its wget loop without errors throughout the restart

### Outcome

✅ **PASS** — HAProxy restart caused sub-10-second kubectl disruption. In-cluster workloads were completely unaffected. All backends restored immediately on HAProxy start.

---

## ⚡ Quick Reference — Recovery Timelines

| Service Stopped | Node | Command | Expected Recovery |
|---|---|---|---|
| `kube-apiserver` | Any 1 master | `systemctl stop kube-apiserver` | HAProxy reroutes in ~10s |
| `etcd` | Any 1 master | `systemctl stop etcd` | Quorum holds (2/3), auto-syncs on restart |
| `kube-controller-manager` | Leader master | `systemctl stop kube-controller-manager` | New leader elected in ~15s |
| `kube-scheduler` | Leader master | `systemctl stop kube-scheduler` | New leader elected in ~15s |
| All 4 services | Any 1 master | `stop` all 4 above | Cluster continues; restart etcd first |
| `kubelet` | Any worker | `systemctl stop kubelet` | Node `NotReady` in ~40s; pods evict in ~5m |
| `haproxy` | HAProxy host | `systemctl restart haproxy` | `kubectl` reconnects in <10s |

---

## 📚 Key Learnings

### 1. HAProxy Health Checks Are Critical Path
HAProxy's Layer4 TCP health checks on port 6443 are what make API server failover fast. The default check interval and fall/rise thresholds directly control how quickly a dead backend is removed from rotation. With our configuration, failover occurred in ~10 seconds — well within acceptable operational tolerance.

### 2. etcd Quorum Is a Hard Boundary
With 3 etcd members, the fault tolerance is exactly 1. This is non-negotiable — losing 2 members simultaneously puts the entire cluster into read-only mode. For clusters requiring higher fault tolerance (losing 2 of N), a 5-member etcd cluster is required (tolerates 2 failures). For this cluster size and use case, 3 members is the correct trade-off.

### 3. Leader Election Is Passive-Standby, Not Active-Active
`kube-controller-manager` and `kube-scheduler` run on all masters, but only one instance is **active** at any time. The others are in a hot-standby loop, periodically trying to acquire the Kubernetes `Lease` object. The lease duration (15s) and renewal period are what bound the maximum failover time. This architecture means there is a real (but bounded) window of no controller/scheduler activity during failover.

### 4. Node Status and Process Status Are Decoupled
Stopping control plane processes (`kube-apiserver`, `etcd`, etc.) on a node does **not** immediately show that node as `NotReady`. Node status reflects **kubelet** liveness. A node can appear `Ready` while its control plane services are completely down — this is expected behaviour and not a bug. Operators must monitor component health independently of node status for control plane nodes.

### 5. Restart Order Matters for Full Recovery
After a full master node failure, etcd **must** be started before `kube-apiserver`. The API server makes an immediate connection to etcd on startup — if etcd is not yet ready, the API server exits and requires a restart. Systemd `Requires=` and `After=` directives in the unit files should enforce this, but manual restarts should always follow the order: `etcd → kube-apiserver → kube-controller-manager → kube-scheduler`.

### 6. In-Cluster Traffic Is Resilient to HAProxy Loss
The Kubernetes data plane (pod networking, `kube-proxy` iptables rules, Calico overlays) is entirely independent of the control plane load balancer. HAProxy going down only affects external API access (`kubectl`, CI/CD pipelines, admission webhooks). Running workloads are completely unaffected. This is an important operational distinction: HAProxy is a control plane concern, not a data plane concern.

---

## ✅ Conclusion

All six HA validation tests passed against this manually assembled multi-master Kubernetes cluster. The cluster demonstrates:

- **Control plane resilience** — any single master node can fail without service interruption
- **etcd quorum integrity** — fault boundary of 1/3 members correctly observed
- **Automatic leader election** — controller-manager and scheduler failover within 15 seconds
- **HAProxy reliability** — load balancer can be restarted with minimal impact to active operations
- **Workload continuity** — running pods survive all tested failure scenarios without eviction or disruption

This cluster is validated as production-grade for single-node-failure scenarios. For multi-node failure tolerance, scaling to 5 master nodes and a 5-member etcd cluster would be the next architectural step.

---

*learning project — extended for AWS multi-master HA on the path to CKA certification.*
