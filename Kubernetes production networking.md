# ☸️ Kubernetes Ingress Explained with Flipkart 🛒

> **One Public IP → Hundreds of Services → Zero Confusion**
> 
> A real-world explanation of how Kubernetes Ingress works using Flipkart as an example.

---

## 📌 Table of Contents

- [The Problem — Without Ingress](#-the-problem--without-ingress)
- [The Solution — With Ingress](#-the-solution--with-ingress)
- [Complete Traffic Flow — With Ingress](#-complete-traffic-flow--with-ingress)
- [The Secret — HOST Header Magic](#-the-secret--host-header-magic)
- [Ingress Routing Rule — YAML](#-ingress-routing-rule--yaml)
- [Key Takeaway](#-key-takeaway)

---

## ❌ The Problem — Without Ingress

When you **don't use Ingress**, every service needs its **own LoadBalancer**.
That means a **separate public IP** and a **separate cloud charge** per service.

```
User hits www.flipkart.com
          ↓
    [LoadBalancer 1]  ← Public IP: 103.21.45.10  💸 Pay
          ↓
   frontend-service
          ↓
       [Pods]

──────────────────────────────────────────────

User hits payment.flipkart.com
          ↓
    [LoadBalancer 2]  ← Public IP: 103.21.45.11  💸 Pay
          ↓
   payment-service
          ↓
       [Pods]

──────────────────────────────────────────────

User hits api.flipkart.com
          ↓
    [LoadBalancer 3]  ← Public IP: 103.21.45.12  💸 Pay
          ↓
    api-service
          ↓
       [Pods]

──────────────────────────────────────────────

User hits search.flipkart.com
          ↓
    [LoadBalancer 4]  ← Public IP: 103.21.45.13  💸 Pay
          ↓
   search-service
          ↓
       [Pods]

──────────────────────────────────────────────

User hits offers.flipkart.com
          ↓
    [LoadBalancer 5]  ← Public IP: 103.21.45.14  💸 Pay
          ↓
   offers-service
          ↓
       [Pods]

──────────────────────────────────────────────

User hits seller.flipkart.com
          ↓
    [LoadBalancer 6]  ← Public IP: 103.21.45.15  💸 Pay
          ↓
   seller-service
          ↓
       [Pods]
```

### 💸 Cost Without Ingress

| Service | LoadBalancer | Public IP | Cost |
|---|---|---|---|
| www.flipkart.com | LoadBalancer 1 | 103.21.45.10 | 💸 Pay |
| payment.flipkart.com | LoadBalancer 2 | 103.21.45.11 | 💸 Pay |
| api.flipkart.com | LoadBalancer 3 | 103.21.45.12 | 💸 Pay |
| search.flipkart.com | LoadBalancer 4 | 103.21.45.13 | 💸 Pay |
| offers.flipkart.com | LoadBalancer 5 | 103.21.45.14 | 💸 Pay |
| seller.flipkart.com | LoadBalancer 6 | 103.21.45.15 | 💸 Pay |
| **Total** | **6 LoadBalancers** | **6 Public IPs** | **💸💸💸 Expensive!** |

> ⚠️ **6 services = 6 public IPs = 6 cloud LB charges. Very expensive and hard to manage!**

---

## ✅ The Solution — With Ingress

With Ingress, you pay for **ONE LoadBalancer only**.
The **Ingress Controller** handles all routing internally.

```
User hits www.flipkart.com
User hits payment.flipkart.com       All traffic hits
User hits api.flipkart.com      →    ONE public IP
User hits search.flipkart.com        103.21.45.10
User hits offers.flipkart.com
User hits seller.flipkart.com
          ↓
   [LoadBalancer]                ← ONE Public IP  🎉 Pay Once
   103.21.45.10
          ↓
  [Ingress Controller]           ← nginx / Traefik / HAProxy
  reads HOST header
  matches routing rules
          ↓
 ┌────────────────────────────────────────────┐
 │                                            │
 ↓            ↓            ↓                  ↓
[frontend] [payment]   [api]    ...    [seller]
 service    service    service          service
(ClusterIP)(ClusterIP)(ClusterIP)     (ClusterIP)
    ↓           ↓          ↓                  ↓
  [Pods]     [Pods]     [Pods]             [Pods]
```

### 🎉 Cost With Ingress

| Service | LoadBalancer | Public IP | Cost |
|---|---|---|---|
| www.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| payment.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| api.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| search.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| offers.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| seller.flipkart.com | Ingress Controller | 103.21.45.10 | ✅ Shared |
| **Total** | **1 LoadBalancer** | **1 Public IP** | **🎉 Maximum Saving!** |

> ✅ **6 services = 1 public IP = 1 cloud LB charge. Cost saved. Easy to manage!**

---

## 🔄 Complete Traffic Flow — With Ingress

### Step 1: DNS Resolution
```
──────────────────────────────────────────────
www.flipkart.com        → resolves to → 103.21.45.10  (public IP)
payment.flipkart.com    → resolves to → 103.21.45.10  (same public IP)
api.flipkart.com        → resolves to → 103.21.45.10  (same public IP)
search.flipkart.com     → resolves to → 103.21.45.10  (same public IP)
offers.flipkart.com     → resolves to → 103.21.45.10  (same public IP)
seller.flipkart.com     → resolves to → 103.21.45.10  (same public IP)

✨ All URLs point to SAME public IP  ←  this is the magic
──────────────────────────────────────────────
```

### Step 2: Hits LoadBalancer
```
──────────────────────────────────────────────
103.21.45.10  (LoadBalancer)
receives all requests from all URLs
forwards to → Ingress Controller
──────────────────────────────────────────────
```

### Step 3: Ingress Controller reads HOST header
```
──────────────────────────────────────────────
Browser always sends HOST header in every HTTP request

User 1 request carries → Host: www.flipkart.com
User 2 request carries → Host: payment.flipkart.com
User 3 request carries → Host: api.flipkart.com
User 4 request carries → Host: search.flipkart.com
User 5 request carries → Host: offers.flipkart.com
User 6 request carries → Host: seller.flipkart.com

Ingress Controller reads this HOST header
and matches against routing rules
──────────────────────────────────────────────
```

### Step 4: Ingress Routing Rules
```
──────────────────────────────────────────────
www.flipkart.com        → frontend-service  (ClusterIP)
payment.flipkart.com    → payment-service   (ClusterIP)
api.flipkart.com        → api-service       (ClusterIP)
search.flipkart.com     → search-service    (ClusterIP)
offers.flipkart.com     → offers-service    (ClusterIP)
seller.flipkart.com     → seller-service    (ClusterIP)
──────────────────────────────────────────────
```

### Step 5: ClusterIP forwards to Pod
```
──────────────────────────────────────────────
frontend-service  → Pod 10.0.0.1  or  10.0.0.2  or  10.0.0.3
payment-service   → Pod 10.0.1.1  or  10.0.1.2
api-service       → Pod 10.0.2.1  or  10.0.2.2
search-service    → Pod 10.0.3.1  or  10.0.3.2  or  10.0.3.3
offers-service    → Pod 10.0.4.1  or  10.0.4.2
seller-service    → Pod 10.0.5.1  or  10.0.5.2
──────────────────────────────────────────────
```

---

## 💡 The Secret — HOST Header Magic

```
Same public IP for everyone
         ↓
But every browser sends HOST header in HTTP request
         ↓
Host: www.flipkart.com     → goes to frontend-service
Host: payment.flipkart.com → goes to payment-service
Host: api.flipkart.com     → goes to api-service
         ↓
Same IP + different HOST header = different service
         ↓
That's the magic! 🎩
```

> 💡 **Even though all URLs resolve to the same IP `103.21.45.10`, the HOST header tells the Ingress Controller exactly which service to route to. The IP gets you to the door — the HOST header tells you which room to go to.**

---

## 📄 Ingress Routing Rule — YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flipkart-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:

    - host: www.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 3000

    - host: payment.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 8080

    - host: api.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 9000

    - host: search.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: search-service
                port:
                  number: 8081

    - host: offers.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: offers-service
                port:
                  number: 8082

    - host: seller.flipkart.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: seller-service
                port:
                  number: 8083
```

---

## 🏆 Key Takeaway

```
WITHOUT Ingress                    WITH Ingress
──────────────────                 ──────────────────
6 services                         6 services
6 LoadBalancers                    1 LoadBalancer
6 Public IPs                       1 Public IP
6 cloud LB charges 💸💸💸          1 cloud LB charge 🎉
Hard to manage                     Easy to manage
No central TLS                     1 TLS certificate
No central routing                 1 routing config
```

> 🔑 **Simple Rule:**
> - **ClusterIP** → for internal service-to-service communication
> - **Ingress** → for HTTP/HTTPS external traffic (use this in production!)
> - **LoadBalancer** → for non-HTTP external traffic (databases, gRPC, etc.)
> - **One public IP, one LoadBalancer, one TLS certificate — Ingress handles everything at the edge.**

---

## 📚 Related Topics

- [Kubernetes Services — ClusterIP, NodePort, LoadBalancer, ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

---

## 🏷️ Tags

`kubernetes` `k8s` `ingress` `devops` `cloud-native` `networking` `flipkart` `loadbalancer` `microservices`

---

> ⭐ If this explanation helped you, please **star this repo** and share it with your team!
