
--------------------------------------------------------------------------------
CLUSTER TOPOLOGY
--------------------------------------------------------------------------------

  Control Plane Nodes:
    master-01  -->  192.168.1.10
    master-02  -->  192.168.1.11
    master-03  -->  192.168.1.12

  Worker Nodes:
    worker-01  -->  192.168.1.20
    worker-02  -->  192.168.1.21
    worker-03  -->  192.168.1.22

  Load Balancer (HAProxy/Keepalived):
    lb-vip     -->  192.168.1.100  (Virtual IP for API Server)

  Pod CIDR     :  10.244.0.0/16
  Service CIDR :  10.96.0.0/12
  CNI Plugin   :  Calico

================================================================================
PHASE 1 -- PREPARE ALL NODES (Run on EVERY node: masters + workers)
================================================================================

--- 1.1  Set Hostnames ---

  On master-01:
    $ hostnamectl set-hostname master-01

  On master-02:
    $ hostnamectl set-hostname master-02

  On master-03:
    $ hostnamectl set-hostname master-03

  On worker-01:
    $ hostnamectl set-hostname worker-01

  On worker-02:
    $ hostnamectl set-hostname worker-02

  On worker-03:
    $ hostnamectl set-hostname worker-03


--- 1.2  Update /etc/hosts on ALL nodes ---

  $ cat >> /etc/hosts <<EOF
  192.168.1.10  master-01
  192.168.1.11  master-02
  192.168.1.12  master-03
  192.168.1.20  worker-01
  192.168.1.21  worker-02
  192.168.1.22  worker-03
  192.168.1.100 k8s-api.prod.local
  EOF


--- 1.3  Disable Swap (MANDATORY for Kubernetes) ---

  $ swapoff -a
  $ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  Verify:
    $ free -h
      --> Swap should show 0B


--- 1.4  Load Required Kernel Modules ---

  $ cat > /etc/modules-load.d/k8s.conf <<EOF
  overlay
  br_netfilter
  EOF

  $ modprobe overlay
  $ modprobe br_netfilter

  Verify modules are loaded:
    $ lsmod | grep -E "overlay|br_netfilter"


--- 1.5  Configure Kernel Parameters for Networking ---

  $ cat > /etc/sysctl.d/k8s.conf <<EOF
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF

  Apply the settings:
    $ sysctl --system

  Verify:
    $ sysctl net.ipv4.ip_forward
      --> Should return: net.ipv4.ip_forward = 1


--- 1.6  Install Container Runtime (containerd) ---

  Remove old Docker packages if any:
    $ apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null

  Install prerequisites:
    $ apt-get update
    $ apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https

  Add Docker's official GPG key:
    $ install -m 0755 -d /etc/apt/keyrings
    $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    $ chmod a+r /etc/apt/keyrings/docker.gpg

  Add Docker repository:
    $ echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
        https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null

  Install containerd:
    $ apt-get update
    $ apt-get install -y containerd.io

  Configure containerd for Kubernetes:
    $ containerd config default > /etc/containerd/config.toml
    $ sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

  Enable and start containerd:
    $ systemctl enable containerd
    $ systemctl restart containerd
    $ systemctl status containerd

  Verify:
    $ ctr version
      --> Should show containerd version info


--- 1.7  Install kubeadm, kubelet, kubectl ---

  Add Kubernetes apt repository:
    $ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
        gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    $ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
        https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
        tee /etc/apt/sources.list.d/kubernetes.list

  Install tools:
    $ apt-get update
    $ apt-get install -y kubelet kubeadm kubectl

  Pin versions to prevent accidental upgrades:
    $ apt-mark hold kubelet kubeadm kubectl

  Enable kubelet:
    $ systemctl enable kubelet

  Verify installation:
    $ kubeadm version
    $ kubectl version --client
    $ kubelet --version


================================================================================
PHASE 2 -- SETUP LOAD BALANCER FOR HA (Run on dedicated LB node)
================================================================================

--- 2.1  Install HAProxy and Keepalived ---

  $ apt-get update
  $ apt-get install -y haproxy keepalived


