# Kubernetes The Hard Way — Production-Grade Self-Managed Cluster

> **Target Environment:** Ubuntu Servers (Production)
> **Security Model:** Only HAProxy is public-facing. All cluster traffic is internal.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Node Reference Table](#node-reference-table)
- [Phase 1 — Single Master + Single Worker](#phase-1--single-master--single-worker)
  - [Section 1 — Download Binaries](#section-1--master-node-download-binaries)
  - [Section 2 — CA TLS Certificates](#section-2--master-node-generate-tls-certificates-ca)
  - [Section 3 — etcd TLS Certificates](#section-3--master-node-etcd-tls-certificates)
  - [Section 4 — Install & Start etcd](#section-4--master-node-install--start-etcd)
  - [Section 5 — kube-apiserver](#section-5--master-node-kube-apiserver)
  - [Section 6 — kube-controller-manager](#section-6--master-node-kube-controller-manager)
  - [Section 7 — kube-scheduler](#section-7--master-node-kube-scheduler)
  - [Section 8 — Admin kubeconfig](#section-8--master-node-admin-kubeconfig)
  - [Section 9 — containerd](#section-9--master-node-containerd-container-runtime)
  - [Section 10 — kubelet (Master)](#section-10--master-node-kubelet)
  - [Section 11 — kube-proxy (Master)](#section-11--master-node-kube-proxy)
  - [Section 12 — Install Calico CNI](#section-12--install-calico-cni)
  - [Section 13 — Install CoreDNS](#section-13--install-coredns)
  - [Section 14 — Worker Node Setup](#section-14--worker-node-setup)

---

## Architecture Overview

```
Internet
   │
   ▼
[HAProxy]  ◄── Only public-facing node (port 6443)
   │
   ├──► [master-1]  (etcd + control plane)
            │
            ▼
      [worker-1]  [worker-2]
```

**Phase 1:** 1 Master + 1 Worker → Calico CNI → Both nodes in `Ready` state
---


> All inter-node communication uses private IPs only.

---

# PHASE 1 — Single Master + Single Worker

---

## SECTION 1 — Master Node: Download Binaries

> **Run on:** `master-1` as `root`

```bash
# Step 1 — Switch to root to avoid permission issues
sudo su

# Step 2 — Update package lists
apt update -y

# Step 3 — Create a folder to hold all downloaded binaries
mkdir -p /root/binaries
cd /root/binaries

# Step 4 — Download Kubernetes server binaries
# (includes apiserver, scheduler, controller-manager, kubelet, etc.)
wget https://dl.k8s.io/v1.34.0/kubernetes-server-linux-amd64.tar.gz

# Step 5 — Extract the Kubernetes tarball
tar -xzvf kubernetes-server-linux-amd64.tar.gz

# Step 6 — Download etcd (the distributed key-value store Kubernetes uses as its database)
wget https://github.com/etcd-io/etcd/releases/download/v3.5.21/etcd-v3.5.21-linux-amd64.tar.gz

# Step 7 — Extract etcd binaries
tar -xzvf etcd-v3.5.21-linux-amd64.tar.gz
```

---

## SECTION 2 — Master Node: Generate TLS Certificates (CA)

> All Kubernetes components communicate over TLS. We generate our own Certificate Authority (CA) to sign all certs.
> **Run on:** `master-1`

```bash
# Step 8 — Create the PKI directory where all certificates will live
mkdir -p /etc/kubernetes/pki
cd /etc/kubernetes/pki

# Step 9 — Generate the CA private key (this is the root of trust for your entire cluster)
openssl genrsa -out ca.key 2048

# Step 10 — Generate the self-signed CA certificate (valid 1000 days)
openssl req -x509 -new -noenc -key ca.key -subj "/CN=KUBERNETES-CA" -days 1000 -out ca.crt

# Step 11 — Verify the CA certificate looks correct
openssl x509 -noout -text -in ca.crt
```

---

## SECTION 3 — Master Node: etcd TLS Certificates

> etcd needs its own cert so it can communicate securely.
> **Run on:** `master-1`
> Replace `<MASTER_IP>` with master-1's private IP.

```bash
# Step 12 — Create directory for etcd certs
mkdir -p /etc/kubernetes/pki/etcd
cd /etc/kubernetes/pki/etcd

# Step 13 — Generate etcd's private key
openssl genrsa -out server.key 2048

# Step 14 — Create a CSR config for etcd (defines who the cert is for, SANs, etc.)
# Replace <country>, <state>, <city>, <organization>, <organization unit>, <MASTER_IP>
vi csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = <country>
ST = <state>
L = <city>
O = <organization>
OU = <organization unit>
CN = etcd

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = <MASTER_IP>
IP.2 = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

```bash
# Step 15 — Generate the Certificate Signing Request from the config
openssl req -new -key server.key -out server.csr -config csr.conf

# Step 16 — Copy the CA cert into the etcd directory so etcd can reference it
cp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/etcd/

# Step 17 — Sign the etcd certificate using our CA (makes it trusted by everything in the cluster)
openssl x509 -req -in server.csr -CA ca.crt -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out server.crt -days 1000 \
  -extensions v3_ext -extfile csr.conf -sha256

# Step 18 — Verify the CSR (optional sanity check)
openssl req -noout -text -in ./server.csr

# Step 19 — Verify the signed certificate
openssl x509 -noout -text -in ./server.crt

# Security cleanup: remove .csr files — they're temporary request files, not needed after signing
rm -f server.csr
```

---

## SECTION 4 — Master Node: Install & Start etcd

> etcd is Kubernetes' brain — it stores all cluster state. We run it as a systemd service.
> **Run on:** `master-1`

```bash
# Step 20 — Move etcd binaries to system PATH so they can be run from anywhere
cd /root/binaries/etcd-v3.5.21-linux-amd64
mv etcd etcdctl etcdutl /usr/local/bin

# Step 21 — Create the etcd systemd service file
# Replace <Hostname> with master-1's hostname, <Host-ip> with master-1's private IP
vi /etc/systemd/system/etcd.service
```

```ini
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/v3.5/op-guide/configuration/

[Service]
ExecStart=/usr/local/bin/etcd \
  --name <Hostname> \
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \
  --key-file=/etc/kubernetes/pki/etcd/server.key \
  --peer-cert-file=/etc/kubernetes/pki/etcd/server.crt \
  --peer-key-file=/etc/kubernetes/pki/etcd/server.key \
  --trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-trusted-ca-file=/etc/kubernetes/pki/ca.crt \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://<Host-ip>:2380 \
  --listen-peer-urls https://<Host-ip>:2380 \
  --listen-client-urls https://<Host-ip>:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://<Host-ip>:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster <Hostname>=https://<Host-ip>:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 22 — Enable etcd to start on boot, then start it now
systemctl enable etcd
systemctl start etcd
# If you edit the service file after starting, always daemon-reload first:
# systemctl daemon-reload && systemctl restart etcd

# Step 23 — Confirm etcd is running without errors
systemctl status etcd

# Step 24 — Health check: confirm etcd is accepting requests
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Step 25 — Smoke test: write a key into etcd
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  put mykey "Test to manually put the key-value into etcd db."

# Step 26 — Read it back to confirm reads work
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get mykey
```

---

## SECTION 5 — Master Node: kube-apiserver

> The API server is the front door to Kubernetes — all `kubectl` commands and component communications go through it.
> **Run on:** `master-1`

```bash
# Step 27 — Copy kube-apiserver binary to system PATH
cp /root/binaries/kubernetes/server/bin/kube-apiserver /usr/local/bin/

# Step 28 — Confirm the version
kube-apiserver --version

# Step 29 — Create the apiserver cert directory and generate its private key
mkdir -p /etc/kubernetes/pki/kube-apiserver
cd /etc/kubernetes/pki/kube-apiserver
openssl genrsa -out apiserver.key 2048

# Step 30 — Create CSR config for the API server
# Replace <MASTER_IP> with master-1's private IP and <clusterDNS IP> with the cluster DNS IP (e.g. 10.96.0.1)
vi apiserver-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kube-apiserver

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = <MASTER_IP>
IP.2 = 127.0.0.1
IP.3 = <clusterDNS IP>
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names
```

```bash
# Step 31 — Generate the apiserver CSR
openssl req -new -key apiserver.key -out apiserver.csr -config apiserver-csr.conf

# Step 32 — Sign the apiserver certificate with our CA
openssl x509 -req -in apiserver.csr \
  -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial \
  -out apiserver.crt -days 1000 \
  -extensions v3_ext -extfile apiserver-csr.conf

# Cleanup CSR
rm -f apiserver.csr

# Step 33 — Generate service account signing keys (used to sign tokens for pods/services)
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub
```

### Enable Encryption-at-Rest (Encrypt Secrets in etcd)

```bash
# Step 34 — Generate a random 256-bit encryption key and base64-encode it
head -c 32 /dev/urandom | base64
# Copy the output — you'll paste it below as <base-64-key>

# Step 35 — Create the encryption config
# Kubernetes will use this to encrypt all Secrets stored in etcd
vi /etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml
```

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base-64-key>
      - identity: {}
```

### Create Audit Policy (Track Who Did What in the Cluster)

```bash
# Step 36 — Create an audit policy file
# Logs all requests at Metadata level — tune as needed for production
vi /etc/kubernetes/pki/kube-apiserver/policy.yaml
```

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
```

```bash
# Step 37 — Create the log directory for audit logs
mkdir -p /var/log/kubernetes
```

### Create apiserver-kubelet-client Certificate

```bash
# Step 38 — Create a certificate that kube-apiserver will use to talk to kubelet
cd /etc/kubernetes/pki/kube-apiserver

openssl genrsa -out apiserver-kubelet-client.key 2048

vi apiserver-kubelet-client-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = v3_req
distinguished_name = dn

[ dn ]
CN = kube-apiserver-kubelet-client
O  = system:masters

[ v3_req ]
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
```

```bash
# Generate CSR
openssl req -new -key apiserver-kubelet-client.key \
  -out apiserver-kubelet-client.csr \
  -config apiserver-kubelet-client-csr.conf

# Sign with CA
openssl x509 -req -in apiserver-kubelet-client.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out apiserver-kubelet-client.crt \
  -days 1000 \
  -extensions v3_req \
  -extfile apiserver-kubelet-client-csr.conf

# Verify
openssl x509 -noout -text -in apiserver-kubelet-client.crt | grep -E "Subject:|Extended"

# Cleanup
rm -f apiserver-kubelet-client.csr
```

### Create the kube-apiserver Systemd Service

```bash
# Step 39 — Create the kube-apiserver systemd service file
# Replace <Master-Ip> with master-1's private IP
vi /etc/systemd/system/kube-apiserver.service
```

```ini
[Unit]
Description=Kubernetes API Server
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=<Master-Ip> \
  --bind-address=0.0.0.0 \
  --apiserver-count=3 \
  --etcd-servers=https://127.0.0.1:2379 \
  --etcd-cafile=/etc/kubernetes/pki/ca.crt \
  --etcd-certfile=/etc/kubernetes/pki/etcd/server.crt \
  --etcd-keyfile=/etc/kubernetes/pki/etcd/server.key \
  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --tls-cert-file=/etc/kubernetes/pki/kube-apiserver/apiserver.crt \
  --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver/apiserver.key \
  --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt \
  --kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.crt \
  --kubelet-client-key=/etc/kubernetes/pki/kube-apiserver/apiserver-kubelet-client.key \
  --service-account-key-file=/etc/kubernetes/pki/kube-apiserver/sa.pub \
  --service-account-signing-key-file=/etc/kubernetes/pki/kube-apiserver/sa.key \
  --service-account-issuer=https://kubernetes.default.svc \
  --service-cluster-ip-range=10.96.0.0/12 \
  --service-node-port-range=30000-32767 \
  --authorization-mode=Node,RBAC \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --encryption-provider-config=/etc/kubernetes/pki/kube-apiserver/encryption-at-rest.yaml \
  --event-ttl=1h \
  --audit-policy-file=/etc/kubernetes/pki/kube-apiserver/policy.yaml \
  --audit-log-path=/var/log/kubernetes/apiserver-audit.log \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=10 \
  --audit-log-maxsize=100 \
  --runtime-config=api/all=true \
  --allow-privileged=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 40 — Start kube-apiserver
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

---

## SECTION 6 — Master Node: kube-controller-manager

> The controller manager watches cluster state and ensures reality matches desired state (e.g., always 3 replicas running).
> **Run on:** `master-1`

```bash
# Step 41 — Copy binary to PATH
cp /root/binaries/kubernetes/server/bin/kube-controller-manager /usr/local/bin/

# Step 42 — Also copy kubectl (the CLI tool to talk to the cluster)
cp /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin

# Step 43 — Create cert directory and generate private key
mkdir -p /etc/kubernetes/pki/kube-controller-manager
cd /etc/kubernetes/pki/kube-controller-manager
openssl genrsa -out kube-controller-manager.key 2048

# Step 44 — Create CSR config
vi kube-controller-manager-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = v3_req
distinguished_name = dn

[ dn ]
CN = system:kube-controller-manager

[ v3_req ]
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth,clientAuth
```

```bash
# Step 45 — Generate CSR and sign the certificate
openssl req -new -key kube-controller-manager.key \
  -out kube-controller-manager.csr -config kube-controller-manager-csr.conf

openssl x509 -req -in kube-controller-manager.csr \
  -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial \
  -out kube-controller-manager.crt -days 1000 \
  -extensions v3_req -extfile kube-controller-manager-csr.conf

rm -f kube-controller-manager.csr

# Step 46 — Create a kubeconfig so the controller-manager can authenticate to the API server
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

# Step 47 — Create the controller-manager service file
vi /etc/systemd/system/kube-controller-manager.service
```

```ini
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --authentication-kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig \
  --authorization-kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig \
  --bind-address=0.0.0.0 \
  --cluster-cidr=192.168.0.0/16 \
  --allocate-node-cidrs=true \
  --cluster-name=production-cluster \
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager/controller-manager.kubeconfig \
  --leader-elect=true \
  --node-monitor-period=5s \
  --root-ca-file=/etc/kubernetes/pki/ca.crt \
  --service-account-private-key-file=/etc/kubernetes/pki/kube-apiserver/sa.key \
  --service-cluster-ip-range=10.96.0.0/12 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 48 — Start controller-manager
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

---

## SECTION 7 — Master Node: kube-scheduler

> The scheduler picks which worker node a new pod should run on based on resource availability.
> **Run on:** `master-1`

```bash
# Step 49 — Copy scheduler binary to PATH
cp /root/binaries/kubernetes/server/bin/kube-scheduler /usr/local/bin/

# Step 50 — Create scheduler directories and config
mkdir -p /etc/kubernetes/pki/kube-scheduler
cd /etc/kubernetes/pki/kube-scheduler

vi kube-scheduler.yaml
```

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig"
leaderElection:
  leaderElect: true
```

```bash
# Step 51 — Create a kubeconfig for the scheduler to authenticate to the API server
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.crt \
  --client-key=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-scheduler \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-scheduler/scheduler.kubeconfig

# Step 52 — Create the scheduler service file
vi /etc/systemd/system/kube-scheduler.service
```

```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/pki/kube-scheduler/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 53 — Start the scheduler
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

---

## SECTION 8 — Master Node: Admin kubeconfig

> This gives you (the admin) the ability to run `kubectl` commands from master-1.
> **Run on:** `master-1`

```bash
# Step 54 — Create the .kube directory where kubectl looks for config by default
mkdir -p ~/.kube

# Step 55 — Create admin cert directory and private key
mkdir -p /etc/kubernetes/pki/admin
cd /etc/kubernetes/pki/admin
openssl genrsa -out admin.key 2048

# Step 56 — CSR config for the admin user
# O=system:masters gives full cluster access
vi admin-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
CN = admin
O = system:masters
```

```bash
# Step 57 — Generate and sign the admin cert
openssl req -new -key admin.key -out admin.csr -config admin-csr.conf

openssl x509 -req -in admin.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out admin.crt \
  -days 1000

rm -f admin.csr

# Step 58 — Create the admin kubeconfig at the default location (~/.kube/config)
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
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

kubectl config use-context default \
  --kubeconfig=$HOME/.kube/config

# Step 59 — Fix file ownership (needed if you ran earlier commands with sudo)
chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## SECTION 9 — Master Node: containerd (Container Runtime)

> Kubernetes doesn't run containers itself — it delegates to containerd. This is the CRI (Container Runtime Interface).
> **Run on:** `master-1`

```bash
# Step 60 — Install containerd from Ubuntu packages
apt install -y containerd

# Step 61 — Load required kernel modules for networking
# overlay = layered filesystems, br_netfilter = bridge networking
modprobe overlay
modprobe br_netfilter

# Persist the modules so they load on reboot
tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Step 62 — Enable IP forwarding and bridge networking in the kernel
# Required for pod networking
vi /etc/sysctl.d/k8s.conf
```

```ini
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

```bash
# Apply the sysctl settings immediately without rebooting
sysctl --system

# Step 63 — Generate the default containerd config and enable SystemdCgroup
# Must match kubelet's cgroup driver
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Start and enable containerd
systemctl restart containerd
systemctl enable containerd

# Step 64 — Confirm containerd is running
systemctl status containerd
```

---

## SECTION 10 — Master Node: kubelet

> kubelet runs on every node (including master). It's the agent that talks to the API server and manages pods on that node.
> **Run on:** `master-1`

```bash
# Step 65 — Copy kubelet binary to PATH
cp /root/binaries/kubernetes/server/bin/kubelet /usr/local/bin/

# Step 66 — Create kubelet directories
mkdir -p /var/lib/kubelet
mkdir -p /etc/kubernetes/pki/kubelet

# Step 67 — Generate kubelet's private key
cd /etc/kubernetes/pki/kubelet
openssl genrsa -out kubelet.key 2048

# Step 68 — Create CSR config for kubelet
# CN must match node hostname exactly — Kubernetes uses it for node identity
# Replace <MASTER_HOSTNAME> with: hostname
# Replace <MASTER_IP> with the server's private IP
vi kubelet-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
CN = system:node:<MASTER_HOSTNAME>
O = system:nodes

[ alt_names ]
DNS.1 = <MASTER_HOSTNAME>   # hostname — apiserver uses this when calling kubectl logs/exec
IP.1  = <MASTER_IP>         # node private IP — for IP-based TLS verification
```

```bash
# Step 69 — Generate and sign the kubelet cert
openssl req -new -key kubelet.key -out kubelet.csr -config kubelet-csr.conf

openssl x509 -req -in kubelet.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kubelet.crt \
  -days 1000

rm -f kubelet.csr

# Step 70 — Create kubeconfig for kubelet so it can authenticate to the API server
# Replace <MASTER_HOSTNAME>
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-credentials system:node:<MASTER_HOSTNAME> \
  --client-certificate=/etc/kubernetes/pki/kubelet/kubelet.crt \
  --client-key=/etc/kubernetes/pki/kubelet/kubelet.key \
  --embed-certs=true \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:node:<MASTER_HOSTNAME> \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config use-context default \
  --kubeconfig=/var/lib/kubelet/kubeconfig

# Step 71 — Create kubelet configuration file
# Behaviour settings for this node's kubelet
vi /var/lib/kubelet/kubelet-config.yaml
```

```yaml
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
```

```bash
# Step 72 — Create the kubelet systemd service
# Replace <MASTER_IP> with master-1's private IP
vi /etc/systemd/system/kubelet.service
```

```ini
[Unit]
Description=Kubelet
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --register-node=true \
  --node-ip=<MASTER_IP> \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 73 — Start kubelet
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

---

## SECTION 11 — Master Node: kube-proxy

> kube-proxy runs on every node and manages the iptables rules that route traffic to the right pods/services.
> **Run on:** `master-1`

```bash
# Step 74 — Copy kube-proxy binary to PATH
cp /root/binaries/kubernetes/server/bin/kube-proxy /usr/local/bin/

# Step 75 — Create the kube-proxy directory and kubeconfig
mkdir -p /etc/kubernetes/pki/kube-proxy

kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.crt \
  --client-key=/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-proxy \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

# Step 76 — Create the kube-proxy service
# cluster-cidr must match Calico's pod CIDR
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
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 77 — Start kube-proxy
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy

# Step 78 — Verify master node is visible
# May show NotReady until CNI is installed
kubectl get nodes
kubectl get pods -A
```

---

## SECTION 12 — Install Calico CNI

> Calico is the network plugin that gives each pod its own IP and enables pod-to-pod communication.
> Without CNI, nodes stay in `NotReady` state.
> **Run on:** `master-1`

```bash
# Step 79 — Apply Calico manifest
# Uses 192.168.0.0/16 pod CIDR by default — matches our config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/calico.yaml
```

### Troubleshooting Calico

```bash
# Read Calico pod logs directly
find /var/log/pods -name "*.log" | grep calico-node
ls /var/log/pods/ | grep calico

# Check the install-cni container logs
cat /var/log/pods/kube-system_calico-node-*/install-cni/0.log
cat /var/log/pods/kube-system_calico-node-*/install-cni/*.log

# Read the calico-node main container logs
cat /var/log/pods/kube-system_calico-node-*/calico-node/*.log | tail -50

# For PoC over a virtual machine: check your network interface name
ip addr show

# Replace eth0 with whatever your interface name actually is
kubectl set env daemonset/calico-node -n kube-system \
  IP_AUTODETECTION_METHOD=interface=eth0 \
  IP=<IP-address>

# Remove any hardcoded IP= that was pinned to master-1's IP
kubectl set env daemonset/calico-node -n kube-system IP-
```

```bash
# Step 80 — Watch Calico pods come up (wait until all are Running)
kubectl get pods -n kube-system -w

# Step 81 — Confirm master node is now in Ready state
kubectl get nodes
```

> **Expected output:**
> ```
> NAME       STATUS   ROLES    AGE   VERSION
> master-1   Ready    <none>   Xm    v1.34.0
> ```

---

## SECTION 13 — Install CoreDNS

> CoreDNS provides DNS-based service discovery for pods inside the cluster.
> Without it, pods cannot resolve service names (e.g. `kubernetes.default.svc.cluster.local`).
> **Run on:** `master-1`

```bash
# Step 82 — Create the CoreDNS directory
mkdir -p /etc/kubernetes/coredns
cd /etc/kubernetes/coredns

# Step 83 — Create all CoreDNS manifests in one file
vi coredns.yaml
```

```yaml
---
# ServiceAccount — gives CoreDNS pods an identity inside the cluster
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system

---
# ClusterRole — defines what CoreDNS is allowed to do
# (read endpoints, services, pods, namespaces)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
rules:
  - apiGroups: [""]
    resources:
      - endpoints
      - services
      - pods
      - namespaces
    verbs: ["list", "watch"]
  - apiGroups: ["discovery.k8s.io"]
    resources:
      - endpointslices
    verbs: ["list", "watch"]

---
# ClusterRoleBinding — binds the ClusterRole to the CoreDNS ServiceAccount
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
# ConfigMap — CoreDNS configuration (Corefile)
# cluster.local is the cluster domain
# 10.96.0.0/12 is your service CIDR — CoreDNS won't forward internal service IPs to upstream
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors                    # log DNS errors
        health {                  # health check endpoint on port 8080
          lameduck 5s
        }
        ready                     # readiness endpoint on port 8181
        kubernetes cluster.local in-addr.arpa ip6.arpa {  # handle in-cluster DNS
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        prometheus :9153          # expose metrics for Prometheus
        forward . /etc/resolv.conf {  # forward external DNS queries to the node's resolver
          max_concurrent 1000
        }
        cache 30                  # cache responses for 30 seconds
        loop                      # detect forwarding loops
        reload                    # auto-reload Corefile on change
        loadbalance               # round-robin DNS load balancing
    }

---
# Deployment — runs CoreDNS pods (2 replicas for redundancy)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: CoreDNS
spec:
  replicas: 2
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
      priorityClassName: system-cluster-critical  # ensures CoreDNS is not evicted under pressure
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
        podAntiAffinity:              # spread replicas across different nodes
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
            successThreshold: 1
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
              add: ["NET_BIND_SERVICE"]   # allows CoreDNS to bind to port 53
              drop: ["ALL"]
            readOnlyRootFilesystem: true
      dnsPolicy: Default              # CoreDNS itself uses the node's DNS, not itself
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
              - key: Corefile
                path: Corefile

---
# Service — this is what pods actually send DNS queries to
# clusterIP MUST match clusterDNS in kubelet-config.yaml on every node (10.96.0.10)
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
  clusterIP: 10.96.0.10      # MUST match clusterDNS in kubelet-config.yaml on ALL nodes
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
# Step 84 — Apply the CoreDNS manifest
kubectl apply -f coredns.yaml

# Step 85 — Verify CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns -w

kubectl get svc kube-dns -n kube-system

# Step 86 — Test DNS resolution is working
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- /bin/sh
```

Inside the pod, run these commands:

```sh
nslookup kubernetes                            # should resolve to 10.96.0.1
nslookup kubernetes.default.svc.cluster.local  # fully qualified — same result
nslookup google.com                            # external DNS — confirms forwarding works
cat /etc/resolv.conf                           # should show nameserver 10.96.0.10
exit
```

```bash
# Step 87 — Verify all nodes have correct clusterDNS configured
cat /var/lib/kubelet/kubelet-config.yaml | grep -A2 clusterDNS

# Troubleshooting: check node resolv.conf if DNS issues arise
cat /etc/resolv.conf
```

---

## SECTION 14 — Worker Node Setup

> The worker node runs your actual workloads (pods). It needs: containerd, kubelet, and kube-proxy.
> **Run all steps in this section on:** `worker-1` as `root`

```bash
# Step 88 — Switch to root
sudo su

# Step 89 — Update and install containerd
apt update -y
apt install -y containerd

# Step 90 — Same kernel/networking setup as master
modprobe overlay
modprobe br_netfilter

tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

vi /etc/sysctl.d/k8s.conf
```

```ini
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

```bash
sysctl --system

# Step 91 — Configure containerd with systemd cgroup driver
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

# Step 92 — Download Kubernetes binaries on the worker
mkdir -p /root/binaries
cd /root/binaries
wget https://dl.k8s.io/v1.34.0/kubernetes-node-linux-amd64.tar.gz
tar -xzvf kubernetes-node-linux-amd64.tar.gz

# Copy required worker binaries
cp kubernetes/node/bin/kubelet /usr/local/bin/
cp kubernetes/node/bin/kube-proxy /usr/local/bin/
cp kubernetes/node/bin/kubectl /usr/local/bin/
```

### Copy Certificates from master-1 to worker-1

> **Run on:** `master-1` — push certs to worker-1 using scp (use worker's private IP)

```bash
# Step 93 — Copy the CA cert to worker
# Worker needs it to trust the cluster
scp /etc/kubernetes/pki/ca.crt root@<WORKER_IP>:/etc/kubernetes/pki/ca.crt
```

> **Back on `worker-1`:** generate kubelet cert and kubeconfig

```bash
# Step 94 — Create PKI dirs on worker
mkdir -p /etc/kubernetes/pki/kubelet
mkdir -p /var/lib/kubelet

cd /etc/kubernetes/pki/kubelet

# Step 95 — Generate worker kubelet key
openssl genrsa -out kubelet.key 2048

# Step 96 — Create CSR config for worker kubelet
# Replace <WORKER_HOSTNAME> with: hostname
vi kubelet-csr.conf
```

```ini
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
CN = system:node:<WORKER_HOSTNAME>
O = system:nodes
```

```bash
# Step 97 — Sign the worker cert using the CA
# Copy ca.key temporarily from master — delete it from worker after signing

# From master-1:
scp /etc/kubernetes/pki/ca.key root@<WORKER_IP>:/etc/kubernetes/pki/ca.key

# On worker-1:
openssl req -new -key kubelet.key -out kubelet.csr -config kubelet-csr.conf
openssl x509 -req -in kubelet.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out kubelet.crt -days 1000
rm -f kubelet.csr

# SECURITY: Remove ca.key from the worker after cert is signed
# It should only live on master
rm -f /etc/kubernetes/pki/ca.key

# Step 98 — Create kubeconfig for kubelet (pointing to master-1's private IP)
# Replace <MASTER_IP> and <WORKER_HOSTNAME>
kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<MASTER_IP>:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-credentials system:node:<WORKER_HOSTNAME> \
  --client-certificate=/etc/kubernetes/pki/kubelet/kubelet.crt \
  --client-key=/etc/kubernetes/pki/kubelet/kubelet.key \
  --embed-certs=true \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:node:<WORKER_HOSTNAME> \
  --kubeconfig=/var/lib/kubelet/kubeconfig

kubectl config use-context default --kubeconfig=/var/lib/kubelet/kubeconfig

# Step 99 — Create kubelet-config.yaml on the worker
vi /var/lib/kubelet/kubelet-config.yaml
```

```yaml
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
```

```bash
# Step 100 — Create kubelet service on worker
# Replace <WORKER_IP>
vi /etc/systemd/system/kubelet.service
```

```ini
[Unit]
Description=Kubelet
Documentation=https://kubernetes.io/docs/

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --register-node=true \
  --node-ip=<WORKER_IP> \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 101 — Create kube-proxy kubeconfig on worker (pointing to master)
mkdir -p /etc/kubernetes/pki/kube-proxy

# Copy kube-controller-manager certs from master for kube-proxy auth
# (Simplest for PoC — generate dedicated proxy certs for production)
scp root@<MASTER_IP>:/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.crt \
    root@<MASTER_IP>:/etc/kubernetes/pki/kube-controller-manager/kube-controller-manager.key \
    /etc/kubernetes/pki/kube-proxy/

kubectl config set-cluster production-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<MASTER_IP>:6443 \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=/etc/kubernetes/pki/kube-proxy/kube-controller-manager.crt \
  --client-key=/etc/kubernetes/pki/kube-proxy/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=production-cluster \
  --user=system:kube-proxy \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=/etc/kubernetes/pki/kube-proxy/kube-proxy.kubeconfig

# Step 102 — Create kube-proxy service on worker
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
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Step 103 — Start kubelet and kube-proxy on worker
systemctl daemon-reload
systemctl enable kubelet kube-proxy
systemctl start kubelet kube-proxy
systemctl status kubelet
systemctl status kube-proxy
```

### Verify Phase 1 Complete

> **Run on:** `master-1`

```bash
# Step 104 — Both nodes should show Ready
kubectl get nodes

# Step 105 — All system pods (Calico, kube-proxy, CoreDNS) should be Running
kubectl get pods -A
```

> **Expected output:**
> ```
> NAME       STATUS   ROLES    AGE   VERSION
> master-1   Ready    <none>   Xm    v1.34.0
> worker-1   Ready    <none>   Xm    v1.34.0
> ```

---

*Phase 1 complete.*