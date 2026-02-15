# Kubernetes Cluster Architecture, Installation & Configuration

**Author:** Sajal Jana

A production-focused guide for platform engineers preparing for CKA/CKAD certification.

---

## Table of Contents

- [1. Cluster Architecture](#1-cluster-architecture)
  - [1.1 Control Plane Components](#11-control-plane-components)
    - [kube-apiserver](#kube-apiserver)
    - [etcd](#etcd)
    - [kube-scheduler](#kube-scheduler)
    - [kube-controller-manager](#kube-controller-manager)
  - [1.2 Worker Node Components](#12-worker-node-components)
    - [kubelet](#kubelet)
    - [kube-proxy](#kube-proxy)
    - [Container Runtime](#container-runtime)
- [2. Installation with kubeadm](#2-installation-with-kubeadm)
  - [2.1 Prerequisites](#21-prerequisites)
  - [2.2 Install kubeadm, kubelet, kubectl](#22-install-kubeadm-kubelet-kubectl)
  - [2.3 Bootstrap Control Plane](#23-bootstrap-control-plane)
  - [2.4 Join Worker Nodes](#24-join-worker-nodes)
  - [2.5 Common Pitfalls](#25-common-pitfalls)
- [3. Infrastructure & High Availability](#3-infrastructure--high-availability)
  - [3.1 Infrastructure Provisioning](#31-infrastructure-provisioning)
  - [3.2 HA Control Plane Setup](#32-ha-control-plane-setup)
  - [3.3 etcd Considerations](#33-etcd-considerations)
- [4. Lifecycle Management](#4-lifecycle-management)
  - [4.1 Kubernetes Version Upgrades](#41-kubernetes-version-upgrades)
  - [4.2 etcd Backup and Restore](#42-etcd-backup-and-restore)
- [5. Quick Reference](#5-quick-reference)
  - [5.1 Essential Commands](#51-essential-commands)
  - [5.2 Key File Locations](#52-key-file-locations)
- [6. Production Checklist](#6-production-checklist)
- [7. Further Reading](#7-further-reading)

---

## 1. Cluster Architecture

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                           │
│  ┌──────────────┐  ┌──────┐  ┌───────────┐  ┌────────────┐│
│  │ kube-apiserver│  │ etcd │  │ scheduler │  │ controller ││
│  │   (REST API) │  │(store)│  │  (pods)   │  │  manager   ││
│  └──────┬───────┘  └──┬───┘  └─────┬─────┘  └──────┬─────┘│
│         │             │             │                │      │
└─────────┼─────────────┼─────────────┼────────────────┼──────┘
          │             │             │                │
          └─────────────┴─────────────┴────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │         WORKER NODES                  │
          │  ┌──────────┐  ┌───────────┐         │
          │  │ kubelet  │  │kube-proxy │         │
          │  │(pod mgmt)│  │(network)  │         │
          │  └────┬─────┘  └─────┬─────┘         │
          │       │              │               │
          │  ┌────┴──────────────┴─────┐         │
          │  │   Container Runtime      │         │
          │  │  (containerd/CRI-O)      │         │
          │  └──────────────────────────┘         │
          └───────────────────────────────────────┘
```

### 1.1 Control Plane Components

#### kube-apiserver

The central management hub of Kubernetes. All cluster components communicate through the API server.

**Key Functions:**
- Validates and processes REST requests
- Only component that directly communicates with etcd
- Serves as the frontend for the cluster's shared state
- Handles authentication, authorization, and admission control

**Health Check Commands:**

```bash
# Check apiserver health
kubectl get --raw /healthz

# View cluster info
kubectl cluster-info

# Check apiserver logs
kubectl logs -n kube-system kube-apiserver-<node-name>
```

#### etcd

Distributed, consistent key-value store that stores all cluster data.

**Key Features:**
- Uses Raft consensus algorithm
- Stores entire cluster state (pods, services, secrets, etc.)
- Strongly consistent and highly available
- Critical component - requires regular backups

**Health Check Commands:**

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

#### kube-scheduler

Watches for newly created pods with no assigned node and selects a node for them to run on.

**Selection Factors:**
- Resource requirements (CPU, memory)
- Hardware/software/policy constraints
- Affinity and anti-affinity specifications
- Data locality
- Inter-workload interference
- Taints and tolerations

**Monitoring Commands:**

```bash
# View scheduler decisions in events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-<node-name>
```

#### kube-controller-manager

Runs controller processes that regulate the state of the cluster.

**Key Controllers:**
- **Node Controller**: Monitors node health
- **Replication Controller**: Maintains correct number of pods
- **Endpoints Controller**: Populates Endpoints objects
- **Service Account Controller**: Creates default accounts and API access tokens
- **Deployment Controller**: Manages deployment rollouts
- **StatefulSet Controller**: Manages stateful applications

**Monitoring Commands:**

```bash
# Check controller-manager status
kubectl get componentstatuses  # deprecated but useful

# View controller-manager logs
kubectl logs -n kube-system kube-controller-manager-<node-name>
```

### 1.2 Worker Node Components

#### kubelet

The primary node agent that runs on each worker node.

**Key Responsibilities:**
- Registers node with the API server
- Watches for PodSpecs from the API server
- Ensures containers described in PodSpecs are running and healthy
- Reports node and pod status back to API server
- Manages pod lifecycle via Container Runtime Interface (CRI)

**Management Commands:**

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f

# Check kubelet config
ps aux | grep kubelet

# View kubelet metrics
curl http://localhost:10255/metrics
```

#### kube-proxy

Network proxy that runs on each node, maintaining network rules.

**Key Functions:**
- Implements Kubernetes Service concept
- Maintains network rules for pod communication
- Handles connection forwarding
- Supports multiple proxy modes: iptables (default), IPVS, userspace

**Monitoring Commands:**

```bash
# Check kube-proxy mode
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"

# View kube-proxy configuration
kubectl get configmap -n kube-system kube-proxy -o yaml

# Check iptables rules (if using iptables mode)
sudo iptables-save | grep KUBE
```

#### Container Runtime

Software responsible for running containers.

**Supported Runtimes:**
- **containerd** (most common, CNCF graduated project)
- **CRI-O** (lightweight, OCI-compliant)
- **Docker** (deprecated as of Kubernetes 1.24)

**Runtime Commands:**

```bash
# containerd (using crictl)
crictl ps          # List running containers
crictl pods        # List pods
crictl images      # List images
crictl logs <id>   # View container logs

# Check runtime version
crictl version

# Runtime info
crictl info
```

---

## 2. Installation with kubeadm

### 2.1 Prerequisites

#### Disable Swap

Kubernetes requires swap to be disabled for proper resource management.

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify swap is off
free -h
```

#### Load Required Kernel Modules

```bash
# Create modules configuration
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay
```

#### Configure Sysctl Parameters

```bash
# Create sysctl configuration
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify configuration
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

#### Install Container Runtime (containerd)

```bash
# Update package index
sudo apt-get update

# Install containerd
sudo apt-get install -y containerd

# Create default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Configure systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

### 2.2 Install kubeadm, kubelet, kubectl

```bash
# Install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes components
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
kubelet --version
kubectl version --client
```

### 2.3 Bootstrap Control Plane

```bash
# Initialize the cluster (run on control plane node)
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<CONTROL_PLANE_IP> \
  --kubernetes-version=v1.29.0

# IMPORTANT: Save the kubeadm join command from the output!
# Example output:
# kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
#   --discovery-token-ca-cert-hash sha256:xxxxx
```

#### Configure kubectl Access

```bash
# For regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# For root user
export KUBECONFIG=/etc/kubernetes/admin.conf

# Verify cluster access
kubectl cluster-info
kubectl get nodes
```

#### Install CNI Plugin (Flannel Example)

```bash
# Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Wait for CoreDNS pods to be ready
kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=300s

# Verify CNI is working
kubectl get pods -n kube-system
kubectl get nodes  # Should show Ready status
```

**Alternative CNI Options:**

```bash
# Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Cilium (with Helm)
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system
```

### 2.4 Join Worker Nodes

```bash
# On each worker node, run the join command from kubeadm init output:
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>

# Example:
sudo kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

#### If Token Expired

```bash
# Generate new token (on control plane)
kubeadm token create --print-join-command

# List existing tokens
kubeadm token list

# Create token with custom TTL (default is 24h)
kubeadm token create --ttl 2h
```

#### Verify Node Join

```bash
# Check nodes (on control plane)
kubectl get nodes

# Check node details
kubectl describe node <worker-node-name>

# View kubelet logs on worker node if issues
journalctl -u kubelet -f
```

### 2.5 Common Pitfalls

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Swap enabled** | kubelet fails to start | `swapoff -a` and comment swap line in `/etc/fstab` |
| **Containerd cgroup driver** | Pods stuck in ContainerCreating | Edit `/etc/containerd/config.toml`, set `SystemdCgroup = true`, restart containerd |
| **Pod CIDR conflict** | Pods can't get IPs | Ensure `--pod-network-cidr` doesn't overlap with node network |
| **Firewall blocking** | Nodes can't join cluster | Open required ports: 6443, 2379-2380, 10250-10252, 30000-32767 |
| **Node NotReady** | Node shows NotReady status | Check CNI plugin installed: `kubectl get pods -n kube-system` |
| **CoreDNS CrashLoopBackOff** | DNS not working | CNI not installed or configured incorrectly |
| **Certificate errors** | API server connection fails | Check system time is synchronized (NTP) |
| **Port already in use** | kubeadm init fails | Another service using required ports, check with `netstat -tulpn` |

#### Required Ports

**Control Plane:**
| Port | Purpose | Used By |
|------|---------|---------|
| 6443 | Kubernetes API server | All |
| 2379-2380 | etcd server client API | kube-apiserver, etcd |
| 10250 | Kubelet API | Self, control plane |
| 10259 | kube-scheduler | Self |
| 10257 | kube-controller-manager | Self |

**Worker Nodes:**
| Port | Purpose | Used By |
|------|---------|---------|
| 10250 | Kubelet API | Self, control plane |
| 30000-32767 | NodePort Services | All |

---

## 3. Infrastructure & High Availability

### 3.1 Infrastructure Provisioning

#### Infrastructure Options

**1. Bare Metal (On-Premises)**
- Full control over hardware
- Best performance
- Higher initial cost
- Requires physical management

**2. Virtual Machines**
- VMware vSphere
- Proxmox
- KVM/QEMU
- Hyper-V

**3. Cloud Providers**
- AWS EC2
- Google Compute Engine
- Azure Virtual Machines
- DigitalOcean Droplets

#### Minimum Production Architecture

```
┌────────────────────────────────────────┐
│  Load Balancer (HAProxy/NGINX/Cloud)  │
│         (VIP: 192.168.1.100:6443)      │
└────────────┬───────────────────────────┘
             │
     ┌───────┴────────┬────────────┐
     │                │            │
┌────▼────┐    ┌──────▼───┐   ┌───▼──────┐
│Master-1 │    │Master-2  │   │Master-3  │
│ (etcd)  │    │ (etcd)   │   │ (etcd)   │
│ 2C/4GB  │    │ 2C/4GB   │   │ 2C/4GB   │
└─────────┘    └──────────┘   └──────────┘

┌──────────┐   ┌──────────┐   ┌──────────┐
│Worker-1  │   │Worker-2  │   │Worker-N  │
│ 4C/8GB   │   │ 4C/8GB   │   │ 4C/8GB   │
└──────────┘   └──────────┘   └──────────┘
```

#### Node Specifications

**Control Plane Nodes (Minimum):**
- CPU: 2 cores
- RAM: 4 GB
- Disk: 50 GB (SSD recommended for etcd)
- Network: 1 Gbps

**Worker Nodes (Minimum):**
- CPU: 2 cores
- RAM: 4 GB
- Disk: 100 GB
- Network: 1 Gbps

**Production Recommendations:**
- Control Plane: 4 CPU, 8 GB RAM, 100 GB SSD
- Worker Nodes: 8+ CPU, 16+ GB RAM, 200+ GB SSD
- Dedicated etcd nodes for large clusters (1000+ nodes)

### 3.2 HA Control Plane Setup

#### Step 1: Provision Load Balancer

**Option A: HAProxy Configuration**

```bash
# Install HAProxy
sudo apt-get install -y haproxy

# Configure HAProxy
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 192.168.1.101:6443 check fall 3 rise 2
    server master2 192.168.1.102:6443 check fall 3 rise 2
    server master3 192.168.1.103:6443 check fall 3 rise 2
EOF

# Restart HAProxy
sudo systemctl restart haproxy
sudo systemctl enable haproxy

# Verify HAProxy is listening
sudo netstat -tulpn | grep 6443
```

**Option B: NGINX Configuration**

```bash
# Install NGINX
sudo apt-get install -y nginx

# Configure NGINX
sudo tee /etc/nginx/nginx.conf <<EOF
events {}

stream {
    upstream k8s_api {
        server 192.168.1.101:6443 max_fails=3 fail_timeout=30s;
        server 192.168.1.102:6443 max_fails=3 fail_timeout=30s;
        server 192.168.1.103:6443 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6443;
        proxy_pass k8s_api;
        proxy_timeout 10s;
        proxy_connect_timeout 5s;
    }
}
EOF

# Test configuration
sudo nginx -t

# Restart NGINX
sudo systemctl restart nginx
sudo systemctl enable nginx
```

**Option C: Cloud Load Balancer**

```bash
# AWS Network Load Balancer (via AWS CLI)
aws elbv2 create-load-balancer \
  --name k8s-control-plane-lb \
  --type network \
  --subnets subnet-xxxxx subnet-yyyyy

# Create target group
aws elbv2 create-target-group \
  --name k8s-api-targets \
  --protocol TCP \
  --port 6443 \
  --vpc-id vpc-xxxxx

# Register control plane nodes as targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-master1 Id=i-master2 Id=i-master3
```

#### Step 2: Initialize First Control Plane Node

```bash
# On first control plane node
sudo kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.1.101

# Save THREE outputs from this command:
# 1. kubectl config setup commands
# 2. Control plane join command (with --control-plane flag)
# 3. Worker node join command
```

**Important Flags:**
- `--control-plane-endpoint`: Load balancer VIP/DNS (used by all nodes)
- `--upload-certs`: Uploads certificates to cluster secret for other control planes
- `--apiserver-advertise-address`: This specific node's IP

```bash
# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI plugin
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify first control plane is ready
kubectl get nodes
kubectl get pods -n kube-system
```

#### Step 3: Join Additional Control Plane Nodes

```bash
# On second and third control plane nodes
sudo kubeadm join 192.168.1.100:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --apiserver-advertise-address=<THIS_NODE_IP>

# Example:
sudo kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234... \
  --control-plane \
  --certificate-key 5678... \
  --apiserver-advertise-address=192.168.1.102
```

**If certificate-key expired (2 hours TTL):**

```bash
# Re-upload certificates (on existing control plane)
sudo kubeadm init phase upload-certs --upload-certs

# Use the new certificate-key in join command
```

#### Step 4: Join Worker Nodes

```bash
# On each worker node (same as single control plane)
sudo kubeadm join 192.168.1.100:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Step 5: Verify HA Setup

```bash
# Check all control plane nodes are ready
kubectl get nodes

# Check control plane components
kubectl get pods -n kube-system

# Check etcd cluster health
kubectl exec -it -n kube-system etcd-master1 -- sh -c \
  "ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

# Test HA: stop one control plane node and verify cluster still works
```

### 3.3 etcd Considerations

#### Stacked etcd Topology (Default)

```
┌─────────────────────────────────────────┐
│           Control Plane Node            │
│  ┌────────────┐    ┌─────────────────┐ │
│  │ etcd       │◄───┤ kube-apiserver  │ │
│  └────────────┘    └─────────────────┘ │
│  ┌────────────┐    ┌─────────────────┐ │
│  │ scheduler  │    │ controller-mgr  │ │
│  └────────────┘    └─────────────────┘ │
└─────────────────────────────────────────┘
```

**Pros:**
- Simpler setup, fewer servers
- Easier to manage
- Sufficient for most use cases

**Cons:**
- Coupled failure (if node dies, both control plane and etcd member lost)
- Limited scalability

**When to Use:**
- Small to medium clusters (<100 nodes)
- Budget constraints
- Simple operations

#### External etcd Topology

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  etcd Node 1 │  │  etcd Node 2 │  │  etcd Node 3 │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┴─────────────────┘
                         │
       ┌─────────────────┴─────────────────┐
       │                                   │
┌──────▼─────────┐  ┌──────────────┐  ┌───▼───────────┐
│ Control Plane 1│  │Control Plane2│  │Control Plane 3│
│  (no etcd)     │  │  (no etcd)   │  │  (no etcd)    │
└────────────────┘  └──────────────┘  └───────────────┘
```

**Pros:**
- Decoupled failure (control plane failure doesn't affect etcd)
- Better scalability
- Independent maintenance

**Cons:**
- More complex setup
- Additional infrastructure required
- Higher costs

**When to Use:**
- Large clusters (>100 nodes)
- Mission-critical workloads
- Need independent scaling of control plane and storage

#### etcd Health Monitoring

```bash
# Check cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd performance
ETCDCTL_API=3 etcdctl endpoint status \
  --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd metrics
curl -L http://localhost:2379/metrics
```

#### etcd Best Practices

**Cluster Sizing:**
- Always use **odd numbers** (3, 5, 7)
- 3 members: Tolerates 1 failure
- 5 members: Tolerates 2 failures (recommended for production)
- 7+ members: Only for very large, critical clusters

**Performance:**
- Use SSD storage
- Dedicated disk for etcd (don't share with OS)
- Network latency < 10ms between members
- Monitor disk latency (use `fio` for testing)

**Placement:**
- Distribute across failure domains (different racks, availability zones)
- Keep members close (same datacenter/region)
- Avoid placing all members on same physical host

**Resource Requirements:**
```
Small cluster (<100 nodes):   2 CPU, 8GB RAM, 20GB SSD
Medium cluster (<1000 nodes): 4 CPU, 16GB RAM, 50GB SSD
Large cluster (>1000 nodes):  8 CPU, 32GB RAM, 100GB SSD
```

**Quotas and Limits:**
```bash
# Default etcd database size is 2GB
# Increase if needed (not recommended >8GB)
etcd --quota-backend-bytes=8589934592  # 8GB

# Check database size
ETCDCTL_API=3 etcdctl endpoint status --write-out=table

# Compact and defragment if size is growing
ETCDCTL_API=3 etcdctl compact <revision>
ETCDCTL_API=3 etcdctl defrag
```

---

## 4. Lifecycle Management

### 4.1 Kubernetes Version Upgrades

#### Upgrade Strategy

**Order of Operations:**
1. Backup etcd
2. Upgrade first control plane node
3. Upgrade additional control plane nodes (one at a time)
4. Upgrade worker nodes (one at a time, draining workloads first)

**Version Skew Policy:**
- Control plane components: Can be N+1 version ahead of workers
- Never skip minor versions (1.27 → 1.28 → 1.29, NOT 1.27 → 1.29)
- Always read release notes for breaking changes

#### Pre-Upgrade Checklist

```bash
# 1. Check current version
kubectl version --short
kubectl get nodes

# 2. Review release notes
# Visit: https://github.com/kubernetes/kubernetes/releases

# 3. Check component health
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get componentstatuses  # deprecated but useful

# 4. Backup etcd (see section 4.2)

# 5. Check available versions
apt-cache madison kubeadm
```

#### Step 1: Upgrade First Control Plane Node

```bash
# 1. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.29.1-00  # Replace with target version
apt-mark hold kubeadm

# 2. Verify upgrade plan
sudo kubeadm upgrade plan

# Example output shows:
# - Current version
# - Target version
# - Components to be upgraded
# - Certificate expiration info

# 3. Apply upgrade
sudo kubeadm upgrade apply v1.29.1

# 4. Drain node
kubectl drain <control-plane-node> --ignore-daemonsets --delete-emptydir-data

# 5. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.29.1-00 kubectl=1.29.1-00
apt-mark hold kubelet kubectl

# 6. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 7. Verify kubelet is running
sudo systemctl status kubelet

# 8. Uncordon node
kubectl uncordon <control-plane-node>

# 9. Verify upgrade
kubectl get nodes
kubectl version --short
```

#### Step 2: Upgrade Additional Control Plane Nodes

```bash
# On each additional control plane node:

# 1. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.29.1-00
apt-mark hold kubeadm

# 2. Upgrade node (simpler than first node)
sudo kubeadm upgrade node

# 3. Drain node (from another control plane)
kubectl drain <control-plane-node> --ignore-daemonsets --delete-emptydir-data

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.29.1-00 kubectl=1.29.1-00
apt-mark hold kubelet kubectl

# 5. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Uncordon node
kubectl uncordon <control-plane-node>

# 7. Verify
kubectl get nodes
```

#### Step 3: Upgrade Worker Nodes

**Strategy Options:**
- **Rolling upgrade**: One node at a time (zero downtime)
- **Blue-green**: Provision new nodes, migrate workloads, decommission old
- **Canary**: Upgrade one node, test, then proceed with others

**Rolling Upgrade (Recommended):**

```bash
# For each worker node:

# 1. Drain node (from control plane)
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --force

# Verify pods have been evicted
kubectl get pods -o wide | grep <worker-node>

# 2. Upgrade kubeadm (on worker node)
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.29.1-00
apt-mark hold kubeadm

# 3. Upgrade node configuration
sudo kubeadm upgrade node

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.29.1-00 kubectl=1.29.1-00
apt-mark hold kubelet kubectl

# 5. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Verify kubelet is healthy
sudo systemctl status kubelet

# 7. Uncordon node (from control plane)
kubectl uncordon <worker-node>

# 8. Verify pods are rescheduled
kubectl get pods -o wide | grep <worker-node>

# 9. Wait and monitor before proceeding to next node
kubectl get nodes
kubectl get pods --all-namespaces
```

#### Post-Upgrade Verification

```bash
# Check all nodes are upgraded
kubectl get nodes

# Check all pods are running
kubectl get pods --all-namespaces

# Check component versions
kubectl version --short
kubelet --version
kubeadm version

# Run cluster health checks
kubectl get componentstatuses
kubectl cluster-info

# Check for any issues
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Verify workloads
kubectl get deployments --all-namespaces
kubectl get statefulsets --all-namespaces
kubectl get daemonsets --all-namespaces
```

#### Troubleshooting Upgrade Issues

```bash
# If upgrade fails on control plane:
# 1. Check kubeadm logs
journalctl -xeu kubeadm

# 2. Check kubelet logs
journalctl -xeu kubelet

# 3. Rollback is NOT officially supported
# Best practice: restore from etcd backup

# If pods don't start after worker upgrade:
# 1. Check node status
kubectl describe node <node-name>

# 2. Check pod events
kubectl describe pod <pod-name>

# 3. Check kubelet logs
journalctl -u kubelet -f

# 4. Check container runtime
crictl ps
crictl logs <container-id>
```

### 4.2 etcd Backup and Restore

#### Why Backup etcd?

etcd stores:
- All cluster objects (pods, services, deployments)
- Secrets and ConfigMaps
- RBAC policies
- Network policies
- Custom resources

**Losing etcd = Losing entire cluster state**

#### Backup etcd

**Manual Snapshot:**

```bash
# Create snapshot directory
sudo mkdir -p /backup/etcd

# Take snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd/etcd-snapshot-*.db --write-out=table

# Example output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 12345678 |    10000 |       1500 |     5.0 MB |
# +----------+----------+------------+------------+
```

**Automated Backup Script:**

```bash
# Create backup script
sudo tee /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash

# Configuration
BACKUP_DIR="/backup/etcd"
RETENTION_DAYS=7
ENDPOINTS="https://127.0.0.1:2379"
CACERT="/etc/kubernetes/pki/etcd/ca.crt"
CERT="/etc/kubernetes/pki/etcd/server.crt"
KEY="/etc/kubernetes/pki/etcd/server.key"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Generate backup filename
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-snapshot-${DATE}.db"

# Take snapshot
ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_FILE} \
  --endpoints=${ENDPOINTS} \
  --cacert=${CACERT} \
  --cert=${CERT} \
  --key=${KEY}

# Verify snapshot
if [ $? -eq 0 ]; then
    echo "Backup successful: ${BACKUP_FILE}"
    ETCDCTL_API=3 etcdctl snapshot status ${BACKUP_FILE}
else
    echo "Backup failed!"
    exit 1
fi

# Remove old backups
find ${BACKUP_DIR} -name "etcd-snapshot-*.db" -mtime +${RETENTION_DAYS} -delete

# Optional: Upload to remote storage (S3, GCS, etc.)
# aws s3 cp ${BACKUP_FILE} s3://my-backup-bucket/etcd/
EOF

# Make script executable
sudo chmod +x /usr/local/bin/etcd-backup.sh

# Test the script
sudo /usr/local/bin/etcd-backup.sh
```

**Schedule Automated Backups:**

```bash
# Create cron job (daily at 2 AM)
sudo tee /etc/cron.d/etcd-backup <<EOF
0 2 * * * root /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1
EOF

# Or use systemd timer
sudo tee /etc/systemd/system/etcd-backup.service <<EOF
[Unit]
Description=etcd Backup
Wants=etcd-backup.timer

[Service]
Type=oneshot
ExecStart=/usr/local/bin/etcd-backup.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/etcd-backup.timer <<EOF
[Unit]
Description=etcd Backup Timer
Requires=etcd-backup.service

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable etcd-backup.timer
sudo systemctl start etcd-backup.timer
sudo systemctl list-timers etcd-backup.timer
```

#### Restore etcd

**⚠️ WARNING: This is a DESTRUCTIVE operation. Only perform in disaster recovery scenarios.**

**Single Control Plane Restore:**

```bash
# 1. Stop kube-apiserver (move manifest out)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Stop etcd (if running as service)
# sudo systemctl stop etcd

# 3. Backup current etcd data (if salvageable)
sudo mv /var/lib/etcd /var/lib/etcd.backup

# 4. Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd/etcd-snapshot-20240215.db \
  --data-dir=/var/lib/etcd-restore \
  --name=master1 \
  --initial-cluster=master1=https://192.168.1.101:2380 \
  --initial-advertise-peer-urls=https://192.168.1.101:2380

# 5. Update etcd data directory in manifest
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd
# To: --data-dir=/var/lib/etcd-restore

# 6. Set correct ownership
sudo chown -R etcd:etcd /var/lib/etcd-restore  # if etcd user exists

# 7. Restart etcd (move kube-apiserver back)
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 8. Wait for cluster to come up
watch kubectl get nodes

# 9. Verify cluster state
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get ns
```

**HA Cluster Restore (All Control Planes):**

```bash
# THIS MUST BE DONE ON ALL CONTROL PLANE NODES

# On each control plane node:

# 1. Stop kube-apiserver
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restore snapshot with correct cluster info
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd/etcd-snapshot-20240215.db \
  --data-dir=/var/lib/etcd-restore \
  --name=<THIS_NODE_NAME> \
  --initial-cluster=master1=https://192.168.1.101:2380,master2=https://192.168.1.102:2380,master3=https://192.168.1.103:2380 \
  --initial-advertise-peer-urls=https://<THIS_NODE_IP>:2380 \
  --initial-cluster-token=etcd-cluster-1

# 3. Update etcd manifest
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change data-dir to /var/lib/etcd-restore

# 4. Set ownership
sudo chown -R etcd:etcd /var/lib/etcd-restore

# 5. Start etcd (restore kube-apiserver)
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 6. Verify etcd cluster
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 7. Check cluster health
kubectl get nodes
kubectl cluster-info
```

#### Restore Verification

```bash
# 1. Check all nodes are Ready
kubectl get nodes

# 2. Check system pods
kubectl get pods -n kube-system

# 3. Verify etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 4. Check application workloads
kubectl get pods --all-namespaces
kubectl get deployments --all-namespaces
kubectl get statefulsets --all-namespaces

# 5. Verify specific resources
kubectl get secrets --all-namespaces
kubectl get configmaps --all-namespaces
kubectl get pvc --all-namespaces

# 6. Test cluster functionality
kubectl run test-pod --image=nginx --restart=Never
kubectl get pod test-pod
kubectl delete pod test-pod
```

#### Backup and Restore Best Practices

**Backup:**
- ✅ Automate daily backups
- ✅ Store backups off-cluster (S3, GCS, NFS)
- ✅ Retain backups for 7-30 days
- ✅ Test restore procedure regularly (at least quarterly)
- ✅ Document restore steps
- ✅ Monitor backup job success
- ✅ Encrypt backup files
- ✅ Include backup as part of disaster recovery plan

**Restore:**
- ⚠️ Only restore in true disaster scenarios
- ⚠️ Test restores in non-production first
- ⚠️ Communicate with team before restore
- ⚠️ Have rollback plan
- ⚠️ Verify data consistency after restore
- ⚠️ Update documentation with lessons learned

#### Backup Alternatives

**Velero (Recommended for Production):**

```bash
# Install Velero with Helm
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-backup-bucket \
  --set configuration.volumeSnapshotLocation.config.region=us-east-1

# Create backup schedule
velero schedule create daily-backup --schedule="0 2 * * *"

# Manual backup
velero backup create manual-backup-$(date +%Y%m%d)

# Restore from backup
velero restore create --from-backup manual-backup-20240215
```

**Advantages of Velero:**
- Backs up entire namespaces, including persistent volumes
- Application-consistent backups
- Disaster recovery across clusters
- Migration between clusters

---

## 5. Quick Reference

### 5.1 Essential Commands

#### Cluster Information

```bash
# Cluster info
kubectl cluster-info
kubectl cluster-info dump  # Detailed cluster state

# Node information
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes  # Requires metrics-server

# Component status (deprecated but useful)
kubectl get componentstatuses
kubectl get cs
```

#### Certificate Management

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Example output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY
# admin.conf                 Feb 15, 2025 12:00 UTC   364d            ca
# apiserver                  Feb 15, 2025 12:00 UTC   364d            ca
# apiserver-etcd-client      Feb 15, 2025 12:00 UTC   364d            etcd-ca
# ...

# Renew all certificates
sudo kubeadm certs renew all

# Renew specific certificate
sudo kubeadm certs renew apiserver

# After renewing, restart control plane components
sudo systemctl restart kubelet
```

#### Token Management

```bash
# List existing tokens
kubeadm token list

# Create new token (24h default TTL)
kubeadm token create

# Create token with custom TTL
kubeadm token create --ttl 2h

# Generate complete join command
kubeadm token create --print-join-command

# Delete token
kubeadm token delete <token-id>
```

#### Static Pod Management

```bash
# View static pod manifests
ls -la /etc/kubernetes/manifests/

# Edit static pod (automatically restarted)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Temporarily disable static pod (move manifest out)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Re-enable static pod
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

#### Kubelet Management

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs (last 100 lines)
journalctl -u kubelet -n 100

# Follow kubelet logs
journalctl -u kubelet -f

# View kubelet config
ps aux | grep kubelet
cat /var/lib/kubelet/config.yaml

# Restart kubelet
sudo systemctl restart kubelet

# View kubelet version
kubelet --version
```

#### etcd Management

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# List etcd members
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd endpoint status
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Compact etcd database
ETCDCTL_API=3 etcdctl compact <revision> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Defragment etcd
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

#### Container Runtime Commands

```bash
# Using crictl (works with containerd, CRI-O)

# List running containers
crictl ps

# List all containers (including stopped)
crictl ps -a

# List pods
crictl pods

# View container logs
crictl logs <container-id>
crictl logs --tail=50 <container-id>

# List images
crictl images

# Inspect container
crictl inspect <container-id>

# Execute command in container
crictl exec -it <container-id> /bin/sh

# Pull image
crictl pull nginx:latest

# Remove container
crictl rm <container-id>

# Remove image
crictl rmi <image-id>
```

#### Cluster Reset

```bash
# Reset node (WARNING: removes all Kubernetes components)
sudo kubeadm reset

# Complete cleanup
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
sudo rm -rf /var/lib/etcd

# If using iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

### 5.2 Key File Locations

#### Kubernetes Configuration Files

```
/etc/kubernetes/
├── admin.conf                      # kubectl admin config
├── controller-manager.conf         # controller-manager config
├── kubelet.conf                    # kubelet config
├── scheduler.conf                  # scheduler config
├── manifests/                      # Static pod manifests
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml
│   └── kube-scheduler.yaml
└── pki/                           # TLS certificates
    ├── ca.crt                     # Cluster CA certificate
    ├── ca.key                     # Cluster CA key
    ├── apiserver.crt              # API server certificate
    ├── apiserver.key              # API server key
    ├── apiserver-kubelet-client.crt
    ├── apiserver-kubelet-client.key
    ├── front-proxy-ca.crt
    ├── front-proxy-ca.key
    ├── front-proxy-client.crt
    ├── front-proxy-client.key
    ├── sa.key                     # Service account signing key
    ├── sa.pub                     # Service account public key
    └── etcd/
        ├── ca.crt                 # etcd CA certificate
        ├── ca.key                 # etcd CA key
        ├── server.crt             # etcd server certificate
        ├── server.key             # etcd server key
        ├── peer.crt               # etcd peer certificate
        ├── peer.key               # etcd peer key
        ├── healthcheck-client.crt
        └── healthcheck-client.key
```

#### Kubelet Files

```
/var/lib/kubelet/
├── config.yaml                    # Kubelet configuration
├── kubeadm-flags.env             # Kubelet startup flags
├── pki/
│   ├── kubelet-client-current.pem # Kubelet client certificate
│   ├── kubelet.crt                # Kubelet serving certificate
│   └── kubelet.key                # Kubelet serving key
├── pods/                          # Pod data and volumes
└── plugins/                       # CNI and CSI plugins
```

#### etcd Data

```
/var/lib/etcd/                     # etcd database directory
└── member/
    ├── snap/                      # etcd snapshots
    └── wal/                       # etcd write-ahead log
```

#### CNI Configuration

```
/etc/cni/net.d/                    # CNI plugin configuration
└── 10-flannel.conflist            # Flannel CNI config (example)
```

#### Container Runtime

```
# Containerd
/etc/containerd/config.toml        # Containerd configuration
/var/lib/containerd/               # Containerd data

# CRI-O
/etc/crio/crio.conf               # CRI-O configuration
/var/lib/containers/              # CRI-O data
```

#### Log Locations

```
# Kubelet logs
journalctl -u kubelet

# Container runtime logs
journalctl -u containerd
journalctl -u crio

# Pod logs (via kubectl)
kubectl logs <pod-name>
kubectl logs -f <pod-name>         # Follow logs
kubectl logs --previous <pod-name> # Previous container logs

# Audit logs (if enabled)
/var/log/kubernetes/audit/audit.log
```

---

## 6. Production Checklist

### Infrastructure

- [ ] **High Availability**
  - [ ] Minimum 3 control plane nodes configured
  - [ ] Load balancer configured for API server
  - [ ] Load balancer health checks enabled
  - [ ] Control plane nodes distributed across failure domains

- [ ] **etcd**
  - [ ] etcd cluster has odd number of members (3 or 5)
  - [ ] etcd running on SSD storage
  - [ ] etcd data directory on dedicated disk
  - [ ] etcd automated backups configured (daily minimum)
  - [ ] etcd backup retention policy defined (7-30 days)
  - [ ] etcd restore procedure documented and tested
  - [ ] etcd metrics monitoring enabled

### Security

- [ ] **Certificates**
  - [ ] Certificate expiry monitoring enabled
  - [ ] Certificate renewal procedure documented
  - [ ] Certificates backed up securely

- [ ] **Access Control**
  - [ ] RBAC enabled and configured
  - [ ] Service accounts follow least privilege principle
  - [ ] Pod Security Standards enforced
  - [ ] Network policies defined and applied
  - [ ] API server audit logging enabled

- [ ] **Node Hardening**
  - [ ] OS security updates applied
  - [ ] Firewall rules configured
  - [ ] SELinux/AppArmor enabled
  - [ ] SSH access restricted
  - [ ] Unnecessary services disabled

### Operations

- [ ] **Monitoring**
  - [ ] Metrics server deployed
  - [ ] Prometheus/Grafana stack deployed
  - [ ] Alerting rules configured
  - [ ] Node and pod metrics collection enabled
  - [ ] Log aggregation configured (ELK, Loki, etc.)

- [ ] **Resource Management**
  - [ ] Resource requests and limits defined for all workloads
  - [ ] Resource quotas configured per namespace
  - [ ] Limit ranges defined
  - [ ] Pod Disruption Budgets configured for critical apps

- [ ] **Upgrades**
  - [ ] Upgrade runbook documented
  - [ ] Upgrade procedure tested in staging
  - [ ] Rollback procedure documented
  - [ ] Maintenance window scheduled

- [ ] **Backup and DR**
  - [ ] etcd backup automation configured
  - [ ] Application backup solution deployed (Velero)
  - [ ] Disaster recovery plan documented
  - [ ] Recovery procedures tested quarterly

### Configuration

- [ ] **Kubelet**
  - [ ] Systemd cgroup driver configured
  - [ ] Resource reservation configured
  - [ ] Image garbage collection configured
  - [ ] Container log rotation enabled

- [ ] **Container Runtime**
  - [ ] Systemd cgroup driver enabled
  - [ ] Image pull policies configured
  - [ ] Registry mirrors configured (if applicable)

- [ ] **Networking**
  - [ ] CNI plugin installed and healthy
  - [ ] DNS resolution working
  - [ ] Service mesh deployed (if required)
  - [ ] Ingress controller deployed

### Documentation

- [ ] Architecture diagram created
- [ ] Runbooks documented
  - [ ] Cluster bootstrap
  - [ ] Node addition/removal
  - [ ] Upgrades
  - [ ] Disaster recovery
  - [ ] Troubleshooting
- [ ] Contact information for escalations
- [ ] On-call procedures defined

---

## 7. Further Reading

### Official Documentation

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [High Availability Clusters](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [etcd Documentation](https://etcd.io/docs/)
- [etcd Operations Guide](https://etcd.io/docs/v3.5/op-guide/)

### Best Practices

- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Production Best Practices](https://kubernetes.io/docs/setup/best-practices/)
- [Security Best Practices](https://kubernetes.io/docs/concepts/security/security-checklist/)

### CKA Exam Preparation

- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [CKA Certification](https://www.cncf.io/certification/cka/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

### Community Resources

- [Kubernetes Slack](https://kubernetes.slack.com/)
- [Kubernetes GitHub](https://github.com/kubernetes/kubernetes)
- [CNCF Blog](https://www.cncf.io/blog/)

### Tools

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - Context and namespace switching
- [k9s](https://k9scli.io/) - Terminal UI for Kubernetes
- [Lens](https://k8slens.dev/) - Kubernetes IDE
- [Velero](https://velero.io/) - Backup and disaster recovery
- [Helm](https://helm.sh/) - Package manager for Kubernetes

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Author:** Sajal Jana

**License:** This document is provided as-is for educational purposes. Feel free to use and modify for your learning and production needs.
