# Kubernetes Workloads & Scheduling 

**Author:** Sajal Jana  
**Motto:** *Because reading documentation shouldn't feel like a punishment*

---

## 🎯 Quick Navigation

1. [Workload Resources](#workload-resources) - *The pod wranglers*
2. [Scheduling](#scheduling) - *Playing favorites with nodes*
3. [Resource Management](#resource-management) - *Don't be greedy*
4. [Autoscaling](#autoscaling) - *Set it and forget it (kinda)*
5. [Configuration Management](#configuration-management) - *Secrets and lies*
6. [Cheat Sheet](#cheat-sheet) - *For when you forget everything*

---

## Workload Resources

*Think of these as different ways to babysit your containers*

### Deployments - The Reliable Friend

**Use when:** You have a normal, well-adjusted stateless app  
**Avoid when:** Your app needs to remember things (use StatefulSet)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-awesome-app
spec:
  replicas: 3  # Safety in numbers!
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # "How many extra pods can we handle?"
      maxUnavailable: 1  # "How many can fail before panic?"
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            memory: "64Mi"   # "I promise I need this much"
            cpu: "250m"      # "Pretty please?"
          limits:
            memory: "128Mi"  # "Don't let me eat more than this"
            cpu: "500m"      # "Cut me off here"
```

**Pro tip:** Always set `replicas: 3` minimum. Because one is lonely, two is risky, and three is a party! 🎉

**Common Commands:**
```bash
# The greatest hits
kubectl scale deployment/my-app --replicas=5              # More power!
kubectl set image deployment/my-app app=nginx:1.22        # Upgrade time
kubectl rollout undo deployment/my-app                    # Oops, go back!
kubectl rollout status deployment/my-app                  # Are we there yet?
```

---

### StatefulSets - The Clingy One

**Use when:** Your app has commitment issues (needs stable identity & storage)  
**Perfect for:** Databases, Kafka, anything that remembers you

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless  # "I need my personal space"
  replicas: 3
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:  # Each pod gets its own storage
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi  # "This is MY disk, get your own!"
```

**Key Features:**
- Predictable names: `mysql-0`, `mysql-1`, `mysql-2` (no random suffixes!)
- Ordered operations: Created 0→1→2, deleted 2→1→0 (polite and organized)
- Persistent storage: "Till delete do us part"

**Warning:** Don't use StatefulSets for stateless apps. That's like buying a sports car for grocery shopping. 🏎️🛒

---

### DaemonSets - The Everywhere Agent

**Use when:** Every node needs one (and only one) copy  
**Perfect for:** Monitoring agents, log collectors, that nosy friend who's always around

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule  # "I can sit anywhere, even with the VIPs"
      containers:
      - name: fluentd
        image: fluentd:v1.14
```

**Reality check:** One DaemonSet pod per node, automatically. New node joins? Pod appears. Node leaves? Pod vanishes. It's like magic, but with YAML.

---

### Jobs & CronJobs - The Task Masters

**Jobs:** Run once and peace out ✌️  
**CronJobs:** Run on schedule, like clockwork ⏰

```yaml
# Job - "I'll do it once and I'm done"
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Never  # "No second chances"
      containers:
      - name: migrator
        image: data-migrator:v1
---
# CronJob - "Same time, every day"
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily (when humans sleep)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:v1
```

**Cron schedule decoder:**
```
 ┌─ minute (0-59)
 │ ┌─ hour (0-23)
 │ │ ┌─ day of month (1-31)
 │ │ │ ┌─ month (1-12)
 │ │ │ │ ┌─ day of week (0-6, Sunday=0)
 * * * * *
 
 "*/15 * * * *"  = Every 15 minutes (for the impatient)
 "0 2 * * *"     = Daily at 2 AM (graveyard shift)
 "0 0 1 * *"     = Monthly, 1st day (rent day!)
```

---

### Decision Tree: Which Workload?

```
Running on ALL nodes? 
├─ Yes → DaemonSet (the stalker)
└─ No
   ├─ Needs stable identity/storage?
   │  └─ Yes → StatefulSet (the clingy one)
   └─ No
      ├─ Run to completion?
      │  ├─ Once → Job (one-night stand)
      │  └─ Scheduled → CronJob (committed relationship)
      └─ Long-running → Deployment (marriage material)
```

---

## Scheduling

*Because not all nodes are created equal*

### Node Selectors - The Simple Filter

**For:** "I only want to run on fancy SSD nodes"

```yaml
spec:
  nodeSelector:
    disktype: ssd  # Picky, aren't we?
```

```bash
# Label your nodes first
kubectl label nodes node1 disktype=ssd
kubectl label nodes node2 disktype=hdd  # The budget option
```

---

### Node Affinity - The Sophisticated Filter

**For:** Complex requirements with preferences

```yaml
spec:
  affinity:
    nodeAffinity:
      # MUST have (hard requirement)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme  # Accept either, we're flexible
      # PREFER (soft suggestion)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100  # "Really want this"
        preference:
          matchExpressions:
          - key: gpu
            operator: Exists
```

**Translation:** "I MUST have SSD/NVMe, but I'd REALLY PREFER a GPU too. Pretty please? 🥺"

---

### Taints & Tolerations - The Bouncer System

**Taint:** "Get lost!" (applied to nodes)  
**Toleration:** "But I'm on the list!" (applied to pods)

```bash
# Taint a node (node becomes picky)
kubectl taint nodes gpu-node1 dedicated=gpu:NoSchedule

# Only pods with this toleration can schedule there
```

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"  # "I have a VIP pass"
```

**Taint Effects:**
- `NoSchedule` - "No new pods allowed" (existing ones stay)
- `PreferNoSchedule` - "Please don't, but if you must..." (soft)
- `NoExecute` - "EVERYONE OUT!" (evicts existing pods)

**Common use case:** GPU nodes
```bash
# Taint GPU nodes so only ML workloads use them
kubectl taint nodes gpu-node dedicated=gpu:NoSchedule

# Regular apps: "Can't sit here"
# ML apps with toleration: "Actually, this is my seat"
```

---

### Pod Affinity - The Social Butterfly

**Use when:** You want pods to be BFFs (sit together)

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname  # Same node
```

**Translation:** "Schedule me on the same node as the cache. We're inseparable! 💕"

---

### Pod Anti-Affinity - The Introvert

**Use when:** You want pods to avoid each other (spread out)

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname  # Different nodes
```

**Translation:** "Don't put me near another web pod. I need personal space! 😤"

**Why?** High availability! If one node dies, others survive.

---

## Resource Management

*Don't be a resource hog*

### Requests vs Limits - The Social Contract

```yaml
resources:
  requests:
    memory: "256Mi"  # "I pinky promise I need this"
    cpu: "200m"      # (used for scheduling)
  limits:
    memory: "512Mi"  # "Don't let me use more than this"
    cpu: "500m"      # (CPU = throttle, Memory = OOMKill)
```

**What happens:**
- **Request:** Scheduler finds node with this much free
- **Limit:** Container gets cut off at this point
  - CPU limit → Throttling (slow but alive)
  - Memory limit → OOMKilled (dead) 💀

**CPU Units:**
```yaml
cpu: "1"      # 1 full core (luxury!)
cpu: "500m"   # 0.5 core (half caff)
cpu: "100m"   # 0.1 core (decaf)
```

**Memory Units:**
```yaml
memory: "1Gi"    # 1 gibibyte (1024³ bytes)
memory: "1G"     # 1 gigabyte (1000³ bytes)
memory: "128Mi"  # 128 mebibytes (most common)
```

---

### QoS Classes - The Caste System

Kubernetes assigns priority based on resources:

**1. Guaranteed (First Class)** 🥇
- Requests = Limits for ALL resources
- Last to be evicted
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"  # Same!
    cpu: "500m"      # Same!
```

**2. Burstable (Economy Plus)** 🥈
- Has requests/limits, but not equal
- Middle priority
```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "512Mi"  # Can burst!
```

**3. BestEffort (Standby)** 🥉
- No requests or limits
- First to be evicted
```yaml
# Nothing specified = "YOLO"
```

**Check QoS:**
```bash
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
```

---

### LimitRange - The Namespace Cop

**For:** "No one gets more than X in this namespace!"

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - max:
      memory: "1Gi"    # "Don't be greedy"
    min:
      memory: "64Mi"   # "But don't be cheap"
    default:
      memory: "256Mi"  # "If you don't specify, you get this"
    type: Container
```

---

### ResourceQuota - The Accountant

**For:** "The entire namespace can't use more than X"

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "10"        # Total CPU requests
    requests.memory: "20Gi"   # Total memory requests
    pods: "50"                # Max 50 pods
    services: "10"            # Max 10 services
```

**Check quota:**
```bash
kubectl describe resourcequota -n dev
# Shows: Used vs Hard limit
```

---

## Autoscaling

*Set it and forget it (mostly)*

### HPA - Horizontal Pod Autoscaler

**For:** "Add more pods when busy, remove when idle"

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3   # Never go below this (HA!)
  maxReplicas: 20  # Emergency brake
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Keep it at 70% CPU
```

**Quick version:**
```bash
kubectl autoscale deployment my-app --min=3 --max=10 --cpu-percent=80
```

**How it works:**
1. Metrics server collects CPU usage
2. HPA calculates: `desiredReplicas = currentReplicas × (currentCPU / targetCPU)`
3. If busy → add pods
4. If idle → remove pods
5. Repeat every 15 seconds

**Watch it work:**
```bash
kubectl get hpa -w  # Watch mode (like reality TV)
```

---

### Advanced HPA - The Power User

```yaml
spec:
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60  # Max 1 pod per minute (slow and steady)
    scaleUp:
      policies:
      - type: Percent
        value: 100  # Can double immediately (panic mode!)
        periodSeconds: 30
```

**Translation:** "Scale up fast, scale down slow. Like a caffeinated snail on the way down."

---

### VPA - Vertical Pod Autoscaler

**For:** "Adjust resource requests automatically"

**Warning:** ⚠️ Separate installation required. Can conflict with HPA on CPU/memory.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"  # YOLO mode
```

**Reality:** Most people use HPA, not VPA. VPA is the weird cousin.

---

## Configuration Management

*Separating secrets from code since 2014*

### ConfigMaps - The Public Board

**For:** Non-sensitive configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432/mydb"
  log_level: "info"
  max_connections: "100"
  # Even multi-line files!
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
    }
```

**Quick create:**
```bash
kubectl create configmap app-config \
  --from-literal=api_url=https://api.com \
  --from-literal=timeout=30
```

**Use in pod (environment variables):**
```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    # Now DATABASE_URL, LOG_LEVEL, etc. are environment variables
```

**Use in pod (files):**
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
  # Creates: /etc/config/database_url, /etc/config/log_level, etc.
```

---

### Secrets - The Vault

**For:** Sensitive data (passwords, tokens, keys)

**⚠️ Important:** Secrets are base64-encoded, NOT encrypted by default! Enable encryption at rest!

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:  # Plain text (gets auto-encoded)
  username: admin
  password: supersecret123
```

**Quick create:**
```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret123
```

**Use in pod:**
```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**Decode a secret:**
```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

### Secret Best Practices

**DO:**
- ✅ Enable encryption at rest
- ✅ Use RBAC to limit access
- ✅ Use external secret managers (Vault, AWS Secrets Manager)
- ✅ Rotate secrets regularly
- ✅ Mark as immutable when possible

**DON'T:**
- ❌ Commit secrets to Git (use `.gitignore`)
- ❌ Use ConfigMaps for sensitive data
- ❌ Give everyone access to all secrets
- ❌ Hardcode secrets in images

**Pro tip:** Use External Secrets Operator to sync from AWS/GCP/Azure secret managers. Future you will thank you!

---

## Cheat Sheet

### Deployment Operations
```bash
# Create
kubectl create deployment app --image=nginx --replicas=3

# Scale
kubectl scale deployment/app --replicas=5

# Update
kubectl set image deployment/app nginx=nginx:1.22

# Rollback
kubectl rollout undo deployment/app

# Status
kubectl rollout status deployment/app

# History
kubectl rollout history deployment/app
```

### Resource Viewing
```bash
# Who's using what?
kubectl top nodes
kubectl top pods
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# QoS class
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
```

### Node Operations
```bash
# Label nodes
kubectl label nodes node1 disktype=ssd

# Taint nodes
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint (note the minus)
kubectl taint nodes node1 key-

# View labels/taints
kubectl describe node node1
```

### ConfigMap & Secret
```bash
# ConfigMap
kubectl create configmap app-config --from-literal=key=value
kubectl get configmap app-config -o yaml

# Secret
kubectl create secret generic db-secret --from-literal=password=secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### HPA
```bash
# Create
kubectl autoscale deployment app --min=2 --max=10 --cpu-percent=80

# Watch
kubectl get hpa -w

# Describe
kubectl describe hpa app
```

---

## Production Checklist ✅

### Before You Deploy

**Workloads:**
- [ ] Set resource requests/limits (don't be that person)
- [ ] Min 3 replicas for HA (one is none, two is one, three is some)
- [ ] Configure readiness/liveness probes (health checks are your friend)
- [ ] Set up PodDisruptionBudget (prevent accidental outages)

**Scheduling:**
- [ ] Use node affinity for special hardware
- [ ] Set pod anti-affinity for HA (spread the love)
- [ ] Configure taints/tolerations for dedicated nodes

**Resources:**
- [ ] All pods have requests (scheduler needs to know!)
- [ ] Memory limits set (prevent OOMKiller rampage)
- [ ] LimitRanges defined per namespace (set guardrails)
- [ ] ResourceQuotas enforced (prevent resource hogging)

**Autoscaling:**
- [ ] HPA configured (if applicable)
- [ ] Metrics server installed (required for HPA)
- [ ] Min replicas ≥ 3 (HA, remember?)
- [ ] Monitoring enabled (watch it work!)

**Configuration:**
- [ ] Secrets encrypted at rest (not just base64!)
- [ ] ConfigMaps versioned (no accidental updates)
- [ ] RBAC limits secret access (least privilege)
- [ ] Nothing committed to Git (seriously, check twice)

---

## Final Words of Wisdom

1. **Start simple, iterate:** Don't over-engineer on day one
2. **Monitor everything:** You can't improve what you don't measure
3. **Test your autoscaling:** Generate load and watch it scale
4. **Document your decisions:** Future you will ask "Why did I do this?"
5. **Read the error messages:** They're actually helpful (sometimes)

**Remember:** Kubernetes is complex, but you don't need to use every feature. Start with Deployments, add resources, configure HPA, and you're already ahead of most people!

---

**Happy Kuberneting!** 🚀

*P.S. - If something breaks, try turning it off and on again. Works 60% of the time, every time.*

---

**Author:** Sajal Jana  
**Last Updated:** February 2026  
**Disclaimer:** Humor may vary by reader. Side effects include increased understanding and occasional chuckles.
