# Kubernetes Production Troubleshooting Guide
## Top 20 Scenarios

---

## 1. Pod Stuck in CrashLoopBackOff

**Symptoms:** `STATUS: CrashLoopBackOff`, Restarts incrementing

**Root Causes:** App crashes, missing config, failed probes, insufficient resources

**Troubleshooting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl get pod <pod-name> -o yaml | grep -A 10 resources
```

**Resolution:** Fix application code, add missing env vars, adjust resource limits, fix probes

**Prevention:** Set proper startup probes, validate configs, use init containers

---

## 2. ImagePullBackOff Error

**Symptoms:** `STATUS: ImagePullBackOff`, "pull access denied" error

**Root Causes:** Wrong credentials, image doesn't exist, network issues, rate limiting

**Troubleshooting:**
```bash
kubectl describe pod <pod-name>
kubectl get secret <pull-secret> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
kubectl get pod <pod-name> -o yaml | grep image
```

**Resolution:**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass \
  -n <namespace>
```

**Prevention:** Use IAM roles, automate credential rotation, specific image tags

---

## 3. Node NotReady Status

**Symptoms:** Node shows NotReady, pods not scheduling

**Root Causes:** Kubelet stopped, disk/memory pressure, network issues, runtime failure

**Troubleshooting:**
```bash
kubectl describe node <node-name>
ssh <node> "sudo systemctl status kubelet"
ssh <node> "df -h && free -m"
```

**Resolution:**
```bash
sudo systemctl restart kubelet
sudo crictl rmi --prune
kubectl uncordon <node-name>
```

**Prevention:** Monitor resources, configure garbage collection, use node problem detector

---

## 4. DNS Resolution Failures

**Symptoms:** `nslookup` fails, "could not resolve host" errors

**Root Causes:** CoreDNS pods down, DNS service misconfigured, network policy blocking port 53

**Troubleshooting:**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl run test --rm -it --image=busybox:1.28 -- nslookup kubernetes.default
```

**Resolution:**
```bash
kubectl rollout restart deployment coredns -n kube-system
# Fix CoreDNS ConfigMap if corrupted
kubectl edit configmap coredns -n kube-system
```

**Prevention:** Monitor CoreDNS, use NodeLocal DNSCache, allow DNS in network policies

---

## 5. PersistentVolumeClaim Pending

**Symptoms:** `STATUS: Pending`, "no persistent volumes available" event

**Root Causes:** No matching PV, StorageClass issues, insufficient quota, CSI driver problems

**Troubleshooting:**
```bash
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl get storageclass
kubectl logs -n kube-system <csi-provisioner-pod>
```

**Resolution:**
```bash
# Fix StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

**Prevention:** Use CSI drivers, set volumeBindingMode: WaitForFirstConsumer, monitor quotas

---

## 6. Service Not Accessible

**Symptoms:** Connection refused/timeout, 503 errors

**Root Causes:** Selector mismatch, no endpoints, wrong ports, network policies

**Troubleshooting:**
```bash
kubectl get svc <service-name>
kubectl get endpoints <service-name>
kubectl get pods -l <selector> --show-labels
kubectl describe svc <service-name>
```

**Resolution:**
```bash
# Fix selector to match pod labels
kubectl edit svc <service-name>
# Test connectivity
kubectl run test --rm -it --image=nicolaka/netshoot -- curl http://<service>:8080
```

**Prevention:** Consistent labeling, readiness probes, endpoint monitoring

---

## 7. High CPU/Memory Usage on Nodes

**Symptoms:** Node at 95%+ CPU/Memory, pods evicted

**Root Causes:** No resource limits, memory leaks, too many pods, insufficient sizing

**Troubleshooting:**
```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl describe node <node> | grep -A 20 "Allocated resources"
```

**Resolution:**
```bash
# Add resource limits
kubectl set resources deployment <name> --limits=cpu=1,memory=1Gi --requests=cpu=500m,memory=512Mi
# Create LimitRange
kubectl apply -f limitrange.yaml
```

**Prevention:** Always set requests/limits, use VPA, implement quotas, HPA

---

## 8. Ingress Controller 502/504 Errors

**Symptoms:** HTTP 502 Bad Gateway, 504 Gateway Timeout

**Root Causes:** Backend pods not ready, port mismatch, timeout too short, controller overloaded

