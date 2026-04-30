================================================================================
          KUBERNETES STORAGE -- FIELD NOTES
          Author : Sajal Jana
          
================================================================================

  PersistentVolume  (PV)  = Actual storage unit
  PersistentVolumeClaim (PVC) = Your rental agreement
  StorageClass            = Auto-finds storage for you
  Volume                  = Where your pod keeps its stuff

  Developer          Cluster
     |                  |
   [PVC] <-- matched --> [PV]
     |                  |
   [Pod] <-- accesses -> [Storage Backend]


================================================================================
1. PERSISTENTVOLUMES & CLAIMS
================================================================================

-- Creating a PV (rarely done manually) --

  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: my-storage
  spec:
    capacity:
      storage: 10Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain   # <-- keep data after release
    storageClassName: fast
    nfs:
      server: nfs-server.example.com
      path: "/data"


-- Creating a PVC (what YOU actually create) --

  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-claim
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
    storageClassName: fast


-- Access Modes --

  RWO (ReadWriteOnce) --> One node only, read+write    "My precious!"
  ROX (ReadOnlyMany)  --> Many nodes, read only        "Look, don't touch"
  RWX (ReadWriteMany) --> Many nodes, read+write       "Sharing is caring"

  NOTE: AWS EBS / Azure Disk only support RWO.
        For RWX you need NFS or CephFS.


-- Reclaim Policies --

  Retain  --> Data stays, manual cleanup needed   (use for production DBs)
  Delete  --> Everything deleted automatically    (default -- scary!)
  Recycle --> Deprecated. Don't use. Seriously.


-- PV Lifecycle --

  Available --> Bound --> Released --> Deleted
   "Single"    "Taken"   "It's         "RIP"
                          Complicated"


================================================================================
2. STORAGECLASSES
================================================================================

-- Basic StorageClass --

  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast
  provisioner: ebs.csi.aws.com
  parameters:
    type: gp3
    iops: "3000"
    throughput: "125"
    encrypted: "true"
  volumeBindingMode: WaitForFirstConsumer   # <-- ALWAYS use this
  allowVolumeExpansion: true
  reclaimPolicy: Delete


-- Cloud Provider Quick Ref --

  AWS   --> type: gp3 (balanced) / io2 (high IOPS) / st1 (big data)
  GCP   --> type: pd-ssd / pd-balanced / pd-standard
  Azure --> storageaccounttype: Premium_LRS / StandardSSD_LRS / Standard_LRS


-- volumeBindingMode: WHY IT MATTERS --

  Immediate (bad):
    Volume created in zone-A, pod lands in zone-B --> attach error at 3 AM

  WaitForFirstConsumer (good):
    Pod scheduled first, volume created in SAME zone --> works every time

  Rule: Always use WaitForFirstConsumer. No exceptions.


================================================================================
3. VOLUME TYPES
================================================================================

  emptyDir  --> Temp scratch space. Dies with pod. Don't store anything here.

    volumes:
    - name: cache
      emptyDir: {}


  hostPath  --> Node filesystem. Security risk. Not for production.

    volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket


  PVC       --> The real deal. Use for all important data.

    volumes:
    - name: data
      persistentVolumeClaim:
        claimName: mysql-pvc


================================================================================
4. VOLUME EXPANSION
================================================================================

  1. StorageClass must have:  allowVolumeExpansion: true
  2. Patch the PVC:

       $ kubectl patch pvc mysql-pvc \
           -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

  3. Some volumes need a pod restart to finish expansion.

  NOTE: You can only go BIGGER, never smaller. Like your cloud bill.


================================================================================
5. VOLUME SNAPSHOTS
================================================================================

-- Create snapshot --

  apiVersion: snapshot.storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: before-i-break-everything
  spec:
    volumeSnapshotClassName: csi-snapclass
    source:
      persistentVolumeClaimName: important-data


-- Restore from snapshot --

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

  Rule: Always snapshot before major changes. Your future self will thank you.


================================================================================
6. COMMON PATTERNS
================================================================================