--- 2.2  Configure HAProxy ---

  $ cat > /etc/haproxy/haproxy.cfg <<EOF
  global
      log /dev/log local0
      log /dev/log local1 notice
      daemon

  defaults
      mode                    http
      log                     global
      option                  httplog
      option                  dontlognull
      option http-server-close
      option forwardfor       except 127.0.0.0/8
      option                  redispatch
      retries                 3
      timeout http-request    10s
      timeout queue           1m
      timeout connect         10s
      timeout client          1m
      timeout server          1m
      timeout http-keep-alive 10s
      timeout check           10s
      maxconn                 3000

  frontend kubernetes-apiserver
      bind *:6443
      mode tcp
      option tcplog
      default_backend kubernetes-apiserver

  backend kubernetes-apiserver
      mode tcp
      option tcp-check
      balance roundrobin
      server master-01 192.168.1.10:6443 check fall 3 rise 2
      server master-02 192.168.1.11:6443 check fall 3 rise 2
      server master-03 192.168.1.12:6443 check fall 3 rise 2

  listen stats
      bind *:8080
      stats enable
      stats uri /haproxy_stats
      stats auth admin:StrongPassword@2026
  EOF

  $ systemctl enable haproxy
  $ systemctl restart haproxy
  $ systemctl status haproxy


--- 2.3  Configure Keepalived (for VIP failover) ---

  $ cat > /etc/keepalived/keepalived.conf <<EOF
  vrrp_script chk_haproxy {
      script "killall -0 haproxy"
      interval 2
      weight 2
  }

  vrrp_instance VI_1 {
      state MASTER
      interface eth0
      virtual_router_id 51
      priority 101
      advert_int 1
      authentication {
          auth_type PASS
          auth_pass k8sVIPpass2026!
      }
      virtual_ipaddress {
          192.168.1.100
      }
      track_script {
          chk_haproxy
      }
  }
  EOF

  $ systemctl enable keepalived
  $ systemctl restart keepalived

  Verify VIP is up:
    $ ip addr show eth0 | grep 192.168.1.100


================================================================================
PHASE 3 -- INITIALIZE THE FIRST CONTROL PLANE NODE (Run on master-01 only)
================================================================================

--- 3.1  Pull Required Images First (saves time during init) ---

  $ kubeadm config images pull --kubernetes-version=v1.29.0

  Verify images:
    $ crictl images | grep registry.k8s.io


--- 3.2  Create kubeadm config file ---

  $ cat > /root/kubeadm-config.yaml <<EOF
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: ClusterConfiguration
  kubernetesVersion: v1.29.0
  clusterName: prod-cluster
  controlPlaneEndpoint: "k8s-api.prod.local:6443"
  networking:
    podSubnet: "10.244.0.0/16"
    serviceSubnet: "10.96.0.0/12"
    dnsDomain: "cluster.local"
  apiServer:
    extraArgs:
      audit-log-maxage: "30"
      audit-log-maxbackup: "10"
      audit-log-maxsize: "100"
      audit-log-path: /var/log/audit/audit.log
      enable-admission-plugins: NodeRestriction,PodSecurityAdmission
      profiling: "false"
    certSANs:
      - "k8s-api.prod.local"
      - "192.168.1.100"
      - "master-01"
      - "master-02"
      - "master-03"
      - "192.168.1.10"
      - "192.168.1.11"
      - "192.168.1.12"
  controllerManager:
    extraArgs:
      profiling: "false"
  scheduler:
    extraArgs:
      profiling: "false"
  etcd:
    local:
      dataDir: /var/lib/etcd
      extraArgs:
        quota-backend-bytes: "8589934592"
  ---
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: InitConfiguration
  localAPIEndpoint:
    advertiseAddress: 192.168.1.10
    bindPort: 6443
  nodeRegistration:
    criSocket: unix:///var/run/containerd/containerd.sock
    kubeletExtraArgs:
      node-labels: "node-role=control-plane"
  ---
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  cgroupDriver: systemd
  maxPods: 110
  EOF