**Troubleshooting:**
```bash
kubectl describe ingress <ingress-name>
kubectl get endpoints <backend-service>
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

**Resolution:**
```bash
# Fix service port in ingress
kubectl edit ingress <name>
# Add timeout annotations
kubectl annotate ingress <name> nginx.ingress.kubernetes.io/proxy-read-timeout="600"
```

**Prevention:** Validate configs, proper readiness probes, monitor ingress metrics

---

## 9. etcd Cluster Unhealthy

**Symptoms:** API server slow/unresponsive, "etcdserver: request timed out"

**Root Causes:** etcd member down, high disk latency, database too large, NOSPACE alarm

**Troubleshooting:**
```bash
ETCDCTL_API=3 etcdctl endpoint health --cluster \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Resolution:**
```bash
# Compact and defragment
ETCDCTL_API=3 etcdctl compact <revision>
ETCDCTL_API=3 etcdctl defrag --cluster
ETCDCTL_API=3 etcdctl alarm disarm
```

**Prevention:** Monitor DB size, auto-compaction, SSD storage, regular backups

---

## 10. Certificate Expiration Issues

**Symptoms:** "x509: certificate has expired", authentication failures

**Root Causes:** Certificates expired, not renewed, validity too short

**Troubleshooting:**
```bash
sudo kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text
```

**Resolution:**
```bash
sudo kubeadm certs renew all
sudo systemctl restart kubelet
# Update kubeconfig
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```

**Prevention:** Monitor expiration dates, automate renewal, alerts at 90/60/30 days

---

## 11. RBAC Permission Denied

**Symptoms:** "Forbidden: User cannot list resource", 403 errors

**Root Causes:** Missing Role/RoleBinding, wrong permissions, service account not configured

**Troubleshooting:**
```bash
kubectl auth can-i list pods --as=user@example.com
kubectl get rolebindings -n <namespace>
kubectl describe rolebinding <binding-name>
```

**Resolution:**
```bash
kubectl create role developer --verb=get,list --resource=pods
kubectl create rolebinding dev-binding --role=developer --user=user@example.com
```

**Prevention:** Least privilege, document roles, test in CI/CD, use groups

---

## 12. Pod Eviction Due to Resource Pressure

**Symptoms:** Pods evicted, MemoryPressure/DiskPressure on node

**Root Causes:** No resource requests, node out of memory/disk, too many pods

**Troubleshooting:**
```bash
kubectl get pods -A | grep Evicted
kubectl describe node <node> | grep -A 10 Conditions
kubectl top node <node>
```

**Resolution:**
```bash
# Clean up
kubectl delete pods --field-selector=status.phase=Failed
ssh <node> "sudo crictl rmi --prune"
# Add resources to deployment
kubectl set resources deployment <name> --requests=memory=256Mi
```

**Prevention:** Set requests/limits, LimitRanges, PodDisruptionBudgets, monitor resources

---

## 13. CNI Plugin Failure

**Symptoms:** Pods stuck in ContainerCreating, "failed to setup network" error

**Root Causes:** CNI pods crashed, binaries missing, CIDR conflicts, IP pool exhausted

**Troubleshooting:**
```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl logs -n kube-system <cni-pod>
ssh <node> "ls /opt/cni/bin && cat /etc/cni/net.d/*"
```

**Resolution:**
```bash
# Restart CNI
kubectl rollout restart daemonset calico-node -n kube-system
# Fix IP pool
kubectl apply -f ippool.yaml
```

**Prevention:** Monitor CNI health, correct CIDR planning, test network policies

---

## 14. Horizontal Pod Autoscaler Not Scaling

**Symptoms:** HPA shows `<unknown>` or not scaling despite high CPU

**Root Causes:** Metrics server down, no resource requests, custom metrics unavailable

**Troubleshooting:**
```bash
kubectl get hpa
kubectl describe hpa <hpa-name>
kubectl top pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
```

**Resolution:**
```bash
# Deploy metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# Add resource requests
kubectl set resources deployment <name> --requests=cpu=100m
```

**Prevention:** Ensure metrics-server running, always set requests, test HPA

---

## 15. StatefulSet Pod Stuck in Pending

**Symptoms:** StatefulSet pod 1+ stuck Pending, previous pod not ready

**Root Causes:** PVC not bound, anti-affinity too restrictive, insufficient resources

**Troubleshooting:**
```bash
kubectl get statefulset <name>
kubectl describe pod <pod-name>
kubectl get pvc -l app=<statefulset>
```

**Resolution:**
```bash
# Use WaitForFirstConsumer
kubectl patch storageclass <name> -p '{"volumeBindingMode":"WaitForFirstConsumer"}'
# Change to Parallel if order doesn't matter
kubectl patch statefulset <name> -p '{"spec":{"podManagementPolicy":"Parallel"}}'
```