-- StatefulSet (Database Pattern) --

  StatefulSets auto-create PVCs per pod using volumeClaimTemplates:

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 100Gi

  Result: data-mysql-0, data-mysql-1, data-mysql-2 (each pod = own storage)
  Warning: Delete StatefulSet does NOT delete PVCs. Do it manually!


-- Shared Storage (RWX Pattern) --

  PVC must use ReadWriteMany + NFS/CephFS StorageClass.
  Multiple pod replicas all mount the same PVC. Simple.


-- Daily Backup CronJob --

  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: daily-backup
  spec:
    schedule: "0 2 * * *"
    jobTemplate:
      spec:
        template:
          spec:
            containers:
            - name: backup
              image: amazon/aws-cli
              command: ["/bin/sh","-c","aws s3 sync /data s3://my-backups/$(date +%Y%m%d)/"]
              volumeMounts:
              - name: data
                mountPath: /data
                readOnly: true
            volumes:
            - name: data
              persistentVolumeClaim:
                claimName: important-data
            restartPolicy: OnFailure


================================================================================
7. TROUBLESHOOTING
================================================================================

-- PVC stuck in "Pending" --

  $ kubectl get storageclass          # does it exist?
  $ kubectl describe pvc <name>       # read the Events section

  Causes:
    - No StorageClass          --> create one
    - No matching PV           --> create PV or fix name
    - Provisioner not running  --> check kube-system pods
    - Zone mismatch            --> use WaitForFirstConsumer
    - Quota exceeded           --> ask for more resources


-- Pod stuck in "ContainerCreating" --

  $ kubectl describe pod <name>       # read the Events section

  Causes:
    - Volume still attached to old node  --> wait 5+ min for timeout
    - PVC missing                        --> create it
    - Wrong access mode                  --> check volume type
    - CSI driver crashed                 --> check kube-system pods


-- Permission Denied on write --

  Add fsGroup to pod spec:

    spec:
      securityContext:
        fsGroup: 1000    # match container user ID


-- Slow storage --

  Use faster type (gp3 > gp2, SSD > HDD) and add mountOptions:

    mountOptions:
      - noatime
      - nodiratime
      - discard


================================================================================
8. ESSENTIAL COMMANDS
================================================================================

  kubectl get pv                         # list PersistentVolumes
  kubectl get pvc                        # list PVCs
  kubectl get sc                         # list StorageClasses
  kubectl get volumesnapshots            # list snapshots

  kubectl describe pvc <name>            # debug pending PVC
  kubectl describe pod <name>            # debug stuck pod

  kubectl get pvc -w                     # watch PVCs live

  kubectl patch pvc <name> \
    -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

  kubectl exec -it <pod> -- df -h        # check actual disk usage

  kubectl annotate storageclass fast \
    storageclass.kubernetes.io/is-default-class=true

  kubectl delete pvc <name>              # careful -- may delete PV too!


================================================================================
9. PRODUCTION CHECKLIST
================================================================================

  Before going live:

  [ ] StorageClasses set up (fast / standard / cheap tiers)
  [ ] Default StorageClass annotated
  [ ] WaitForFirstConsumer enabled on all classes
  [ ] allowVolumeExpansion: true on all classes
  [ ] Reclaim policy = Retain for production data
  [ ] Encryption enabled
  [ ] Backups configured (snapshots + external)
  [ ] Backup RESTORE tested (untested = no backup)
  [ ] Monitoring on usage / IOPS / latency
  [ ] Resource quotas defined
  [ ] RBAC for PVC creation configured

  Cost hygiene:
  [ ] Delete orphaned PVs       (they cost money!)
  [ ] Clean up old snapshots    (they cost money!)
  [ ] Right-size volumes        (don't request 1TB for 10GB workloads)


================================================================================
10. DO / DON'T CHEAT SHEET
================================================================================

  DO:
    + Use StorageClasses (dynamic provisioning)
    + Use WaitForFirstConsumer
    + Enable volume expansion
    + Use Retain for production
    + Test backups regularly
    + Set resource quotas

  DON'T:
    - hostPath in production     (security nightmare)
    - emptyDir for real data     (it's gone when pod dies)
    - Forget to delete old PVCs  (billing surprise)
    - Skip backups               (Murphy's Law is real)
    - Trust Delete policy won't  (it will. it always will)
      actually delete

