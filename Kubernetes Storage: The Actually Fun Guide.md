# Kubernetes Storage: The Actually Fun Guide 🎢

**Author:** Sajal Jana  
**Motto:** *"Where your data goes to live happily ever after (hopefully)"*

---

## Table of Contents

- [The Basics (aka "Please Don't Lose My Data")](#the-basics)
- [PersistentVolumes & Claims: The Dating Game](#persistentvolumes--claims-the-dating-game)
- [StorageClasses: Your Storage Matchmaker](#storageclasses-your-storage-matchmaker)
- [Volume Operations: The Technical Stuff](#volume-operations)
- [Common Patterns: What Actually Works](#common-patterns)
- [Troubleshooting: When Things Go Boom](#troubleshooting)
- [Quick Reference: For When You're Panicking](#quick-reference)

---

## The Basics

Think of Kubernetes storage like a fancy storage unit facility:
- **PersistentVolume (PV)** = The actual storage unit 🏢
- **PersistentVolumeClaim (PVC)** = Your rental agreement 📝
- **StorageClass** = The property manager who finds units for you 🤵
- **Volume** = The folder where your pod keeps its stuff 📂

```
Developer              Cluster Admin
    |                        |
    | "I need storage!"      | "Here's a storage unit!"
    ▼                        ▼
  [PVC] ←─── matched ───→ [PV]
    |                        |
    | "Mount me!"            |
    ▼                        ▼
  [Pod] ←─── accesses ──→ [Storage Backend]
                           (AWS/NFS/etc)
```

---

## PersistentVolumes & Claims: The Dating Game

### Creating a PersistentVolume (PV)

**Pro tip:** Most of the time, you don't create these manually. That's like going to a matchmaker and bringing your own date. But here's how anyway:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-storage
spec:
  capacity:
    storage: 10Gi  # Like saying "I want a 10GB storage unit"
  accessModes:
    - ReadWriteOnce  # "Only I can use this, and I can write to it"
  persistentVolumeReclaimPolicy: Retain  # "Don't throw away my stuff!"
  storageClassName: fast
  nfs:
    server: nfs-server.example.com
    path: "/data"
```

### Creating a PersistentVolumeClaim (PVC)

This is what YOU actually create. Think of it as swiping right on a storage unit:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi  # "I need at least 10GB"
  storageClassName: fast  # "Find me something fast!"
```

**The matching process:**
1. You create a PVC with requirements
2. Kubernetes plays matchmaker
3. Finds a PV that meets your standards (size, speed, access mode)
4. They bind together in holy matrimony 💒
5. Your pod can now use the storage

### Access Modes (aka "Who Gets to Touch This?")

| Mode | Code | Meaning | Real Talk |
|------|------|---------|-----------|
| ReadWriteOnce | RWO | One node, read/write | "My precious!" 💍 |
| ReadOnlyMany | ROX | Many nodes, read-only | "Look, don't touch" 👀 |
| ReadWriteMany | RWX | Many nodes, read/write | "Sharing is caring" 🤝 |

**Important:** Most cloud storage (AWS EBS, Azure Disk) only supports RWO. For RWX, you need network storage like NFS. *This is why everyone complains about storage in Kubernetes.*

### PV Lifecycle (The Circle of Life)

```
Available → Bound → Released → Failed/Deleted
   ↓          ↓         ↓           ↓
 "Single"   "Taken"  "It's      "RIP" 💀
                     Complicated"
```

### Reclaim Policies (What Happens After Breakup)

**Retain:** "I'll keep your stuff safe, just in case" 
- Data stays, manual cleanup required
- Use for production databases you actually care about

**Delete:** "I'm burning all your stuff!" 🔥
- Everything gets deleted automatically
- Use for dev/test environments
- *Default for most StorageClasses (scary, right?)*

**Recycle (Deprecated):** "I'll just throw it all in the trash" 🗑️
- Don't use this. Seriously.

---

## StorageClasses: Your Storage Matchmaker

StorageClasses are like having a personal assistant who provisions storage for you. No more begging the admin!

### Basic StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com  # "Use AWS EBS to create volumes"
parameters:
  type: gp3  # General Purpose SSD v3 (the good stuff)
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer  # "Don't create until pod is scheduled"
allowVolumeExpansion: true  # "You can make it bigger later"
reclaimPolicy: Delete  # "Clean up after yourself"
```

### Cloud Provider Examples

**AWS (The Popular Kid):**
```yaml
parameters:
  type: gp3           # Balanced: Good performance, reasonable cost
  # type: io2        # Speed demon: High IOPS, $$$$
  # type: st1        # Big data: Throughput optimized HDD
  # type: sc1        # Cheapskate: Cold storage HDD
  encrypted: "true"  # Because security matters
```

**GCP (The Smart One):**
```yaml
parameters:
  type: pd-ssd        # Fast SSD
  # type: pd-balanced  # Middle ground
  # type: pd-standard  # Slow but cheap
  replication-type: regional-pd  # Replicated across zones (fancy!)
```

**Azure (The Enterprise One):**
```yaml
parameters:
  storageaccounttype: Premium_LRS  # Premium SSD
  # StandardSSD_LRS  # Standard SSD
  # Standard_LRS      # Standard HDD (slow)
  kind: Managed
```

### The Magic of Dynamic Provisioning

**Old Way (Static):**
```
1. Admin creates PV manually
2. User creates PVC
3. User waits
4. Still waiting...
5. "Admin, did you create my volume?" 😴
```

**New Way (Dynamic):**
```
1. User creates PVC with StorageClass
2. *POOF* ✨ Volume appears automatically
3. User is happy
4. Admin is happy (less work)
5. Everyone is happy! 🎉
```

### volumeBindingMode: The Zone Trick

**Immediate (Default):** "Create volume RIGHT NOW!"
- Problem: Volume in us-east-1a, pod scheduled to us-east-1b
- Result: "Error: Can't attach volume" 😭

**WaitForFirstConsumer (Smart):** "Wait until pod is scheduled!"
- Pod scheduled to us-east-1b
- Volume created in us-east-1b
- Everything works! 🎊

*Always use WaitForFirstConsumer unless you enjoy debugging zone issues at 3 AM.*

---

## Volume Operations

### Volume Types (Choose Your Fighter)

**EmptyDir:** Temporary storage (dies with pod)
```yaml
volumes:
- name: cache
  emptyDir: {}  # 💨 Poof! Gone when pod dies
```
*Use for: Scratch space, temp files, "I don't care if I lose this"*

**HostPath:** Node's filesystem (dangerous!)
```yaml
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket
```
*Use for: Accessing Docker socket, being a rebel 🏴‍☠️*
*Don't use for: Anything in production, multi-tenant clusters*

**PersistentVolumeClaim:** The real deal
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: mysql-pvc
```
*Use for: Everything important, databases, actual data*

### Volume Expansion (Making It Bigger)

1. Make sure StorageClass has `allowVolumeExpansion: true`
2. Edit your PVC:
```bash
kubectl patch pvc mysql-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```
3. Magic happens! ✨
4. *Some volumes need pod restart to complete expansion*

**Important:** You can only go bigger, never smaller. Like your cloud bill! 📈

### Volume Snapshots (Time Machine for Data)

**Create a snapshot:**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: before-i-break-everything
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: important-data
```

**Restore from snapshot:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: phew-that-was-close
spec:
  dataSource:
    name: before-i-break-everything
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

*Always snapshot before major changes. Your future self will thank you.* 🙏

---

## Common Patterns

### StatefulSet with Storage (The Database Pattern)

StatefulSets automatically create PVCs for each pod. It's like having a personal storage unit!

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
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
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:  # 🪄 Magic happens here
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 100Gi
```

**Result:** Creates `data-mysql-0`, `data-mysql-1`, `data-mysql-2`
- Each pod gets its own storage
- Storage persists even if StatefulSet is deleted
- *Remember to delete PVCs manually to avoid surprise cloud bills!*

### Shared Storage (The Collaboration Pattern)

Multiple pods sharing the same files. Requires RWX support (NFS, CephFS, etc.):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files
spec:
  accessModes:
    - ReadWriteMany  # Key: Must support RWX!
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs  # NFS or similar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 5  # All 5 pods share the same storage
  template:
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: shared-files
```

### Backup Pattern (For the Paranoid)

**Daily backups to S3:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: amazon/aws-cli
            command:
            - /bin/sh
            - -c
            - |
              aws s3 sync /data s3://my-backups/$(date +%Y%m%d)/
            volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: important-data
          restartPolicy: OnFailure
```

*"The best time to set up backups was yesterday. The second best time is now."* 🕐

---

## Troubleshooting

### PVC Stuck in "Pending" 😰

**Check 1: Does StorageClass exist?**
```bash
kubectl get storageclass
# If empty: "Houston, we have a problem"
```

**Check 2: Any errors?**
```bash
kubectl describe pvc stuck-pvc
# Look for events at bottom
```

**Common Causes:**
- No StorageClass found ➜ Create one or fix name
- No available PVs (static) ➜ Create matching PV
- Provisioner not running ➜ Check kube-system pods
- Zone mismatch ➜ Use `WaitForFirstConsumer`
- Out of quota ➜ Ask for more resources

### Pod Stuck in "ContainerCreating" 😱

```bash
kubectl describe pod stuck-pod
# Events section will tell you what's wrong
```

**Common Causes:**
- Volume still attached to old node ➜ Wait for timeout (can take 5+ min)
- PVC doesn't exist ➜ Create it!
- Wrong access mode ➜ Check volume type supports it
- CSI driver crashed ➜ Check kube-system pods
- Node affinity mismatch ➜ PV locked to specific node

**The Nuclear Option:** Force detach volume (dangerous!)
```bash
# AWS EBS
aws ec2 detach-volume --volume-id vol-xxx --force
# Then delete and recreate pod
```

### "Permission Denied" When Writing 🚫

Add `fsGroup` to your pod:
```yaml
spec:
  securityContext:
    fsGroup: 1000  # Match your container's user ID
  containers:
  - name: app
    image: myapp
```

### Slow Storage Performance 🐌

**Quick Fixes:**
1. Use faster storage type (gp3 > gp2, SSD > HDD)
2. Increase IOPS (for cloud storage)
3. Add mount options:
```yaml
mountOptions:
  - noatime      # Don't update access times
  - nodiratime   # Don't update dir access times
  - discard      # Enable TRIM for SSD
```

---

## Quick Reference

### Essential Commands

```bash
# List all the things
kubectl get pv                    # Show all PersistentVolumes
kubectl get pvc                   # Show all PVCs in current namespace
kubectl get sc                    # Show StorageClasses
kubectl get volumesnapshots       # Show snapshots

# Describe (when things break)
kubectl describe pvc mysql-pvc    # Why is this pending?
kubectl describe pod mysql-pod    # Why won't this start?

# Watch (when you're impatient)
kubectl get pvc -w               # Watch PVCs change

# Expand a PVC
kubectl patch pvc mysql-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Check actual usage
kubectl exec -it pod-name -- df -h

# Delete (with caution!)
kubectl delete pvc my-pvc        # Deletes claim
# PV might be deleted too (if policy is Delete)
```

### Common Scenarios

**Create PVC with default StorageClass:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: quick-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```

**Set default StorageClass:**
```bash
kubectl annotate storageclass fast \
  storageclass.kubernetes.io/is-default-class=true
```

**Create snapshot:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: backup-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: my-data
EOF
```

---

## Production Checklist ✅

**Before You Go Live:**

- [ ] StorageClasses configured with proper tiers (fast/standard/cheap)
- [ ] Default StorageClass set (to avoid confusion)
- [ ] `WaitForFirstConsumer` enabled (save yourself from zone hell)
- [ ] `allowVolumeExpansion: true` (future you will thank you)
- [ ] Reclaim policy set to **Retain** for production data
- [ ] Encryption enabled (because compliance)
- [ ] Backups configured (snapshots or external)
- [ ] Backup restore tested (untested backups = no backups)
- [ ] Monitoring set up (track usage, IOPS, latency)
- [ ] Resource quotas defined (prevent runaway costs)
- [ ] Access control configured (RBAC for PVC creation)
- [ ] Disaster recovery plan written (and actually read by team)

**Cost Optimization:**
- [ ] Delete orphaned PVs (they cost money!)
- [ ] Clean up old snapshots (they cost money!)
- [ ] Right-size volumes (don't request 1TB when you need 10GB)
- [ ] Use cheaper storage tiers for non-critical data

---

## Best Practices (Learned the Hard Way)

### DO ✅
- Always use StorageClasses (dynamic provisioning FTW)
- Always use `WaitForFirstConsumer` binding mode
- Always enable volume expansion
- Always use `Retain` for production data
- Always test your backups
- Always monitor storage usage
- Always set resource quotas

### DON'T ❌
- Don't use `hostPath` in production (security nightmare)
- Don't use `emptyDir` for important data (it's temporary!)
- Don't forget to delete unused PVCs ($$$ adds up)
- Don't skip backups (Murphy's Law is real)
- Don't use `Immediate` binding (zone issues ahead)
- Don't assume `Delete` policy won't actually delete (it will!)

### MAYBE 🤔
- Use local storage for performance-critical apps (but lose portability)
- Use network storage for sharing (but lose some performance)
- Use snapshots for quick backups (but don't rely on them alone)

---

## Further Reading

**When You Want to Go Deeper:**
- [Official K8s Storage Docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [CSI Driver List](https://kubernetes-csi.github.io/docs/drivers.html) (find drivers for your storage)
- [Velero](https://velero.io/) (backup/restore tool - actually good!)
- [Rook](https://rook.io/) (storage orchestration - for the brave)

---

## The End! 🎬

**Remember:**
- Storage is hard
- Backups are important
- `Delete` policy means *delete*
- Zone issues are the worst
- But you got this! 💪

**Questions?** Check the logs. Still stuck? Check the events. Still stuck? Google it. Still stuck? Ask your friendly neighborhood SRE. They've seen it all.

---

**Document Version:** 2.0 (Now with 100% more fun!)  
**Last Updated:** February 17, 2026  
**Author:** Sajal Jana  
**Status:** Caffeinated ☕

*"In production, no one can hear you scream... unless you set up proper monitoring and alerting."* 👻