**Prevention:** WaitForFirstConsumer mode, realistic affinity rules, test scaling

---

## 16. API Server Unresponsive

**Symptoms:** `Unable to connect to server`, timeouts, all kubectl commands fail

**Root Causes:** API server crashed, etcd unhealthy, overloaded, certificates expired

**Troubleshooting:**
```bash
# On master node
curl -k https://localhost:6443/healthz
sudo crictl ps | grep kube-apiserver
sudo crictl logs <apiserver-container>
```

**Resolution:**
```bash
# Restart API server (move manifest temporarily)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 20
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

**Prevention:** Monitor API latency, configure rate limits, multiple API servers, etcd maintenance

---

## 17. ConfigMap/Secret Not Updating in Pods

**Symptoms:** Application still using old configuration after ConfigMap update

**Root Causes:** Environment variables (never update), volume mount delay, app not reloading

**Troubleshooting:**
```bash
kubectl get configmap <name> -o yaml
kubectl exec <pod> -- cat /etc/config/app.conf
kubectl get pod <pod> -o yaml | grep -A 20 volumes
```

**Resolution:**
```bash
# Option 1: Rollout restart
kubectl rollout restart deployment <name>

# Option 2: Use Reloader
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
kubectl annotate deployment <name> configmap.reloader.stakater.com/reload="configmap-name"
```

**Prevention:** Use volume mounts (not env), implement hot reload, use Reloader, version ConfigMaps

---

## 18. Network Policy Blocking Traffic

**Symptoms:** Connection timeouts, "no route to host" between pods

**Root Causes:** Default deny policy, missing ingress/egress rules, DNS blocked

**Troubleshooting:**
```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>
kubectl run test --rm -it --image=nicolaka/netshoot -- curl <service>
```

**Resolution:**
```bash
# Allow DNS
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

**Prevention:** Always allow DNS, test policies, use visualization tools, gradual rollout

---

## 19. Cluster Upgrade Failure

**Symptoms:** Mixed versions, API server won't start, upgrade command fails

**Root Causes:** Version skew >1 minor, deprecated APIs, etcd issues, disk space

**Troubleshooting:**
```bash
kubectl version --short
kubectl get nodes -o wide
sudo kubeadm upgrade plan
kubent  # Check deprecated APIs
```

**Resolution:**
```bash
# Fix deprecated APIs first
kubectl convert -f old-ingress.yaml --output-version networking.k8s.io/v1

# Upgrade control plane
sudo kubeadm upgrade apply v1.28.1

# Upgrade kubelet
sudo apt-get install -y kubelet=1.28.1-00 kubectl=1.28.1-00
sudo systemctl restart kubelet
```

**Prevention:** Backup etcd, test in staging, check deprecations (kubent), one version at a time

---

## 20. Persistent Volume Data Loss

**Symptoms:** Data missing from volumes, empty directories, database errors

**Root Causes:** PV deleted (Delete reclaim policy), storage backend failure, PVC deleted

**Troubleshooting:**
```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pv <pv-name>
aws ec2 describe-volumes --volume-ids <vol-id>  # Check if exists
```

**Resolution:**
```bash
# If volume exists but PV deleted, recreate PV
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: recovered-pv
spec:
  capacity:
    storage: 100Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-xxxxx
    fsType: ext4
EOF

# Restore from backup (Velero)
velero restore create --from-backup backup-name
```

**Prevention:** **ALWAYS use Retain policy**, regular backups (Velero), test restores, immutable backups

---

## 🔧 Quick Commands Reference

```bash
# Health Check
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
kubectl top nodes

# Debugging
kubectl describe <resource> <name>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
kubectl get events --sort-by='.lastTimestamp'

# Network Testing
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Resource Info
kubectl api-resources
kubectl explain <resource>
kubectl get <resource> -o yaml
```

## 📊 Monitoring Alerts (Prometheus)

```yaml
groups:
- name: kubernetes
  rules:
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  - alert: PVCPending
    expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
```

## 🛠️ Essential Tools

- **kubectl** - Kubernetes CLI
- **k9s** - Terminal UI
- **stern** - Multi-pod log tailing
- **kubectx/kubens** - Context/namespace switching
- **kubent** - Check deprecated APIs
- **Lens** - Kubernetes IDE
- **Prometheus/Grafana** - Monitoring
- **Velero** - Backup/restore

---

**Kubernetes Versions:** 1.26 - 1.29  
**Last Updated:** February 2026  
**Author:** Senior K8s Administrator (10+ years production experience)

---

## 📚 Resources

- [Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Troubleshooting Guide](https://kubernetes.io/docs/tasks/debug/)