--- 3.3  Initialize the Cluster ---

  $ kubeadm init --config=/root/kubeadm-config.yaml --upload-certs 2>&1 | tee /root/kubeadm-init.log

  IMPORTANT: Save the output! It contains:
    - The join command for other control-plane nodes
    - The join command for worker nodes
    - The certificate key (expires after 2 hours)

  Example output to save (yours will be different):
  ...
  You can now join any number of control-plane node by running the following:
    kubeadm join k8s-api.prod.local:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash sha256:<HASH> \
        --control-plane --certificate-key <CERT-KEY>

  Then you can join any number of worker nodes by running:
    kubeadm join k8s-api.prod.local:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash sha256:<HASH>
  ...


--- 3.4  Configure kubectl for Admin Access ---

  $ mkdir -p $HOME/.kube
  $ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ chown $(id -u):$(id -g) $HOME/.kube/config

  Verify cluster is reachable:
    $ kubectl get nodes
      --> Should show master-01 in NotReady state (CNI not installed yet)

    $ kubectl get pods -n kube-system
      --> All system pods should be Running or Pending


================================================================================
PHASE 4 -- JOIN REMAINING CONTROL PLANE NODES (master-02 and master-03)
================================================================================

--- 4.1  Run join command on master-02 ---

  Replace <TOKEN>, <HASH>, <CERT-KEY> with values from Phase 3 output:

  $ kubeadm join k8s-api.prod.local:6443 \
      --token <TOKEN> \
      --discovery-token-ca-cert-hash sha256:<HASH> \
      --control-plane \
      --certificate-key <CERT-KEY> \
      --apiserver-advertise-address 192.168.1.11 \
      --cri-socket unix:///var/run/containerd/containerd.sock

  Setup kubectl on master-02:
    $ mkdir -p $HOME/.kube
    $ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ chown $(id -u):$(id -g) $HOME/.kube/config


--- 4.2  Run join command on master-03 ---

  $ kubeadm join k8s-api.prod.local:6443 \
      --token <TOKEN> \
      --discovery-token-ca-cert-hash sha256:<HASH> \
      --control-plane \
      --certificate-key <CERT-KEY> \
      --apiserver-advertise-address 192.168.1.12 \
      --cri-socket unix:///var/run/containerd/containerd.sock

  Setup kubectl on master-03:
    $ mkdir -p $HOME/.kube
    $ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ chown $(id -u):$(id -g) $HOME/.kube/config


--- 4.3  Verify all control plane nodes joined ---

  (Run on master-01)
    $ kubectl get nodes
      --> master-01, master-02, master-03 should all appear (NotReady is OK)

    $ kubectl get pods -n kube-system | grep etcd
      --> 3 etcd pods should be running


================================================================================
PHASE 5 -- INSTALL CNI PLUGIN (Calico) -- Run on master-01
================================================================================

--- 5.1  Download Calico manifests ---

  $ curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml \
      -o /root/calico.yaml


--- 5.2  Set correct Pod CIDR in Calico config ---

  $ sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|' /root/calico.yaml
  $ sed -i 's|#   value: "192.168.0.0/16"|  value: "10.244.0.0/16"|' /root/calico.yaml


--- 5.3  Apply Calico ---

  $ kubectl apply -f /root/calico.yaml

  Wait for Calico to come up (may take 2-3 minutes):
    $ kubectl rollout status daemonset calico-node -n kube-system --timeout=5m
    $ kubectl get pods -n kube-system -l k8s-app=calico-node


--- 5.4  Verify all nodes go Ready ---

  $ kubectl get nodes -o wide
    --> All 3 master nodes should show STATUS = Ready after Calico is up


================================================================================
PHASE 6 -- JOIN WORKER NODES (Run on worker-01, worker-02, worker-03)
================================================================================

--- 6.1  Run join command on each worker ---

  On worker-01, worker-02, worker-03 (one at a time or simultaneously):

  $ kubeadm join k8s-api.prod.local:6443 \
      --token <TOKEN> \
      --discovery-token-ca-cert-hash sha256:<HASH> \
      --cri-socket unix:///var/run/containerd/containerd.sock

  If token has expired (tokens expire after 24h), regenerate:
    $ kubeadm token create --print-join-command


--- 6.2  Label worker nodes ---

  (Run on master-01)
  $ kubectl label node worker-01 node-role.kubernetes.io/worker=worker
  $ kubectl label node worker-02 node-role.kubernetes.io/worker=worker
  $ kubectl label node worker-03 node-role.kubernetes.io/worker=worker


