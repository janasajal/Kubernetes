# ELK Stack on Kubernetes — Complete Hands-On Lab Guide

**Author:** Sajal Jana  
**Platform:** Minikube on AWS EC2 (Ubuntu 24.04, 2 vCPU, 8GB RAM)  
**Stack Version:** Elasticsearch 8.5.1 | Kibana 8.5.1 | Logstash 8.5.1 | Filebeat 8.5.1  

---

## Table of Contents

1. [What is the ELK Stack?](#1-what-is-the-elk-stack)
2. [How ELK Works](#2-how-elk-works)
3. [Architecture & Flow Diagram](#3-architecture--flow-diagram)
4. [Prerequisites](#4-prerequisites)
5. [Environment Setup](#5-environment-setup)
6. [Deploy Elasticsearch](#6-deploy-elasticsearch)
7. [Deploy Kibana](#7-deploy-kibana)
8. [Deploy Logstash](#8-deploy-logstash)
9. [Deploy Filebeat](#9-deploy-filebeat)
10. [Deploy Sample Application](#10-deploy-sample-application)
11. [Verify Log Flow](#11-verify-log-flow)
12. [Kibana — Explore Logs](#12-kibana--explore-logs)
13. [Production Patterns](#13-production-patterns)
14. [Troubleshooting Reference](#14-troubleshooting-reference)

---

## 1. What is the ELK Stack?

The **ELK Stack** is a collection of three open-source tools — **Elasticsearch**, **Logstash**, and **Kibana** — combined with **Filebeat** (a lightweight log shipper) to form a complete log management and observability platform used in production environments worldwide.

| Component | Role | Description |
|-----------|------|-------------|
| **Elasticsearch** | Storage & Search | Distributed search and analytics engine. Stores all log data as JSON documents and provides powerful full-text search and aggregations. |
| **Logstash** | Processing Pipeline | Ingests data from multiple sources, transforms it using filters (Grok, mutate, date), and ships it to Elasticsearch. |
| **Kibana** | Visualization | Web UI for exploring, searching, and visualizing data stored in Elasticsearch. Provides Discover, Dashboards, Alerts, and APM. |
| **Filebeat** | Log Shipper | Lightweight agent that runs on each Kubernetes node, tails container log files, enriches them with Kubernetes metadata, and forwards to Logstash. |

### Why ELK in Production?

- **Centralized logging** — All logs from hundreds of microservices in one place
- **Real-time search** — Find any log in milliseconds across terabytes of data
- **Alerting** — Notify teams when error rates spike or anomalies occur
- **Compliance** — Retain logs with automated lifecycle policies
- **Debugging** — Trace requests across distributed services

---

## 2. How ELK Works

### Data Flow

```
Application Pods
      │
      │  stdout/stderr logs written to
      ▼
/var/log/containers/*.log   (on each K8s node)
      │
      │  Filebeat tails these files
      ▼
   Filebeat (DaemonSet)
      │
      │  Adds Kubernetes metadata:
      │  namespace, pod name, container name
      │
      │  Ships via Beats protocol (port 5044)
      ▼
   Logstash (Deployment)
      │
      │  Grok filter: parses raw log strings
      │  into structured fields:
      │  log_level, service, request_id, user_id
      │
      │  Date filter: normalizes timestamps
      │  Mutate filter: adds/renames fields
      ▼
 Elasticsearch (StatefulSet)
      │
      │  Stores as JSON documents
      │  Index: k8s-logs-YYYY.MM.DD
      │  ILM: hot → warm → cold → delete
      ▼
    Kibana (Deployment)
      │
      │  Data View: k8s-logs-*
      │  Discover: search & filter logs
      │  Dashboard: visualize trends
      │  Alerts: notify on anomalies
      ▼
   Browser (port 5601)
```

### Key Concepts

**Index** — A collection of documents in Elasticsearch, similar to a database table. We use daily indices: `k8s-logs-2026.03.05`.

**Document** — A single log entry stored as JSON in Elasticsearch.

**Shard** — An index is split into shards for distributed storage and parallel search. We use 1 shard for a single-node lab.

**Mapping** — Defines field types in an index (keyword, text, date). Applied automatically via Index Templates.

**Pipeline** — A Logstash configuration defining input → filter → output processing steps.

**DaemonSet** — A Kubernetes resource that ensures one pod runs on every node. Perfect for Filebeat — every node must ship its logs.

---

## 3. Architecture & Flow Diagram

### Full Stack Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS EC2 Instance                          │
│                 Ubuntu 24.04 | 2 vCPU | 8GB                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Minikube (Kubernetes v1.35.1)            │   │
│  │                                                      │   │
│  │  ┌─────────────────┐   ┌──────────────────────────┐  │   │
│  │  │  sample-app ns  │   │        elk ns             │  │   │
│  │  │                 │   │                           │  │   │
│  │  │  log-generator  │   │  ┌─────────────────────┐  │  │   │
│  │  │  (Deployment)   │   │  │  Filebeat            │  │  │   │
│  │  │                 │   │  │  (DaemonSet)         │  │  │   │
│  │  │  Generates:     │   │  │  Reads: /var/log/    │  │  │   │
│  │  │  INFO/WARN/ERROR│──▶│  │  containers/*.log    │  │  │   │
│  │  │  logs with      │   │  │  Adds K8s metadata   │  │  │   │
│  │  │  reqId, userId  │   │  └──────────┬──────────┘  │  │   │
│  │  └─────────────────┘   │             │ port 5044   │  │   │
│  │                        │             ▼             │  │   │
│  │                        │  ┌─────────────────────┐  │  │   │
│  │                        │  │  Logstash            │  │  │   │
│  │                        │  │  (Deployment)        │  │  │   │
│  │                        │  │  Grok parsing        │  │  │   │
│  │                        │  │  Field extraction    │  │  │   │
│  │                        │  └──────────┬──────────┘  │  │   │
│  │                        │             │ port 9200   │  │   │
│  │                        │             ▼             │  │   │
│  │                        │  ┌─────────────────────┐  │  │   │
│  │                        │  │  Elasticsearch       │  │  │   │
│  │                        │  │  (StatefulSet)       │  │  │   │
│  │                        │  │  Index: k8s-logs-*   │  │  │   │
│  │                        │  │  ILM Policy          │  │  │   │
│  │                        │  │  Index Template      │  │  │   │
│  │                        │  └──────────┬──────────┘  │  │   │
│  │                        │             │             │  │   │
│  │                        │             ▼             │  │   │
│  │                        │  ┌─────────────────────┐  │  │   │
│  │                        │  │  Kibana              │  │  │   │
│  │                        │  │  (Deployment)        │  │  │   │
│  │                        │  │  Discover            │  │  │   │
│  │                        │  │  Dashboards          │  │  │   │
│  │                        │  │  Alerts              │  │  │   │
│  │                        │  └──────────┬──────────┘  │  │   │
│  │                        └─────────────┼─────────────┘  │   │
│  └──────────────────────────────────────┼─────────────────┘   │
│                                         │ port 5601           │
└─────────────────────────────────────────┼─────────────────────┘
                                          │
                                          ▼
                                    Your Browser
                              http://<EC2-PUBLIC-IP>:5601
```

### ILM (Index Lifecycle) Flow

```
Day 0                Day 1               Day 7              Day 30
  │                    │                   │                   │
  ▼                    ▼                   ▼                   ▼
┌─────┐            ┌──────┐           ┌──────┐           ┌────────┐
│ HOT │───────────▶│ WARM │──────────▶│ COLD │──────────▶│ DELETE │
│     │  rollover  │      │  7 days   │      │  30 days  │        │
│Write│  1d/10GB   │Read  │           │Rare  │           │ Auto   │
│Read │            │Only  │           │Access│           │Removed │
└─────┘            └──────┘           └──────┘           └────────┘
```

---

## 4. Prerequisites

### EC2 Instance Requirements

| Resource | Minimum | Used in Lab |
|----------|---------|-------------|
| vCPU | 2 | 2 |
| RAM | 8 GB | 8 GB |
| Disk | 20 GB | 20 GB |
| OS | Ubuntu 20.04+ | Ubuntu 24.04 |

### Required Tools

| Tool | Version Used | Purpose |
|------|-------------|---------|
| Docker | 29.2.1 | Minikube container runtime |
| Minikube | v1.38.1 | Local Kubernetes cluster |
| kubectl | v1.35.2 | Kubernetes CLI |
| Helm | v3.20.0 | Kubernetes package manager |

### AWS Security Group — Required Inbound Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH access |
| 5601 | TCP | Kibana UI |
| 9200 | TCP | Elasticsearch API (optional) |

---

## 5. Environment Setup

### Step 1 — Install Docker

Installs Docker CE (Community Edition) using the official Docker apt repository for Ubuntu.

```bash
sudo apt-get update && \
sudo apt-get install -y ca-certificates curl gnupg && \
sudo install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
sudo chmod a+r /etc/apt/keyrings/docker.gpg && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
sudo apt-get update && \
sudo apt-get install -y docker-ce docker-ce-cli containerd.io && \
sudo usermod -aG docker $USER && \
newgrp docker && \
docker --version
```

**Expected output:** `Docker version 29.x.x`

### Step 2 — Install Minikube

Downloads the Minikube binary for linux/amd64 and installs it to `/usr/local/bin`.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && \
sudo install minikube-linux-amd64 /usr/local/bin/minikube && \
rm minikube-linux-amd64 && \
minikube version
```

**Expected output:** `minikube version: v1.38.x`

### Step 3 — Install kubectl

Downloads the latest stable kubectl binary from the official Kubernetes release server.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
rm kubectl && \
kubectl version --client
```

**Expected output:** `Client Version: v1.35.x`

### Step 4 — Install Helm

Installs Helm 3 (Kubernetes package manager) using the official install script.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash && \
helm version --short
```

**Expected output:** `v3.20.x`

### Step 5 — Start Minikube

Starts a single-node Kubernetes cluster using Docker as the driver. Allocates 2 CPUs and 6GB RAM for ELK stack.

> **Note:** Kubernetes recommends disabling swap. We skip swap setup intentionally.

```bash
minikube start \
  --driver=docker \
  --cpus=2 \
  --memory=6144 \
  --disk-size=20g \
  --kubernetes-version=stable
```

**Key flags:**

| Flag | Value | Reason |
|------|-------|--------|
| `--driver=docker` | docker | Works on EC2 without KVM |
| `--cpus=2` | 2 | Matches EC2 vCPU count |
| `--memory=6144` | 6 GB | ELK needs minimum 4GB |
| `--disk-size=20g` | 20 GB | For images + data |

### Step 6 — Verify Cluster

```bash
kubectl get nodes && \
kubectl get pods -n kube-system
```

**Expected:** Node status `Ready`, all kube-system pods `Running`.

### Step 7 — Create ELK Namespace

Creates an isolated namespace for all ELK components — production best practice for resource isolation.

```bash
kubectl create namespace elk && \
kubectl get namespace elk
```

### Step 8 — Enable Metrics Server

Enables the Kubernetes metrics server to allow `kubectl top` commands for resource monitoring.

```bash
minikube addons enable metrics-server && \
sleep 60 && \
kubectl top nodes
```

### Step 9 — Permanent Port Forwards

Sets up persistent port-forwards using `nohup` so they survive terminal session changes.

```bash
nohup kubectl port-forward svc/elasticsearch -n elk 9200:9200 --address 0.0.0.0 > /tmp/es-pf.log 2>&1 &
nohup kubectl port-forward svc/kibana -n elk 5601:5601 --address 0.0.0.0 > /tmp/kibana-pf.log 2>&1 &
```

**Check logs anytime:**
```bash
tail -f /tmp/es-pf.log
tail -f /tmp/kibana-pf.log
```

---

## 6. Deploy Elasticsearch

Elasticsearch is the heart of the ELK stack — it stores, indexes, and searches all log data.

### Why Raw Manifests Instead of Helm?

Elastic's official Helm chart v8.x enforces TLS/security by default via auto-generated certificates injected as environment variables. These override any `extraEnvs` settings. Using raw Kubernetes manifests gives us full control — which is also more representative of real production environments where teams often manage their own manifests.

### Key Lessons Learned During Deployment

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `initial_master_nodes not allowed with single-node` | Helm chart injects this env var automatically | Remove `discovery.type: single-node` or use raw manifests |
| `CgroupInfo.getMountPoint() null` | Elasticsearch 7.x JVM doesn't support cgroup v2 (kernel 6.x) | Use ES 8.x + `-Des.cgroups.hierarchy.override=/` JVM flag |
| `plaintext http on https channel` | Helm chart sets `protocol: https` by default, auto-generates certs | Use raw manifests with explicit security disabled |

### Create Elasticsearch Manifest

```bash
cat > elasticsearch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elk
data:
  elasticsearch.yml: |
    cluster.name: elk-lab
    node.name: elasticsearch-0
    network.host: 0.0.0.0
    discovery.type: single-node
    xpack.security.enabled: false
    xpack.ml.enabled: false
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elk
spec:
  selector:
    app: elasticsearch
  ports:
    - name: http
      port: 9200
      targetPort: 9200
    - name: transport
      port: 9300
      targetPort: 9300
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elk
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
        - name: fix-permissions
          image: busybox
          command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:8.5.1
          ports:
            - containerPort: 9200
            - containerPort: 9300
          env:
            - name: ES_JAVA_OPTS
              value: "-Xms1g -Xmx1g -Des.cgroups.hierarchy.override=/"
            - name: discovery.type
              value: single-node
            - name: xpack.security.enabled
              value: "false"
            - name: xpack.ml.enabled
              value: "false"
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 2Gi
          volumeMounts:
            - name: config
              mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
              subPath: elasticsearch.yml
            - name: data
              mountPath: /usr/share/elasticsearch/data
          readinessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 10
      volumes:
        - name: config
          configMap:
            name: elasticsearch-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
EOF
```

### Deploy and Verify

```bash
# Deploy
kubectl apply -f elasticsearch.yaml

# Watch pod startup (takes ~60-90 seconds)
kubectl get pods -n elk -w
# Wait for: elasticsearch-0   1/1   Running

# Verify cluster health
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
```

**Expected health response:**
```json
{
  "cluster_name": "elk-lab",
  "status": "green",
  "number_of_nodes": 1,
  "active_shards_percent_as_number": 100.0
}
```

**Important env var explained:**

`-Des.cgroups.hierarchy.override=/` — Tells the Elasticsearch JVM to use the root cgroup, bypassing cgroup v2 detection issues on Linux kernel 6.x. Without this, Elasticsearch throws a `NullPointerException` on modern Ubuntu kernels.

---

## 7. Deploy Kibana

Kibana provides the web UI for exploring logs, creating dashboards, and setting up alerts.

### Create Kibana Manifest

```bash
cat > kibana.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    monitoring.ui.container.elasticsearch.enabled: true
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
spec:
  type: NodePort
  selector:
    app: kibana
  ports:
    - port: 5601
      targetPort: 5601
      nodePort: 30601
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:8.5.1
          ports:
            - containerPort: 5601
          env:
            - name: ELASTICSEARCH_HOSTS
              value: "http://elasticsearch:9200"
            - name: SERVER_HOST
              value: "0.0.0.0"
            - name: XPACK_SECURITY_ENABLED
              value: "false"
            - name: XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY
              value: "a7a6311933d3503b89bc2dbc36572c33a6c10925682e591bffcab6911c06786d"
          resources:
            requests:
              cpu: 300m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          volumeMounts:
            - name: config
              mountPath: /usr/share/kibana/config/kibana.yml
              subPath: kibana.yml
          readinessProbe:
            httpGet:
              path: /api/status
              port: 5601
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 10
      volumes:
        - name: config
          configMap:
            name: kibana-config
EOF
```

### Deploy and Access

```bash
# Deploy
kubectl apply -f kibana.yaml

# Watch pod startup (takes ~60-90 seconds)
kubectl get pods -n elk -w
# Wait for: kibana-xxx   1/1   Running

# Start port-forward (if not already running)
nohup kubectl port-forward svc/kibana -n elk 5601:5601 --address 0.0.0.0 > /tmp/kibana-pf.log 2>&1 &
```

**Access Kibana:** Open `http://<EC2-PUBLIC-IP>:5601` in your browser.

---

## 8. Deploy Logstash

Logstash is the processing pipeline — it receives logs from Filebeat, applies Grok patterns to extract structured fields, and ships them to Elasticsearch.

### Logstash Pipeline Explained

```
input {
  beats { port => 5044 }     # Receives from Filebeat
}

filter {
  grok {                     # Parses raw log string into fields
    match => { "message" => "%{TIMESTAMP_ISO8601:log_timestamp} \[%{LOGLEVEL:log_level}\] ..." }
  }
  date {                     # Sets @timestamp from parsed timestamp
    match => ["log_timestamp", "ISO8601"]
  }
  mutate {                   # Adds/renames/removes fields
    add_field => { "k8s_namespace" => "%{[kubernetes][namespace]}" }
  }
}

output {
  elasticsearch {            # Ships to Elasticsearch
    hosts => ["http://elasticsearch:9200"]
    index => "k8s-logs-%{+YYYY.MM.dd}"
  }
}
```

### Create Logstash Manifest

```bash
cat > logstash.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.enabled: false
  pipelines.yml: |
    - pipeline.id: main
      path.config: "/usr/share/logstash/pipeline/logstash.conf"
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }
    filter {
      if [kubernetes] {
        mutate {
          add_field => {
            "k8s_namespace" => "%{[kubernetes][namespace]}"
            "k8s_pod"       => "%{[kubernetes][pod][name]}"
            "k8s_container" => "%{[kubernetes][container][name]}"
          }
        }
      }
      if [k8s_namespace] == "sample-app" {
        grok {
          match => {
            "message" => "%{TIMESTAMP_ISO8601:log_timestamp} \[%{LOGLEVEL:log_level}\] %{WORD:service}-%{WORD:sub_service} - %{DATA:log_message} - reqId=%{DATA:request_id} userId=%{GREEDYDATA:user_id}"
          }
          tag_on_failure => ["_grokparsefailure"]
        }
        if "_grokparsefailure" not in [tags] {
          mutate {
            add_field => { "parsed" => "true" }
          }
        }
      }
      date {
        match => ["log_timestamp", "ISO8601"]
        target => "@timestamp"
        tag_on_failure => ["_dateparsefailure"]
      }
    }
    output {
      elasticsearch {
        hosts => ["http://elasticsearch:9200"]
        index => "k8s-logs-%{+YYYY.MM.dd}"
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elk
spec:
  selector:
    app: logstash
  ports:
    - name: beats
      port: 5044
      targetPort: 5044
    - name: http
      port: 9600
      targetPort: 9600
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
        - name: logstash
          image: docker.elastic.co/logstash/logstash:8.5.1
          ports:
            - containerPort: 5044
            - containerPort: 9600
          env:
            - name: LS_JAVA_OPTS
              value: "-Xms512m -Xmx512m -Des.cgroups.hierarchy.override=/"
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          volumeMounts:
            - name: config
              mountPath: /usr/share/logstash/config/logstash.yml
              subPath: logstash.yml
            - name: config
              mountPath: /usr/share/logstash/config/pipelines.yml
              subPath: pipelines.yml
            - name: pipeline
              mountPath: /usr/share/logstash/pipeline/logstash.conf
              subPath: logstash.conf
          readinessProbe:
            httpGet:
              path: /
              port: 9600
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 10
      volumes:
        - name: config
          configMap:
            name: logstash-config
        - name: pipeline
          configMap:
            name: logstash-config
EOF
```

### Deploy

```bash
kubectl apply -f logstash.yaml
kubectl get pods -n elk -w
# Wait for: logstash-xxx   1/1   Running
```

---

## 9. Deploy Filebeat

Filebeat runs as a **DaemonSet** — one pod per Kubernetes node — collecting logs from all containers and forwarding them to Logstash with Kubernetes metadata enrichment.

### Why DaemonSet?

In Kubernetes, container logs are written to `/var/log/containers/` on each node. Since Filebeat needs to read these files, it must run on every node. A DaemonSet guarantees this automatically, even as nodes are added or removed.

### RBAC Explained

Filebeat needs permission to query the Kubernetes API to enrich logs with pod metadata (namespace, labels, annotations). We create:

- **ServiceAccount** — Identity for the Filebeat pods
- **ClusterRole** — Permissions to `get/list/watch` pods, nodes, namespaces
- **ClusterRoleBinding** — Binds the role to the service account

### Create Filebeat Manifest

```bash
cat > filebeat.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: elk
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    output.logstash:
      hosts: ["logstash:5044"]

    logging.level: info
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
  - apiGroups: [""]
    resources: ["namespaces","pods","nodes"]
    verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: elk
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elk
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elk
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: filebeat
          image: docker.elastic.co/beats/filebeat:8.5.1
          args: ["-c", "/etc/filebeat.yml", "-e"]
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          volumeMounts:
            - name: config
              mountPath: /etc/filebeat.yml
              subPath: filebeat.yml
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: varlogcontainers
              mountPath: /var/log/containers
              readOnly: true
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: filebeat-config
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
EOF
```

### Deploy

```bash
kubectl apply -f filebeat.yaml
kubectl get pods -n elk -w
# Wait for: filebeat-xxx   1/1   Running
```

---

## 10. Deploy Sample Application

A log-generating application that simulates a real production microservice, emitting structured logs with INFO, WARN, and ERROR levels.

### Create Sample App Manifest

```bash
cat > sample-app.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: sample-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
        - name: log-generator
          image: busybox
          command:
            - /bin/sh
            - -c
            - |
              LEVELS="INFO INFO INFO WARN ERROR"
              SERVICES="auth-service payment-service user-service order-service inventory-service"
              MESSAGES_INFO="User login successful Payment processed successfully Order created User profile updated Inventory checked"
              MESSAGES_WARN="High memory usage detected Slow query detected Retry attempt Connection pool running low Cache miss rate high"
              MESSAGES_ERROR="Database connection failed Payment gateway timeout Authentication failed Order processing error Inventory sync failed"
              while true; do
                LEVEL=$(echo $LEVELS | tr ' ' '\n' | sed -n "$((RANDOM % 5 + 1))p")
                SERVICE=$(echo $SERVICES | tr ' ' '\n' | sed -n "$((RANDOM % 5 + 1))p")
                if [ "$LEVEL" = "INFO" ]; then
                  MSG=$(echo $MESSAGES_INFO | tr ' ' '\n' | sed -n "$((RANDOM % 5 + 1))p")
                elif [ "$LEVEL" = "WARN" ]; then
                  MSG=$(echo $MESSAGES_WARN | tr ' ' '\n' | sed -n "$((RANDOM % 5 + 1))p")
                else
                  MSG=$(echo $MESSAGES_ERROR | tr ' ' '\n' | sed -n "$((RANDOM % 5 + 1))p")
                fi
                echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) [$LEVEL] $SERVICE - $MSG - reqId=req-$RANDOM userId=user-$((RANDOM % 100))"
                sleep $((RANDOM % 3 + 1))
              done
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
EOF
```

### Deploy

```bash
kubectl apply -f sample-app.yaml
kubectl get pods -n sample-app -w
# Wait for: log-generator-xxx   1/1   Running

# Verify logs are being generated
kubectl logs -n sample-app -l app=log-generator --tail=10
```

**Sample log output:**
```
2026-03-05T11:46:38Z [WARN] order-service - Slow query detected - reqId=req-3476 userId=user-9
2026-03-05T11:46:42Z [ERROR] auth-service - Authentication failed - reqId=req-28092 userId=user-88
2026-03-05T11:46:46Z [INFO] payment-service - Payment processed successfully - reqId=req-4852 userId=user-56
```

---

## 11. Verify Log Flow

### Check All Pods Running

```bash
kubectl get pods -n elk
kubectl get pods -n sample-app
```

**Expected output:**
```
NAME                        READY   STATUS    RESTARTS   AGE
elasticsearch-0             1/1     Running   0          30m
filebeat-xxxxx              1/1     Running   0          20m
kibana-xxxxx                1/1     Running   0          25m
logstash-xxxxx              1/1     Running   0          15m
```

### Check Elasticsearch Indices

```bash
curl -s http://localhost:9200/_cat/indices?v
```

**Expected:** Index `k8s-logs-YYYY.MM.DD` with `docs.count` > 0 and growing.

### Check Cluster Health

```bash
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
```

### Check Error Log Count

```bash
curl -s "http://localhost:9200/k8s-logs-*/_count" \
  -H "Content-Type: application/json" \
  -d '{"query": {"match": {"message": "ERROR"}}}' | python3 -m json.tool
```

### Check Parsed Log Fields

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 3,
    "query": {"term": {"parsed": "true"}},
    "_source": ["log_level","service","log_message","request_id","user_id","k8s_pod"]
  }' | python3 -m json.tool
```

### Node Resource Usage

```bash
kubectl top nodes
kubectl top pods -n elk
kubectl top pods -n sample-app
```

---

## 12. Kibana — Explore Logs

### Create Data View

1. Open `http://<EC2-PUBLIC-IP>:5601`
2. Go to: `☰ Menu → Stack Management → Data Views`
3. Click **Create data view**
4. Fill in:
   - **Name:** `k8s-logs`
   - **Index pattern:** `k8s-logs-*`
   - **Timestamp field:** `@timestamp`
5. Click **Save data view to Kibana**

Or use the direct URL: `http://<EC2-PUBLIC-IP>:5601/app/management/kibana/dataViews`

### Explore in Discover

1. Go to: `☰ Menu → Discover`
2. Select **k8s-logs** data view (top-left dropdown)
3. Set time range to **Last 1 hour** (top right)

### Add Useful Column Fields

In the left sidebar, click `+` next to each field to add as a column:
- `k8s_namespace`
- `k8s_pod`
- `k8s_container`
- `log_level`
- `message`

### Useful KQL Search Queries

```
# All logs from sample-app namespace
k8s_namespace : "sample-app"

# Only ERROR logs
log_level : "ERROR"

# Logs from a specific pod
k8s_pod : "elasticsearch-0"

# Errors in elk namespace
k8s_namespace : "elk" and log_level : "ERROR"

# Logs for a specific user
user_id : "user-88"

# Logs for a specific request
request_id : "req-28092"

# Wildcard pod search
k8s_pod : *kibana*
```

---

## 13. Production Patterns

### ILM — Index Lifecycle Management

Automatically manages index lifecycle to control storage costs. Indices move through phases as they age.

```bash
curl -s -X PUT "http://localhost:9200/_ilm/policy/k8s-logs-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {"max_age": "1d", "max_size": "10gb"},
            "set_priority": {"priority": 100}
          }
        },
        "warm": {
          "min_age": "1d",
          "actions": {
            "shrink": {"number_of_shards": 1},
            "forcemerge": {"max_num_segments": 1},
            "set_priority": {"priority": 50}
          }
        },
        "cold": {
          "min_age": "7d",
          "actions": {"set_priority": {"priority": 0}}
        },
        "delete": {
          "min_age": "30d",
          "actions": {"delete": {}}
        }
      }
    }
  }' | python3 -m json.tool
```

**Verify policy:**
```bash
curl -s "http://localhost:9200/_ilm/policy/k8s-logs-policy" | python3 -m json.tool
```

### Index Template

Automatically applies settings, mappings, and ILM policy to every new `k8s-logs-*` index.

```bash
curl -s -X PUT "http://localhost:9200/_index_template/k8s-logs-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["k8s-logs-*"],
    "template": {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0,
        "index.lifecycle.name": "k8s-logs-policy",
        "index.lifecycle.rollover_alias": "k8s-logs"
      },
      "mappings": {
        "properties": {
          "@timestamp":    {"type": "date"},
          "message":       {"type": "text"},
          "k8s_namespace": {"type": "keyword"},
          "k8s_pod":       {"type": "keyword"},
          "k8s_container": {"type": "keyword"},
          "log_level":     {"type": "keyword"},
          "service":       {"type": "keyword"},
          "request_id":    {"type": "keyword"},
          "user_id":       {"type": "keyword"}
        }
      }
    },
    "priority": 100
  }' | python3 -m json.tool
```

**Verify template:**
```bash
curl -s "http://localhost:9200/_index_template/k8s-logs-template" | python3 -m json.tool | head -30
```

### Alerting in Kibana

**Navigate to:** `☰ → Stack Management → Rules → Create rule`

| Field | Value |
|-------|-------|
| Name | `High Error Rate Alert` |
| Rule type | Elasticsearch query |
| Index | `k8s-logs-*` |
| Time field | `@timestamp` |
| Query | `{"match": {"message": "ERROR"}}` |
| Threshold | IS ABOVE 10 |
| Time window | Last 5 minutes |
| Check every | 1 minute |

In production, connect actions to **Slack**, **PagerDuty**, or **Email** for real-time incident notification.

### Production Monitoring Commands

```bash
# Index stats
curl -s "http://localhost:9200/_cat/indices?v&h=index,health,docs.count,store.size"

# Shard allocation
curl -s "http://localhost:9200/_cat/shards?v"

# Node resource usage
curl -s "http://localhost:9200/_cat/nodes?v&h=name,cpu,ram.percent,heap.percent,load_1m"

# Cluster stats
curl -s "http://localhost:9200/_cluster/stats" | python3 -m json.tool

# ILM status of indices
curl -s "http://localhost:9200/_cat/indices?v&h=index,ilm.phase,ilm.age"
```

---

## 14. Troubleshooting Reference

### Common Issues & Fixes

| Symptom | Command to Diagnose | Fix |
|---------|---------------------|-----|
| Pod stuck in `Pending` | `kubectl describe pod <name> -n elk` | Check Events for CPU/memory/PVC issues |
| Pod in `CrashLoopBackOff` | `kubectl logs <pod> -n elk -c <container>` | Check logs for startup errors |
| Port-forward disconnected | `pgrep -a kubectl` | Restart with `nohup` |
| No logs in Elasticsearch | `kubectl logs -n elk -l app=filebeat` | Check Filebeat connectivity to Logstash |
| Kibana can't connect to ES | `kubectl logs -n elk -l app=kibana` | Verify elasticsearch service name |
| Insufficient CPU scheduling | `kubectl top nodes` | Reduce resource requests or scale down replicas |

### Useful Diagnostic Commands

```bash
# Get all resources in elk namespace
kubectl get all -n elk

# Describe a pod (shows events, resource limits, volumes)
kubectl describe pod <pod-name> -n elk

# Follow live logs
kubectl logs -f <pod-name> -n elk

# Get previous container logs (after crash)
kubectl logs <pod-name> -n elk --previous

# Check actual resource usage
kubectl top pods -n elk

# Check PersistentVolumeClaims
kubectl get pvc -n elk

# Check ConfigMaps
kubectl get configmap -n elk

# Exec into a pod for debugging
kubectl exec -it elasticsearch-0 -n elk -- bash

# Check env vars inside pod
kubectl exec -n elk elasticsearch-0 -- env | grep -i xpack

# Force restart a deployment
kubectl rollout restart deployment/<name> -n elk

# Check rollout status
kubectl rollout status deployment/<name> -n elk
```

### Port-Forward Management

```bash
# Start permanent port-forwards
nohup kubectl port-forward svc/elasticsearch -n elk 9200:9200 --address 0.0.0.0 > /tmp/es-pf.log 2>&1 &
nohup kubectl port-forward svc/kibana -n elk 5601:5601 --address 0.0.0.0 > /tmp/kibana-pf.log 2>&1 &

# Check running port-forwards
pgrep -a kubectl

# Kill all port-forwards
pkill -f "kubectl port-forward"

# View port-forward logs
tail -f /tmp/es-pf.log
tail -f /tmp/kibana-pf.log
```

---

## Summary

You have successfully built a complete ELK Stack on Kubernetes that:

- **Collects** logs from all Kubernetes pods automatically via Filebeat DaemonSet
- **Processes** and parses raw logs into structured fields via Logstash Grok patterns
- **Stores** logs in Elasticsearch with persistent volumes and daily indices
- **Manages** log lifecycle automatically via ILM policies (hot→warm→cold→delete)
- **Visualizes** logs via Kibana Discover and Dashboards
- **Alerts** on error rate thresholds via Kibana Rules

This mirrors a real production ELK deployment used by teams managing hundreds of microservices and terabytes of log data daily.

---

*Author: Sajal Jana | ELK Stack 8.5.1 | Kubernetes v1.35.1 | Minikube v1.38.1*
