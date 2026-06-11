# Kubernetes The Hard Way — Multi-Master HA Production Cluster

> **Target Environment:** Ubuntu 22.04 LTS Servers (Production)
> **Security Model:** Only HAProxy is public-facing. All cluster traffic is internal.
> **Kubernetes Version:** v1.34.0 | **etcd:** v3.5.21 | **Calico:** v3.31.0

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Node Reference Table](#node-reference-table)
- [Security Hardening Principles](#security-hardening-principles)
- [Phase 0 — HAProxy Load Balancer](#phase-0--haproxy-load-balancer)
- [Phase 1 — PKI & Certificate Authority](#phase-1--pki--certificate-authority)
  - [Section 1 — Root CA](#section-1--master-1-generate-root-ca)
  - [Section 2 — etcd Peer Certificates](#section-2--master-1-etcd-peer-tls-certificates)
  - [Section 3 — API Server Certificates](#section-3--master-1-kube-apiserver-certificates)
  - [Section 4 — Component Certificates](#section-4--master-1-component-certificates)
  - [Section 5 — Distribute Certs](#section-5--master-1-distribute-certificates-to-master-2-and-master-3)
- [Phase 2 — etcd Cluster (All 3 Masters)](#phase-2--etcd-cluster)
  - [Section 6 — Download Binaries](#section-6--all-masters-download-binaries)
  - [Section 7 — Install etcd](#section-7--all-masters-install--configure-etcd)
  - [Section 8 — Verify etcd Cluster](#section-8--master-1-verify-etcd-cluster-health)
- [Phase 3 — Control Plane (All 3 Masters)](#phase-3--control-plane)
  - [Section 9 — kube-apiserver](#section-9--all-masters-kube-apiserver)
  - [Section 10 — kube-controller-manager](#section-10--all-masters-kube-controller-manager)
  - [Section 11 — kube-scheduler](#section-11--all-masters-kube-scheduler)
  - [Section 12 — Admin kubeconfig](#section-12--master-1-admin-kubeconfig)
  - [Section 13 — containerd Runtime](#section-13--all-masters-containerd-runtime)
  - [Section 14 — kubelet on Masters](#section-14--all-masters-kubelet)
  - [Section 15 — kube-proxy on Masters](#section-15--all-masters-kube-proxy)
- [Phase 4 — Worker Nodes](#phase-4--worker-nodes)
  - [Section 16 — Worker Certificates & Setup](#section-16--workers-setup)
- [Phase 5 — Networking](#phase-5--networking)
  - [Section 17 — Calico CNI](#section-17--master-1-install-calico-cni)
  - [Section 18 — CoreDNS](#section-18--master-1-install-coredns)
  - [Section 19 — Kubernetes Gateway API](#section-19--master-1-install-kubernetes-gateway-api)
  - [Section 20 — NGINX Fabric Gateway Controller](#section-20--master-1-install-nginx-fabric-gateway-controller)
- [Phase 6 — Security Hardening](#phase-6--security-hardening)
  - [Section 21 — RBAC Policies](#section-21--master-1-rbac-policies)
  - [Section 22 — Network Policies](#section-22--master-1-network-policies)
  - [Section 23 — Pod Security Admission](#section-23--master-1-pod-security-admission)
  - [Section 24 — Node Firewall (UFW)](#section-24--all-nodes-ufw-firewall-rules)
- [Phase 7 — Verification](#phase-7--verification)

---

## Architecture Overview

```
Internet
   │
   ▼
[HAProxy]  ◄── Only public-facing node (port 6443)
   │            IP: <HAPROXY_PUBLIC_IP>
   │            Internal: <HAPROXY_IP>
   │
   ├──► [master-1]  etcd + control plane  <MASTER1_IP>
   ├──► [master-2]  etcd + control plane  <MASTER2_IP>
   └──► [master-3]  etcd + control plane  <MASTER3_IP>
              │
              ▼
   [worker-1] [worker-2] [worker-3] [worker-4] [worker-5]
   <WRK1_IP>  <WRK2_IP>  <WRK3_IP>  <WRK4_IP>  <WRK5_IP>
```

**Traffic Flow:**
- External `kubectl` → HAProxy:6443 → master-{1,2,3}:6443 (round-robin)
- Internal pod traffic → Calico CNI (BGP mesh)
- Service traffic → kube-proxy (iptables) → pods
- Ingress HTTP/HTTPS → NGINX Fabric Gateway Controller → backend services

---

## Node Reference Table

| Role     | Hostname  | Private IP        | Public IP      |
|----------|-----------|-------------------|----------------|
| HAProxy  | haproxy   | `<HAPROXY_IP>`    | `<PUBLIC_IP>`  |
| Master 1 | master-1  | `<MASTER1_IP>`    | none           |
| Master 2 | master-2  | `<MASTER2_IP>`    | none           |
| Master 3 | master-3  | `<MASTER3_IP>`    | none           |
| Worker 1 | worker-1  | `<WRK1_IP>`       | none           |
| Worker 2 | worker-2  | `<WRK2_IP>`       | none           |
| Worker 3 | worker-3  | `<WRK3_IP>`       | none           |
| Worker 4 | worker-4  | `<WRK4_IP>`       | none           |
| Worker 5 | worker-5  | `<WRK5_IP>`       | none           |

> Replace all `<PLACEHOLDER>` values throughout this guide with your actual IPs.

**CIDRs used in this guide:**
- Pod CIDR: `192.168.0.0/16` (Calico)
- Service CIDR: `10.96.0.0/12`
- ClusterDNS IP: `10.96.0.10`
- API Server ClusterIP: `10.96.0.1`

---

## Security Hardening Principles

Before beginning, understand these core security decisions baked into this setup:

1. **No public IPs on masters/workers** — only HAProxy is internet-reachable.
2. **Mutual TLS everywhere** — every component authenticates via x509 certificates.
3. **Encryption at rest** — Secrets in etcd are AES-CBC encrypted.
4. **etcd peer auth** — separate peer certs prevent unauthorized nodes joining etcd.
5. **Least-privilege RBAC** — all components use dedicated service accounts.
6. **Audit logging** — all API server requests are logged with full metadata.
7. **Admission controllers** — NodeRestriction, ResourceQuota, PodSecurity enforced.
8. **UFW firewalls** — all nodes allow only required ports from required sources.
9. **`ca.key` never leaves master-1** — workers sign certs on master-1 and `ca.key` is never copied.
10. **Leader election** — controller-manager and scheduler use built-in HA leader election.

---

## Phase 0 — HAProxy Load Balancer

> **Run on:** `haproxy` node as `root`
> **Why:** HAProxy is the single public entry point. It terminates no TLS (TCP pass-through mode) so master nodes see real client certificates. This enables mutual TLS all the way through.

```bash
# Step 1 — Become root
sudo su

# Step 2 — Update and install HAProxy
apt update -y
apt install -y haproxy

# Step 3 — Back up the default config
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# Step 4 — Write the HAProxy configuration
# Replace all <MASTER_IP> placeholders with actual master private IPs
# TCP mode (layer 4) passes TLS through unchanged — masters do TLS termination
vi /etc/haproxy/haproxy.cfg
```

```ini
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 50000

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# --- Kubernetes API Server ---
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server master-1 <MASTER1_IP>:6443 check
    server master-2 <MASTER2_IP>:6443 check
    server master-3 <MASTER3_IP>:6443 check

# --- HAProxy Stats Page (internal only) ---
frontend stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
```
```bash

# Step 5 — Validate the config before applying
haproxy -c -f /etc/haproxy/haproxy.cfg

# Step 6 — Enable and start HAProxy
systemctl enable haproxy
systemctl restart haproxy
systemctl status haproxy

# Step 7 — Confirm HAProxy is listening on 6443
ss -tlnp | grep 6443
```

### HAProxy TLS Hardening

```bash
# Step 8 — Restrict stats page to localhost only (already done above)
# Additional: enable HAProxy logging to file for audit trail
mkdir -p /var/log/haproxy

cat > /etc/rsyslog.d/49-haproxy.conf <<'EOF'
$AddUnixListenSocket /var/lib/haproxy/dev/log
:programname, startswith, "haproxy" {
  /var/log/haproxy/haproxy.log
  stop
}
EOF

systemctl restart rsyslog
```

---

## Phase 1 — PKI & Certificate Authority

> **All PKI work runs on:** `master-1` as `root`
> **Why:** Centralising cert generation on one node is the secure hard-way approach. Certs are then distributed via `scp`. `ca.key` is **never** copied to any other node.

---

## SECTION 1 — master-1: Generate Root CA

> The CA is the root of trust for the entire cluster. Every certificate is signed by this CA.

```bash
# Step 9 — Become root
sudo su

# Step 10 — Create the PKI directory
mkdir -p /etc/kubernetes/pki
cd /etc/kubernetes/pki

# Step 11 — Generate the CA private key (4096-bit for production)
openssl genrsa -out ca.key 4096

# Step 12 — Generate the self-signed CA certificate (valid 10 years for production)
openssl req -x509 -new -noenc -key ca.key \
  -subj "/CN=KUBERNETES-CA/O=Kubernetes" \
  -days 3650 -out ca.crt

# Step 13 — Lock down CA key permissions — only root should read this
chmod 600 /etc/kubernetes/pki/ca.key

# Step 14 — Verify the CA cert
openssl x509 -noout -text -in ca.crt | grep -E "Subject:|Validity|Not After"

# Step 14 - Copy this ca.crt and ca.key on other masters under directory
/etc/kubernetes/pki
```

---

## SECTION 2 — master-1: etcd Peer TLS Certificates

> etcd requires two types of certs: **server** (client-to-etcd TLS) and **peer** (etcd-to-etcd TLS).
> In a 3-node etcd cluster, peers authenticate each other via mTLS — this prevents rogue nodes joining.
> We generate ONE cert per master node, with that node's IP and hostname in the SANs.

### etcd Certificate for master-1

```bash
# Step 15 — Create etcd PKI directory
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd

# Step 16 — Generate etcd server/peer private key for master-1
openssl genrsa -out master-1.key 2048

# Step 17 — Create CSR config for master-1 etcd cert
# This cert covers both server auth (clients→etcd) and peer auth (etcd↔etcd)
# Replace <MASTER1_IP> with master-1's private IP
vi master-1-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = etcd

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = master-1
DNS.2 = localhost
IP.1  = <MASTER1_IP>
IP.2  = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = CA:FALSE
keyUsage               = keyEncipherment,dataEncipherment,digitalSignature
extendedKeyUsage       = serverAuth,clientAuth
subjectAltName         = @alt_names
```
```bash

# Step 18 — Generate CSR and sign with CA
openssl req -new -key master-1.key -out master-1.csr -config master-1-csr.conf

openssl x509 -req -in master-1.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out master-1.crt \
  -days 1825 \
  -extensions v3_ext -extfile master-1-csr.conf -sha256

rm -f master-1.csr

openssl x509 -noout -text -in master-1.crt | grep -A6 "Subject Alternative"
```

### etcd Certificate for master-2

```bash
# Step 19 — Generate etcd cert for master-2 
# Replace <MASTER2_IP> with master-2's private IP
mkdir -p /etc/kubernetes/pki/etcd

cd /etc/kubernetes/pki/etcd

openssl genrsa -out master-2.key 2048

vi master-2-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = etcd

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = master-2
DNS.2 = localhost
IP.1  = <MASTER2_IP>
IP.2  = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = CA:FALSE
keyUsage               = keyEncipherment,dataEncipherment,digitalSignature
extendedKeyUsage       = serverAuth,clientAuth
subjectAltName         = @alt_names
```
```bash

openssl req -new -key master-2.key -out master-2.csr -config master-2-csr.conf

openssl x509 -req -in master-2.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out master-2.crt \
  -days 1825 \
  -extensions v3_ext -extfile master-2-csr.conf -sha256

rm -f master-2.csr
```

### etcd Certificate for master-3

```bash
# Step 20 — Generate etcd cert for master-3
# Replace <MASTER3_IP> with master-3's private IP
mkdir -p /etc/kubernetes/pki/etcd

cd /etc/kubernetes/pki/etcd

openssl genrsa -out master-3.key 2048

vi master-3-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = etcd

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = master-3
DNS.2 = localhost
IP.1  = <MASTER3_IP>
IP.2  = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = CA:FALSE
keyUsage               = keyEncipherment,dataEncipherment,digitalSignature
extendedKeyUsage       = serverAuth,clientAuth
subjectAltName         = @alt_names
```
```bash

openssl req -new -key master-3.key -out master-3.csr -config master-3-csr.conf

openssl x509 -req -in master-3.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out master-3.crt \
  -days 1825 \
  -extensions v3_ext -extfile master-3-csr.conf -sha256

rm -f master-3.csr
```

---

## SECTION 3 — master-1: kube-apiserver Certificates

> The API server cert must include SANs for ALL master IPs, the HAProxy IP (the VIP that workers use),
> and the Kubernetes service ClusterIP (10.96.0.1). This lets clients connect via any path.

```bash
# Step 21 — Create apiserver cert directory
mkdir -p /etc/kubernetes/pki/kube-apiserver
cd /etc/kubernetes/pki/kube-apiserver

# Step 22 — Generate apiserver private key
openssl genrsa -out apiserver.key 2048

# Step 23 — Create CSR config — include ALL master IPs, HAProxy IP, localhost, and service IP
# Replace all <IP> placeholders
vi apiserver-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = kube-apiserver

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1  = <MASTER1_IP>
IP.2  = <MASTER2_IP>
IP.3  = <MASTER3_IP>
IP.4  = <HAPROXY_IP>
IP.5  = 127.0.0.1
IP.6  = 10.96.0.1
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local

[ v3_ext ]
authorityKeyIdentifier = keyid,issuer
basicConstraints       = CA:FALSE
keyUsage               = critical,digitalSignature,keyEncipherment
extendedKeyUsage       = serverAuth
subjectAltName         = @alt_names
```
```bash

# Step 24 — Sign the apiserver certificate
openssl req -new -key apiserver.key -out apiserver.csr -config apiserver-csr.conf

openssl x509 -req -in apiserver.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out apiserver.crt \
  -days 1825 \
  -extensions v3_ext -extfile apiserver-csr.conf

rm -f apiserver.csr

# Step 25 — Generate service account signing key pair
# Used by apiserver to sign ServiceAccount tokens (pods use these to call the API)
openssl genrsa -out sa.key 2048

openssl rsa -in sa.key -pubout -out sa.pub

# Step 26 — Generate apiserver-kubelet-client cert
# apiserver uses this when calling kubelet endpoints (logs, exec, port-forward)
openssl genrsa -out apiserver-kubelet-client.key 2048

vi apiserver-kubelet-client-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = kube-apiserver-kubelet-client
O  = system:masters

[ v3_req ]
keyUsage         = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
```
```bash

openssl req -new -key apiserver-kubelet-client.key \
  -out apiserver-kubelet-client.csr \
  -config apiserver-kubelet-client-csr.conf

openssl x509 -req -in apiserver-kubelet-client.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out apiserver-kubelet-client.crt \
  -days 1825 \
  -extensions v3_req -extfile apiserver-kubelet-client-csr.conf

rm -f apiserver-kubelet-client.csr
```

### Encryption at Rest Configuration

```bash
# Step 27 — Generate a 256-bit AES encryption key for etcd secrets encryption
# Store this key securely — losing it means losing access to all encrypted secrets
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "Save this key securely: $ENCRYPTION_KEY"

# Step 28 — Create the encryption config file
vi /etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml
```

```ini
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
```
```bash

chmod 600 /etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml
```

### Audit Policy Configuration

```bash
# Step 29 — Create a comprehensive audit policy
# Logs metadata for all requests; full RequestResponse for sensitive operations
mkdir -p /var/log/kubernetes

vi /etc/kubernetes/pki/kube-apiserver/audit-policy.yaml
```

```ini
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  # Log secret/configmap access at full RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  # Log pod exec/attach/port-forward at Metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  # Log all other requests at Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
```

---

## SECTION 4 — master-1: Component Certificates

> Each Kubernetes component authenticates to the API server using its own dedicated client certificate.
> These are NOT copied to other masters — each master generates its own component certs in Phase 3.
> Here we only generate the certs that need to be identical across masters (shared by design).

### kube-controller-manager Certificate

```bash
# Step 30 — Create controller-manager cert
mkdir -p /etc/kubernetes/pki/kube-controller-manager

cd /etc/kubernetes/pki/kube-controller-manager

openssl genrsa -out kube-controller-manager.key 2048

cat > kube-controller-manager-csr.conf <<EOF
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = system:kube-controller-manager

[ v3_req ]
keyUsage         = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
EOF

openssl req -new -key kube-controller-manager.key \
  -out kube-controller-manager.csr \
  -config kube-controller-manager-csr.conf

openssl x509 -req -in kube-controller-manager.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kube-controller-manager.crt \
  -days 1825 \
  -extensions v3_req -extfile kube-controller-manager-csr.conf

rm -f kube-controller-manager.csr
```

### kube-scheduler Certificate

```bash
# Step 31 — Create scheduler cert
mkdir -p /etc/kubernetes/pki/kube-scheduler
cd /etc/kubernetes/pki/kube-scheduler
openssl genrsa -out kube-scheduler.key 2048

vi kube-scheduler-csr.conf 
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = system:kube-scheduler

[ v3_req ]
keyUsage         = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
```
```bash

openssl req -new -key kube-scheduler.key \
  -out kube-scheduler.csr \
  -config kube-scheduler-csr.conf

openssl x509 -req -in kube-scheduler.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kube-scheduler.crt \
  -days 1825 \
  -extensions v3_req -extfile kube-scheduler-csr.conf

rm -f kube-scheduler.csr
```

### kube-proxy Certificate

```bash
# Step 32 — Create a dedicated kube-proxy client cert
mkdir -p /etc/kubernetes/pki/kube-proxy
cd /etc/kubernetes/pki/kube-proxy
openssl genrsa -out kube-proxy.key 2048

vi kube-proxy-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = system:kube-proxy
O  = system:node-proxier

[ v3_req ]
keyUsage         = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
```
```bash

openssl req -new -key kube-proxy.key \
  -out kube-proxy.csr -config kube-proxy-csr.conf

openssl x509 -req -in kube-proxy.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kube-proxy.crt \
  -days 1825 \
  -extensions v3_req -extfile kube-proxy-csr.conf

rm -f kube-proxy.csr
```

### Admin Certificate

```bash
# Step 33 — Create the admin user cert (gives full cluster access)
mkdir -p /etc/kubernetes/pki/admin
cd /etc/kubernetes/pki/admin
openssl genrsa -out admin.key 2048

vi admin-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn

[ dn ]
CN = admin
O  = system:masters
```
```bash

openssl req -new -key admin.key -out admin.csr -config admin-csr.conf

openssl x509 -req -in admin.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out admin.crt \
  -days 1825

rm -f admin.csr
```

---

## SECTION 5 — master-1: Distribute Certificates to master-2 and master-3

> All shared PKI material is pushed from master-1 to the other masters.
> `ca.key` is NEVER distributed — only the public CA cert is shared.
> master-2 and master-3 will use the distributed certs as-is and add their own etcd peer certs.

```bash
# Step 34 — Create PKI directory structure on master-2 and master-3
# Run from master-1; replace IPs with actual values
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  ssh root@${MASTER_IP} "mkdir -p \
    /etc/kubernetes/pki/etcd \
    /etc/kubernetes/pki/kube-apiserver \
    /etc/kubernetes/pki/kube-controller-manager \
    /etc/kubernetes/pki/kube-scheduler \
    /etc/kubernetes/pki/kube-proxy \
    /var/log/kubernetes"
done

# Step 35 — Distribute CA cert (NOT the key) to all masters
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/ca.crt root@${MASTER_IP}:/etc/kubernetes/pki/ca.crt
done

# Step 36 — Distribute etcd certs — each master gets all 3 peer certs
# (etcd needs to know all peer certs for mTLS verification)
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/etcd/master-1.crt \
      /etc/kubernetes/pki/etcd/master-1.key \
      /etc/kubernetes/pki/etcd/master-2.crt \
      /etc/kubernetes/pki/etcd/master-2.key \
      /etc/kubernetes/pki/etcd/master-3.crt \
      /etc/kubernetes/pki/etcd/master-3.key \
      root@${MASTER_IP}:/etc/kubernetes/pki/etcd/
done

# Step 37 — Distribute apiserver certs (same cert on all masters — all IPs are in the SANs)
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/kube-apiserver/apiserver.crt \
      /etc/kubernetes/pki/kube-apiserver/apiserver.key \
      /etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.crt \
      /etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.key \
      /etc/kubernetes/pki/kube-apiserver/sa.key \
      /etc/kubernetes/pki/kube-apiserver/sa.pub \
      /etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml \
      /etc/kubernetes/pki/kube-apiserver/audit-policy.yaml \
      root@${MASTER_IP}:/etc/kubernetes/pki/kube-apiserver/
done

# Step 38 — Distribute component certs
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.crt \
      /etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.key \
      root@${MASTER_IP}:/etc/kubernetes/pki/kube-controller-manager/

  scp /etc/kubernetes/pki/kube-scheduler/kube-scheduler.crt \
      /etc/kubernetes/pki/kube-scheduler/kube-scheduler.key \
      root@${MASTER_IP}:/etc/kubernetes/pki/kube-scheduler/

  scp /etc/kubernetes/pki/kube-proxy/kube-proxy.crt \
      /etc/kubernetes/pki/kube-proxy/kube-proxy.key \
      root@${MASTER_IP}:/etc/kubernetes/pki/kube-proxy/
done

# Step 39 — Lock down file permissions on all masters
for MASTER_IP in <MASTER1_IP> <MASTER2_IP> <MASTER3_IP>; do
  ssh root@${MASTER_IP} "find /etc/kubernetes/pki -name '*.key' -exec chmod 600 {} \;"
  ssh root@${MASTER_IP} "find /etc/kubernetes/pki -name '*.crt' -exec chmod 644 {} \;"
done
```

---

## Phase 2 — etcd Cluster

---

## SECTION 6 — All Masters: Download Binaries

> **Run on:** `master-1`, `master-2`, `master-3` as `root` (run the same commands on each)
> **Why:** Each master runs all Kubernetes control plane binaries locally — no NFS, no shared storage.

```bash
# Step 40 — Run on EACH master node individually
sudo su
apt update -y

# Create binaries staging directory
mkdir -p /root/binaries
cd /root/binaries

# Download Kubernetes server binaries (apiserver, scheduler, controller-manager, kubelet, kubectl, kube-proxy)
wget https://dl.k8s.io/v1.34.0/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz

# Download etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.5.21/etcd-v3.5.21-linux-amd64.tar.gz
tar -xzvf etcd-v3.5.21-linux-amd64.tar.gz

# Copy all Kubernetes binaries to system PATH
cp kubernetes/server/bin/kube-apiserver \
   kubernetes/server/bin/kube-controller-manager \
   kubernetes/server/bin/kube-scheduler \
   kubernetes/server/bin/kubelet \
   kubernetes/server/bin/kube-proxy \
   kubernetes/server/bin/kubectl \
   /usr/local/bin/

# Copy etcd binaries
cp etcd-v3.5.21-linux-amd64/etcd \
   etcd-v3.5.21-linux-amd64/etcdctl \
   etcd-v3.5.21-linux-amd64/etcdutl \
   /usr/local/bin/

# Verify versions
kube-apiserver --version
etcd --version
```

---

## SECTION 7 — All Masters: Install & Configure etcd

> **Why:** etcd runs as a 3-node cluster where any 1 node can fail and the cluster remains operational (quorum = 2 of 3).
> Each node's etcd service must know ALL peers via `--initial-cluster`.
> All peer communication is authenticated via mTLS using the per-node etcd certs.

### Create Data Directory on All Masters

```bash
# Step 41 — Run on EACH master: create etcd data directory
mkdir -p /var/lib/etcd
chmod 700 /var/lib/etcd
```

### etcd Service on master-1

> **Run on:** `master-1`

```bash
# Step 42 — Create the etcd systemd service on master-1
# Replace all <IP> values with actual private IPs
vi /etc/systemd/system/etcd.service
```

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/v3.5/op-guide/configuration/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name master-1 \
  --cert-file=/etc/kubernetes/pki/etcd/master-1.crt \
  --key-file=/etc/kubernetes/pki/etcd/master-1.key \
  --peer-cert-file=/etc/kubernetes/pki/etcd/master-1.crt \
  --peer-key-file=/etc/kubernetes/pki/etcd/master-1.key \
  --trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-client-cert-auth=true \
  --client-cert-auth=true \
  --initial-advertise-peer-urls https://<MASTER1_IP>:2380 \
  --listen-peer-urls https://<MASTER1_IP>:2380 \
  --listen-client-urls https://<MASTER1_IP>:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://<MASTER1_IP>:2379 \
  --initial-cluster-token etcd-cluster-prod-01 \
  --initial-cluster master-1=https://<MASTER1_IP>:2380,master-2=https://<MASTER2_IP>:2380,master-3=https://<MASTER3_IP>:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd \
  --auto-compaction-retention=1 \
  --snapshot-count=10000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable etcd
# DO NOT start etcd yet — start all 3 nodes near-simultaneously in Step 45
```

### etcd Service on master-2

> **Run on:** `master-2`

```bash
# Step 43 — Create etcd service on master-2
# Replace <IP> values
vi /etc/systemd/system/etcd.service
```

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/v3.5/op-guide/configuration/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name master-2 \
  --cert-file=/etc/kubernetes/pki/etcd/master-2.crt \
  --key-file=/etc/kubernetes/pki/etcd/master-2.key \
  --peer-cert-file=/etc/kubernetes/pki/etcd/master-2.crt \
  --peer-key-file=/etc/kubernetes/pki/etcd/master-2.key \
  --trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-client-cert-auth=true \
  --client-cert-auth=true \
  --initial-advertise-peer-urls https://<MASTER2_IP>:2380 \
  --listen-peer-urls https://<MASTER2_IP>:2380 \
  --listen-client-urls https://<MASTER2_IP>:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://<MASTER2_IP>:2379 \
  --initial-cluster-token etcd-cluster-prod-01 \
  --initial-cluster master-1=https://<MASTER1_IP>:2380,master-2=https://<MASTER2_IP>:2380,master-3=https://<MASTER3_IP>:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd \
  --auto-compaction-retention=1 \
  --snapshot-count=10000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable etcd
```

### etcd Service on master-3

> **Run on:** `master-3`

```bash
# Step 44 — Create etcd service on master-3
# Replace <IP> values
vi /etc/systemd/system/etcd.service
```

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/v3.5/op-guide/configuration/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name master-3 \
  --cert-file=/etc/kubernetes/pki/etcd/master-3.crt \
  --key-file=/etc/kubernetes/pki/etcd/master-3.key \
  --peer-cert-file=/etc/kubernetes/pki/etcd/master-3.crt \
  --peer-key-file=/etc/kubernetes/pki/etcd/master-3.key \
  --trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-client-cert-auth=true \
  --client-cert-auth=true \
  --initial-advertise-peer-urls https://<MASTER3_IP>:2380 \
  --listen-peer-urls https://<MASTER3_IP>:2380 \
  --listen-client-urls https://<MASTER3_IP>:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://<MASTER3_IP>:2379 \
  --initial-cluster-token etcd-cluster-prod-01 \
  --initial-cluster master-1=https://<MASTER1_IP>:2380,master-2=https://<MASTER2_IP>:2380,master-3=https://<MASTER3_IP>:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd \
  --auto-compaction-retention=1 \
  --snapshot-count=10000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable etcd
```

### Start etcd on All Masters Simultaneously

```bash
# Step 45 — Start etcd on ALL THREE masters at roughly the same time
# etcd bootstraps as a cluster — all 3 nodes need to be up within the election timeout
# Open 3 terminal windows/tmux panes and run this on each master simultaneously:
systemctl start etcd

# Check status immediately after
systemctl status etcd
```

---

## SECTION 8 — master-1: Verify etcd Cluster Health

> **Run on:** `master-1`

```bash
# Step 46 — Check all 3 etcd members are registered
ETCDCTL_API=3 etcdctl \
  --endpoints=https://<MASTER1_IP>:2379,https://<MASTER2_IP>:2379,https://<MASTER3_IP>:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  member list

# Step 47 — Check health of all endpoints
ETCDCTL_API=3 etcdctl \
  --endpoints=https://<MASTER1_IP>:2379,https://<MASTER2_IP>:2379,https://<MASTER3_IP>:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  endpoint health

# Step 48 — Verify quorum with a write test
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  put /test/cluster "ha-ready"

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  get /test/cluster

# Expected output: ha-ready
```

> **Expected `member list` output:**
> ```
> <id>, started, master-1, https://<MASTER1_IP>:2380, https://<MASTER1_IP>:2379, false
> <id>, started, master-2, https://<MASTER2_IP>:2380, https://<MASTER2_IP>:2379, false
> <id>, started, master-3, https://<MASTER3_IP>:2380, https://<MASTER3_IP>:2379, false
> ```

---

## Phase 3 — Control Plane

---

## SECTION 9 — All Masters: kube-apiserver

> **Why:** Each master runs its own apiserver. All 3 talk to the local etcd (127.0.0.1:2379) but etcd is clustered, so all state is replicated. HAProxy routes external traffic to whichever apiserver is healthy.

### kube-apiserver on master-1

> **Run on:** `master-1`

```bash
# Step 49 — Create kubeconfigs for controller-manager, scheduler, and proxy on master-1
# These kubeconfigs point to the LOCAL apiserver (127.0.0.1) — they don't go through HAProxy

# kube-controller-manager kubeconfig
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.crt \
  --client-key=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-controller-manager \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig

# kube-scheduler kubeconfig
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/etc/kubernetes/pki/kube-scheduler/kube-scheduler.crt \
  --client-key=/etc/kubernetes/pki/kube-scheduler/kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-scheduler \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

# kube-proxy kubeconfig (points to HAProxy — workers use HAProxy to reach the API)
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<HAPROXY_IP>:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=/etc/kubernetes/pki/kube-proxy/kube-proxy.crt \
  --client-key=/etc/kubernetes/pki/kube-proxy/kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-proxy \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

# Step 50 — Distribute kubeconfigs to master-2 and master-3
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig \
    root@${MASTER_IP}:/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig

  scp /etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig \
    root@${MASTER_IP}:/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

  scp /etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig \
    root@${MASTER_IP}:/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig
done
```

### Create kube-apiserver Service — Run on EACH Master

> Run the following on **master-1**, **master-2**, and **master-3**.
> The only difference per node is `--advertise-address` — use that node's own IP.

```bash
# Step 51 — On master-1: create apiserver service
# Replace <THIS_MASTER_IP> with master-1's IP
# Replace all three etcd endpoints with actual IPs
vi /etc/systemd/system/kube-apiserver.service
```

```ini
[Unit]
Description=Kubernetes API Server
Documentation=https://kubernetes.io/docs/
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=<THIS_MASTER_IP> \
  --bind-address=0.0.0.0 \
  --apiserver-count=3 \
  --etcd-servers=https://<MASTER1_IP>:2379,https://<MASTER2_IP>:2379,https://<MASTER3_IP>:2379 \
  --etcd-cafile=/etc/kubernetes/pki/ca.crt \
  --etcd-certfile=/etc/kubernetes/pki/etcd/<THIS_MASTER_NAME>.crt \
  --etcd-keyfile=/etc/kubernetes/pki/etcd/<THIS_MASTER_NAME>.key \
  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --tls-cert-file=/etc/kubernetes/pki/kube-apiserver/apiserver.crt \
  --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver/apiserver.key \
  --tls-min-version=VersionTLS12 \
  --tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 \
  --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt \
  --kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.crt \
  --kubelet-client-key=/etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.key \
  --service-account-key-file=/etc/kubernetes/pki/kube-apiserver/sa.pub \
  --service-account-signing-key-file=/etc/kubernetes/pki/kube-apiserver/sa.key \
  --service-account-issuer=https://kubernetes.default.svc \
  --service-cluster-ip-range=10.96.0.0/12 \
  --service-node-port-range=30000-32767 \
  --authorization-mode=Node,RBAC \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,PodSecurity \
  --encryption-provider-config=/etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml \
  --event-ttl=1h \
  --audit-policy-file=/etc/kubernetes/pki/kube-apiserver/audit-policy.yaml \
  --audit-log-path=/var/log/kubernetes/apiserver-audit.log \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=10 \
  --audit-log-maxsize=100 \
  --runtime-config=api/all=true \
  --allow-privileged=true \
  --anonymous-auth=false \
  --profiling=false \
  --request-timeout=60s \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

# Step 52 — On master-1: start apiserver
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

> **On master-2:** Run Step 51 with `--advertise-address=<MASTER2_IP>` and `--etcd-certfile=.../etcd/master-2.crt`, `--etcd-keyfile=.../etcd/master-2.key`

> **On master-3:** Run Step 51 with `--advertise-address=<MASTER3_IP>` and `--etcd-certfile=.../etcd/master-3.crt`, `--etcd-keyfile=.../etcd/master-3.key`

---

## SECTION 10 — All Masters: kube-controller-manager

> **Why:** The controller manager uses leader election — only ONE of the 3 instances is active at a time. The other 2 are in standby and take over automatically if the leader fails. All 3 instances register as a Lease object in etcd.

```bash
# Step 53 — Create scheduler config (same on all masters)
vi /etc/kubernetes/pki/kube-scheduler/kube-scheduler.yaml
```

```ini
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig"
leaderElection:
  leaderElect: true
```
```bash

# Step 54 — Create controller-manager service (run on EACH master, content is identical)
vi /etc/systemd/system/kube-controller-manager.service
```

```ini
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://kubernetes.io/docs/
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig \
  --service-account-private-key-file=/etc/kubernetes/pki/kube-apiserver/sa.key \
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
  --root-ca-file=/etc/kubernetes/pki/ca.crt \
  --cluster-cidr=192.168.0.0/16 \
  --service-cluster-ip-range=10.96.0.0/12 \
  --cluster-name=production-cluster \
  --use-service-account-credentials=true \
  --leader-elect=true \
  --leader-elect-lease-duration=15s \
  --leader-elect-renew-deadline=10s \
  --leader-elect-retry-period=2s \
  --allocate-node-cidrs=true \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --profiling=false \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

> **IMPORTANT:** On master-2 and master-3, `--cluster-signing-key-file` must also point to `ca.key`.
> The CA key must be copied to master-2 and master-3 for the controller-manager to sign kubelet certs.
> This is acceptable because all master nodes are equally trusted and equally hardened.

```bash
# Step 55 — Copy ca.key to master-2 and master-3 (masters only — never to workers)
for MASTER_IP in <MASTER2_IP> <MASTER3_IP>; do
  scp /etc/kubernetes/pki/ca.key root@${MASTER_IP}:/etc/kubernetes/pki/ca.key
  ssh root@${MASTER_IP} "chmod 600 /etc/kubernetes/pki/ca.key"
done
```

---

## SECTION 11 — All Masters: kube-scheduler

> **Why:** Like the controller-manager, the scheduler uses leader election. Only one is active; the others are hot standby.

```bash
# Step 56 — Create scheduler service (run on EACH master, content is identical)
vi /etc/systemd/system/kube-scheduler.service 
```

```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://kubernetes.io/docs/
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/pki/kube-scheduler/kube-scheduler.yaml \
  --leader-elect=true \
  --profiling=false \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

---

## SECTION 12 — master-1: Admin kubeconfig

> **Run on:** `master-1`
> **Why:** The admin kubeconfig allows you to run `kubectl` commands. We configure it to point to HAProxy so it works even if master-1 is down.

```bash
# Step 57 — Create .kube directory
mkdir -p ~/.kube

# Step 58 — Create admin kubeconfig pointing to HAProxy
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<HAPROXY_IP>:6443 \
  --kubeconfig=$HOME/.kube/config

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/pki/admin/admin.crt \
  --client-key=/etc/kubernetes/pki/admin/admin.key \
  --embed-certs=true \
  --kubeconfig=$HOME/.kube/config

kubectl config set-context default \
  --cluster=production-cluster \
  --user=admin \
  --kubeconfig=$HOME/.kube/config

kubectl config use-context default --kubeconfig=$HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Step 59 — Verify API server access via HAProxy
kubectl get componentstatuses
kubectl cluster-info
```

---

## SECTION 13 — All Masters: containerd Runtime

> **Run on:** `master-1`, `master-2`, `master-3`
> **Why:** Kubernetes delegates container execution to containerd via the CRI. The systemd cgroup driver must match kubelet's cgroup driver setting.

```bash
# Step 60 — Install containerd and required kernel modules on EACH master
apt install -y containerd

# Load kernel modules required for pod networking
modprobe overlay
modprobe br_netfilter

# Persist modules across reboots
tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Enable IP forwarding and bridge netfilter — required for pod-to-pod and service traffic
tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1
EOF

sysctl --system

# Step 61 — Configure containerd with systemd cgroup driver
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
systemctl status containerd
```

---

## SECTION 14 — All Masters: kubelet

> **Run on:** `master-1`, `master-2`, `master-3`
> **Why:** kubelet runs on every node including masters. On masters it registers the node with the API server so master nodes can schedule critical system pods (Calico, CoreDNS).
> Each master gets its own kubelet cert with its own hostname and IP in the CN/SAN.

### Generate kubelet Cert — Run on Each Master With Its Own Details

```bash
# Step 62 — On EACH master, generate a kubelet cert for that node
# Run this block on master-1 first, then repeat on master-2/3 with correct values

# On master-1:
HOSTNAME=$(hostname)    # Should output: master-1
echo $HOSTNAME
NODE_IP=<MASTER_IP>   # Set to this master's private IP

mkdir -p /etc/kubernetes/pki/kubelet /var/lib/kubelet
cd /etc/kubernetes/pki/kubelet

openssl genrsa -out kubelet.key 2048

vi kubelet-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = system:node:$<HOSTNAME>
O  = system:nodes

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = <HOSTNAME>
IP.1  = <NODE_IP>
```
```bash

openssl req -new -key kubelet.key -out kubelet.csr -config kubelet-csr.conf

openssl x509 -req -in kubelet.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kubelet.crt \
  -days 1825 \
  -extensions v3_req -extfile kubelet-csr.conf

rm -f kubelet.csr

# Step 63 — Create kubelet kubeconfig pointing to LOCAL apiserver
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-credentials system:node:<HOSTNAME> \
  --client-certificate=/etc/kubernetes/pki/kubelet/kubelet.crt \
  --client-key=/etc/kubernetes/pki/kubelet/kubelet.key \
  --embed-certs=true \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:node:<HOSTNAME> \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config use-context default --kubeconfig=/var/lib/kubelet/kubeconfig

# Step 64 — Create kubelet config
# Replace <THIS_NODE_IP> with this master's private IP
vi /var/lib/kubelet/kubelet-config.yaml
```

```ini
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1

containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
cgroupDriver: systemd

authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"

authorization:
  mode: Webhook

clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"

tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet.key"

rotateCertificates: true
failSwapOn: false
protectKernelDefaults: true
readOnlyPort: 0
```
```bash

# Step 65 — Create kubelet systemd service
# Replace <THIS_MASTER_IP>
vi /etc/systemd/system/kubelet.service
```

```ini
[Unit]
Description=Kubelet
Documentation=https://kubernetes.io/docs/
After=network.target containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --register-node=true \
  --node-ip=<NODE_IP> \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

---

## SECTION 15 — All Masters: kube-proxy

> **Run on:** `master-1`, `master-2`, `master-3`
> **Why:** kube-proxy manages iptables rules that implement Service VIPs. It runs on masters too so master-to-service traffic (e.g., apiserver calling CoreDNS) is correctly routed.

```bash
# Step 66 — Create kube-proxy service on EACH master
# The kubeconfig was distributed from master-1 in Step 50
vi /etc/systemd/system/kube-proxy.service
```

```ini
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://kubernetes.io/docs/
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig \
  --cluster-cidr=192.168.0.0/16 \
  --proxy-mode=iptables \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy

# Step 67 — Verify master nodes are visible (may show NotReady until CNI installed)
kubectl get nodes
```

---

## Phase 4 — Worker Nodes

---

## SECTION 16 — Workers: Setup

> **Run all steps in this section on each worker node** (`worker-1` through `n - workers`)
> Worker nodes need: containerd, kubelet, and kube-proxy.
> Each worker gets a dedicated kubelet cert signed by the CA.
> **Cert signing MUST be done on master-1 — `ca.key` is never copied to workers.**

### Step A — Worker Node Base Setup (Run on Each Worker)

```bash
# Step 68 — On EACH worker: base setup
sudo su
apt update -y
apt install -y containerd

# Kernel modules and networking
modprobe overlay
modprobe br_netfilter

tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1
EOF

sysctl --system

# Configure containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

# Download Kubernetes node binaries
mkdir -p /root/binaries
cd /root/binaries

wget https://dl.k8s.io/v1.34.0/kubernetes-node-linux-amd64.tar.gz
tar -xzvf kubernetes-node-linux-amd64.tar.gz

cp kubernetes/node/bin/kubelet /usr/local/bin/
cp kubernetes/node/bin/kube-proxy /usr/local/bin/
cp kubernetes/node/bin/kubectl /usr/local/bin/

# Create PKI directories
mkdir -p /etc/kubernetes/pki/kubelet \
         /etc/kubernetes/pki/kube-proxy \
         /var/lib/kubelet
```

### Step B — Push CA Cert to Worker (Run on master-1)

```bash
# Step 69 — From master-1: copy CA cert (not key) to each worker
# Repeat for worker-2 through worker-5
scp /etc/kubernetes/pki/ca.crt root@<WORKER_IP>:/etc/kubernetes/pki/ca.crt
scp /etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig \
    root@<WORKER_IP>:/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig
```

### Step C — Generate Kubelet Cert on Each Worker (Run on Each Worker)

```bash
# Step 70 — On EACH worker: generate kubelet key and CSR
HOSTNAME=$(hostname)
NODE_IP=<THIS_WORKER_IP>

cd /etc/kubernetes/pki/kubelet

openssl genrsa -out kubelet.key 2048

vi kubelet-csr.conf
```

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = v3_req
distinguished_name = dn

[ dn ]
CN = system:node:<HOSTNAME>
O  = system:nodes

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = <HOSTNAME>
IP.1  = <NODE_IP>
```
```bash

openssl req -new -key kubelet.key -out kubelet.csr -config kubelet-csr.conf

# Step 71 — Securely transfer CSR to master-1 for signing
scp kubelet.csr root@<MASTER1_IP>:/tmp/${HOSTNAME}-kubelet.csr
```

### Step D — Sign Worker Cert on master-1 (Run on master-1)

```bash
# Step 72 — On master-1: sign the worker kubelet CSR
# Replace <WORKER_HOSTNAME> with the worker's actual hostname
WORKER_HOSTNAME=<WORKER_HOSTNAME>

openssl x509 -req \
  -in /tmp/${WORKER_HOSTNAME}-kubelet.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out /tmp/${WORKER_HOSTNAME}-kubelet.crt \
  -days 1825

# Transfer signed cert back to worker
scp /tmp/${WORKER_HOSTNAME}-kubelet.crt root@<WORKER_IP>:/etc/kubernetes/pki/kubelet/kubelet.crt

# Clean up temp files on master-1
rm -f /tmp/${WORKER_HOSTNAME}-kubelet.csr /tmp/${WORKER_HOSTNAME}-kubelet.crt
```

### Step E — Configure kubelet and kube-proxy (Run on Each Worker)

```bash
# Step 73 — On EACH worker: create kubelet kubeconfig
# Points to HAProxy — if one master is down, traffic routes to another
HOSTNAME=$(hostname)

kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<HAPROXY_IP>:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-credentials system:node:${HOSTNAME} \
  --client-certificate=/etc/kubernetes/pki/kubelet/kubelet.crt \
  --client-key=/etc/kubernetes/pki/kubelet/kubelet.key \
  --embed-certs=true \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:node:${HOSTNAME} \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config use-context default --kubeconfig=/var/lib/kubelet/kubeconfig

# Step 74 — Create kubelet config
vi /var/lib/kubelet/kubelet-config.yaml
```

```ini
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1

containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
cgroupDriver: systemd

authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"

authorization:
  mode: Webhook

clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"

tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet.key"

rotateCertificates: true
failSwapOn: false
protectKernelDefaults: true
readOnlyPort: 0
```
```bash

# Step 75 — Create kubelet service
NODE_IP=<THIS_WORKER_IP>

vi /etc/systemd/system/kubelet.service
```

```ini
[Unit]
Description=Kubelet
Documentation=https://kubernetes.io/docs/
After=network.target containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --register-node=true \
  --node-ip=${NODE_IP} \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

# Step 76 — Create kube-proxy service
vi /etc/systemd/system/kube-proxy.service
```

```ini
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig \
  --cluster-cidr=192.168.0.0/16 \
  --proxy-mode=iptables \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash

# Step 77 — Start all services on worker
systemctl daemon-reload
systemctl enable kubelet kube-proxy
systemctl start kubelet kube-proxy
systemctl status kubelet
systemctl status kube-proxy
```

### Verify All Nodes Joined

> **Run on:** `master-1`

```bash
# Step 78 — All 8 nodes (3 masters + 5 workers) should appear
# Status is NotReady until CNI (Calico) is installed
kubectl get nodes -o wide
```

---

## Phase 5 — Networking

---

## SECTION 17 — master-1: Install Calico CNI

> **Run on:** `master-1`
> **Why:** Without a CNI, all nodes stay in `NotReady` state. Calico assigns each pod a unique IP from the `192.168.0.0/16` pod CIDR and programs iptables/BGP for pod-to-pod routing.

```bash
# Step 79 — Apply the Calico operator (manages Calico lifecycle)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/tigera-operator.yaml

# Step 80 — Apply the Calico custom resources (configures pod CIDR to match our setup)
vi /etc/kubernetes/calico-installation.yaml
```

```ini
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 192.168.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
```
```bash

kubectl create -f /etc/kubernetes/calico-installation.yaml

# Step 81 — Watch Calico pods come up (wait until all are Running ~2-3 minutes)
kubectl get pods -n calico-system -w

# Step 82 — Verify all nodes are now Ready
kubectl get nodes -o wide
```

> **Expected output:**
> ```
> NAME       STATUS   ROLES    AGE   VERSION
> master-1   Ready    <none>   Xm    v1.34.0
> master-2   Ready    <none>   Xm    v1.34.0
> master-3   Ready    <none>   Xm    v1.34.0
> worker-1   Ready    <none>   Xm    v1.34.0
> ...
> ```

### Label Master Nodes

```bash
# Step 83 — Add control-plane role labels and taint masters so workloads don't schedule there
for NODE in master-1 master-2 master-3; do
  kubectl label node ${NODE} node-role.kubernetes.io/control-plane=
  kubectl taint node ${NODE} node-role.kubernetes.io/control-plane:NoSchedule
done

# Add worker labels
for i in 1 2 3 4 5; do
  kubectl label node worker-${i} node-role.kubernetes.io/worker=
done

kubectl get nodes
```

---

## SECTION 18 — master-1: Install CoreDNS

> **Run on:** `master-1`
> **Why:** CoreDNS provides in-cluster DNS. Pods resolve service names (e.g. `my-svc.default.svc.cluster.local`) via CoreDNS. The clusterIP `10.96.0.10` must match `clusterDNS` in every node's `kubelet-config.yaml`.

```bash
# Step 84 — Create CoreDNS directory and manifest
mkdir -p /etc/kubernetes/coredns
vi /etc/kubernetes/coredns/coredns.yaml
```

```ini
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
rules:
  - apiGroups: [""]
    resources: [endpoints, services, pods, namespaces]
    verbs: [list, watch]
  - apiGroups: [discovery.k8s.io]
    resources: [endpointslices]
    verbs: [list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:coredns
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
  - kind: ServiceAccount
    name: coredns
    namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: CoreDNS
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      serviceAccountName: coredns
      priorityClassName: system-cluster-critical
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: k8s-app
                      operator: In
                      values: ["kube-dns"]
                topologyKey: kubernetes.io/hostname
      containers:
        - name: coredns
          image: registry.k8s.io/coredns/coredns:v1.12.0
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: 170Mi
            requests:
              cpu: 100m
              memory: 70Mi
          args: ["-conf", "/etc/coredns/Corefile"]
          volumeMounts:
            - name: config-volume
              mountPath: /etc/coredns
              readOnly: true
          ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
            - containerPort: 9153
              name: metrics
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8181
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              add: ["NET_BIND_SERVICE"]
              drop: ["ALL"]
            readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
              - key: Corefile
                path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: CoreDNS
    kubernetes.io/cluster-service: "true"
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9153"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.10
  ports:
    - name: dns
      port: 53
      protocol: UDP
      targetPort: 53
    - name: dns-tcp
      port: 53
      protocol: TCP
      targetPort: 53
    - name: metrics
      port: 9153
      protocol: TCP
      targetPort: 9153
```
```bash

# Step 85 — Apply CoreDNS
kubectl apply -f /etc/kubernetes/coredns/coredns.yaml

# Step 86 — Wait for CoreDNS to be Running
kubectl get pods -n kube-system -l k8s-app=kube-dns -w

# Step 87 — Test DNS resolution
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- /bin/sh
# Inside the pod:
# nslookup kubernetes
# nslookup kubernetes.default.svc.cluster.local
# nslookup google.com
# cat /etc/resolv.conf
# exit
```

---

## SECTION 19 — master-1: Install Kubernetes Gateway API

> **Run on:** `master-1`
> **Why:** Gateway API is the modern successor to Ingress. It provides expressive, role-oriented routing primitives (GatewayClass, Gateway, HTTPRoute). Installing the CRDs enables the NGINX Fabric Gateway Controller to function.

```bash
# Step 88 — Install the Gateway API CRDs (standard channel)
# The standard channel includes GatewayClass, Gateway, HTTPRoute, ReferenceGrant, etc.
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml

# Step 89 — Verify the CRDs are installed
kubectl get crd | grep gateway

# Expected CRDs:
# gatewayclasses.gateway.networking.k8s.io
# gateways.gateway.networking.k8s.io
# httproutes.gateway.networking.k8s.io
# referencegrants.gateway.networking.k8s.io
# grpcroutes.gateway.networking.k8s.io
```

---

## SECTION 20 — master-1: Install NGINX Fabric Gateway Controller

> **Run on:** `master-1`
> **Why:** NGINX Gateway Fabric is a Kubernetes-native ingress controller built on Gateway API. It replaces traditional Ingress with proper role separation: platform admins manage Gateways, developers manage Routes. It uses NGINX as the data plane for high-performance HTTP/HTTPS routing.

```bash
# Step 90 — Create the nginx-gateway namespace
kubectl create namespace nginx-gateway

# Step 91 — Deploy NGINX Gateway Fabric using the official manifest
# This installs the controller, RBAC, and the GatewayClass
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.5.1/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.5.1/deploy/default/deploy.yaml

# Step 92 — Verify NGINX Gateway Fabric is running
kubectl get pods -n nginx-gateway -w

# Step 93 — Verify the GatewayClass is accepted
kubectl get gatewayclass nginx
# Expected: ACCEPTED = True

# Step 94 — Create a production Gateway (entry point for external traffic)
# This binds to port 80 and 443 on all worker nodes
mkdir -p /etc/kubernetes/gateway
vi /etc/kubernetes/gateway/central-gateway.yaml
```

```ini
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: central-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: tls-secret
            namespace: nginx-gateway
      allowedRoutes:
        namespaces:
          from: All
```
```bash

kubectl apply -f /etc/kubernetes/gateway/central-gateway.yaml
kubectl get gateway -n nginx-gateway

# Step 95 — Example HTTPRoute (deploy per application)
vi /etc/kubernetes/gateway/app1-httproute.yaml
```

```ini
# Reference example — deploy this per application team
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
    - name: central-gateway
      namespace: nginx-gateway
  hostnames:
    - "myapp.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-service
          port: 8080
```
```bash

# NOTE: Apply the above per application — don't apply the example above to production
```

---

## Phase 6 — Security Hardening

---

## SECTION 21 — master-1: RBAC Policies

> **Run on:** `master-1`
> **Why:** Default RBAC is permissive. These policies lock down what service accounts and users can do.

```bash
# Step 96 — Prevent pods from using the default service account for API calls
# Apply to all namespaces where you deploy workloads
vi /etc/kubernetes/security/restrict-default-sa.yaml
```

```ini
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
automountServiceAccountToken: false
```
```bash

kubectl apply -f /etc/kubernetes/security/restrict-default-sa.yaml

# Step 97 — Create a read-only cluster viewer role for monitoring/ops
vi /etc/kubernetes/security/viewer-clusterrole.yaml
```

```ini
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
  - apiGroups: [""]
    resources: [nodes, pods, services, endpoints, namespaces, configmaps]
    verbs: [get, list, watch]
  - apiGroups: [apps]
    resources: [deployments, daemonsets, replicasets, statefulsets]
    verbs: [get, list, watch]
  - apiGroups: [batch]
    resources: [jobs, cronjobs]
    verbs: [get, list, watch]
```
```bash

kubectl apply -f /etc/kubernetes/security/viewer-clusterrole.yaml

# Step 98 — Disable anonymous API server access (already done via --anonymous-auth=false)
# Verify:
kubectl get --as=system:anonymous nodes 2>&1 | grep Forbidden
```

---

## SECTION 22 — master-1: Network Policies

> **Run on:** `master-1`
> **Why:** Without NetworkPolicy, any pod can talk to any other pod. These policies implement a default-deny posture with explicit allow rules.

```bash
mkdir -p /etc/kubernetes/netpolicies

# Step 99 — Default deny all ingress and egress in the default namespace
vi /etc/kubernetes/netpolicies/default-deny.yaml
```

```ini
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```
```bash

kubectl apply -f /etc/kubernetes/netpolicies/default-deny.yaml

# Step 100 — Allow DNS egress for all pods (pods must be able to reach CoreDNS)
vi /etc/kubernetes/netpolicies/allow-dns.yaml
```

```ini
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```
```bash

kubectl apply -f /etc/kubernetes/netpolicies/allow-dns.yaml

# Step 101 — Isolate kube-system namespace from user workloads
vi /etc/kubernetes/netpolicies/kube-system-isolation.yaml
```

```ini
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: kube-system
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
        - podSelector: {}
```
```bash

kubectl apply -f /etc/kubernetes/netpolicies/kube-system-isolation.yaml
```

---

## SECTION 23 — master-1: Pod Security Admission

> **Run on:** `master-1`
> **Why:** Pod Security Admission (PSA) replaced PodSecurityPolicy. It enforces security standards at the namespace level. We enable `baseline` enforcement cluster-wide and `restricted` for production namespaces.

```bash
# Step 102 — Label the default namespace with baseline enforcement
# Pods that violate baseline security will be REJECTED
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# Step 103 — Label production namespaces with restricted enforcement
# Create a production namespace with full restricted policy
kubectl create namespace production
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Step 104 — Verify PSA labels
kubectl get namespace -L pod-security.kubernetes.io/enforce
```

---

## SECTION 24 — All Nodes: UFW Firewall Rules

> **Run on:** Each node type — run the appropriate block for each node's role
> **Why:** Defense-in-depth. Even if an attacker reaches the private network, firewall rules limit lateral movement.

### HAProxy Node

```bash
# Step 105 — On haproxy node
ufw --force reset
ufw default deny incoming
ufw default allow outgoing

# Allow SSH from management network only
ufw allow from <MGMT_CIDR> to any port 22

# Allow Kubernetes API (public entry point)
ufw allow 6443/tcp

# Allow HAProxy stats from management only
ufw allow from <MGMT_CIDR> to any port 8404

ufw --force enable
ufw status verbose
```

### Master Nodes

```bash
# Step 106 — On EACH master node
ufw --force reset
ufw default deny incoming
ufw default allow outgoing

# SSH from management only
ufw allow from <MGMT_CIDR> to any port 22

# Kubernetes API server — from HAProxy and other masters only
ufw allow from <HAPROXY_IP> to any port 6443
ufw allow from <MASTER1_IP> to any port 6443
ufw allow from <MASTER2_IP> to any port 6443
ufw allow from <MASTER3_IP> to any port 6443
# Allow workers to reach API via HAProxy (they connect to HAPROXY not directly to masters)

# etcd — only master-to-master
ufw allow from <MASTER1_IP> to any port 2379:2380
ufw allow from <MASTER2_IP> to any port 2379:2380
ufw allow from <MASTER3_IP> to any port 2379:2380

# kubelet API — from masters (apiserver calls kubelet for logs/exec)
ufw allow from <MASTER1_IP> to any port 10250
ufw allow from <MASTER2_IP> to any port 10250
ufw allow from <MASTER3_IP> to any port 10250

# Calico BGP — all cluster nodes
ufw allow from <POD_CIDR> to any
ufw allow from <MASTER1_IP> to any port 179
ufw allow from <MASTER2_IP> to any port 179
ufw allow from <MASTER3_IP> to any port 179

ufw --force enable
ufw status verbose
```

### Worker Nodes

```bash
# Step 107 — On EACH worker node
ufw --force reset
ufw default deny incoming
ufw default allow outgoing

# SSH from management only
ufw allow from <MGMT_CIDR> to any port 22

# kubelet API — masters call workers for logs/exec/portforward
ufw allow from <MASTER1_IP> to any port 10250
ufw allow from <MASTER2_IP> to any port 10250
ufw allow from <MASTER3_IP> to any port 10250

# NodePort services — allow if needed (or restrict to LB/HAProxy IP)
ufw allow 30000:32767/tcp

# Calico — pod CIDR traffic (pod-to-pod across nodes)
ufw allow from <POD_CIDR> to any

ufw --force enable
ufw status verbose
```

---

## Phase 7 — Verification

> **Run on:** `master-1`

### Cluster Health

```bash
# Step 108 — Verify all nodes are Ready
kubectl get nodes -o wide

# Step 109 — Verify all system pods are Running
kubectl get pods -A

# Step 110 — Verify etcd cluster is healthy
ETCDCTL_API=3 etcdctl \
  --endpoints=https://<MASTER1_IP>:2379,https://<MASTER2_IP>:2379,https://<MASTER3_IP>:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  endpoint status --write-out=table

# Step 111 — Verify controller-manager leader election (only 1 should be leader)
kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml

# Step 112 — Verify apiserver is accepting requests through HAProxy
curl -k https://<HAPROXY_IP>:6443/healthz
# Expected: ok

# Step 113 — Verify encryption at rest is working
# Create a secret, then check it's stored as encrypted binary in etcd
kubectl create secret generic test-secret --from-literal=key=value
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  get /registry/secrets/default/test-secret | strings | head -5
# Should show "k8s:enc:aescbc" — confirming encryption is active
kubectl delete secret test-secret
```

### DNS & Connectivity

```bash
# Step 114 — Deploy a test pod and verify DNS + inter-pod connectivity
kubectl run test-pod --image=busybox:1.28 --restart=Never -- sleep 3600
kubectl exec test-pod -- nslookup kubernetes
kubectl exec test-pod -- nslookup kubernetes.default.svc.cluster.local
kubectl exec test-pod -- wget -qO- http://kubernetes.default.svc.cluster.local
kubectl delete pod test-pod
```

### HA Failover Test

```bash
# Step 115 — Test that the cluster survives a master node failure
# On master-2 (or master-3), stop the apiserver:
# systemctl stop kube-apiserver

# Back on master-1, verify cluster still functions:
kubectl get nodes     # Should still work via HAProxy routing to remaining masters
kubectl get pods -A   # Should still show running pods

# Restart the stopped apiserver:
# systemctl start kube-apiserver

# Verify all 3 apiservers are back online:
for IP in <MASTER1_IP> <MASTER2_IP> <MASTER3_IP>; do
  echo -n "${IP}: "; curl -sk https://${IP}:6443/healthz
done
```

### Final Expected State

> ```
> NAME       STATUS   ROLES           AGE   VERSION
> master-1   Ready    control-plane   Xm    v1.34.0
> master-2   Ready    control-plane   Xm    v1.34.0
> master-3   Ready    control-plane   Xm    v1.34.0
> worker-1   Ready    worker          Xm    v1.34.0
> worker-2   Ready    worker          Xm    v1.34.0
> worker-3   Ready    worker          Xm    v1.34.0
> worker-4   Ready    worker          Xm    v1.34.0
> worker-5   Ready    worker          Xm    v1.34.0
> ```

---

## Appendix: Placeholder Reference

Replace all occurrences before running:

| Placeholder        | Replace with                                  |
|--------------------|-----------------------------------------------|
| `<HAPROXY_IP>`     | HAProxy server's private IP                   |
| `<PUBLIC_IP>`      | HAProxy server's public IP (internet-facing)  |
| `<MASTER1_IP>`     | master-1 private IP                           |
| `<MASTER2_IP>`     | master-2 private IP                           |
| `<MASTER3_IP>`     | master-3 private IP                           |
| `<WRK1_IP>`        | worker-1 private IP                           |
| `<WRK2_IP>`        | worker-2 private IP                           |
| `<WRK3_IP>`        | worker-3 private IP                           |
| `<WRK4_IP>`        | worker-4 private IP                           |
| `<WRK5_IP>`        | worker-5 private IP                           |
| `<WORKER_IP>`      | The specific worker's private IP              |
| `<WORKER_HOSTNAME>`| The specific worker's hostname                |
| `<THIS_MASTER_IP>` | The current master's own private IP           |
| `<THIS_MASTER_NAME>`| The current master's hostname (master-1 etc) |
| `<THIS_WORKER_IP>` | The current worker's own private IP           |
| `<MGMT_CIDR>`      | Your management/ops network CIDR             |
| `<POD_CIDR>`       | `192.168.0.0/16`                              |

---

## Appendix: Quick Reference Commands

```bash
# Check all services on a master
systemctl status etcd kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy

# Watch cluster events in real time
kubectl get events -A --sort-by='.lastTimestamp' -w

# Check audit logs
tail -f /var/log/kubernetes/apiserver-audit.log | python3 -m json.tool

# etcd cluster status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://<MASTER1_IP>:2379,https://<MASTER2_IP>:2379,https://<MASTER3_IP>:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/master-1.crt \
  --key=/etc/kubernetes/pki/etcd/master-1.key \
  endpoint status --write-out=table

# Force HAProxy to re-check backend health
echo "show stat" | socat stdio /run/haproxy/admin.sock

# Check certificate expiration dates
find /etc/kubernetes/pki -name "*.crt" -exec sh -c \
  'echo "=== {} ===" && openssl x509 -noout -enddate -in {}' \;
```