--- 6.3  Taint control plane nodes (prevent workload scheduling on masters) ---

  $ kubectl taint nodes master-01 node-role.kubernetes.io/control-plane=:NoSchedule
  $ kubectl taint nodes master-02 node-role.kubernetes.io/control-plane=:NoSchedule
  $ kubectl taint nodes master-03 node-role.kubernetes.io/control-plane=:NoSchedule


--- 6.4  Verify full cluster ---

  $ kubectl get nodes -o wide
    NAME        STATUS   ROLES           AGE   VERSION
    master-01   Ready    control-plane   Xm    v1.29.x
    master-02   Ready    control-plane   Xm    v1.29.x
    master-03   Ready    control-plane   Xm    v1.29.x
    worker-01   Ready    worker          Xm    v1.29.x
    worker-02   Ready    worker          Xm    v1.29.x
    worker-03   Ready    worker          Xm    v1.29.x


================================================================================
PHASE 7 -- PRODUCTION ESSENTIALS SETUP
================================================================================

--- 7.1  Install Metrics Server ---

  $ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

  Patch for production (add --kubelet-insecure-tls if using self-signed certs):
    $ kubectl patch deployment metrics-server -n kube-system \
        --type='json' \
        -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

  Verify:
    $ kubectl top nodes
    $ kubectl top pods -A


--- 7.2  Install NGINX Ingress Controller ---

  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml

  Verify:
    $ kubectl get pods -n ingress-nginx
    $ kubectl get svc -n ingress-nginx


--- 7.3  Install cert-manager (for TLS certificates) ---

  $ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

  Wait for cert-manager to be ready:
    $ kubectl rollout status deployment cert-manager -n cert-manager --timeout=3m
    $ kubectl rollout status deployment cert-manager-webhook -n cert-manager --timeout=3m

  Verify:
    $ kubectl get pods -n cert-manager


--- 7.4  Install Helm (Package Manager) ---

  $ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

  Verify:
    $ helm version


--- 7.5  Setup Storage Class with local-path-provisioner ---

  $ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

  Set as default StorageClass:
    $ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

  Verify:
    $ kubectl get storageclass


--- 7.6  Deploy Kubernetes Dashboard (optional but useful) ---

  $ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
  $ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
      --create-namespace --namespace kubernetes-dashboard

  Create admin service account:
    $ cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

  Get dashboard access token:
    $ kubectl -n kubernetes-dashboard create token admin-user


================================================================================
PHASE 8 -- PRODUCTION HARDENING
================================================================================

--- 8.1  Enable Pod Security Standards ---

  Label namespaces with appropriate security policies:

  For production namespaces (restrict):
    $ kubectl label namespace production \
        pod-security.kubernetes.io/enforce=restricted \
        pod-security.kubernetes.io/audit=restricted \
        pod-security.kubernetes.io/warn=restricted

  For system namespaces (baseline):
    $ kubectl label namespace kube-system \
        pod-security.kubernetes.io/enforce=privileged


--- 8.2  Setup Resource Quotas for namespaces ---

  $ cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: prod-quota
    namespace: production
  spec:
    hard:
      requests.cpu: "20"
      requests.memory: 40Gi
      limits.cpu: "40"
      limits.memory: 80Gi
      pods: "100"
      persistentvolumeclaims: "20"
  EOF


--- 8.3  Setup LimitRange ---

  $ cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: prod-limits
    namespace: production
  spec:
    limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
  EOF


--- 8.4  Configure etcd backup (CRITICAL for production) ---

  Create backup script:
    $ cat > /usr/local/bin/etcd-backup.sh <<'EOF'
    #!/bin/bash
    BACKUP_DIR=/var/backups/etcd
    DATE=$(date +%Y%m%d-%H%M%S)
    mkdir -p $BACKUP_DIR

    ETCDCTL_API=3 etcdctl snapshot save $BACKUP_DIR/etcd-snapshot-$DATE.db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

    # Keep only last 7 days of backups
    find $BACKUP_DIR -name "*.db" -mtime +7 -delete

    echo "Backup completed: $BACKUP_DIR/etcd-snapshot-$DATE.db"
    EOF

  $ chmod +x /usr/local/bin/etcd-backup.sh

  Schedule backup via cron (daily at 2 AM):
    $ echo "0 2 * * * root /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1" \
        >> /etc/crontab

  Test backup manually:
    $ /usr/local/bin/etcd-backup.sh


