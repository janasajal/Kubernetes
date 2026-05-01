================================================================================
        KUBERNETES -- FIELD NOTES
        Author      : Sajal Jana
        Last Updated: February 2026
================================================================================

CLUSTER ARCHITECTURE
--------------------

  CONTROL PLANE
    kube-apiserver       --> The boss. Validates every request. Talks to etcd.
    etcd                 --> The brain. Stores everything. If this dies, RIP.
    kube-scheduler       --> Decides where pods land (resources, affinity, taints)
    kube-controller-mgr  --> Helicopter parent. Watches nodes, replicas, deployments.

  WORKER NODES
    kubelet              --> Does actual work. Registers node, runs containers.
    kube-proxy           --> Routes traffic, manages iptables rules.
    container runtime    --> containerd (common) / CRI-O (lightweight) / Docker (retired)


CONTROL PLANE LAYOUT
--------------------

  Load Balancer  :6443
       |
  +---------+---------+
  |         |         |
  Master1   Master2   Master3   <-- odd numbers only (Raft consensus)
  (+etcd)   (+etcd)   (+etcd)


KEY CHECKS
----------

  API server alive?
    $ kubectl get --raw /healthz

  etcd healthy?
    $ ETCDCTL_API=3 etcdctl endpoint health \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key

  kubelet alive?
    $ systemctl status kubelet
    $ journalctl -u kubelet -f

  kube-proxy mode?
    $ kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"

  Container runtime?
    $ crictl ps
    $ crictl pods
    $ crictl images


================================================================================
INSTALLATION WITH KUBEADM
================================================================================

-- Step 1: Disable Swap (mandatory) --

  $ sudo swapoff -a
  $ sudo sed -i '/ swap / s/^/#/' /etc/fstab
  $ free -h     --> Swap line should show 0

  Why: Kubernetes resource management breaks with swap enabled.


-- Step 2: Kernel Modules --

  $ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF

  $ sudo modprobe overlay
  $ sudo modprobe br_netfilter
  $ lsmod | grep br_netfilter
  $ lsmod | grep overlay


-- Step 3: Sysctl (network settings) --

  $ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF

  $ sudo sysctl --system


-- Step 4: Install containerd --

  $ sudo apt-get update && sudo apt-get install -y containerd
  $ sudo mkdir -p /etc/containerd
  $ containerd config default | sudo tee /etc/containerd/config.toml

  CRITICAL -- do not skip this line:
    $ sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
        /etc/containerd/config.toml

  $ sudo systemctl restart containerd
  $ sudo systemctl enable containerd

  NOTE: Forget the SystemdCgroup line = pods stuck in ContainerCreating forever.


-- Step 5: Install kubeadm / kubelet / kubectl --

  $ sudo mkdir -p /etc/apt/keyrings
  $ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
      sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

  $ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
      https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
      sudo tee /etc/apt/sources.list.d/kubernetes.list

  $ sudo apt-get update
  $ sudo apt-get install -y kubelet kubeadm kubectl
  $ sudo apt-mark hold kubelet kubeadm kubectl   # pin versions!


-- Step 6: Bootstrap control plane --

  $ sudo kubeadm init \
      --pod-network-cidr=10.244.0.0/16 \
      --apiserver-advertise-address=<YOUR_IP> \
      --kubernetes-version=v1.29.0

  SAVE THE JOIN COMMAND from output. You will need it for workers.
  It expires in 24h.


-- Step 7: Configure kubectl --

  $ mkdir -p $HOME/.kube
  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config

  $ kubectl get nodes     --> shows 1 node, NotReady (normal, no CNI yet)


-- Step 8: Install CNI plugin --

  Without CNI pods can't talk to each other. Node stays NotReady.

  $ kubectl apply -f \
      https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

  $ kubectl wait --for=condition=ready pod -l k8s-app=kube-dns \
      -n kube-system --timeout=300s

  $ kubectl get nodes     --> Should now show Ready


