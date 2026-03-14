## 📋 Lab Overview

| Property | Details |
|----------|---------|
| **Namespace** | default |
| **Tasks** | 6 |

---

## 🏗️ Prerequisites

- Minikube installed and running
- `kubectl` configured
- Ingress controller enabled:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

- Backend apps deployed:

```bash
kubectl apply -f 01-deployments-and-services.yaml
kubectl get deploy,svc
```

---

## 📁 Repository Structure

```
.
├── 01-deployments-and-services.yaml   # 3 backend apps + ClusterIP services
├── 02-ingress-basic-path.yaml         # Task 2 — Path-based ingress
├── 03-ingress-host-based.yaml         # Task 4 — Host-based ingress
├── 04-ingress-default-backend.yaml    # Task 6 — Default backend
├── 05-ingress-path-types.yaml         # Task 5 — Exact vs Prefix PathType
├── 06-ingress-full-lab.yaml           # Task 3 — Full lab ingress (3 paths)
└── README.md
```

---

## 🚀 Lab Tasks

### Part 1 — Setup (20 pts)

#### Task 1 — Enable Ingress Controller & Deploy Backend Apps ⭐

Enable the nginx ingress controller on minikube and deploy 3 backend applications with ClusterIP services.

**Key answers:**
- IngressClass name: `nginx` (controller: `k8s.io/ingress-nginx`)
- Services use **ClusterIP** (not NodePort) because they communicate internally — the Ingress controller handles all external traffic routing

---

### Part 2 — Path-based Routing (40 pts)

#### Task 2 — Create a Path-based Ingress ⭐

Single domain `myapp.local` routing traffic to 2 services based on URL path.

```bash
kubectl apply -f 02-ingress-basic-path.yaml
kubectl get ingress
```

**Test:**
```bash
ADDRESS=$(kubectl get ingress basic-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/       # → webapp-svc
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/api    # → api-svc
```

**Routing table:**

| Path | Service | PathType |
|------|---------|----------|
| `/` | webapp-svc:80 | Prefix |
| `/api` | api-svc:80 | Prefix |

> **Why one Ingress replaces 2 NodePort Services?** A single Ingress provides one external endpoint and routes traffic internally to multiple services — eliminating the need for a separate NodePort per service.

---

#### Task 3 — Add a Third Route `/admin` ⭐⭐

Extends routing to 3 services under the same host.

```bash
kubectl delete ingress basic-ingress
kubectl apply -f 06-ingress-full-lab.yaml
```

**Test:**
```bash
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/        # → webapp-svc (app1)
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/api     # → api-svc (app2)
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/admin   # → admin-svc (app3)
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/random  # → falls back to / (app1)
```

**Routing table:**

| Path | Service | PathType |
|------|---------|----------|
| `/` | webapp-svc:80 | Prefix |
| `/api` | api-svc:80 | Prefix |
| `/admin` | admin-svc:80 | Prefix |

> **Unmatched path `/random`** falls through to the `/` Prefix rule and reaches webapp-svc (app1).

---

### Part 3 — Host-based Routing & PathType (40 pts)

#### Task 4 — Host-based Routing: 3 Subdomains ⭐⭐

One Ingress object routing 3 different subdomains to 3 different services.

```bash
kubectl delete ingress lab-ingress
kubectl apply -f 03-ingress-host-based.yaml
```

**Test:**
```bash
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local              # → webapp-svc
curl --resolve "api.myapp.local:80:$ADDRESS" http://api.myapp.local      # → api-svc
curl --resolve "admin.myapp.local:80:$ADDRESS" http://admin.myapp.local  # → admin-svc
curl --resolve "other.myapp.local:80:$ADDRESS" http://other.myapp.local  # → 404 Not Found
```

**Host routing table:**

| Host | Path | Service |
|------|------|---------|
| `myapp.local` | `/` | webapp-svc |
| `api.myapp.local` | `/` | api-svc |
| `admin.myapp.local` | `/` | admin-svc |

> **Path-based vs Host-based:**
> - **Path-based**: Fixed host, multiple paths → `myapp.local/api`, `myapp.local/admin`
> - **Host-based**: Multiple subdomains, each mapped to a different service → `api.myapp.local`, `admin.myapp.local`

---

#### Task 5 — PathType: Exact vs Prefix ⭐⭐

Demonstrates the difference between `Exact` and `Prefix` path matching.

```bash
kubectl delete ingress host-ingress
kubectl apply -f 05-ingress-path-types.yaml
```

**Test:**
```bash
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/api/users      # ✅ Prefix match → api-svc
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/admin          # ✅ Exact match → admin-svc
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/admin/settings # ❌ Exact fails → falls to /
```

**PathType routing table:**

| Path | PathType | Service | Notes |
|------|----------|---------|-------|
| `/api` | Prefix | api-svc | Matches `/api`, `/api/users`, `/api/v2/...` |
| `/admin` | Exact | admin-svc | **Only** matches `/admin` exactly |
| `/` | Prefix | webapp-svc | Catch-all fallback |

> **When to use `Exact`?** Use it when you need strict path control — e.g., a health check endpoint (`/health`) or an admin panel (`/admin`) that should never accidentally match sub-paths.

---

#### Task 6 — Default Backend: Catch-all Route ⭐⭐⭐

Uses `defaultBackend` to catch all requests that match no rule — regardless of host or path.

```bash
kubectl delete ingress pathtype-demo
kubectl apply -f 04-ingress-default-backend.yaml
```

**Test:**
```bash
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/api      # → api-svc
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/admin    # → admin-svc
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local/anything # → webapp-svc (defaultBackend)
curl --resolve "myapp.local:80:$ADDRESS" http://myapp.local          # → webapp-svc (defaultBackend)
```

> **Real-world use case for `defaultBackend`:** Route all unmatched requests to a custom 404/error page or a fallback service, ensuring users always get a meaningful response instead of a raw nginx error.

---

## 🧠 Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **IngressClass** | Identifies which controller handles the Ingress (`nginx`) |
| **ClusterIP** | Internal-only service — Ingress is the single external entry point |
| **Path-based routing** | One host, multiple paths → multiple services |
| **Host-based routing** | Multiple subdomains → multiple services |
| **PathType: Prefix** | Matches the path and all sub-paths |
| **PathType: Exact** | Matches only the exact path string |
| **defaultBackend** | Catch-all for requests that match no defined rule |

---

## ✅ Verification Commands

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl get ingressclass

# Check deployments and services
kubectl get deploy,svc

# Inspect any ingress
kubectl describe ingress <ingress-name>

# Get controller IP
minikube ip
kubectl get ingress
```

---
