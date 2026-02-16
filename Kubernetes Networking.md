# Kubernetes Networking: The Fun Guide 🎢

**Author:** Sajal Jana  
**Disclaimer:** Yes, networking is scary. No, you're not alone. Yes, we'll get through this together.

---

## Table of Contents

1. [Pod & Node Communication](#1-pod--node-communication)
2. [Service Types](#2-service-types)
3. [Ingress & Gateway API](#3-ingress--gateway-api)
4. [Network Security](#4-network-security)
5. [DNS & Service Discovery](#5-dns--service-discovery)
6. [Quick Reference](#quick-reference)

---

## 1. Pod & Node Communication

### CNI: The Matchmaker 💘

CNI (Container Network Interface) is basically Tinder for pods. It gives them IPs and helps them find each other.

**Popular CNIs:**
- **Calico**: The overachiever (does everything)
- **Flannel**: The simple friend (just works)
- **Cilium**: The show-off (eBPF is its personality)
- **Weave**: The security-conscious one (encryption included)

### Kubernetes Networking Rules (Non-Negotiable)

1. All pods talk to each other without NAT (no middle-person drama)
2. Nodes talk to pods without NAT (direct communication only)
3. A pod's IP is the same everywhere (no identity crisis)

**Fun fact:** If your pods can't talk to each other, it's probably MTU. It's always MTU. 🤷

### Common Pitfalls That'll Ruin Your Day

❌ **MTU Mismatch**: Like trying to fit a couch through a door
```bash
# Check MTU - it should be consistent
ip link show | grep mtu
# Cloud providers love MTU 1460. Your cluster? Probably 1500. Chaos ensues.
```

❌ **IP Pool Exhaustion**: Running out of IPs is like running out of plates at a buffet
```bash
kubectl get ippools -o yaml  # For Calico users
```

❌ **Firewall Rules**: "No one shall pass!" - Your firewall, probably
```bash
# Make friends with iptables
iptables -A FORWARD -s 10.244.0.0/16 -j ACCEPT
```

---

## 2. Service Types

Services are like receptionists - they know where everyone is and forward your requests accordingly.

### ClusterIP (The Introvert)

**Personality:** Stays inside, never goes out, only talks to cluster friends.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: ClientIP  # Sticky sessions (clingy mode ON)
```

**When to use:** Databases, internal APIs, that microservice nobody outside should know about

### NodePort (The Extrovert)

**Personality:** "HEY EVERYONE! I'm on port 30080 of EVERY NODE! Come say hi!"

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
  - port: 80
    targetPort: 3000
    nodePort: 30080  # Between 30000-32767 (the weird kid port range)
```

**When to use:** Testing, demos, when you're too lazy to set up a LoadBalancer

**Access:** `http://<any-node-ip>:30080` (yes, ANY node - they're all gossiping)

### LoadBalancer (The Popular Kid)

**Personality:** Has money (cloud provider's), gets a real external IP, doesn't share.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  externalTrafficPolicy: Local  # "Don't bounce me around!"
```

**When to use:** Production, when you need that sweet external IP

**Cost:** $$$$ (your cloud bill will remember)

### Quick Comparison (Because You're Impatient)

| Type | Accessibility | Cost | Your Boss Approves For |
|------|--------------|------|----------------------|
| ClusterIP | Internal only | Free | Internal stuff |
| NodePort | External (port 30000+) | Free | Dev/testing |
| LoadBalancer | External (nice IP) | 💸💸💸 | Production |

### Common Mistakes (We've All Been There)

❌ **No Endpoints = No Service**
```bash
kubectl get endpoints my-service
# Empty? Your selector is lying. Check those labels!
```

❌ **Wrong Target Port**: Container listening on 8080, you set 80 → 💥
```bash
# Investigate the crime scene
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].ports}'
```

❌ **externalTrafficPolicy: Local** gotchas:
- ✅ Preserves source IP (security team happy)
- ❌ Unbalanced traffic if pods aren't evenly distributed (ops team sad)

---

## 3. Ingress & Gateway API

### Ingress: The Smart Router 🧠

Think of Ingress as a bouncer with a clipboard. It checks the domain name and path, then points you to the right service.

**Popular Bouncers (Ingress Controllers):**
- **NGINX**: The veteran (everyone knows them)
- **Traefik**: The modern hipster (automatic HTTPS!)
- **HAProxy**: The performance junkie
- **Istio Gateway**: The overqualified one (brings the whole service mesh)

### Basic Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret  # Cybersecurity enters the chat
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

**Path Types Explained:**
- `Exact`: "/api" only (picky)
- `Prefix`: "/api/anything" (chill)
- `ImplementationSpecific`: "I do what I want" (controller-dependent)

### Useful NGINX Annotations (The Power-Ups)

```yaml
annotations:
  # Rate limiting (calm down, buddy)
  nginx.ingress.kubernetes.io/rate-limit: "100"
  
  # CORS (let the frontend devs be happy)
  nginx.ingress.kubernetes.io/enable-cors: "true"
  
  # Upload limits (no, you can't upload your entire hard drive)
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  
  # Sticky sessions (clingy customers)
  nginx.ingress.kubernetes.io/affinity: "cookie"
```

### Gateway API: Ingress's Cooler Younger Sibling

More expressive, more flexible, more... everything. It's like Ingress went to college.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: production-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-service
      weight: 90      # 90% of traffic
    - name: api-v1-canary
      weight: 10      # 10% to the canary (risky business)
```

**Traffic splitting!** Because YOLO deployments are so 2015.

### Common Ingress Fails

❌ **Missing IngressClass**: Like showing up to a party without an invitation
```bash
kubectl get ingressclass  # Do you even exist?
```

❌ **DNS Not Pointing to Ingress**: Your domain points to... nothing
```bash
nslookup app.example.com  # Should point to ingress external IP
```

❌ **TLS Secret in Wrong Namespace**: Secrets don't teleport between namespaces
```bash
# Secret MUST be in same namespace as Ingress
kubectl get secret app-tls-secret -n production
```

---

## 4. Network Security

### NetworkPolicies: The Firewall Rules That Actually Make Sense

**Default behavior:** Everything's a party! 🎉  
**With NetworkPolicies:** Bouncer at the door. "You're not on the list." 🚫

**IMPORTANT:** Your CNI needs to support this (Calico, Cilium, Weave say yes; Flannel says... maybe?)

### Real-World Scenarios

#### Deny All (The Paranoid Approach)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # ALL pods: "No friends allowed!"
  policyTypes:
  - Ingress
  - Egress
```

Start here. Trust no one. Add exceptions gradually.

#### Three-Tier App (The Proper Way)

**Frontend** → Can talk to backend only  
**Backend** → Can talk to database only  
**Database** → Talks to no one (antisocial)

```yaml
# Frontend policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:  # DNS (don't forget this or nothing works!)
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Pro tip:** ALWAYS include DNS egress. I repeat: ALWAYS. Or pods can't resolve service names and you'll be debugging for hours. 🤦

### Common NetworkPolicy Facepalms

❌ **Forgot DNS Egress**: Pods can't resolve names → Everything breaks → You cry
```yaml
# The magic incantation (include EVERYWHERE)
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

❌ **Wrong Labels**: Policy doesn't apply → You think it's secure → It's not
```bash
# Trust but verify
kubectl get pods --show-labels
```

❌ **CNI Doesn't Support NetworkPolicies**: You're yelling into the void
```bash
# Check if it's actually working
kubectl describe networkpolicy <name>
```

---

## 5. DNS & Service Discovery

### CoreDNS: The Phone Book 📞

Every pod queries CoreDNS to find services. It's like asking "Hey, where's the backend service?" and CoreDNS says "Oh yeah, it's at 10.96.5.123!"

### DNS Format (The Secret Handshake)

```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
```bash
# Lazy (same namespace)
curl http://backend

# Less lazy (different namespace)
curl http://backend.production

# Paranoid (fully qualified)
curl http://backend.production.svc.cluster.local
```

All three work. The first one is just tired of typing.

### Headless Services: When You Need All The IPs

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None  # "I don't need no VIP!"
  selector:
    app: database
```

**Result:** DNS returns ALL pod IPs instead of one service IP. Useful for StatefulSets and distributed databases.

### Troubleshooting DNS (The Debug Dance)

```bash
# Spawn debug pod
kubectl run dnsutils --image=gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 -it --rm

# Inside the pod
nslookup kubernetes.default  # Should work
nslookup backend.production  # Should work
nslookup google.com          # Should work (if internet exists)

# Still broken? Check this
cat /etc/resolv.conf
```

### Common DNS Disasters

❌ **Service Doesn't Resolve**: Does it even exist?
```bash
kubectl get svc -n production backend
kubectl get endpoints -n production backend  # Empty = problem
```

❌ **Slow DNS**: Too many queries because ndots is set too high
```yaml
# Fix: Lower ndots for external domains
dnsConfig:
  options:
  - name: ndots
    value: "2"  # Default is 5 (overkill)
```

**What's ndots?** It's how many dots trigger a "treat this as fully qualified" response. With ndots=5, querying "google.com" tries:
1. google.com.default.svc.cluster.local
2. google.com.svc.cluster.local
3. google.com.cluster.local
4. google.com

See the problem? Yeah.

❌ **NetworkPolicy Blocking DNS**: Classic mistake
```yaml
# Add this to your egress rules or cry
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

---

## Quick Reference

### Essential Commands (Bookmark This)

```bash
# "Is my service alive?"
kubectl get svc -A
kubectl describe svc <name>
kubectl get endpoints <name>  # The truth is here

# "Why isn't my ingress working?"
kubectl get ingress -A
kubectl describe ingress <name>

# "Who can talk to who?"
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>

# "Is DNS broken?"
kubectl get svc -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# "I need to test RIGHT NOW"
kubectl run test --image=busybox -it --rm -- sh
kubectl run curl --image=curlimages/curl -it --rm -- sh

# "Let me see what's happening inside"
kubectl port-forward svc/<name> 8080:80
```

### Debug Checklist (When Everything's On Fire 🔥)

1. ✓ Are pods running? `kubectl get pods`
2. ✓ Do labels match? `kubectl get pods --show-labels`
3. ✓ Endpoints exist? `kubectl get endpoints <svc>`
4. ✓ DNS works? `nslookup <svc>`
5. ✓ NetworkPolicies allowing traffic?
6. ✓ Ports match? (targetPort = container port)
7. ✓ Readiness probes passing?

**If all above check out and it STILL doesn't work:** Take a walk. Pet a dog. The answer will come to you.

### Common Ports (For Your Convenience)

| Service | Port | What It Does |
|---------|------|--------------|
| HTTP | 80 | The internet's favorite |
| HTTPS | 443 | HTTP but with secrets |
| PostgreSQL | 5432 | Database stuff |
| Redis | 6379 | Cache things |
| MongoDB | 27017 | NoSQL chaos |
| Kafka | 9092 | Message madness |

---

## Final Words of Wisdom

1. **DNS issues?** 90% chance you forgot the DNS egress rule
2. **NetworkPolicy not working?** Check if your CNI supports it
3. **Service has no endpoints?** Labels. Always labels.
4. **Ingress returns 404?** DNS probably points to wrong place
5. **Everything broken?** `kubectl get pods -A` - something's not running

**Remember:** Kubernetes networking seems hard because it IS hard. But you got this! 💪

---

**Author:** Sajal Jana  
**Last Updated:** February 2026  
**Therapy Sessions Attended While Writing This:** Too many to count

**Found this helpful?** Star the repo! Found a mistake? Well... that's what PRs are for! 😅

MIT License - Use it, abuse it, just don't blame me when things break.