-- Step 9: Join worker nodes --

  Run on each worker (use command saved from init output):

  $ sudo kubeadm join 192.168.1.100:6443 \
      --token <TOKEN> \
      --discovery-token-ca-cert-hash sha256:<HASH>

  Token expired (24h TTL)?
    $ kubeadm token create --print-join-command    # regenerate
    $ kubeadm token create --ttl 48h               # or make it last longer


PORTS TO KEEP OPEN
------------------

  6443       API server (most important)
  2379-2380  etcd
  10250      kubelet
  30000-32767 NodePort services


COMMON PITFALLS
---------------

  Swap still on          --> kubelet refuses to start. Run swapoff -a + fix fstab.
  Wrong cgroup driver    --> Pods stuck ContainerCreating. Fix SystemdCgroup in containerd.
  No CNI installed       --> Node NotReady, pods stuck. Install Flannel/Calico.
  Firewall blocking      --> Nodes won't join. Open ports above.
  CoreDNS crashing       --> Usually a CNI problem. Fix CNI first.


================================================================================
HIGH AVAILABILITY
================================================================================

-- HAProxy load balancer config --

  $ sudo tee /etc/haproxy/haproxy.cfg <<EOF
  frontend k8s-api
      bind *:6443
      default_backend k8s-api-backend

  backend k8s-api-backend
      balance roundrobin
      server master1 192.168.1.101:6443 check
      server master2 192.168.1.102:6443 check
      server master3 192.168.1.103:6443 check
  EOF

  $ sudo systemctl restart haproxy


-- Init first control plane --

  $ sudo kubeadm init \
      --control-plane-endpoint "192.168.1.100:6443" \
      --upload-certs \
      --pod-network-cidr=10.244.0.0/16

  SAVE THREE THINGS from output:
    1. kubectl config commands
    2. Control plane join command (has --control-plane flag)
    3. Worker join command


-- Join additional control planes --

  $ sudo kubeadm join 192.168.1.100:6443 \
      --token <TOKEN> \
      --discovery-token-ca-cert-hash sha256:<HASH> \
      --control-plane \
      --certificate-key <CERT_KEY>

  Certificate key expires in 2 hours. Regenerate if needed:
    $ sudo kubeadm init phase upload-certs --upload-certs


ETCD TOPOLOGY OPTIONS
---------------------

  Stacked etcd (default):
    etcd lives ON the control plane nodes (roommates)
    Simpler, fewer servers, cheaper
    Risk: lose node = lose control plane + etcd together
    Good for: small/medium clusters

  External etcd:
    etcd on separate dedicated nodes
    Better failure isolation, more scalable
    More servers, more cost, more complexity
    Good for: large clusters, mission-critical

ETCD GOLDEN RULES
-----------------

  Always odd node count    : 3, 5, or 7. Raft needs majority votes.
  Use SSDs                 : etcd on HDD = bad time.
  Backup daily             : Non-negotiable.
  Monitor latency          : >10ms between members = problem.
  Keep under 8GB           : If larger, rethink your data model.


================================================================================
LIFECYCLE MANAGEMENT
================================================================================

UPGRADE RULES
-------------

  Never skip versions      : 1.27 --> 1.28 --> 1.29. Not 1.27 --> 1.29.
  Read release notes       : Actually read them.
  Backup etcd FIRST        : Before anything else.
  Test in staging          : Production is not a test environment.


-- Pre-upgrade checks --

  $ kubectl version --short
  $ kubectl get nodes
  $ kubectl get pods --all-namespaces


-- Backup etcd (do this before every upgrade) --

  $ ETCDCTL_API=3 etcdctl snapshot save \
      /backup/etcd-$(date +%Y%m%d).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

  $ ETCDCTL_API=3 etcdctl snapshot status \
      /backup/etcd-snapshot-*.db --write-out=table


