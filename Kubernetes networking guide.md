# Kubernetes Services & Networking

**Author:** Sajal Jana

A comprehensive guide to Kubernetes networking with practical examples, YAML configurations, and production best practices.

---

## Table of Contents

1. [Connectivity - Pod & Node Communication](#1-connectivity---pod--node-communication)
   - [CNI Basics](#cni-basics)
   - [Pod-to-Pod Communication](#pod-to-pod-communication)
   - [Node-to-Node Communication](#node-to-node-communication)
   - [Common Pitfalls](#connectivity-pitfalls)

2. [Service Types](#2-service-types)
   - [ClusterIP](#clusterip)
   - [NodePort](#nodeport)
   - [LoadBalancer](#loadbalancer)
   - [Service Comparison](#service-comparison-table)
   - [Common Pitfalls](#service-pitfalls)

3. [Ingress & Gateway](#3-ingress--gateway)
   - [Ingress Controllers](#ingress-controllers)
   - [Ingress Resources](#ingress-resources)
   - [Gateway API](#gateway-api)
   - [When to Use Which](#when-to-use-ingress-vs-gateway-api)
   - [Common Pitfalls](#ingress-pitfalls)

4. [Network Security](#4-network-security)
   - [NetworkPolicy Basics](#networkpolicy-basics)
   - [Real-World Scenarios](#real-world-scenarios)
   - [Common Pitfalls](#network-security-pitfalls)

5. [DNS & Service Discovery](#5-dns--service-discovery)
   - [CoreDNS Architecture](#coredns-architecture)
   - [DNS Resolution Patterns](#dns-resolution-patterns)
   - [Troubleshooting DNS](#troubleshooting-dns)
   - [Common Pitfalls](#dns-pitfalls)

6. [Quick Reference](#quick-reference)

---

## 1. Connectivity - Pod & Node Communication

### CNI Basics

The Container Network Interface (CNI) is responsible for allocating IP addresses to pods and establishing connectivity between them.

**Popular CNI Plugins:**
- **Calico**: Feature-rich, supports NetworkPolicies
- **Flannel**: Simple overlay network
- **Cilium**: eBPF-based, high performance
- **Weave Net**: Easy setup, encryption support

**How CNI Works:**

```
┌─────────────────────────────────────────────┐
│              Kubernetes API                  │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│               kubelet                        │
│  Calls CNI plugin when pod is created       │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│            CNI Plugin                        │
│  - Allocates IP address                     │
│  - Sets up network namespace                │
│  - Configures routes                        │
└─────────────────────────────────────────────┘
```

### Pod-to-Pod Communication

Kubernetes enforces the following networking requirements:
1. All pods can communicate with each other without NAT
2. All nodes can communicate with all pods without NAT
3. The IP a pod sees itself as is the same IP others see it as

**Communication Flow:**

```
Pod A (10.244.1.5)                      Pod B (10.244.2.10)
   │                                        │
   │  1. Packet to 10.244.2.10             │
   ▼                                        │
veth0 (bridge)                              │
   │                                        │
   │  2. Routes through Node A              │
   ▼                                        │
Node A eth0                                 │
   │                                        │
   │  3. Physical network                   │
   │  ────────────────────────────────────► │
   │                                        ▼
   │                                   Node B eth0
   │                                        │
   │                                        │  4. Routes to Pod B
   │                                        ▼
   │                                   veth0 (bridge)
   │                                        │
   │                                        ▼
   │                                    Pod B receives packet
```

### Node-to-Node Communication

Nodes communicate over the cluster network using their physical network interfaces. The CNI plugin handles routing pod traffic between nodes.

**Example with Calico BGP:**

```yaml
# View Calico node status
kubectl get nodes -o wide
kubectl get pods -n kube-system -l k8s-app=calico-node

# Check BGP peers
kubectl exec -n kube-system calico-node-xxxxx -- calicoctl node status
```

### Connectivity Pitfalls

❌ **MTU Mismatch**: Ensure MTU is consistent across the cluster
```bash
# Check MTU
ip link show | grep mtu

# Common issue: Cloud providers may use MTU of 1460 instead of 1500
```

❌ **IP Pool Exhaustion**: Monitor IPAM to prevent IP shortage
```bash
# For Calico
kubectl get ippools -o yaml
```

❌ **Firewall Rules**: Ensure node firewalls allow pod CIDR traffic
```bash
# Example: Allow pod CIDR (adjust for your setup)
iptables -A FORWARD -s 10.244.0.0/16 -j ACCEPT
iptables -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```

---

## 2. Service Types

Services provide stable endpoints for accessing pods. They abstract pod IP addresses and provide load balancing.

### ClusterIP

**Default service type**. Accessible only within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    protocol: TCP
    port: 80        # Service port
    targetPort: 8080 # Container port
  sessionAffinity: ClientIP  # Optional: sticky sessions
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

**Use Cases:**
- Internal microservice communication
- Database services
- Cache layers

**Access Pattern:**
```bash
# From within cluster
curl http://backend-service.production.svc.cluster.local
curl http://backend-service.production
curl http://backend-service  # if in same namespace
```

### NodePort

Exposes service on each node's IP at a static port (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
    nodePort: 30080  # Optional: auto-assigned if omitted
```

**Access Pattern:**
```bash
# From outside cluster
curl http://<any-node-ip>:30080

# Get node IPs
kubectl get nodes -o wide
```

**Use Cases:**
- Development/testing environments
- Quick external access without load balancer
- On-premises clusters without cloud LB

### LoadBalancer

Provisions an external load balancer (cloud provider specific).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    # Cloud-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  externalTrafficPolicy: Local  # Preserve source IP
  loadBalancerSourceRanges:     # Restrict access
  - 10.0.0.0/8
  - 192.168.0.0/16
```

**Use Cases:**
- Production applications needing external access
- When you need cloud provider's native load balancer features
- Applications requiring static external IP

**Check Status:**
```bash
kubectl get svc web-loadbalancer
# Look for EXTERNAL-IP field

# View events
kubectl describe svc web-loadbalancer
```

### Service Comparison Table

| Feature | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Accessibility** | Internal only | External via NodeIP:Port | External via cloud LB |
| **Port Range** | Any | 30000-32767 | Any |
| **Load Balancing** | Internal only | Manual/external | Cloud provider |
| **Cost** | Free | Free | Paid (cloud LB) |
| **Production Use** | Yes (internal) | Limited | Yes (external) |
| **Source IP Preservation** | Yes | With externalTrafficPolicy: Local | With externalTrafficPolicy: Local |

### Service Pitfalls

❌ **No Endpoint Pods**: Service won't work if selector doesn't match any pods
```bash
# Debug
kubectl get endpoints web-loadbalancer
# Should show pod IPs; if empty, check selectors

kubectl get pods -l app=web --show-labels
```

❌ **Wrong Target Port**: Ensure targetPort matches container's listening port
```bash
# Check container port
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].ports}'
```

❌ **NodePort Firewall**: Node firewalls must allow NodePort range
```bash
# Check if port is accessible
nc -zv <node-ip> 30080
```

❌ **externalTrafficPolicy: Local Issues**:
- Pros: Preserves source IP, no extra hop
- Cons: Imbalanced traffic if pods aren't evenly distributed

```yaml
# Health check issues with Local policy
# Ensure readiness probes are properly configured
spec:
  externalTrafficPolicy: Local
  healthCheckNodePort: 32000  # Auto-assigned if not specified
```

---

## 3. Ingress & Gateway

### Ingress Controllers

Popular ingress controllers:
- **NGINX Ingress**: Most widely used
- **Traefik**: Modern, automatic HTTPS
- **HAProxy**: High performance
- **Istio Gateway**: Service mesh integration
- **Contour**: Envoy-based

**Install NGINX Ingress Controller:**

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Ingress Resources

**Basic Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Path Types:**
- `Exact`: Matches exact path only
- `Prefix`: Matches path prefix (most common)
- `ImplementationSpecific`: Depends on IngressClass

**Advanced: Multiple Hosts with SSL:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - admin.example.com
    secretName: wildcard-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

**Common NGINX Annotations:**

```yaml
annotations:
  # Rate limiting
  nginx.ingress.kubernetes.io/rate-limit: "100"
  
  # CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
  
  # Request/Response modifications
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  
  # Authentication
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
  
  # Backend protocol
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  
  # Sticky sessions
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "route"
```

### Gateway API

The Gateway API is the next-generation Ingress API, offering more expressiveness and role-oriented design.

**Install Gateway API CRDs:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

**Gateway Resource:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: gateway-system
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: tls-secret
    allowedRoutes:
      namespaces:
        from: All
```

**HTTPRoute Resource:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: production
spec:
  parentRefs:
  - name: production-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-service
      port: 80
      weight: 90
    - name: api-v1-canary-service
      port: 80
      weight: 10  # Traffic split for canary
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2-service
      port: 80
```

**Advanced: Header-based Routing:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-routing
spec:
  parentRefs:
  - name: production-gateway
  rules:
  - matches:
    - headers:
      - name: X-Beta-User
        value: "true"
    backendRefs:
    - name: beta-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: stable-service
      port: 80
```

### When to Use Ingress vs Gateway API

**Use Ingress when:**
- Simple HTTP/HTTPS routing needs
- Existing setup with Ingress controllers
- Team is familiar with Ingress
- Quick setup required

**Use Gateway API when:**
- Complex traffic routing (header-based, weighted)
- Multiple teams managing different routes
- Advanced features like traffic splitting
- TCP/UDP/TLS routing needed
- Future-proofing your infrastructure

**Migration Path:**
Many Ingress controllers (NGINX, Traefik) support both APIs, allowing gradual migration.

### Ingress Pitfalls

❌ **Missing IngressClass**: Ingress won't work without proper class
```bash
# List available classes
kubectl get ingressclass

# Verify ingress
kubectl describe ingress web-ingress
```

❌ **DNS Not Configured**: Ensure DNS points to ingress controller's external IP
```bash
# Get ingress address
kubectl get ingress web-ingress

# Test DNS
nslookup app.example.com
```

❌ **TLS Secret Issues**: Wrong namespace or missing secret
```bash
# Secret must be in same namespace as Ingress
kubectl get secret app-tls-secret -n production

# Create TLS secret
kubectl create secret tls app-tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/key.key \
  -n production
```

❌ **Backend Service Mismatch**: Service name/port must match exactly
```bash
# Verify service exists
kubectl get svc -n production

# Check ingress backend configuration
kubectl get ingress web-ingress -o yaml
```

---

## 4. Network Security

NetworkPolicies define how pods can communicate with each other and external endpoints.

### NetworkPolicy Basics

**Default Behavior:**
- Without NetworkPolicies: All traffic allowed
- With NetworkPolicies: Default deny + explicit allow

**Important**: NetworkPolicies require a CNI plugin that supports them (Calico, Cilium, Weave).

### Real-World Scenarios

#### Scenario 1: Deny All Traffic (Baseline Security)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

#### Scenario 2: Allow Frontend → Backend Communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### Scenario 3: Three-Tier Application

```yaml
---
# Frontend: Allow ingress from anywhere on port 80
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx  # From ingress controller
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:  # DNS resolution
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Backend: Allow only from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Database: Allow only from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:  # Database typically doesn't need egress except DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

#### Scenario 4: Allow External API Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}  # DNS
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:  # Specific external API
    - ipBlock:
        cidr: 203.0.113.0/24  # Replace with actual API CIDR
    ports:
    - protocol: TCP
      port: 443
  - to:  # Fallback for dynamic IPs
    ports:
    - protocol: TCP
      port: 443
```

#### Scenario 5: Namespace Isolation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          environment: production
  - to:  # DNS and external
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Network Security Pitfalls

❌ **Forgetting DNS Egress**: Pods need DNS access to resolve service names
```yaml
# Always include DNS egress rule
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  - podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
```

❌ **CNI Not Supporting NetworkPolicies**: Verify your CNI supports them
```bash
# Test with a simple policy
kubectl apply -f test-networkpolicy.yaml
kubectl describe networkpolicy test-networkpolicy

# Check CNI documentation
```

❌ **Label Selector Mistakes**: Wrong labels = no effect
```bash
# Verify pod labels
kubectl get pods --show-labels -n production

# Test connectivity
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://backend-service:8080
```

❌ **Overlapping Policies**: Multiple policies = union of allowed traffic
```bash
# List all policies affecting a pod
kubectl get networkpolicies -n production
kubectl describe networkpolicy -n production
```

**Testing NetworkPolicies:**

```bash
# Deploy a test pod
kubectl run test-$RANDOM --image=busybox -it --rm -- sh

# Inside the pod, test connectivity
wget -O- --timeout=2 http://service-name:port

# Test DNS
nslookup service-name
```

---

## 5. DNS & Service Discovery

### CoreDNS Architecture

CoreDNS is the default DNS server in Kubernetes (replaced kube-dns).

```
┌─────────────────────────────────────────────┐
│                  Pod                         │
│  Queries: backend.production.svc.cluster.local│
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│          /etc/resolv.conf                    │
│  nameserver 10.96.0.10 (ClusterIP)          │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│          CoreDNS Service                     │
│  ClusterIP: 10.96.0.10                      │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│          CoreDNS Pods                        │
│  - Resolves cluster.local queries          │
│  - Forwards external queries to upstream    │
│  - Caches responses                         │
└─────────────────────────────────────────────┘
```

**Check CoreDNS Status:**

```bash
# View CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS service
kubectl get svc -n kube-system kube-dns

# Check CoreDNS configmap
kubectl get configmap coredns -n kube-system -o yaml
```

### DNS Resolution Patterns

**Service DNS Format:**

```
<service-name>.<namespace>.svc.cluster.local
```

**Resolution Examples:**

```bash
# Same namespace
curl http://backend

# Different namespace
curl http://backend.production

# Fully qualified (always works)
curl http://backend.production.svc.cluster.local

# External DNS (forwarded to upstream)
curl http://google.com
```

**Pod DNS:**

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local

# Example: Pod IP 10.244.1.5 in namespace 'default'
# DNS: 10-244-1-5.default.pod.cluster.local
```

**Headless Service DNS:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: database
  ports:
  - port: 5432
```

DNS returns individual pod IPs instead of service VIP:
```bash
# Returns all pod IPs
nslookup database-headless.production.svc.cluster.local

# Individual pod DNS
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

### Troubleshooting DNS

**Test DNS Resolution:**

```bash
# Deploy debug pod
kubectl run dnsutils --image=gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 \
  --restart=Never -it --rm

# Inside the pod
nslookup kubernetes.default
nslookup backend.production
nslookup google.com

# Check /etc/resolv.conf
cat /etc/resolv.conf
```

**Common DNS Issues:**

#### Issue 1: Service Not Resolving

```bash
# Check if service exists
kubectl get svc -n production backend

# Check endpoints
kubectl get endpoints -n production backend

# Verify DNS service is running
kubectl get svc -n kube-system kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

#### Issue 2: Slow DNS Resolution

```yaml
# Edit CoreDNS config to increase cache
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30  # Increased cache time
        loop
        reload
        loadbalance
    }
```

#### Issue 3: External DNS Not Working

```bash
# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test upstream DNS
kubectl exec -n kube-system -it coredns-xxxxx -- nslookup google.com

# Check forward configuration in CoreDNS ConfigMap
kubectl get cm coredns -n kube-system -o yaml | grep forward
```

#### Issue 4: Pod DNS Resolution Failures

```yaml
# Check pod's dnsPolicy
kubectl get pod <pod-name> -o jsonpath='{.spec.dnsPolicy}'

# DNSPolicy options:
# - ClusterFirst (default): Use cluster DNS
# - Default: Use node's DNS
# - ClusterFirstWithHostNet: For hostNetwork pods
# - None: Custom DNS config
```

**Custom DNS Configuration:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  containers:
  - name: app
    image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 1.1.1.1
    - 8.8.8.8
    searches:
    - production.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "5"
```

### DNS Pitfalls

❌ **ndots Too High**: Can cause multiple DNS queries
```yaml
# Default ndots is 5, which means:
# Query "backend" becomes:
# 1. backend.default.svc.cluster.local
# 2. backend.svc.cluster.local
# 3. backend.cluster.local
# 4. backend (external)

# Reduce for external domains
dnsConfig:
  options:
  - name: ndots
    value: "2"
```

❌ **CoreDNS Pod Issues**: Check resource limits
```bash
# CoreDNS may be OOMKilled
kubectl describe pod -n kube-system coredns-xxxxx

# Increase resources if needed
kubectl edit deployment coredns -n kube-system
```

❌ **Network Policy Blocking DNS**: Ensure egress to kube-dns
```yaml
# Add to NetworkPolicy
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

❌ **Stale DNS Cache**: Restart CoreDNS if cache is stale
```bash
kubectl rollout restart deployment coredns -n kube-system
```

---

## Quick Reference

### Essential Commands

```bash
# Services
kubectl get svc -A
kubectl describe svc <service-name>
kubectl get endpoints <service-name>

# Ingress
kubectl get ingress -A
kubectl describe ingress <ingress-name>
kubectl get ingressclass

# NetworkPolicies
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy-name>

# DNS
kubectl get svc -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Connectivity Testing
kubectl run test-pod --image=busybox -it --rm -- sh
kubectl run curl --image=curlimages/curl -it --rm -- sh

# Port Forward (for debugging)
kubectl port-forward svc/<service-name> 8080:80
kubectl port-forward pod/<pod-name> 8080:8080

# Logs
kubectl logs -f <pod-name>
kubectl logs -f <pod-name> -c <container-name>
```

### Debug Checklist

When services don't work:
1. ✓ Pods are running: `kubectl get pods`
2. ✓ Service selector matches pods: `kubectl get pods --show-labels`
3. ✓ Endpoints exist: `kubectl get endpoints <service-name>`
4. ✓ DNS resolves: `nslookup <service-name>`
5. ✓ Network policies allow traffic
6. ✓ Container port matches targetPort
7. ✓ Readiness probes passing

### Port Reference

| Service | Default Port | Purpose |
|---------|-------------|----------|
| HTTP | 80 | Web traffic |
| HTTPS | 443 | Secure web traffic |
| PostgreSQL | 5432 | Database |
| MySQL | 3306 | Database |
| MongoDB | 27017 | Database |
| Redis | 6379 | Cache |
| Kafka | 9092 | Message broker |
| Elasticsearch | 9200 | Search engine |

### Common CIDR Blocks

| Block | Description |
|-------|-------------|
| 10.0.0.0/8 | Private Class A |
| 172.16.0.0/12 | Private Class B |
| 192.168.0.0/16 | Private Class C |
| 10.244.0.0/16 | Common pod CIDR (Flannel) |
| 10.96.0.0/12 | Common service CIDR |

---

## Additional Resources

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/services-networking/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [CoreDNS Documentation](https://coredns.io/manual/toc/)
- [Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)

---

**Author:** Sajal Jana  
**Last Updated:** February 2026  
**License:** MIT

Feel free to contribute or report issues!
