# Kubernetes: The "Not So Scary" Guide 🚀
**Author:** Sajal Jana
---

## Table of Contents
- [1. Cluster Architecture (The Gang's All Here)](#1-cluster-architecture)
- [2. Installation with kubeadm (Let's Build This Thing)](#2-installation-with-kubeadm)
- [3. High Availability (Because One is the Loneliest Number)](#3-high-availability)
- [4. Lifecycle Management (Keep It Fresh)](#4-lifecycle-management)
- [5. Quick Reference (Your Cheat Sheet)](#5-quick-reference)
- [6. Production Checklist (Don't Skip This!)](#6-production-checklist)

---

## 1. Cluster Architecture

### The Squad

Think of Kubernetes like a well-organized restaurant:

```
┌─────────────────────────────────────┐
│    CONTROL PLANE (The Kitchen)      │
│  📋 API Server  💾 etcd  📅 Scheduler│
│       (Head Chef & Recipe Book)      │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│     WORKER NODES (Dining Room)      │
│  🏃 kubelet  🌐 kube-proxy           │
│     (Waiters serving containers)     │
└─────────────────────────────────────┘
```

### Control Plane Components

#### kube-apiserver (The Boss)
*The one who talks to everyone but does no actual work*

- Validates every request (yes, EVERY single one)
- Only one allowed to talk to etcd (it's exclusive like that)
- If this dies, your cluster is basically a paperweight

```bash
# Is the boss alive?
kubectl get --raw /healthz
```

#### etcd (The Brain)
*Where all your secrets live... literally*

- Uses Raft consensus (fancy way of saying "we vote on stuff")
- Stores EVERYTHING (pods, secrets, that embarrassing typo in your deployment)
- **No etcd = No cluster** 💀
- Think of it as your cluster's brain - backup or suffer!

```bash
# Brain health check
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Pro Tip:** Those certificate paths are longer than a CVS receipt. Get used to it.

#### kube-scheduler (The Matchmaker)
*"This pod would look PERFECT on that node!"*

Decides where your pods live based on:
- Resources (does it fit?)
- Affinity (does it like its neighbors?)
- Taints (is it too picky?)
- Your tears (just kidding... or am I?)

#### kube-controller-manager (The Helicopter Parent)
*Constantly watching, always judging*

Runs a bunch of controllers:
- **Node Controller**: "Is everyone alive?!"
- **Replication Controller**: "We need exactly 3 replicas, WHERE'S THE THIRD ONE?!"
- **Deployment Controller**: "Time to roll out... slowly... carefully..."

### Worker Node Components

#### kubelet (The Worker Bee)
*Does all the actual work while the control plane takes credit*

- Registers nodes (like checking in at a hotel)
- Ensures containers are running (or at least pretending to)
- Reports status back to the boss
- Never gets a day off

```bash
# Is the worker bee still buzzing?
systemctl status kubelet
journalctl -u kubelet -f  # Watch it complain in real-time
```

#### kube-proxy (The Network Ninja)
*Routes traffic like a bouncer at a club*

- Implements Services (the concept, not actual work ethic)
- Manages iptables rules (prepare for a LOT of rules)
- Handles connection forwarding
- Works in the shadows (literally, check those iptables)

```bash
# What mode is this ninja using?
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"
```

#### Container Runtime
*The actual muscle doing the heavy lifting*

- **containerd**: The popular kid (most common)
- **CRI-O**: The hipster (lightweight and cool)
- **Docker**: The retired veteran (deprecated as of 1.24, F in chat)

```bash
# List what's actually running
crictl ps
crictl pods
crictl images
```

---

## 2. Installation with kubeadm

### Pre-Flight Checklist (Don't Skip or Cry Later)

#### 1. Disable Swap (Kubernetes Hates Swap)
*"Swap is like training wheels for RAM. We don't do that here."*

```bash
# Kill swap dead
sudo swapoff -a

# Make sure it stays dead after reboot
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify it's really dead
free -h  # Should show 0 swap
```

**Why?** Because Kubernetes is a resource management diva and swap messes with its vibes.

#### 2. Load Kernel Modules (Make Linux Do Linux Things)

```bash
# Tell Linux we need some special powers
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Did it work? Let's check
lsmod | grep br_netfilter
lsmod | grep overlay
```

#### 3. Sysctl Parameters (Network Magic Settings)
*Without this, pods can't talk to each other. Awkward.*

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

#### 4. Install containerd (The Container Whisperer)

```bash
sudo apt-get update
sudo apt-get install -y containerd

# Create config (it's picky about this)
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# THE MOST IMPORTANT LINE (seriously, don't skip)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Start it up
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Fun Fact:** Forget that systemd cgroup line and enjoy hours of debugging pods stuck in "ContainerCreating". You're welcome.

### Install The Holy Trinity

```bash
# Add Kubernetes repo
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet, kubectl (the dream team)
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# HOLD THEM (don't let apt update these by accident)
sudo apt-mark hold kubelet kubeadm kubectl
```

### Bootstrap Control Plane (The Big Moment)

```bash
# Deep breath... here we go!
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<YOUR_IP> \
  --kubernetes-version=v1.29.0

# 🚨 SAVE THAT JOIN COMMAND! 🚨
# It looks like: kubeadm join 192.168.1.100:6443 --token abc...
# You'll need it for worker nodes (and you WILL forget it)
```

#### Configure kubectl (So You Can Actually Use It)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# The moment of truth
kubectl get nodes
# Should show 1 node... but NotReady (don't panic)
```

#### Install CNI Plugin (Because Pods Need to Talk)
*No CNI = Sad pods stuck in "ContainerCreating" purgatory*

```bash
# Flannel is popular and easy (like pizza)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Wait for pods to be ready (patience is a virtue)
kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=300s

# NOW check nodes
kubectl get nodes  # Should be Ready! 🎉
```

### Join Worker Nodes (Growing the Family)

```bash
# On each worker node, use that command you definitely saved:
sudo kubeadm join 192.168.1.100:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Token Expired? (Of Course It Did)
*Default TTL is 24 hours because Kubernetes doesn't trust you*

```bash
# Generate new join command (on control plane)
kubeadm token create --print-join-command

# Or make token last longer
kubeadm token create --ttl 48h
```

### Common Pitfalls (A.K.A. Why Isn't This Working?!)

| Problem | You'll See | Fix It |
|---------|-----------|---------|
| **Swap still on** | kubelet refuses to start | `swapoff -a` (did you edit fstab though?) |
| **Wrong cgroup driver** | Pods: "ContainerCreating" forever | Edit containerd config, set SystemdCgroup=true |
| **No CNI** | Node: NotReady, Pods: sad | Install Flannel/Calico/your favorite CNI |
| **Firewall blocking** | Nodes won't join | Open ports 6443, 10250, etc. |
| **CoreDNS dying** | DNS doesn't work | CNI issue, fix that first |

**Ports You Need Open (Don't Firewall These):**
- **6443**: API server (kind of important)
- **2379-2380**: etcd (super important)
- **10250**: kubelet (very important)
- **30000-32767**: NodePort services (nice to have)

---

## 3. High Availability

### Why HA? (Because Single Points of Failure Are Scary)

One control plane = One coffee spill away from disaster 

```
         ┌──────────────┐
         │ Load Balancer│  ← Your traffic cop
         └──────┬───────┘
                │
      ┌─────────┼─────────┐
      │         │         │
  ┌───▼───┐ ┌──▼────┐ ┌──▼────┐
  │Master1│ │Master2│ │Master3│  ← The council of wise nodes
  │(+etcd)│ │(+etcd)│ │(+etcd)│
  └───────┘ └───────┘ └───────┘
```

**Rule of Thumb:** 
- 3 control planes = Can lose 1 (like a three-legged stool)
- 5 control planes = Can lose 2 (very stable, very expensive)
- 7+ control planes = You're either Google or paranoid

### Setup HA (The Professional Way)

#### Step 1: Load Balancer (The Traffic Director)

**HAProxy Config:**
```bash
# Simple HAProxy setup
sudo tee /etc/haproxy/haproxy.cfg <<EOF
frontend k8s-api
    bind *:6443
    default_backend k8s-api-backend

backend k8s-api-backend
    balance roundrobin
    server master1 192.168.1.101:6443 check
    server master2 192.168.1.102:6443 check
    server master3 192.168.1.103:6443 check
EOF

sudo systemctl restart haproxy
```

#### Step 2: Init First Control Plane

```bash
sudo kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \  # LB address
  --upload-certs \  # Share certs with other masters
  --pod-network-cidr=10.244.0.0/16

# SAVE THREE THINGS:
# 1. kubectl config commands
# 2. Control plane join command (--control-plane flag)
# 3. Worker join command
```

#### Step 3: Join Additional Control Planes

```bash
# On master2 and master3
sudo kubeadm join 192.168.1.100:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \  # This makes it a control plane!
  --certificate-key <CERT_KEY>
```

**Certificate Key Expired?** (They last 2 hours because reasons)
```bash
# Re-upload certs on existing master
sudo kubeadm init phase upload-certs --upload-certs
```

### etcd Topology Choices

#### Stacked etcd (Default - Easy Mode)
*etcd lives WITH control plane (like roommates)*

**Pros:** Simpler, fewer servers, lower cost
**Cons:** If node dies, you lose both control plane AND etcd

**Good for:** Small-medium clusters, tight budgets

#### External etcd (Hard Mode - Pro Edition)
*etcd gets its own VIP section*

**Pros:** Decoupled failures, better scaling
**Cons:** More servers, more complexity, more $$$ 

**Good for:** Large clusters, mission-critical stuff, showing off

### etcd Best Practices (Listen Carefully)

**The Golden Rules:**
1. **Always odd numbers** (3, 5, 7) - Raft consensus isn't into even numbers
2. **Use SSDs** - etcd on HDD is like running in flip-flops
3. **Backup daily** - Or enjoy explaining data loss to your boss
4. **Monitor latency** - >10ms between members = bad time
5. **Don't exceed 8GB** - If you need bigger, you're doing it wrong

```bash
# Check etcd health (do this often)
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## 4. Lifecycle Management

### Kubernetes Upgrades (Approach with Caution)

**The Rules:**
- ⚠️ **Never skip versions** (1.27 → 1.28 → 1.29, not 1.27 → 1.29)
- 📖 **Read release notes** (yes, actually read them)
- 💾 **Backup etcd first** (seriously, FIRST)
- 🧪 **Test in staging** (production is not a test environment)

#### Pre-Upgrade Ritual

```bash
# 1. What version am I on?
kubectl version --short

# 2. Backup etcd (DO NOT SKIP)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. Check everything is healthy
kubectl get nodes
kubectl get pods --all-namespaces
```

#### Upgrade Control Plane (First Master)

```bash
# Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.1-00
apt-mark hold kubeadm

# Check upgrade plan (what's gonna happen)
sudo kubeadm upgrade plan

# Apply upgrade (hold your breath)
sudo kubeadm upgrade apply v1.29.1

# Drain node (kick pods off)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.1-00 kubectl=1.29.1-00
apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Bring node back
kubectl uncordon <node>
```

#### Upgrade Additional Masters

```bash
# Similar but easier (no "apply", just "node")
sudo kubeadm upgrade node

# Then drain, upgrade kubelet/kubectl, restart, uncordon
```

#### Upgrade Workers (One at a Time!)

```bash
# Drain worker
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data --force

# On worker node
apt-mark unhold kubeadm kubelet kubectl
apt-get update && apt-get install -y kubeadm=1.29.1-00 kubelet=1.29.1-00 kubectl=1.29.1-00
apt-mark hold kubeadm kubelet kubectl

sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon
kubectl uncordon <worker>

# Wait a bit, watch pods, then next worker
```

### etcd Backup (Your Cluster's Life Insurance)

#### Quick Backup

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify it worked
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-*.db --write-out=table
```

#### Automated Backup Script (Set It and Forget It)

```bash
sudo tee /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/etcd"
mkdir -p ${BACKUP_DIR}

ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_DIR}/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Delete old backups (keep 7 days)
find ${BACKUP_DIR} -name "etcd-*.db" -mtime +7 -delete
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh

# Cron it (daily at 2 AM)
echo "0 2 * * * root /usr/local/bin/etcd-backup.sh" | sudo tee /etc/cron.d/etcd-backup
```

#### Restore etcd (Break Glass Only)

**⚠️ WARNING:** This is DESTRUCTIVE. Only do this when everything is on fire.

```bash
# Stop API server
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --name=master1 \
  --initial-cluster=master1=https://192.168.1.101:2380

# Update etcd manifest to use new data dir
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-restore

# Start API server
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Cross fingers, pray to the cluster gods
kubectl get nodes
```

---

## 5. Quick Reference

### Essential Commands (Copy-Paste Paradise)

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# Component health
kubectl get componentstatuses
kubectl get pods -n kube-system

# Certificate expiration (don't let these expire)
kubeadm certs check-expiration

# Renew all certs
sudo kubeadm certs renew all

# Token management
kubeadm token list
kubeadm token create --print-join-command

# Kubelet logs (when things go wrong)
journalctl -u kubelet -f

# Container runtime
crictl ps
crictl pods
crictl logs <container-id>

# etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Key File Locations (Where Things Live)

```
/etc/kubernetes/
├── admin.conf              # kubectl config
├── manifests/              # Static pod manifests
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml
│   └── kube-scheduler.yaml
└── pki/                    # All the certs (don't lose these)

/var/lib/kubelet/           # Kubelet data
/var/lib/etcd/              # etcd database (THE PRECIOUS)
/etc/containerd/config.toml # Container runtime config
```

---

## 6. Production Checklist

### Before You Go Live (Don't Skip This!)

**Infrastructure:**
- [ ] At least 3 control plane nodes (odd numbers!)
- [ ] Load balancer for API server
- [ ] etcd on SSDs (seriously, SSDs)
- [ ] Nodes across different failure domains

**Security:**
- [ ] RBAC enabled
- [ ] Network policies configured
- [ ] Pod Security Standards enforced
- [ ] Audit logging enabled
- [ ] Certificates won't expire soon

**Operations:**
- [ ] Monitoring configured (Prometheus recommended)
- [ ] Log aggregation setup (ELK, Loki, etc.)
- [ ] Resource quotas per namespace
- [ ] Pod Disruption Budgets for critical apps

**Backup & DR:**
- [ ] Automated etcd backups (DAILY)
- [ ] Backup retention policy (7-30 days)
- [ ] Restore procedure documented AND TESTED
- [ ] Disaster recovery plan exists (not just in your head)

**Documentation:**
- [ ] Architecture diagram
- [ ] Runbooks for common tasks
- [ ] On-call procedures
- [ ] Escalation contacts

---

## Final Words of Wisdom

1. **Backup etcd** - Did I mention this? BACKUP ETCD.
2. **Test in staging** - Production is not your playground
3. **Read the docs** - Kubernetes changes fast
4. **Monitor everything** - If you can't see it, it's broken
5. **Keep it simple** - You're not Google (probably)
6. **Document everything** - Future you will thank you
7. **Join the community** - Kubernetes Slack is your friend

### Useful Resources

- [Official Docs](https://kubernetes.io/docs/) - Your bible
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Pain builds character
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) - Bookmark this
- [k9s](https://k9scli.io/) - CLI tool that doesn't hate you
- [CKA Curriculum](https://github.com/cncf/curriculum) - If you're studying

---

**Remember:** Every expert was once a beginner who didn't give up after their first "CrashLoopBackOff"

**Good luck, and may your pods always be Running! 🎉**

---


*Last Updated: February 2026*  
*Author: Sajal Jana 