-- Upgrade first control plane --

  $ apt-mark unhold kubeadm
  $ apt-get update && apt-get install -y kubeadm=1.29.1-00
  $ apt-mark hold kubeadm

  $ sudo kubeadm upgrade plan
  $ sudo kubeadm upgrade apply v1.29.1

  $ kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

  $ apt-mark unhold kubelet kubectl
  $ apt-get install -y kubelet=1.29.1-00 kubectl=1.29.1-00
  $ apt-mark hold kubelet kubectl

  $ sudo systemctl daemon-reload
  $ sudo systemctl restart kubelet

  $ kubectl uncordon <node>


-- Upgrade additional control planes --

  Same as above but replace:
    kubeadm upgrade apply  -->  kubeadm upgrade node


-- Upgrade workers (one at a time!) --

  $ kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data --force

  On the worker node:
    $ apt-mark unhold kubeadm kubelet kubectl
    $ apt-get install -y kubeadm=1.29.1-00 kubelet=1.29.1-00 kubectl=1.29.1-00
    $ apt-mark hold kubeadm kubelet kubectl
    $ sudo kubeadm upgrade node
    $ sudo systemctl daemon-reload && sudo systemctl restart kubelet

  $ kubectl uncordon <worker>

  Wait, watch pods, then move to next worker.


ETCD BACKUP AUTOMATION
----------------------

  $ sudo tee /usr/local/bin/etcd-backup.sh <<'EOF'
  #!/bin/bash
  BACKUP_DIR="/backup/etcd"
  mkdir -p ${BACKUP_DIR}

  ETCDCTL_API=3 etcdctl snapshot save \
    ${BACKUP_DIR}/etcd-$(date +%Y%m%d-%H%M%S).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

  find ${BACKUP_DIR} -name "etcd-*.db" -mtime +7 -delete
  EOF

  $ sudo chmod +x /usr/local/bin/etcd-backup.sh
  $ echo "0 2 * * * root /usr/local/bin/etcd-backup.sh" | \
      sudo tee /etc/cron.d/etcd-backup


ETCD RESTORE (BREAK GLASS ONLY)
--------------------------------

  WARNING: Destructive operation. Only run when cluster is fully broken.

  $ sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

  $ ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
      --data-dir=/var/lib/etcd-restore \
      --name=master1 \
      --initial-cluster=master1=https://192.168.1.101:2380

  Edit etcd manifest to point to new data dir:
    $ sudo vi /etc/kubernetes/manifests/etcd.yaml
      --> change: --data-dir=/var/lib/etcd-restore

  $ sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

  $ kubectl get nodes     --> verify recovery


================================================================================
QUICK REFERENCE COMMANDS
================================================================================

  kubectl cluster-info
  kubectl get nodes -o wide
  kubectl get componentstatuses
  kubectl get pods -n kube-system

  kubeadm certs check-expiration
  sudo kubeadm certs renew all

  kubeadm token list
  kubeadm token create --print-join-command

  journalctl -u kubelet -f

  crictl ps
  crictl pods
  crictl logs <container-id>


KEY FILE LOCATIONS
------------------

  /etc/kubernetes/admin.conf                        kubectl config
  /etc/kubernetes/manifests/etcd.yaml               etcd static pod
  /etc/kubernetes/manifests/kube-apiserver.yaml     apiserver static pod
  /etc/kubernetes/manifests/kube-controller-manager.yaml
  /etc/kubernetes/manifests/kube-scheduler.yaml
  /etc/kubernetes/pki/                              all certs (don't lose!)
  /var/lib/kubelet/                                 kubelet data
  /var/lib/etcd/                                    etcd data (THE PRECIOUS)
  /etc/containerd/config.toml                       container runtime config


================================================================================
WORDS TO LIVE BY
================================================================================

  Backup etcd           -- daily, automated, tested
  Never skip versions   -- one minor at a time
  Test in staging       -- production is not a playground
  Monitor everything    -- if you can't see it, it's broken
  Document everything   -- future you will be grateful
  Odd etcd nodes only   -- 3, 5, 7. Never even.
  SSDs for etcd         -- non-negotiable
  Read release notes    -- Kubernetes changes fast

================================================================================
  END OF NOTES
================================================================================
