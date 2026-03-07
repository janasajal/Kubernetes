# Kubernetes Monitoring Lab: Minikube + Prometheus + Grafana on AWS EC2

> Author : Sajal Jana

> **Hands-on lab documenting a complete Kubernetes monitoring setup from scratch.**  
> Deployed on AWS EC2, using Minikube as the Kubernetes engine, Prometheus for metrics collection, and Grafana for visualization.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tools & Versions Used](#tools--versions-used)
- [Phase 1: Infrastructure Setup](#phase-1-infrastructure-setup)
- [Phase 2: Install Dependencies](#phase-2-install-dependencies)
- [Phase 3: Start Minikube Cluster](#phase-3-start-minikube-cluster)
- [Phase 4: Deploy Monitoring Stack](#phase-4-deploy-monitoring-stack)
- [Phase 5: Access Dashboards](#phase-5-access-dashboards)
- [Phase 6: Deploy Sample Application](#phase-6-deploy-sample-application)
- [Phase 7: Metrics Debugging with PromQL](#phase-7-metrics-debugging-with-promql)
- [Phase 8: Custom Grafana Dashboard](#phase-8-custom-grafana-dashboard)
- [Phase 9: Alerting](#phase-9-alerting)
- [Phase 10: Kubernetes Self-Healing Demo](#phase-10-kubernetes-self-healing-demo)
- [Key Learnings](#key-learnings)
- [Cleanup](#cleanup)

---

## Architecture Overview

```
AWS EC2 (t3.large - Ubuntu 22.04)
└── Minikube Cluster (Docker driver)
    ├── kube-system namespace
    │   ├── coredns
    │   ├── etcd
    │   ├── kube-apiserver
    │   ├── kube-scheduler
    │   ├── kube-controller-manager
    │   └── metrics-server (addon)
    ├── monitoring namespace
    │   ├── Prometheus         (metrics scraping & storage)
    │   ├── Alertmanager       (alert routing)
    │   ├── Grafana            (visualization)
    │   ├── kube-state-metrics (K8s object metrics)
    │   └── node-exporter      (node level metrics)
    └── demo-app namespace
        ├── nginx-demo (6 replicas)
        └── load-gen (traffic generator)
```

---

## Tools & Versions Used

| Tool | Version |
|------|---------|
| Ubuntu | 22.04 LTS |
| Docker | 27.x |
| kubectl | v1.35.2 |
| Helm | v3.20.0 |
| Minikube | v1.38.1 |
| Kubernetes | v1.35.1 |
| kube-prometheus-stack | latest (Helm chart) |

---

## Phase 1: Infrastructure Setup

### EC2 Instance Configuration

Launched an EC2 instance via AWS Console with the following settings:

| Setting | Value |
|---------|-------|
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Instance Type | t3.large (2 vCPU, 8GB RAM) |
| Storage | 30 GB gp3 |
| Key Pair | k8s-lab-key.pem |

**Security Group Inbound Rules:**

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH access |
| 3000 | TCP | Grafana UI |
| 9090 | TCP | Prometheus UI |
| 9093 | TCP | Alertmanager UI |

### SSH into EC2

```bash
chmod 400 k8s-lab-key.pem
ssh -i k8s-lab-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## Phase 2: Install Dependencies

### System Update

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker

```bash
curl -fsSL https://get.docker.com | sudo bash
sudo usermod -aG docker ubuntu
newgrp docker
```

**Verify:**
```bash
docker ps
```

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

**Verify:**
```bash
kubectl version --client
```

```
Client Version: v1.35.2
Kustomize Version: v5.7.1
```

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Verify:**
```bash
helm version
```

```
version.BuildInfo{Version:"v3.20.0", GitCommit:"b2e4314fa0f229a1de7b4c981273f61d69ee5a59", GoVersion:"go1.25.6"}
```

### Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Verify:**
```bash
minikube version
```

```
minikube version: v1.38.1
commit: c93a4cb9311efc66b90d33ea03f75f2c4120e9b0
```

---

## Phase 3: Start Minikube Cluster

```bash
minikube start --driver=docker --cpus=2 --memory=3800
```

### Enable metrics-server addon

```bash
minikube addons enable metrics-server
```

### Verify cluster health

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   53s   v1.35.1
```

```bash
kubectl get pods -A
```

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS
kube-system   coredns-7d764666f9-swndq           1/1     Running   0
kube-system   etcd-minikube                      1/1     Running   0
kube-system   kube-apiserver-minikube            1/1     Running   0
kube-system   kube-controller-manager-minikube   1/1     Running   0
kube-system   kube-proxy-xwf54                   1/1     Running   0
kube-system   kube-scheduler-minikube            1/1     Running   0
kube-system   storage-provisioner                1/1     Running   1
```

All core components confirmed running ✅

---

## Phase 4: Deploy Monitoring Stack

### Add Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### Deploy kube-prometheus-stack

This single Helm chart deploys the complete monitoring stack: Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter.

```bash
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=7d
```

### Verify all pods are running

```bash
kubectl get pods -n monitoring
```

```
NAME                                                        READY   STATUS
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running
kube-prometheus-stack-grafana-59966b5786-fh8w8              3/3     Running
kube-prometheus-stack-kube-state-metrics-5cc5cc8bf4-rnj5d   1/1     Running
kube-prometheus-stack-operator-789b7f8576-75pfs             1/1     Running
kube-prometheus-stack-prometheus-node-exporter-ltbfp        1/1     Running
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running
```

All 6 monitoring components running ✅

---

## Phase 5: Access Dashboards

Port-forwarding exposes cluster services to the EC2 network so the browser can reach them.

### Expose Grafana

```bash
nohup kubectl port-forward svc/kube-prometheus-stack-grafana \
  3000:80 -n monitoring --address 0.0.0.0 \
  > /tmp/grafana-pf.log 2>&1 &
```

### Expose Prometheus

```bash
nohup kubectl port-forward svc/kube-prometheus-stack-prometheus \
  9090:9090 -n monitoring --address 0.0.0.0 \
  > /tmp/prometheus-pf.log 2>&1 &
```

### Expose Alertmanager

```bash
nohup kubectl port-forward svc/kube-prometheus-stack-alertmanager \
  9093:9093 -n monitoring --address 0.0.0.0 \
  > /tmp/alertmanager-pf.log 2>&1 &
```

### Access URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Grafana | `http://<EC2_IP>:3000` | admin / admin123 |
| Prometheus | `http://<EC2_IP>:9090` | No login required |
| Alertmanager | `http://<EC2_IP>:9093` | No login required |

### Verify all port-forwards

```bash
ps aux | grep port-forward
```

```
ubuntu  22325  kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 ...
ubuntu  23065  kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 ...
ubuntu  152204 kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 ...
```

---

## Phase 6: Deploy Sample Application

Deployed a real workload to the cluster to observe live metrics.

### Create namespace and deployment

```bash
kubectl create namespace demo-app

kubectl create deployment nginx-demo \
  --image=nginx:latest \
  --replicas=3 \
  --namespace=demo-app

kubectl expose deployment nginx-demo \
  --port=80 \
  --type=ClusterIP \
  --namespace=demo-app
```

### Watch pods come up

```bash
kubectl get pods -n demo-app -w
```

```
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-5dc66cb97d-8m67x   1/1     Running   0          53s
nginx-demo-5dc66cb97d-hdqwg   1/1     Running   0          53s
nginx-demo-5dc66cb97d-s66g5   1/1     Running   0          53s
```

### Generate traffic

```bash
kubectl run load-gen \
  --image=busybox \
  --restart=Never \
  --namespace=demo-app \
  -- /bin/sh -c "while true; do wget -q -O- http://nginx-demo; sleep 0.1; done"
```

**Verify traffic is flowing:**

```bash
kubectl logs load-gen -n demo-app | head -5
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Confirmed nginx is responding to load generator ✅

---

## Phase 7: Metrics Debugging with PromQL

### Important Discovery

Pre-built Grafana dashboards showed **"No data"** for the `demo-app` namespace. Root cause identified by querying Prometheus directly:

Minikube's cadvisor metrics use a `cpu="total"` label that pre-built dashboards don't filter for. This is a real-world scenario where dashboards built for production clusters may not match local Minikube label structures.

### Verified metrics exist in Prometheus

```promql
container_cpu_usage_seconds_total{namespace="demo-app"}
```

**Result:** 4 series returned — confirmed all pods being scraped ✅

```promql
count(container_cpu_usage_seconds_total) by (namespace)
```

**Result:** All namespaces confirmed (demo-app, kube-system, monitoring)

### Correct PromQL queries for this setup

```promql
# CPU usage per pod (Minikube specific - requires cpu="total" filter)
rate(container_cpu_usage_seconds_total{namespace="demo-app", cpu="total"}[5m])

# Memory usage per pod
container_memory_usage_bytes{namespace="demo-app"}

# Pod count per namespace
count(kube_pod_info) by (namespace)

# Node CPU usage %
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Pod restarts in last hour
increase(kube_pod_container_status_restarts_total[1h])
```

**Key lesson:** Always verify metrics exist in Prometheus before debugging Grafana. Use the Explore view in Grafana to test queries before building panels.

---

## Phase 8: Custom Grafana Dashboard

Since pre-built dashboards didn't match our Minikube label structure, built a custom dashboard from scratch using Grafana Explore.

### Steps taken

1. Grafana → **Explore** → switched to **Code** mode
2. Entered verified PromQL query
3. Confirmed graph appeared with all pod lines
4. Used **"Add to dashboard"** → **New dashboard**
5. Saved as **"demo-app CPU Monitoring"**

### CPU Panel Query

```promql
rate(container_cpu_usage_seconds_total{namespace="demo-app", cpu="total"}[5m])
```

### Memory Panel Query

```promql
container_memory_usage_bytes{namespace="demo-app"}
```

- Set unit to **bytes(IEC)** for human-readable MB display
- Set refresh to **10s** for live updates
- Set time range to **Last 15 minutes**

**Result:** Both CPU and Memory panels working live ✅

---

## Phase 9: Alerting

### Created Alert Rule in Grafana

**Alert Name:** `High CPU Usage - demo-app`

**Query:**
```promql
rate(container_cpu_usage_seconds_total{namespace="demo-app", cpu="total"}[5m])
```

**Conditions:**
- Threshold: IS ABOVE `0.5`
- Evaluate every: `1m`
- For: `2m`
- Label: `severity = warning`

### Triggered the alert with heavy load

```bash
kubectl run load-gen-heavy \
  --image=busybox \
  --restart=Never \
  --namespace=demo-app \
  -- /bin/sh -c "while true; do \
      wget -q -O- http://nginx-demo & \
      wget -q -O- http://nginx-demo & \
      wget -q -O- http://nginx-demo; \
      sleep 0.01; done"
```

### Observed full alert lifecycle

```
Normal → Pending → Firing 🔥
```

### Removed load and observed recovery

```bash
kubectl delete pod load-gen load-gen-heavy -n demo-app
```

```
Firing → Pending → Normal ✅
```

**Full alert lifecycle observed end to end ✅**

---

## Phase 10: Kubernetes Self-Healing Demo

### Scale deployment up

```bash
kubectl scale deployment nginx-demo --replicas=6 -n demo-app
kubectl get pods -n demo-app -w
```

All 6 pods reached `Running` state automatically.

### Force delete a pod

```bash
kubectl delete pod $(kubectl get pods -n demo-app \
  -l app=nginx-demo \
  -o jsonpath='{.items[0].metadata.name}') -n demo-app
```

### Observe automatic replacement

```bash
kubectl get pods -n demo-app -w
```

```
nginx-demo-5dc66cb97d-s66g5   Terminating
nginx-demo-5dc66cb97d-newpod  Pending         ← K8s detected mismatch
nginx-demo-5dc66cb97d-newpod  ContainerCreating
nginx-demo-5dc66cb97d-newpod  Running         ← back to 6 replicas
```

**K8s self-healing confirmed ✅** — desired state (6 replicas) maintained automatically with zero manual intervention.

---

## Key Learnings

### 1. Minikube vs Production Metric Labels
Minikube's cadvisor exposes `cpu="total"` label that production clusters don't use. Pre-built dashboards that work in production may show "No data" in Minikube. Always verify in Prometheus first.

### 2. Always Verify Data in Prometheus Before Debugging Grafana
The debugging workflow used in this lab:
- Prometheus → confirm metric exists
- Prometheus → find exact labels
- Grafana Explore → test query
- Grafana → build panel

### 3. kube-prometheus-stack is the Industry Standard
One Helm chart deploys the full observability stack — Prometheus, Grafana, Alertmanager, kube-state-metrics, node-exporter — with 20+ pre-built dashboards and alert rules.

### 4. PromQL is the Core Skill
Grafana is just a visualization layer. The real skill is writing PromQL queries that correctly filter by namespace, pod, container, and other labels.

### 5. Kubernetes Self-Healing
The controller-manager continuously reconciles actual state with desired state. Pod deletion triggers immediate replacement — no human intervention required.

---

## Cleanup

```bash
# Delete all workloads
kubectl delete namespace demo-app

# Uninstall monitoring stack
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring

# Stop Minikube
minikube stop

# Exit EC2
exit
```

**Stop EC2 in AWS Console to avoid charges:**
```
EC2 → Instances → Select → Instance State → Stop
```

> ⚠️ Use **Stop** not **Terminate** to preserve your work.

---

## References

- [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus documentation](https://prometheus.io/docs/)
- [Grafana documentation](https://grafana.com/docs/)
- [Minikube documentation](https://minikube.sigs.k8s.io/docs/)
- [PromQL cheat sheet](https://promlabs.com/promql-cheat-sheet/)
