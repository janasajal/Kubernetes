# Kubernetes Workloads & Scheduling

**Author:** Sajal Jana

A production-focused guide for Kubernetes workload management and scheduling strategies.

---

## Table of Contents

- [1. Workload Resources](#1-workload-resources)
  - [1.1 Deployments](#11-deployments)
    - [When to Use Deployments](#when-to-use-deployments)
    - [Creating a Deployment](#creating-a-deployment)
    - [Deployment Strategies](#deployment-strategies)
    - [Rolling Updates and Rollbacks](#rolling-updates-and-rollbacks)
    - [Scaling Deployments](#scaling-deployments)
  - [1.2 ReplicaSets](#12-replicasets)
    - [Understanding ReplicaSets](#understanding-replicasets)
    - [When to Use ReplicaSets](#when-to-use-replicasets)
    - [Creating a ReplicaSet](#creating-a-replicaset)
  - [1.3 StatefulSets](#13-statefulsets)
    - [When to Use StatefulSets](#when-to-use-statefulsets)
    - [StatefulSet Features](#statefulset-features)
    - [Creating a StatefulSet](#creating-a-statefulset)
    - [Headless Services](#headless-services)
    - [Update Strategies](#update-strategies)
  - [1.4 DaemonSets](#14-daemonsets)
    - [When to Use DaemonSets](#when-to-use-daemonsets)
    - [Creating a DaemonSet](#creating-a-daemonset)
    - [Updating DaemonSets](#updating-daemonsets)
  - [1.5 Jobs and CronJobs](#15-jobs-and-cronjobs)
    - [Jobs](#jobs)
    - [CronJobs](#cronjobs)
  - [1.6 Workload Comparison](#16-workload-comparison)
- [2. Scheduling](#2-scheduling)
  - [2.1 Node Selectors](#21-node-selectors)
  - [2.2 Node Affinity](#22-node-affinity)
    - [Required vs Preferred](#required-vs-preferred)
    - [Node Affinity Examples](#node-affinity-examples)
  - [2.3 Taints and Tolerations](#23-taints-and-tolerations)
    - [Understanding Taints](#understanding-taints)
    - [Taint Effects](#taint-effects)
    - [Adding Taints](#adding-taints)
    - [Pod Tolerations](#pod-tolerations)
    - [Common Use Cases](#common-use-cases)
  - [2.4 Pod Affinity and Anti-Affinity](#24-pod-affinity-and-anti-affinity)
    - [Pod Affinity](#pod-affinity)
    - [Pod Anti-Affinity](#pod-anti-affinity)
  - [2.5 Topology Spread Constraints](#25-topology-spread-constraints)
  - [2.6 Manual Scheduling](#26-manual-scheduling)
- [3. Resource Management](#3-resource-management)
  - [3.1 Resource Requests and Limits](#31-resource-requests-and-limits)
    - [Understanding Requests](#understanding-requests)
    - [Understanding Limits](#understanding-limits)
    - [Resource Units](#resource-units)
    - [Setting Requests and Limits](#setting-requests-and-limits)
  - [3.2 Quality of Service (QoS) Classes](#32-quality-of-service-qos-classes)
  - [3.3 LimitRanges](#33-limitranges)
  - [3.4 ResourceQuotas](#34-resourcequotas)
  - [3.5 Resource Management Best Practices](#35-resource-management-best-practices)
- [4. Autoscaling](#4-autoscaling)
  - [4.1 Horizontal Pod Autoscaler (HPA)](#41-horizontal-pod-autoscaler-hpa)
    - [Prerequisites](#prerequisites)
    - [Creating an HPA](#creating-an-hpa)
    - [HPA Based on Custom Metrics](#hpa-based-on-custom-metrics)
    - [HPA Behavior Configuration](#hpa-behavior-configuration)
  - [4.2 Vertical Pod Autoscaler (VPA)](#42-vertical-pod-autoscaler-vpa)
  - [4.3 Cluster Autoscaler](#43-cluster-autoscaler)
  - [4.4 Autoscaling Best Practices](#44-autoscaling-best-practices)
- [5. Configuration Management](#5-configuration-management)
  - [5.1 ConfigMaps](#51-configmaps)
    - [Creating ConfigMaps](#creating-configmaps)
    - [Using ConfigMaps in Pods](#using-configmaps-in-pods)
    - [ConfigMap Best Practices](#configmap-best-practices)
  - [5.2 Secrets](#52-secrets)
    - [Creating Secrets](#creating-secrets)
    - [Using Secrets in Pods](#using-secrets-in-pods)
    - [Secret Best Practices](#secret-best-practices)
  - [5.3 Immutable ConfigMaps and Secrets](#53-immutable-configmaps-and-secrets)
  - [5.4 Configuration Management Patterns](#54-configuration-management-patterns)
- [6. Quick Reference](#6-quick-reference)
  - [6.1 Essential Commands](#61-essential-commands)
  - [6.2 Common Patterns](#62-common-patterns)
- [7. Production Checklist](#7-production-checklist)
- [8. Further Reading](#8-further-reading)

---

## 1. Workload Resources

Kubernetes provides several workload resources to manage application lifecycle. Each serves a specific purpose.

### 1.1 Deployments

The most common workload resource for stateless applications.

#### When to Use Deployments

- **Stateless applications** (web servers, APIs, microservices)
- Applications requiring **rolling updates** and **rollbacks**
- Applications that need **easy scaling**
- Most production workloads

**Key Features:**
- Declarative updates
- Rolling updates and rollbacks
- Revision history
- Automatic ReplicaSet management
- Pause and resume capabilities

#### Creating a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

```bash
# Create deployment
kubectl apply -f nginx-deployment.yaml

# Verify deployment
kubectl get deployments
kubectl get pods
kubectl describe deployment nginx-deployment
```

#### Deployment Strategies

**1. RollingUpdate (Default)**

Updates pods gradually to ensure zero downtime.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max new pods above desired count
      maxUnavailable: 1  # Max pods unavailable during update
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**How it works:**
1. Creates new pods with updated spec
2. Waits for new pods to be ready
3. Terminates old pods
4. Repeats until all pods updated

**2. Recreate**

Terminates all existing pods before creating new ones (causes downtime).

```yaml
spec:
  strategy:
    type: Recreate
```

**When to use:**
- Development environments
- Applications that can't run multiple versions simultaneously
- Database schema migrations

#### Rolling Updates and Rollbacks

```bash
# Update deployment image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment

# View specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=3

# Pause rollout (useful during troubleshooting)
kubectl rollout pause deployment/nginx-deployment

# Resume rollout
kubectl rollout resume deployment/nginx-deployment
```

#### Scaling Deployments

```bash
# Scale manually
kubectl scale deployment/nginx-deployment --replicas=5

# Autoscale (covered in section 4)
kubectl autoscale deployment/nginx-deployment --min=3 --max=10 --cpu-percent=80
```

**Declarative Scaling:**

```yaml
spec:
  replicas: 5  # Update this value
```

```bash
kubectl apply -f nginx-deployment.yaml
```

### 1.2 ReplicaSets

Ensures a specified number of pod replicas are running at all times.

#### Understanding ReplicaSets

- **Low-level** primitive that Deployments use
- Manages pod replication
- Self-healing (restarts failed pods)

```
Deployment → ReplicaSet → Pods
```

#### When to Use ReplicaSets

**Directly:** Rarely recommended
- Only when you need custom update orchestration
- Advanced scenarios not covered by Deployments

**Indirectly:** Always (through Deployments)
- Deployments automatically create and manage ReplicaSets

#### Creating a ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# View ReplicaSets
kubectl get rs

# Describe ReplicaSet
kubectl describe rs nginx-replicaset

# Scale ReplicaSet
kubectl scale rs/nginx-replicaset --replicas=5
```

**Important:** ReplicaSets don't support rolling updates. Changing the template doesn't update existing pods.

### 1.3 StatefulSets

Manages stateful applications with stable identities and persistent storage.

#### When to Use StatefulSets

- **Databases** (MySQL, PostgreSQL, MongoDB)
- **Distributed systems** (Kafka, ZooKeeper, etcd)
- **Applications requiring:**
  - Stable network identities
  - Stable persistent storage
  - Ordered deployment and scaling
  - Ordered rolling updates

#### StatefulSet Features

**Stable Network Identity:**
- Predictable pod names: `<statefulset-name>-<ordinal>`
- Example: `mysql-0`, `mysql-1`, `mysql-2`
- DNS names: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

**Ordered Operations:**
- Pods created sequentially (0, 1, 2...)
- Pods deleted in reverse order (2, 1, 0...)
- Updates applied in order

**Persistent Storage:**
- Each pod gets dedicated PersistentVolume
- Storage persists even if pod is deleted

#### Creating a StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # Headless service
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

```bash
# Create StatefulSet
kubectl apply -f mysql-statefulset.yaml

# View StatefulSets
kubectl get statefulsets
kubectl get sts  # Short form

# View pods (note ordered names)
kubectl get pods

# View PVCs (one per pod)
kubectl get pvc

# Scale StatefulSet
kubectl scale statefulset/mysql --replicas=5

# Delete StatefulSet (keeps PVCs)
kubectl delete statefulset mysql

# Delete StatefulSet and PVCs
kubectl delete statefulset mysql
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

#### Headless Services

Required for StatefulSets to provide stable network identities.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # Makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

**DNS Resolution:**
```bash
# Individual pod DNS
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-0.mysql-headless
```

#### Update Strategies

**RollingUpdate (Default):**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods
```

**OnDelete:**
```yaml
spec:
  updateStrategy:
    type: OnDelete  # Manual update by deleting pods
```

**Partitioned Rolling Update:**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # Only update pods >= ordinal 2
```

### 1.4 DaemonSets

Ensures a copy of a pod runs on all (or selected) nodes.

#### When to Use DaemonSets

- **Monitoring agents** (Prometheus node-exporter, Datadog agent)
- **Log collectors** (Fluentd, Filebeat)
- **Node-level storage** (Ceph, GlusterFS)
- **Network plugins** (Calico, Flannel)
- **Security agents** (Falco)

**Key Characteristics:**
- One pod per node (automatically)
- Pods scheduled on new nodes automatically
- Pods removed when nodes are deleted

#### Creating a DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
      # Run on control plane nodes too
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.14
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```bash
# Create DaemonSet
kubectl apply -f fluentd-daemonset.yaml

# View DaemonSets
kubectl get daemonsets -n kube-system
kubectl get ds -n kube-system  # Short form

# Describe DaemonSet
kubectl describe ds/fluentd -n kube-system

# Check DaemonSet pods
kubectl get pods -n kube-system -l app=fluentd -o wide
```

#### Updating DaemonSets

**RollingUpdate (Default):**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Max nodes without the daemon pod
```

**OnDelete:**
```yaml
spec:
  updateStrategy:
    type: OnDelete  # Manual update by deleting pods
```

**Update Commands:**
```bash
# Update DaemonSet image
kubectl set image ds/fluentd fluentd=fluentd:v1.15 -n kube-system

# Check rollout status
kubectl rollout status ds/fluentd -n kube-system

# View rollout history
kubectl rollout history ds/fluentd -n kube-system

# Rollback
kubectl rollout undo ds/fluentd -n kube-system
```

**Selective Deployment:**
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd  # Only on nodes with this label
```

### 1.5 Jobs and CronJobs

#### Jobs

Runs pods to completion (for batch processing, one-off tasks).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  completions: 5      # Total successful completions needed
  parallelism: 2      # Max pods running in parallel
  backoffLimit: 3     # Max retries before marking failed
  template:
    spec:
      restartPolicy: Never  # or OnFailure
      containers:
      - name: importer
        image: data-importer:v1
        command: ["python", "import.py"]
```

```bash
# Create Job
kubectl apply -f data-import-job.yaml

# View Jobs
kubectl get jobs

# View Job pods
kubectl get pods --selector=job-name=data-import

# View Job logs
kubectl logs job/data-import

# Delete Job (and pods)
kubectl delete job data-import

# Delete Job but keep pods
kubectl delete job data-import --cascade=orphan
```

#### CronJobs

Runs Jobs on a schedule (like cron).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM (UTC)
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:v1
            command: ["/bin/sh"]
            args: ["-c", "backup-script.sh"]
  successfulJobsHistoryLimit: 3  # Keep last 3 successful
  failedJobsHistoryLimit: 1      # Keep last 1 failed
```

```bash
# Create CronJob
kubectl apply -f backup-cronjob.yaml

# View CronJobs
kubectl get cronjobs

# View Jobs created by CronJob
kubectl get jobs

# Manually trigger CronJob
kubectl create job backup-manual --from=cronjob/backup-job

# Suspend CronJob
kubectl patch cronjob backup-job -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob backup-job -p '{"spec":{"suspend":false}}'
```

**Cron Schedule Format:**
```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
# │ │ │ │ │
# * * * * *

Examples:
"0 * * * *"      # Every hour
"*/15 * * * *"   # Every 15 minutes
"0 2 * * *"      # Daily at 2 AM
"0 2 * * 0"      # Weekly on Sunday at 2 AM
"0 2 1 * *"      # Monthly on 1st at 2 AM
```

### 1.6 Workload Comparison

| Workload | Use Case | Pod Names | Scaling | Storage | Updates |
|----------|----------|-----------|---------|---------|---------|
| **Deployment** | Stateless apps | Random | Easy | Ephemeral | Rolling, Recreate |
| **ReplicaSet** | Low-level control | Random | Manual | Ephemeral | None (manual) |
| **StatefulSet** | Stateful apps | Ordered | Ordered | Persistent | Rolling, OnDelete |
| **DaemonSet** | Node agents | Per-node | Automatic | Host or ephemeral | Rolling, OnDelete |
| **Job** | Batch tasks | Job-based | Fixed | Ephemeral | N/A |
| **CronJob** | Scheduled tasks | Job-based | Per schedule | Ephemeral | N/A |

**Decision Tree:**

```
Need to run on all nodes?
├─ Yes → DaemonSet
└─ No
   ├─ Need stable identity/storage?
   │  └─ Yes → StatefulSet
   └─ No
      ├─ Run to completion?
      │  ├─ Once → Job
      │  └─ Scheduled → CronJob
      └─ Long-running → Deployment
```

---

## 2. Scheduling

The Kubernetes scheduler assigns pods to nodes based on resource requirements, constraints, and policies.

### 2.1 Node Selectors

Simplest way to constrain pods to nodes with specific labels.

**Label a Node:**
```bash
# Add label to node
kubectl label nodes node1 disktype=ssd

# View node labels
kubectl get nodes --show-labels

# Remove label
kubectl label nodes node1 disktype-
```

**Use Node Selector:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd  # Only schedule on nodes with this label
  containers:
  - name: nginx
    image: nginx
```

**Common Use Cases:**
```yaml
# GPU nodes
nodeSelector:
  accelerator: nvidia-tesla-v100

# High-memory nodes
nodeSelector:
  node.kubernetes.io/instance-type: m5.8xlarge

# Availability zone
nodeSelector:
  topology.kubernetes.io/zone: us-east-1a
```

**Limitations:**
- Only exact matches (no OR/NOT logic)
- Cannot express preferences
- → Use Node Affinity for complex rules

### 2.2 Node Affinity

More expressive than nodeSelector, supports required and preferred rules.

#### Required vs Preferred

**Required:** Must match (hard constraint)
**Preferred:** Try to match (soft constraint)

#### Node Affinity Examples

**Required Affinity:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx
```

**Preferred Affinity:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100  # Higher weight = stronger preference (1-100)
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50
        preference:
          matchExpressions:
          - key: cpu-type
            operator: In
            values:
            - high-performance
  containers:
  - name: nginx
    image: nginx
```

**Combined Required and Preferred:**
```yaml
spec:
  affinity:
    nodeAffinity:
      # MUST be in us-east-1a or us-east-1b
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      # PREFER ssd over hdd
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

**Operators:**
- `In` - Label value in list
- `NotIn` - Label value not in list
- `Exists` - Label key exists (any value)
- `DoesNotExist` - Label key doesn't exist
- `Gt` - Label value greater than (integer)
- `Lt` - Label value less than (integer)

**Examples:**
```yaml
# Node has GPU
- key: accelerator
  operator: Exists

# Node is NOT spot instance
- key: node.kubernetes.io/instance-lifecycle
  operator: NotIn
  values:
  - spot

# Node has more than 8 CPUs
- key: node.kubernetes.io/cpu-count
  operator: Gt
  values:
  - "8"
```

### 2.3 Taints and Tolerations

Control which pods can be scheduled on which nodes (inverse of node affinity).

#### Understanding Taints

**Taint:** Applied to nodes to repel pods
**Toleration:** Applied to pods to tolerate (allow) taints

```
Node with taint → Repels all pods without toleration
Pod with toleration → Can be scheduled on tainted node
```

#### Taint Effects

1. **NoSchedule:** Pods without toleration won't be scheduled
2. **PreferNoSchedule:** Avoid scheduling if possible (soft)
3. **NoExecute:** Evict existing pods without toleration

#### Adding Taints

```bash
# Add taint to node
kubectl taint nodes node1 key=value:NoSchedule

# Examples:
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node1 environment=production:NoSchedule
kubectl taint nodes node1 maintenance=true:NoExecute

# View node taints
kubectl describe node node1 | grep Taints

# Remove taint (note the minus sign)
kubectl taint nodes node1 key=value:NoSchedule-
kubectl taint nodes node1 dedicated-  # Remove all taints with key
```

#### Pod Tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: cuda-app
    image: nvidia/cuda:11.0
```

**Toleration Operators:**

**Equal (default):**
```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

**Exists (wildcard):**
```yaml
# Tolerate any value for this key
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerate all taints (any key/value)
tolerations:
- operator: "Exists"
```

**Tolerate NoExecute with grace period:**
```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 3600  # Stay for 1 hour before eviction
```

#### Common Use Cases

**1. Dedicated Nodes (GPU, high-memory):**
```bash
# Taint GPU nodes
kubectl taint nodes gpu-node1 dedicated=gpu:NoSchedule

# Only GPU workloads tolerate this
tolerations:
- key: "dedicated"
  value: "gpu"
  effect: "NoSchedule"
```

**2. Production vs Non-Production:**
```bash
# Taint production nodes
kubectl taint nodes prod-node1 environment=production:NoSchedule

# Production pods tolerate
tolerations:
- key: "environment"
  value: "production"
  effect: "NoSchedule"
```

**3. Node Maintenance:**
```bash
# Evict all pods for maintenance
kubectl taint nodes node1 maintenance=true:NoExecute

# Critical pods tolerate
tolerations:
- key: "maintenance"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

**4. Control Plane Nodes:**
```yaml
# Default taint on control plane nodes
# Most pods cannot schedule there
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  effect: "NoSchedule"
```

### 2.4 Pod Affinity and Anti-Affinity

Control pod placement relative to other pods.

#### Pod Affinity

Schedule pods close to other pods (same node/zone/rack).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname  # Same node
  containers:
  - name: nginx
    image: nginx
```

**Use Cases:**
- Co-locate web servers with cache (reduce latency)
- Place related microservices together
- Group pods in same availability zone

#### Pod Anti-Affinity

Schedule pods away from other pods (different nodes/zones).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: kubernetes.io/hostname  # Different nodes
      containers:
      - name: nginx
        image: nginx
```

**Use Cases:**
- Spread replicas across nodes (high availability)
- Avoid resource contention
- Distribute load

**Topology Keys:**
```yaml
# Same/different node
topologyKey: kubernetes.io/hostname

# Same/different availability zone
topologyKey: topology.kubernetes.io/zone

# Same/different region
topologyKey: topology.kubernetes.io/region

# Custom topology
topologyKey: rack
```

**Preferred Anti-Affinity (soft):**
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
```

### 2.5 Topology Spread Constraints

Evenly distribute pods across topology domains (zones, nodes, racks).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1  # Max difference between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule  # or ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx
```

**How it works:**
- `maxSkew: 1` - Zones can differ by at most 1 pod
- With 9 replicas and 3 zones → 3 pods per zone

**Multiple Constraints:**
```yaml
topologySpreadConstraints:
# Spread across zones (strict)
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: web
# Spread across nodes (best effort)
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

### 2.6 Manual Scheduling

Bypass scheduler and assign pod to specific node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: node1  # Directly assign to node1
  containers:
  - name: nginx
    image: nginx
```

**Use Cases:**
- Testing
- Troubleshooting
- Special hardware requirements

**⚠️ Warning:** Bypasses resource checks, taints, and affinity rules.

---

## 3. Resource Management

Proper resource management ensures efficient cluster utilization and application performance.

### 3.1 Resource Requests and Limits

#### Understanding Requests

**Request:** Guaranteed resources for a container

**Scheduler uses requests to:**
- Find nodes with sufficient resources
- Decide pod placement

**Kubelet uses requests to:**
- Reserve resources on node
- Prevent resource overcommitment

#### Understanding Limits

**Limit:** Maximum resources a container can use

**What happens when limit is reached:**
- **CPU:** Throttling (container slowed down)
- **Memory:** OOMKilled (container terminated)

#### Resource Units

**CPU:**
- `1` or `1000m` = 1 CPU core
- `500m` = 0.5 CPU core (half a core)
- `100m` = 0.1 CPU core (10% of a core)

**Memory:**
- `128Mi` = 128 mebibytes (binary)
- `128M` = 128 megabytes (decimal)
- `1Gi` = 1 gibibyte
- `1G` = 1 gigabyte

**Common Units:**
```yaml
# CPU
cpu: "1"      # 1 core
cpu: "500m"   # 0.5 core
cpu: "100m"   # 0.1 core

# Memory
memory: "128Mi"   # 128 MiB
memory: "1Gi"     # 1 GiB
memory: "512Mi"   # 512 MiB
```

#### Setting Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Best Practices:**

```yaml
# Production application
resources:
  requests:
    memory: "256Mi"  # Minimum needed
    cpu: "200m"
  limits:
    memory: "512Mi"  # 2x requests (some headroom)
    cpu: "1000m"     # Allow bursting

# Background job
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"      # Limit bursting

# Memory-intensive app
resources:
  requests:
    memory: "2Gi"
    cpu: "500m"
  limits:
    memory: "4Gi"    # Prevent OOM
    cpu: "2000m"
```

**Container with no limits (not recommended):**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  # No limits - can use unlimited resources
```

**Multiple Containers:**
```yaml
spec:
  containers:
  - name: app
    image: app:v1
    resources:
      requests:
        memory: "256Mi"
        cpu: "200m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  - name: sidecar
    image: sidecar:v1
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
  # Total pod request: 320Mi memory, 250m CPU
  # Total pod limit: 640Mi memory, 600m CPU
```

### 3.2 Quality of Service (QoS) Classes

Kubernetes assigns QoS class to pods based on resources. Determines eviction priority.

**QoS Classes (highest to lowest priority):**

**1. Guaranteed (highest priority)**

Requirements:
- Every container has CPU and memory requests
- Requests equal limits

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"  # Equal to request
    cpu: "500m"      # Equal to request
```

**Characteristics:**
- Never evicted unless exceeding limits
- Best performance guarantees
- Use for critical workloads

**2. Burstable (medium priority)**

Requirements:
- At least one container has request or limit
- Requests not equal to limits

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "200m"
  limits:
    memory: "256Mi"  # Different from request
    cpu: "500m"      # Different from request
```

**Characteristics:**
- Evicted if node runs out of resources
- Can burst above requests
- Use for most workloads

**3. BestEffort (lowest priority)**

Requirements:
- No requests or limits set

```yaml
# No resources specified
containers:
- name: app
  image: nginx
```

**Characteristics:**
- First to be evicted
- No guarantees
- Use for non-critical, batch jobs

**Check Pod QoS:**
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
kubectl describe pod <pod-name> | grep QoS
```

**Eviction Order:**
1. BestEffort pods (lowest priority)
2. Burstable pods exceeding requests
3. Burstable pods within requests
4. Guaranteed pods (last resort)

### 3.3 LimitRanges

Set default and constrain resource values per namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limit-range
  namespace: dev
spec:
  limits:
  - max:  # Max per container
      memory: "1Gi"
      cpu: "1000m"
    min:  # Min per container
      memory: "64Mi"
      cpu: "100m"
    default:  # Default limits (if not specified)
      memory: "256Mi"
      cpu: "500m"
    defaultRequest:  # Default requests (if not specified)
      memory: "128Mi"
      cpu: "200m"
    type: Container
  - max:  # Max per pod
      memory: "2Gi"
      cpu: "2000m"
    type: Pod
```

```bash
# Create LimitRange
kubectl apply -f limitrange.yaml

# View LimitRanges
kubectl get limitranges -n dev
kubectl describe limitrange mem-cpu-limit-range -n dev
```

**Use Cases:**
- Prevent resource hogging
- Set sensible defaults
- Enforce resource policies

### 3.4 ResourceQuotas

Limit aggregate resource consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    # Compute resources
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    
    # Object counts
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"
    
    # Storage
    requests.storage: "100Gi"
```

```bash
# Create ResourceQuota
kubectl apply -f resourcequota.yaml

# View quotas
kubectl get resourcequota -n dev
kubectl describe resourcequota compute-quota -n dev

# Check usage
kubectl get resourcequota compute-quota -n dev -o yaml
```

**Example with Object Counts:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: dev
spec:
  hard:
    configmaps: "10"
    secrets: "10"
    services.loadbalancers: "2"
    services.nodeports: "5"
```

**Scope-based Quotas:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-quota
  namespace: dev
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high"]
```

### 3.5 Resource Management Best Practices

**1. Always Set Requests:**
```yaml
# ✅ Good
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"

# ❌ Bad
resources: {}
```

**2. Set Limits for Memory:**
```yaml
# ✅ Good - Prevent OOM killing other pods
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"

# ⚠️ Acceptable - But monitor closely
resources:
  requests:
    memory: "256Mi"
  # No limit
```

**3. CPU Limits (Careful):**
```yaml
# ✅ Good for predictable workloads
resources:
  requests:
    cpu: "200m"
  limits:
    cpu: "500m"

# ✅ Also good - Allow bursting
resources:
  requests:
    cpu: "200m"
  # No CPU limit - can burst to node capacity
```

**4. Start Conservative:**
```yaml
# Initial deployment
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"

# Monitor and adjust based on actual usage
```

**5. Use Monitoring:**
```bash
# Check actual usage
kubectl top pod <pod-name>
kubectl top node

# Check over time
# Use Prometheus, Grafana, or vendor tools
```

**6. Profile Applications:**
```yaml
# After profiling, right-size resources
resources:
  requests:
    memory: "384Mi"  # Based on P95 usage
    cpu: "300m"
  limits:
    memory: "512Mi"  # 1.3x request (headroom)
    cpu: "600m"
```

---

## 4. Autoscaling

Automatically adjust workload replicas or resources based on metrics.

### 4.1 Horizontal Pod Autoscaler (HPA)

Automatically scales pod replicas based on CPU, memory, or custom metrics.

#### Prerequisites

**1. Metrics Server (required):**
```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify metrics-server
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

**2. Resource Requests (required):**
```yaml
# Deployment must have resource requests
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "200m"  # Required for CPU-based HPA
```

#### Creating an HPA

**Using kubectl:**
```bash
# Autoscale deployment based on CPU
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# View HPA
kubectl get hpa

# Describe HPA
kubectl describe hpa nginx
```

**Using YAML (CPU):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80  # Target 80% CPU
```

**Memory-based HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75  # Target 75% memory
```

**Multiple Metrics (CPU and Memory):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # HPA scales if ANY metric exceeds target
```

#### HPA Based on Custom Metrics

**Example: Requests per Second**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # 1000 req/s per pod
```

**Requires:**
- Custom metrics API (Prometheus Adapter, Datadog, etc.)
- Application exposing metrics

#### HPA Behavior Configuration

Control scale-up and scale-down behavior.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50  # Max 50% of pods at a time
        periodSeconds: 60
      - type: Pods
        value: 2  # Max 2 pods at a time
        periodSeconds: 60
      selectPolicy: Min  # Use the most conservative policy
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Max 100% (double) at a time
        periodSeconds: 30
      - type: Pods
        value: 4  # Max 4 pods at a time
        periodSeconds: 30
      selectPolicy: Max  # Use the most aggressive policy
```

**Monitoring HPA:**
```bash
# Watch HPA status
kubectl get hpa -w

# View HPA events
kubectl describe hpa nginx-hpa

# Check current metrics
kubectl get hpa nginx-hpa -o yaml

# View pod count over time
kubectl get deployment nginx -w
```

**Testing HPA:**
```bash
# Generate load
kubectl run -it --rm load-generator --image=busybox -- /bin/sh
# Inside container:
while true; do wget -q -O- http://nginx-service; done

# Watch HPA scale up
kubectl get hpa nginx-hpa -w
```

### 4.2 Vertical Pod Autoscaler (VPA)

Automatically adjusts CPU and memory requests/limits.

**⚠️ Note:** VPA requires separate installation and can conflict with HPA on CPU/memory.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"  # or "Initial", "Recreate", "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "100m"
        memory: "50Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

### 4.3 Cluster Autoscaler

Automatically adds/removes nodes based on pod scheduling needs.

**Cloud-specific implementation:**
- AWS: Cluster Autoscaler for Auto Scaling Groups
- GCP: Cluster Autoscaler for Instance Groups
- Azure: Cluster Autoscaler for Scale Sets

**Behavior:**
- Scales up when pods fail to schedule (insufficient resources)
- Scales down when nodes are underutilized (<50% for 10 minutes)

### 4.4 Autoscaling Best Practices

**1. Start with HPA:**
```yaml
# Most applications benefit from horizontal scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  minReplicas: 3  # Always maintain minimum for HA
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**2. Set Conservative Minimums:**
```yaml
minReplicas: 3  # Not 1 - ensure high availability
```

**3. Set Realistic Maximums:**
```yaml
maxReplicas: 20  # Based on load tests and cost limits
```

**4. Configure Scale-Down Carefully:**
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Prevent flapping
    policies:
    - type: Pods
      value: 1
      periodSeconds: 60  # Slow, gradual scale-down
```

**5. Monitor and Tune:**
```bash
# Check HPA effectiveness
kubectl describe hpa app-hpa

# Look for:
# - Frequent scaling up/down (flapping)
# - Hitting max replicas (increase max)
# - Never scaling (adjust targets)
```

**6. Combine with PDB:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # Always keep 2 pods during disruptions
  selector:
    matchLabels:
      app: nginx
```

---

## 5. Configuration Management

Separate configuration from application code using ConfigMaps and Secrets.

### 5.1 ConfigMaps

Store non-sensitive configuration data in key-value pairs.

#### Creating ConfigMaps

**From Literal Values:**
```bash
kubectl create configmap app-config \
  --from-literal=database_url=postgres://localhost:5432/mydb \
  --from-literal=log_level=info \
  --from-literal=max_connections=100
```

**From File:**
```bash
# config.properties
database_url=postgres://localhost:5432/mydb
log_level=info
max_connections=100

kubectl create configmap app-config --from-file=config.properties
```

**From Directory:**
```bash
kubectl create configmap app-config --from-file=./config/
```

**From YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://localhost:5432/mydb"
  log_level: "info"
  max_connections: "100"
  # Multi-line configuration file
  app.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        proxy_pass http://backend;
      }
    }
```

```bash
kubectl apply -f configmap.yaml
```

#### Using ConfigMaps in Pods

**1. Environment Variables (Individual Keys):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
```

**2. Environment Variables (All Keys):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    envFrom:
    - configMapRef:
        name: app-config
    # All keys become environment variables
    # database_url → DATABASE_URL
    # log_level → LOG_LEVEL
```

**3. Volume Mount (Files):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
    # ConfigMap keys become files:
    # /etc/config/database_url
    # /etc/config/log_level
    # /etc/config/app.conf
  volumes:
  - name: config
    configMap:
      name: app-config
```

**4. Volume Mount (Specific Keys):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
      items:
      - key: app.conf
        path: nginx.conf  # Rename file
      # Only /etc/config/nginx.conf is created
```

**Managing ConfigMaps:**
```bash
# View ConfigMaps
kubectl get configmaps
kubectl get cm  # Short form

# Describe ConfigMap
kubectl describe configmap app-config

# View ConfigMap data
kubectl get configmap app-config -o yaml

# Edit ConfigMap
kubectl edit configmap app-config

# Delete ConfigMap
kubectl delete configmap app-config
```

#### ConfigMap Best Practices

**1. Use Descriptive Names:**
```yaml
# ✅ Good
name: nginx-config
name: database-credentials
name: app-settings-v2

# ❌ Bad
name: config
name: data
name: cm1
```

**2. Version ConfigMaps:**
```yaml
# Include version in name
name: app-config-v2

# Update deployment to use new version
env:
- name: CONFIG_VERSION
  value: "v2"
envFrom:
- configMapRef:
    name: app-config-v2
```

**3. Separate Concerns:**
```yaml
# ✅ Good - Separate ConfigMaps
kind: ConfigMap
metadata:
  name: app-settings
---
kind: ConfigMap
metadata:
  name: nginx-config
---
kind: ConfigMap
metadata:
  name: database-config

# ❌ Bad - One giant ConfigMap
kind: ConfigMap
metadata:
  name: all-config
```

**4. Use for Non-Sensitive Data Only:**
```yaml
# ✅ Good - Public configuration
data:
  api_endpoint: "https://api.example.com"
  timeout: "30"
  log_level: "info"

# ❌ Bad - Sensitive data (use Secrets instead)
data:
  database_password: "supersecret"
  api_key: "abc123"
```

### 5.2 Secrets

Store sensitive data (passwords, tokens, keys).

**⚠️ Important:** Secrets are base64-encoded, NOT encrypted by default. Use encryption at rest.

#### Creating Secrets

**From Literal Values:**
```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret123
```

**From Files:**
```bash
echo -n 'admin' > username.txt
echo -n 'supersecret123' > password.txt

kubectl create secret generic db-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt
```

**From YAML (base64 encoded):**
```bash
# Encode values
echo -n 'admin' | base64
# Output: YWRtaW4=

echo -n 'supersecret123' | base64
# Output: c3VwZXJzZWNyZXQxMjM=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=
  password: c3VwZXJzZWNyZXQxMjM=
```

**From YAML (plain text, auto-encoded):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:  # Plain text, auto-encoded
  username: admin
  password: supersecret123
```

**Secret Types:**
```bash
# Generic (Opaque)
kubectl create secret generic my-secret --from-literal=key=value

# Docker registry
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# TLS
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# SSH
kubectl create secret generic ssh-secret \
  --from-file=ssh-privatekey=~/.ssh/id_rsa \
  --type=kubernetes.io/ssh-auth

# Basic auth
kubectl create secret generic basic-auth \
  --from-literal=username=admin \
  --from-literal=password=secret \
  --type=kubernetes.io/basic-auth
```

#### Using Secrets in Pods

**1. Environment Variables (Individual Keys):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**2. Environment Variables (All Keys):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    envFrom:
    - secretRef:
        name: db-secret
```

**3. Volume Mount:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: secret
      mountPath: /etc/secrets
      readOnly: true
    # Files created:
    # /etc/secrets/username
    # /etc/secrets/password
  volumes:
  - name: secret
    secret:
      secretName: db-secret
```

**4. ImagePullSecrets (Docker Registry):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: private-registry.io/myapp:v1
  imagePullSecrets:
  - name: regcred
```

**Managing Secrets:**
```bash
# View Secrets (data is hidden)
kubectl get secrets

# Describe Secret (data is hidden)
kubectl describe secret db-secret

# View Secret data (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode Secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# Edit Secret
kubectl edit secret db-secret

# Delete Secret
kubectl delete secret db-secret
```

#### Secret Best Practices

**1. Never Commit Secrets to Git:**
```bash
# .gitignore
secrets/
*.secret.yaml
```

**2. Use External Secret Management:**
- AWS Secrets Manager + External Secrets Operator
- HashiCorp Vault + Vault CSI Driver
- Azure Key Vault + Secrets Store CSI Driver
- Google Secret Manager

**3. Enable Encryption at Rest:**
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-secret>
  - identity: {}
```

**4. Limit RBAC Access:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret"]  # Specific secrets only
  verbs: ["get"]
```

**5. Rotate Secrets Regularly:**
```bash
# Update secret
kubectl create secret generic db-secret \
  --from-literal=password=newsecret123 \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods to pick up new secret
kubectl rollout restart deployment myapp
```

**6. Use Immutable Secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
immutable: true  # Cannot be updated
data:
  password: c3VwZXJzZWNyZXQxMjM=
```

### 5.3 Immutable ConfigMaps and Secrets

Prevent accidental changes to configuration.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  database_url: "postgres://localhost:5432/mydb"
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
immutable: true
data:
  password: c3VwZXJzZWNyZXQxMjM=
```

**Benefits:**
- Prevent accidental updates
- Improve cluster performance (kubelet doesn't watch for changes)
- Require explicit ConfigMap/Secret recreation

**To Update:**
```bash
# Delete old
kubectl delete configmap app-config

# Create new
kubectl create configmap app-config --from-literal=key=newvalue

# Restart pods
kubectl rollout restart deployment myapp
```

### 5.4 Configuration Management Patterns

**1. ConfigMap + Secret Pattern:**
```yaml
# ConfigMap for non-sensitive data
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "postgres.example.com"
  database_port: "5432"
  database_name: "mydb"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: "admin"
  password: "supersecret123"
---
# Pod uses both
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-credentials
```

**2. Versioned Configuration Pattern:**
```yaml
# v1 configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
data:
  feature_flag: "false"
---
# v2 configuration (blue-green deployment)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  feature_flag: "true"
---
# Deployment v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config-v1
---
# Deployment v2 (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: app-config-v2
```

**3. ConfigMap for Application Files:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    http {
      server {
        listen 80;
        location / {
          proxy_pass http://backend:8080;
        }
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

---

## 6. Quick Reference

### 6.1 Essential Commands

**Deployments:**
```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# Rollout status
kubectl rollout status deployment/nginx

# Rollout history
kubectl rollout history deployment/nginx

# Rollback
kubectl rollout undo deployment/nginx

# Autoscale
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

**StatefulSets:**
```bash
# Create StatefulSet
kubectl apply -f statefulset.yaml

# Scale StatefulSet
kubectl scale statefulset mysql --replicas=5

# Delete StatefulSet (keep PVCs)
kubectl delete statefulset mysql

# Delete StatefulSet and PVCs
kubectl delete statefulset mysql
kubectl delete pvc -l app=mysql
```

**DaemonSets:**
```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# Update DaemonSet
kubectl set image ds/fluentd fluentd=fluentd:v1.15 -n kube-system

# View DaemonSet pods
kubectl get pods -l app=fluentd -o wide
```

**Jobs and CronJobs:**
```bash
# Create Job
kubectl create job test --image=busybox -- /bin/sh -c "echo hello"

# Create CronJob
kubectl create cronjob backup --schedule="0 2 * * *" --image=backup-tool

# Manually trigger CronJob
kubectl create job backup-manual --from=cronjob/backup

# Suspend CronJob
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'
```

**Scheduling:**
```bash
# Label node
kubectl label nodes node1 disktype=ssd

# Taint node
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-

# View node labels and taints
kubectl describe node node1
```

**Resources:**
```bash
# View resource usage
kubectl top nodes
kubectl top pods
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Check QoS class
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

**ConfigMaps:**
```bash
# Create ConfigMap
kubectl create configmap app-config --from-literal=key=value

# Create from file
kubectl create configmap app-config --from-file=config.properties

# View ConfigMap
kubectl get configmap app-config -o yaml

# Edit ConfigMap
kubectl edit configmap app-config
```

**Secrets:**
```bash
# Create Secret
kubectl create secret generic db-secret --from-literal=password=secret

# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass

# View Secret (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode Secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### 6.2 Common Patterns

**High Availability Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Spread across nodes
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: myapp
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:v1
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

**Sidecar Container Pattern:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Main application
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  # Log collector sidecar
  - name: log-collector
    image: fluent-bit:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
  volumes:
  - name: logs
    emptyDir: {}
```

**Init Container Pattern:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  # Run before main container
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: myapp:v1
```

---

## 7. Production Checklist

### Workloads

- [ ] **Deployments**
  - [ ] Resource requests and limits set
  - [ ] Multiple replicas for HA (minimum 3)
  - [ ] Rolling update strategy configured
  - [ ] maxSurge and maxUnavailable tuned
  - [ ] Readiness and liveness probes configured
  - [ ] PodDisruptionBudget defined

- [ ] **StatefulSets**
  - [ ] Headless service created
  - [ ] VolumeClaimTemplates configured
  - [ ] Update strategy defined
  - [ ] Backup strategy for PVCs
  - [ ] Pod Management Policy set appropriately

- [ ] **DaemonSets**
  - [ ] Resource limits set (prevent node exhaustion)
  - [ ] Tolerations configured for all required nodes
  - [ ] Update strategy defined
  - [ ] Host path mounts secured

### Scheduling

- [ ] **Node Selection**
  - [ ] Critical workloads use node affinity
  - [ ] Taints applied to specialized nodes
  - [ ] Tolerations configured correctly
  - [ ] Pod anti-affinity for HA workloads

- [ ] **Resource Management**
  - [ ] All pods have resource requests
  - [ ] Memory limits set (prevent OOM)
  - [ ] LimitRanges defined per namespace
  - [ ] ResourceQuotas enforced

### Autoscaling

- [ ] **HPA**
  - [ ] Metrics server installed
  - [ ] HPA configured for scalable workloads
  - [ ] Min replicas ≥ 3 for HA
  - [ ] Max replicas based on capacity planning
  - [ ] Scale-down behavior configured
  - [ ] Monitoring and alerting enabled

### Configuration

- [ ] **ConfigMaps**
  - [ ] Non-sensitive data only
  - [ ] Versioned naming convention
  - [ ] Immutable for critical config
  - [ ] Separated by concern

- [ ] **Secrets**
  - [ ] Encryption at rest enabled
  - [ ] RBAC policies restrict access
  - [ ] External secret management considered
  - [ ] Rotation policy defined
  - [ ] Never committed to version control

### Monitoring

- [ ] Resource usage monitored
- [ ] HPA metrics tracked
- [ ] ConfigMap/Secret changes audited
- [ ] Pod scheduling failures alerted
- [ ] Node resource pressure monitored

---

## 8. Further Reading

### Official Documentation

- [Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Configuration](https://kubernetes.io/docs/concepts/configuration/)
- [Managing Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

### Best Practices

- [Resource Management Best Practices](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Running Production Applications](https://kubernetes.io/docs/tasks/run-application/)

### Tools

- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [HashiCorp Vault](https://www.vaultproject.io/)

### CKA/CKAD Preparation

- [CKA Curriculum](https://github.com/cncf/curriculum)
- [CKAD Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Author:** Sajal Jana

**License:** This document is provided as-is for educational purposes.