--- 8.5  Install etcdctl tool ---

  $ ETCD_VER=v3.5.11
  $ curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
      -o /tmp/etcd.tar.gz
  $ tar xzvf /tmp/etcd.tar.gz -C /tmp
  $ mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/

  Verify etcd cluster health:
    $ ETCDCTL_API=3 etcdctl member list \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key


================================================================================
PHASE 9 -- MONITORING SETUP (Prometheus + Grafana)
================================================================================

--- 9.1  Add Helm repos ---

  $ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  $ helm repo add grafana https://grafana.github.io/helm-charts
  $ helm repo update


--- 9.2  Create monitoring namespace ---

  $ kubectl create namespace monitoring


--- 9.3  Install kube-prometheus-stack ---

  $ cat > /root/prometheus-values.yaml <<EOF
  prometheus:
    prometheusSpec:
      retention: 30d
      storageSpec:
        volumeClaimTemplate:
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 50Gi

  grafana:
    adminPassword: "SecureGrafanaPass@2026!"
    persistence:
      enabled: true
      size: 10Gi
    ingress:
      enabled: true
      ingressClassName: nginx
      hosts:
        - grafana.prod.local

  alertmanager:
    alertmanagerSpec:
      storage:
        volumeClaimTemplate:
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 10Gi
  EOF

  $ helm install prometheus prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --values /root/prometheus-values.yaml

  Wait for rollout:
    $ kubectl rollout status deployment prometheus-grafana -n monitoring --timeout=5m

  Verify all monitoring pods are running:
    $ kubectl get pods -n monitoring


================================================================================
PHASE 10 -- FINAL VERIFICATION
================================================================================

--- 10.1  Full cluster health check ---

  Check all nodes:
    $ kubectl get nodes -o wide

  Check all system pods:
    $ kubectl get pods -A

  Check component status:
    $ kubectl get componentstatuses

  Check cluster info:
    $ kubectl cluster-info

  Check API server:
    $ curl -k https://k8s-api.prod.local:6443/healthz
      --> Should return: ok

  Check etcd health:
    $ ETCDCTL_API=3 etcdctl endpoint health \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key


--- 10.2  Run a test deployment ---

  $ kubectl create namespace test
  $ kubectl run nginx-test --image=nginx:alpine -n test
  $ kubectl expose pod nginx-test --port=80 --type=ClusterIP -n test

  Verify pod is running:
    $ kubectl get pods -n test
    $ kubectl get svc -n test

  Clean up test:
    $ kubectl delete namespace test


--- 10.3  Verify HA control plane (simulate master failure) ---

  On master-01, temporarily stop kubelet:
    $ systemctl stop kubelet

  From master-02, verify cluster is still functional:
    $ kubectl get nodes
    $ kubectl get pods -A

  Restart kubelet on master-01:
    $ systemctl start kubelet

  Verify master-01 rejoins:
    $ kubectl get nodes


================================================================================
APPENDIX -- USEFUL DAILY COMMANDS
================================================================================

  Get all resources in a namespace:
    $ kubectl get all -n <namespace>

  Describe a node:
    $ kubectl describe node <node-name>

  Get resource usage:
    $ kubectl top nodes
    $ kubectl top pods -A --sort-by=memory

  Check cluster events:
    $ kubectl get events -A --sort-by='.lastTimestamp'

  Check certificates expiry:
    $ kubeadm certs check-expiration

  Renew certificates (before expiry):
    $ kubeadm certs renew all

  Drain a node for maintenance:
    $ kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

  Uncordon node after maintenance:
    $ kubectl uncordon <node-name>

  Get kubeconfig for a user:
    $ kubectl config view --raw > ~/custom-kubeconfig.yaml

  Upgrade cluster (kubeadm workflow):
    $ apt-get update && apt-get install -y kubeadm=1.30.0-*
    $ kubeadm upgrade plan
    $ kubeadm upgrade apply v1.30.0

