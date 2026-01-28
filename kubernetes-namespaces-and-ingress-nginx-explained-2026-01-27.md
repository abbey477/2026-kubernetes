# Understanding Kubernetes Namespaces and Ingress-NGINX

**Created:** January 27, 2026  
**Purpose:** Learn how Kubernetes namespaces work and their relationship to ingress-nginx

---

## Table of Contents
1. [What is a Kubernetes Namespace?](#what-is-a-kubernetes-namespace)
2. [Why Use Namespaces?](#why-use-namespaces)
3. [Default Namespaces](#default-namespaces-in-kubernetes)
4. [How Namespaces Link to Ingress-NGINX](#how-namespaces-link-to-ingress-nginx)
5. [Visual Architecture](#visual-example-how-it-all-works-together)
6. [Practical Examples](#practical-examples)
7. [Common Patterns](#common-namespace-patterns-for-ingress-nginx)
8. [Important Rules](#important-rules-about-namespaces--ingress)
9. [Useful Commands](#commands-to-explore-namespaces--ingress-nginx)
10. [Migration Considerations](#why-this-matters-for-your-migration)

---

## What is a Kubernetes Namespace?

Think of namespaces as **virtual clusters within a physical cluster** - they're like folders or apartments in a building.

### Real-World Analogy:
```
Kubernetes Cluster = Apartment Building
├── Namespace: production (Floor 3)
│   ├── App A pods
│   ├── App A services
│   └── App A configs
├── Namespace: staging (Floor 2)
│   ├── App B pods
│   └── App B services
└── Namespace: development (Floor 1)
    └── Test apps
```

Each "floor" (namespace) is isolated but shares the same building infrastructure.

### Key Characteristics:
- **Logical separation** within a single physical cluster
- **Resource isolation** between different environments or teams
- **Access control** at the namespace level
- **Resource quotas** can be applied per namespace
- Resources in one namespace cannot directly access resources in another (by default)

---

## Why Use Namespaces?

### 1. **Isolation**
Separate different environments or stages of your application lifecycle:

```bash
# Production apps stay separate from development
production namespace → prod database, prod APIs
staging namespace → staging database, staging APIs
development namespace → test database, test APIs
```

**Example:**
- Your dev team can experiment freely in the `development` namespace
- Changes won't affect the `production` namespace
- Each namespace has its own resources (pods, services, configs)

### 2. **Organization**
Group related resources by team, project, or function:

```bash
# Team-based organization
team-frontend namespace → React apps, Next.js apps
team-backend namespace → Java services, Python APIs
team-data namespace → Kafka, databases, data pipelines

# Or project-based organization
project-ecommerce namespace → Shopping cart, payment services
project-analytics namespace → Reporting, dashboards
project-mobile-api namespace → Mobile backend services
```

**Benefits:**
- Easy to find related resources
- Clear ownership and responsibility
- Simplified resource management

### 3. **Access Control (RBAC)**
Control who can access what using Role-Based Access Control:

```bash
# Different teams get access to different namespaces
DevTeam → can read/write in 'development' namespace
OpsTeam → can read/write in all namespaces
ProductTeam → can only read 'production' namespace
```

**Example RBAC:**
```yaml
# Give developer access only to development namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: development
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### 4. **Resource Quotas**
Limit resources to prevent one namespace from consuming all cluster resources:

```bash
# Set limits per namespace
development namespace → max 10 pods, 4GB RAM, 2 CPUs
staging namespace → max 30 pods, 16GB RAM, 8 CPUs
production namespace → max 100 pods, 64GB RAM, 32 CPUs
```

**Example Resource Quota:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    pods: "10"
```

### 5. **Environment Separation**
Run multiple versions of the same application:

```bash
# Same application, different namespaces
production namespace
  └── myapp:v1.5.0 (stable)

staging namespace
  └── myapp:v1.6.0-rc1 (testing)

development namespace
  └── myapp:v1.7.0-dev (in development)
```

---

## Default Namespaces in Kubernetes

When you create a Kubernetes cluster, you get these namespaces by default:

```bash
# List all namespaces
kubectl get namespaces
```

**Output:**
```
NAME              STATUS   AGE     DESCRIPTION
default           Active   30d     Where resources go if no namespace specified
kube-system       Active   30d     Kubernetes system components (DNS, metrics)
kube-public       Active   30d     Resources that should be publicly accessible
kube-node-lease   Active   30d     Node heartbeat data (for node health)
```

### Namespace Details:

#### 1. **default**
- Where your resources go if you don't specify a namespace
- Most user applications start here
- Not recommended for production use (too generic)

```bash
# These two commands are equivalent:
kubectl get pods
kubectl get pods -n default
```

#### 2. **kube-system**
- Contains Kubernetes control plane components
- Examples: kube-dns, kube-proxy, metrics-server
- Critical for cluster operation
- **Important:** ingress-nginx controller is sometimes installed here

```bash
# View system components
kubectl get pods -n kube-system
```

#### 3. **kube-public**
- Publicly readable by all users (even unauthenticated)
- Used for cluster information
- Rarely used in practice

#### 4. **kube-node-lease**
- Stores node heartbeat information
- Used by Kubernetes to detect node failures
- Automatic, no user interaction needed

---

## How Namespaces Link to Ingress-NGINX

Here's where it gets interesting! There are **two ways** ingress-nginx relates to namespaces:

### Understanding the Two Layers:

```
Layer 1: Controller Layer (Where ingress-nginx runs)
    └── Usually in 'ingress-nginx' or 'kube-system' namespace

Layer 2: Application Layer (Where your apps and ingress rules are)
    ├── production namespace (with ingress rules)
    ├── staging namespace (with ingress rules)
    └── development namespace (with ingress rules)
```

---

### 1. **Where Ingress-NGINX Controller Runs** (Controller Namespace)

The ingress-nginx **controller itself** runs as pods in a namespace, typically:

```bash
# Most common: dedicated ingress-nginx namespace
kubectl get pods -n ingress-nginx

# Output:
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-abc123             1/1     Running   0          5d
ingress-nginx-controller-xyz789             1/1     Running   0          5d
```

**This is where the "traffic cop" lives** - the actual ingress controller that manages routing.

#### What the Controller Does:
- **Watches** all ingress resources across all namespaces (by default)
- **Configures** NGINX based on ingress rules it finds
- **Routes** incoming traffic to the correct services
- **Runs** as one or more pods for high availability

#### View the Controller:
```bash
# Check where ingress-nginx controller is installed
kubectl get pods --all-namespaces | grep ingress-nginx

# Get detailed information
kubectl get deployment -n ingress-nginx

# Check controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

#### Common Controller Namespaces:
| Namespace | When Used | Notes |
|-----------|-----------|-------|
| `ingress-nginx` | Most common | Dedicated namespace for ingress controller |
| `kube-system` | Legacy installations | Older setups, not recommended |
| `ingress` | Alternative naming | Some organizations prefer this |
| `default` | Rarely | Not recommended, too generic |

---

### 2. **Where Ingress Resources Are Defined** (Application Namespaces)

Your **Ingress resources** (routing rules) can exist in ANY namespace where your apps are:

```bash
# Example: You have apps in different namespaces

production namespace
  ├── my-app (pod)                  # Your application
  ├── my-app-service (service)      # Service exposing the pod
  └── my-app-ingress (ingress)      # Routes traffic TO my-app-service

staging namespace
  ├── test-app (pod)
  ├── test-app-service (service)
  └── test-app-ingress (ingress)    # Routes traffic TO test-app-service

development namespace
  ├── dev-app (pod)
  ├── dev-app-service (service)
  └── dev-app-ingress (ingress)     # Routes traffic TO dev-app-service
```

#### Key Points:
- **One controller** in its namespace watches **all ingress resources** in all namespaces
- Each ingress resource must be in the **same namespace** as the service it routes to
- You can have multiple ingress resources with the same name in different namespaces
- Each namespace manages its own routing rules independently

---

## Visual Example: How It All Works Together

```
                        INTERNET
                           |
                           ↓
        ┌──────────────────────────────────┐
        │  Load Balancer (AWS ELB/NLB)     │
        │  External IP: 52.1.2.3           │
        └──────────────┬───────────────────┘
                       |
                       ↓
┌────────────────────────────────────────────────────────────┐
│  Namespace: ingress-nginx                                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Ingress-NGINX Controller (Pods)                      │ │
│  │                                                       │ │
│  │ • Watches ALL ingress resources in ALL namespaces    │ │
│  │ • Routes traffic based on Host/Path rules            │ │
│  │ • Configures NGINX dynamically                       │ │
│  │                                                       │ │
│  │ Controller Logic:                                    │ │
│  │ IF host == "api.prod.com"                            │ │
│  │   → Route to production/api-service                  │ │
│  │ IF host == "api.staging.com"                         │ │
│  │   → Route to staging/api-service                     │ │
│  │ IF host == "api.dev.com"                             │ │
│  │   → Route to development/api-service                 │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────┬───────────────────────────────────────┘
                     |
        ┌────────────┴────────────┬────────────────┐
        ↓                         ↓                ↓
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Namespace:      │    │ Namespace:      │    │ Namespace:      │
│ production      │    │ staging         │    │ development     │
│                 │    │                 │    │                 │
│ Ingress:        │    │ Ingress:        │    │ Ingress:        │
│ api-ingress     │    │ api-ingress     │    │ api-ingress     │
│ host:           │    │ host:           │    │ host:           │
│ api.prod.com    │    │ api.staging.com │    │ api.dev.com     │
│   ↓             │    │   ↓             │    │   ↓             │
│ Service:        │    │ Service:        │    │ Service:        │
│ api-service     │    │ api-service     │    │ api-service     │
│ port: 80        │    │ port: 80        │    │ port: 80        │
│   ↓             │    │   ↓             │    │   ↓             │
│ Pods:           │    │ Pods:           │    │ Pods:           │
│ [api-pod-1]     │    │ [api-pod-1]     │    │ [api-pod-1]     │
│ [api-pod-2]     │    │ [api-pod-2]     │    │ [api-pod-2]     │
│ [api-pod-3]     │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Traffic Flow Explanation:

1. **User Request:** `http://api.prod.com/users`
2. **DNS Resolution:** Resolves to Load Balancer IP (52.1.2.3)
3. **Load Balancer:** Forwards to ingress-nginx controller
4. **Controller Logic:**
   - Reads Host header: `api.prod.com`
   - Finds matching ingress in `production` namespace
   - Routes to `api-service` in `production` namespace
5. **Service:** Load balances to one of the api pods
6. **Pod:** Processes request and returns response

---

## Practical Examples

### Example 1: Ingress in Production Namespace

```yaml
# File: production-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production  # ← Ingress is in 'production' namespace
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service  # ← Must be a service in 'production' namespace
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert  # ← Secret must also be in 'production' namespace
```

**Key Points:**
- The ingress is in the `production` namespace
- It can ONLY route to services in the `production` namespace
- TLS secrets must also be in the `production` namespace
- The ingress-nginx controller (wherever it runs) will detect and configure this

**Apply the ingress:**
```bash
# Create the ingress
kubectl apply -f production-ingress.yaml

# Verify it was created
kubectl get ingress -n production

# View details
kubectl describe ingress api-ingress -n production
```

---

### Example 2: Multiple Namespaces with Ingress

Let's create the same application in three different environments:

#### Production Environment
```yaml
# File: prod-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: api.example.com  # Production domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

#### Staging Environment
```yaml
# File: staging-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Service
metadata:
  name: api-service  # Same name as production, but different namespace!
  namespace: staging
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress  # Same name as production, but different namespace!
  namespace: staging
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: staging-api.example.com  # Different domain for staging
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service  # Routes to api-service in staging namespace
            port:
              number: 80
```

#### Development Environment
```yaml
# File: dev-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: development
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: development
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: dev-api.example.com  # Development domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Apply all environments:**
```bash
# Create all three environments
kubectl apply -f prod-namespace.yaml
kubectl apply -f staging-namespace.yaml
kubectl apply -f dev-namespace.yaml

# Verify ingresses were created in each namespace
kubectl get ingress --all-namespaces

# Output:
# NAMESPACE     NAME          CLASS   HOSTS                    ADDRESS
# production    api-ingress   nginx   api.example.com          52.1.2.3
# staging       api-ingress   nginx   staging-api.example.com  52.1.2.3
# development   api-ingress   nginx   dev-api.example.com      52.1.2.3
```

**Result:**
- `api.example.com` → routes to `production/api-service`
- `staging-api.example.com` → routes to `staging/api-service`
- `dev-api.example.com` → routes to `development/api-service`
- **One ingress-nginx controller** manages all three!
- Each environment is completely isolated

---

### Example 3: Path-Based Routing Within a Namespace

You can have multiple services in one namespace with different paths:

```yaml
# File: microservices-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service  # Routes /users/* to user-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service  # Routes /products/* to product-service
            port:
              number: 80
      - path: /orders
        pathType Prefix
        backend:
          service:
            name: order-service  # Routes /orders/* to order-service
            port:
              number: 80
```

**Traffic routing:**
- `api.example.com/users/123` → `production/user-service`
- `api.example.com/products/456` → `production/product-service`
- `api.example.com/orders/789` → `production/order-service`

All services must be in the `production` namespace!

---

## How Ingress-NGINX Controller Watches Multiple Namespaces

The ingress-nginx controller (wherever it's installed) watches **ALL namespaces** by default:

```bash
# The controller automatically discovers ingress resources everywhere:
Namespace: production     → Ingress: prod-api (watches and configures)
Namespace: staging        → Ingress: staging-api (watches and configures)
Namespace: development    → Ingress: dev-api (watches and configures)
Namespace: team-frontend  → Ingress: frontend-ingress (watches and configures)
Namespace: team-backend   → Ingress: backend-ingress (watches and configures)
```

### How It Works:

1. **Controller starts** in its namespace (e.g., `ingress-nginx`)
2. **Kubernetes API watch** is established for ingress resources cluster-wide
3. **When ingress is created/updated** in ANY namespace:
   - Controller detects the change
   - Reads the ingress specification
   - Updates NGINX configuration
   - Reloads NGINX
4. **Traffic flows** according to the new rules

It's like a **central traffic controller at an airport** - it doesn't matter which terminal (namespace) the plane (request) is going to; the controller manages all of them.

### Limiting Controller Scope (Optional)

You can configure the controller to watch only specific namespaces:

```yaml
# Controller deployment with namespace restriction
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  template:
    spec:
      containers:
      - name: controller
        args:
        - /nginx-ingress-controller
        - --watch-namespace=production,staging  # Only watch these namespaces
```

**When to limit scope:**
- Multi-tenant clusters with separate teams
- Security requirements for namespace isolation
- Different ingress configurations per environment

---

## Common Namespace Patterns for Ingress-NGINX

### Pattern 1: One Controller for Everything (Most Common)

**Architecture:**
```
Cluster: my-eks-cluster
├── ingress-nginx namespace
│   └── Controller pods (watches ALL namespaces)
│
├── production namespace
│   ├── App pods
│   ├── Services
│   └── Ingress resources
│
├── staging namespace
│   ├── App pods
│   ├── Services
│   └── Ingress resources
│
└── development namespace
    ├── App pods
    ├── Services
    └── Ingress resources
```

**Pros:**
- Simple architecture
- Single point of management
- Cost-effective (one load balancer)
- Easy to maintain

**Cons:**
- All environments share same controller
- Controller issues affect all namespaces
- Less isolation between environments

**Best for:**
- Small to medium teams
- Standard web applications
- Cost-conscious deployments

---

### Pattern 2: Environment-Specific Controllers

**Architecture:**
```
Cluster: my-eks-cluster
├── ingress-nginx-prod namespace
│   └── Production controller (watches only 'production' namespace)
│
├── ingress-nginx-staging namespace
│   └── Staging controller (watches only 'staging' namespace)
│
├── ingress-nginx-dev namespace
│   └── Dev controller (watches only 'development' namespace)
│
├── production namespace
│   └── App ingresses (managed by prod controller)
│
├── staging namespace
│   └── App ingresses (managed by staging controller)
│
└── development namespace
    └── App ingresses (managed by dev controller)
```

**Pros:**
- Complete isolation between environments
- Independent upgrades per environment
- Different configurations per environment
- Issues in one don't affect others

**Cons:**
- More complex to manage
- Higher cost (multiple load balancers)
- More resources consumed

**Best for:**
- Large enterprises
- Strict compliance requirements
- High-availability production systems

---

### Pattern 3: Team-Based Namespaces

**Architecture:**
```
Cluster: my-eks-cluster
├── ingress-nginx namespace
│   └── Shared controller (watches all team namespaces)
│
├── team-alpha namespace
│   ├── Team A applications
│   ├── Team A services
│   └── Team A ingresses
│
├── team-beta namespace
│   ├── Team B applications
│   ├── Team B services
│   └── Team B ingresses
│
├── team-gamma namespace
│   ├── Team C applications
│   ├── Team C services
│   └── Team C ingresses
│
└── shared-services namespace
    └── Common services (databases, caches, etc.)
```

**Pros:**
- Clear team ownership
- Resource quotas per team
- Access control per team
- Team autonomy

**Cons:**
- Teams can affect each other via shared controller
- Requires good RBAC setup
- Need coordination for changes

**Best for:**
- Organizations with multiple teams
- Microservices architectures
- Teams with different applications

---

### Pattern 4: Mixed Approach

**Architecture:**
```
Cluster: my-eks-cluster
├── ingress-nginx-public namespace
│   └── Public-facing controller
│       (watches: production, staging, public-services)
│
├── ingress-nginx-internal namespace
│   └── Internal-only controller
│       (watches: internal-services, admin-tools)
│
├── production namespace (public)
├── staging namespace (public)
├── public-services namespace
├── internal-services namespace (internal)
└── admin-tools namespace (internal)
```

**Pros:**
- Security separation (public vs internal)
- Different configurations for different use cases
- Flexible routing rules

**Best for:**
- Applications with both public and internal APIs
- Security-conscious organizations
- Complex routing requirements

---

## Important Rules About Namespaces & Ingress

### ❌ Rule 1: Cannot Cross Namespace Boundaries

**This DOES NOT work:**
```yaml
# Ingress in 'production' namespace
# CANNOT route to service in 'staging' namespace
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bad-example
  namespace: production  # Ingress is here
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: api-service
            namespace: staging  # ← This field doesn't exist! Won't work!
            port:
              number: 80
```

**Error you'll see:**
```
Error: Service "api-service" not found in namespace "production"
```

**Why?** Kubernetes ingress can only reference services in the same namespace for security and isolation.

---

### ✅ Rule 2: Keep Ingress and Service Together

**This DOES work:**
```yaml
# Both ingress and service in same namespace
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: good-example
  namespace: production  # ← Same namespace
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: api-service  # ← Service is also in 'production'
            port:
              number: 80
```

---

### ✅ Rule 3: Same Name in Different Namespaces is OK

You can have resources with the same name in different namespaces:

```bash
# These don't conflict:
production/api-service (running v1.5.0)
staging/api-service (running v1.6.0-rc)
development/api-service (running v1.7.0-dev)

# Same with ingresses:
production/api-ingress
staging/api-ingress
development/api-ingress
```

Each namespace is independent!

---

### ✅ Rule 4: TLS Secrets Must Be in Same Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert  # ← Must exist in 'production' namespace
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: api-service
            port:
              number: 443
```

**Create the secret in the correct namespace:**
```bash
kubectl create secret tls api-tls-cert \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  -n production  # ← Same namespace as ingress
```

---

### ⚠️ Rule 5: Controller Can Watch Multiple, But You Configure It

By default, the ingress-nginx controller watches ALL namespaces, but you can restrict it:

```bash
# Controller that watches everything (default)
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml
# (no --watch-namespace argument means watch all)

# Controller that watches specific namespaces only
# Edit the deployment and add:
args:
  - --watch-namespace=production,staging
```

---

### ✅ Rule 6: IngressClass Works Across Namespaces

IngressClass is a cluster-wide resource (not namespaced):

```yaml
# This IngressClass can be used by ingresses in any namespace
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx  # No namespace!
spec:
  controller: k8s.io/ingress-nginx
```

```yaml
# Ingress in any namespace can use it
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: production
spec:
  ingressClassName: nginx  # References cluster-wide IngressClass
  rules:
  - host: api.example.com
    # ...
```

---

## Commands to Explore Namespaces & Ingress-NGINX

### Namespace Discovery Commands

```bash
# 1. List all namespaces in the cluster
kubectl get namespaces
# Shows all namespaces with their status and age

# 2. List namespaces with labels
kubectl get namespaces --show-labels
# Helpful to see how namespaces are organized

# 3. Get detailed info about a specific namespace
kubectl describe namespace production
# Shows quotas, limits, and other namespace details

# 4. Create a new namespace
kubectl create namespace my-new-namespace

# 5. Create namespace from YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  labels:
    environment: test
EOF

# 6. Delete a namespace (careful - deletes everything in it!)
kubectl delete namespace my-old-namespace
```

---

### Finding Ingress-NGINX Controller

```bash
# 1. Find where ingress-nginx controller runs
kubectl get pods --all-namespaces | grep ingress-nginx
# Shows controller pods and their namespace

# 2. Get controller pods with labels
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
# More precise - uses official labels

# 3. Check controller deployment
kubectl get deployment --all-namespaces | grep ingress
# Shows the deployment managing controller pods

# 4. Get detailed controller info
kubectl describe deployment ingress-nginx-controller -n ingress-nginx
# Shows configuration, replicas, resources

# 5. View controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100
# See what the controller is doing

# 6. Check controller version
kubectl get deployment ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.template.spec.containers[0].image}'
# Shows the image/version being used
```

---

### Working with Ingress Resources

```bash
# 1. List ingress resources in a specific namespace
kubectl get ingress -n production
# Shows ingresses only in production namespace

# 2. List ALL ingress resources across all namespaces
kubectl get ingress --all-namespaces
# Overview of all routing rules in the cluster

# 3. Get ingress with more details
kubectl get ingress --all-namespaces -o wide
# Shows hosts, addresses, ports, and age

# 4. Describe a specific ingress (shows rules, backends, events)
kubectl describe ingress api-ingress -n production
# Detailed view including routing rules and TLS

# 5. Get ingress in YAML format
kubectl get ingress api-ingress -n production -o yaml
# Full specification - useful for migration

# 6. Get ingress in JSON format
kubectl get ingress api-ingress -n production -o json
# Programmatic access to ingress configuration

# 7. Watch ingress changes in real-time
kubectl get ingress -n production --watch
# See changes as they happen

# 8. Filter ingresses by label
kubectl get ingress -n production -l app=api
# Only show ingresses with app=api label
```

---

### Checking Namespace Context

```bash
# 1. See what namespace you're currently using (default)
kubectl config view --minify | grep namespace
# Shows current default namespace

# 2. Switch default namespace (changes where kubectl looks by default)
kubectl config set-context --current --namespace=production
# Now 'kubectl get pods' shows production pods

# 3. Verify current context and namespace
kubectl config get-contexts
# Shows all contexts and which is active

# 4. Temporarily use different namespace (doesn't change default)
kubectl get pods -n staging
# -n flag overrides default namespace for one command

# 5. Reset to default namespace
kubectl config set-context --current --namespace=default
```

---

### Viewing Resources in Namespaces

```bash
# 1. See all resources in a specific namespace
kubectl get all -n production
# Shows pods, services, deployments, etc.

# 2. See only specific resource types
kubectl get pods,services,ingress -n production
# Comma-separated list of resources

# 3. Get resource counts per namespace
kubectl get all --all-namespaces | awk '{print $1}' | sort | uniq -c
# Shows how many resources in each namespace

# 4. List services in all namespaces
kubectl get services --all-namespaces
# Useful to see all exposed services

# 5. Check configmaps and secrets
kubectl get configmap,secret -n production
# See configuration and credentials

# 6. View events in a namespace
kubectl get events -n production --sort-by='.lastTimestamp'
# See what's happening (errors, warnings, info)
```

---

### Advanced Ingress Discovery

```bash
# 1. Find all ingresses using nginx ingress class
kubectl get ingress --all-namespaces -o json | \
  jq '.items[] | select(.spec.ingressClassName=="nginx") | {namespace:.metadata.namespace, name:.metadata.name, host:.spec.rules[0].host}'
# Uses jq to filter and format results

# 2. List all unique hosts across all ingresses
kubectl get ingress --all-namespaces -o jsonpath='{range .items[*]}{.spec.rules[*].host}{"\n"}{end}' | sort -u
# Shows all domains being routed

# 3. Find ingresses without TLS
kubectl get ingress --all-namespaces -o json | \
  jq '.items[] | select(.spec.tls == null) | {namespace:.metadata.namespace, name:.metadata.name}'
# Security audit - find unencrypted ingresses

# 4. Check which services are exposed via ingress
kubectl get ingress --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{" → "}{.spec.rules[*].http.paths[*].backend.service.name}{"\n"}{end}'
# Map namespaces to backend services

# 5. Find ingresses with specific annotations
kubectl get ingress --all-namespaces -o json | \
  jq '.items[] | select(.metadata.annotations["kubernetes.io/ingress.class"]=="nginx")'
# Filter by annotation (legacy ingress class method)
```

---

### Troubleshooting Commands

```bash
# 1. Check if ingress controller is running
kubectl get pods -n ingress-nginx
# Should show controller pods in Running state

# 2. Check controller service and external IP
kubectl get svc -n ingress-nginx
# Shows the LoadBalancer service and its external IP

# 3. Test DNS resolution
nslookup api.example.com
# Should resolve to the load balancer IP

# 4. Check ingress status
kubectl get ingress api-ingress -n production
# ADDRESS column should show the load balancer IP

# 5. View controller configuration
kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml
# See how the controller is configured

# 6. Check for errors in ingress resource
kubectl describe ingress api-ingress -n production | grep -i error
# Look for error messages

# 7. Validate ingress connectivity
kubectl run test-pod --image=curlimages/curl --rm -it -- curl -H "Host: api.example.com" http://<CONTROLLER-IP>
# Test from inside the cluster

# 8. Check service endpoints
kubectl get endpoints api-service -n production
# Verify service has backend pods

# 9. View recent events
kubectl get events -n production --sort-by='.lastTimestamp' | tail -20
# See recent activity and errors
```

---

### Comprehensive Inventory Script

```bash
#!/bin/bash
# Generate complete ingress-nginx inventory across namespaces

OUTPUT="namespace-ingress-inventory-$(date +%Y%m%d-%H%M%S).txt"

echo "=== KUBERNETES NAMESPACES & INGRESS-NGINX INVENTORY ===" > $OUTPUT
echo "Generated: $(date)" >> $OUTPUT
echo "" >> $OUTPUT

echo "=== ALL NAMESPACES ===" >> $OUTPUT
kubectl get namespaces >> $OUTPUT
echo "" >> $OUTPUT

echo "=== INGRESS-NGINX CONTROLLER LOCATION ===" >> $OUTPUT
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx >> $OUTPUT
echo "" >> $OUTPUT

echo "=== INGRESS-NGINX SERVICES ===" >> $OUTPUT
kubectl get svc --all-namespaces -l app.kubernetes.io/name=ingress-nginx >> $OUTPUT
echo "" >> $OUTPUT

echo "=== ALL INGRESS RESOURCES BY NAMESPACE ===" >> $OUTPUT
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  ingresses=$(kubectl get ingress -n $ns --no-headers 2>/dev/null | wc -l)
  if [ $ingresses -gt 0 ]; then
    echo "--- Namespace: $ns ---" >> $OUTPUT
    kubectl get ingress -n $ns -o wide >> $OUTPUT
    echo "" >> $OUTPUT
  fi
done

echo "=== INGRESS CLASSES ===" >> $OUTPUT
kubectl get ingressclass >> $OUTPUT
echo "" >> $OUTPUT

echo "=== SERVICES WITH TYPE LoadBalancer ===" >> $OUTPUT
kubectl get svc --all-namespaces --field-selector spec.type=LoadBalancer >> $OUTPUT

echo "" >> $OUTPUT
echo "Report saved to: $OUTPUT"
cat $OUTPUT
```

**What this script does:**
1. Lists all namespaces
2. Finds ingress-nginx controller pods
3. Lists controller services
4. Iterates through each namespace and lists ingresses
5. Shows ingress classes
6. Lists LoadBalancer services
7. Saves everything to a timestamped file

---

## Why This Matters for Your Migration

Understanding namespaces is crucial for migrating from ingress-nginx because:

### 1. **Locate All Ingress Resources**
You need to find ingress resources in EVERY namespace:

```bash
# Don't just check one namespace
kubectl get ingress -n production  # ❌ Incomplete

# Check ALL namespaces
kubectl get ingress --all-namespaces  # ✅ Complete picture
```

**Why it matters:**
- Missing an ingress means missing an application
- Could result in downtime during migration
- Need complete inventory for planning

---

### 2. **Understand the Blast Radius**
When you remove ingress-nginx controller:

```bash
# If controller is in 'ingress-nginx' namespace:
kubectl delete namespace ingress-nginx  # Affects ALL ingresses in ALL namespaces!
```

**Impact:**
- Controller removal affects every namespace using it
- All ingresses stop working simultaneously
- Need to plan downtime or blue-green migration

---

### 3. **Plan Per-Namespace Migration**
You can migrate namespace by namespace:

```bash
# Phase 1: Migrate development (lowest risk)
# - Update ingresses in development namespace
# - Test thoroughly

# Phase 2: Migrate staging
# - Update ingresses in staging namespace
# - Validate against staging apps

# Phase 3: Migrate production (last)
# - Update ingresses in production namespace
# - Monitor closely
```

**Strategy:**
- Start with non-critical namespaces
- Learn from each migration
- Minimize production risk

---

### 4. **Identify Application Owners**
Different namespaces often have different owners:

```bash
production namespace → DevOps team
team-alpha namespace → Team Alpha
team-beta namespace → Team Beta
staging namespace → QA team
```

**Coordination needed:**
- Each team needs to test after migration
- Different schedules and priorities
- Clear communication channels

---

### 5. **Handle Namespace-Specific Configurations**
Each namespace might have unique ingress configurations:

```bash
# Production might have:
- WAF rules
- Rate limiting
- TLS certificates
- Custom annotations

# Development might have:
- Basic auth
- IP whitelisting
- No TLS
```

**Migration considerations:**
- AWS ALB supports different features than ingress-nginx
- Some annotations don't translate directly
- Need namespace-specific migration plans

---

### 6. **Resource Quotas and Limits**
Namespaces have resource quotas that affect new controllers:

```bash
# Check namespace quotas
kubectl describe namespace production | grep -A 10 "Resource Quotas"

# If quota is maxed out:
# - Can't install AWS Load Balancer Controller
# - Need to increase limits
# - Or remove old ingress-nginx first
```

---

### 7. **RBAC and Permissions**
Your access might be limited per namespace:

```bash
# You might have:
✅ Full access to development namespace
⚠️  Read-only access to staging namespace
❌ No access to production namespace

# Impact on migration:
# - Can't migrate production yourself
# - Need to coordinate with ops team
# - Requires elevated permissions or handoff
```

---

## Summary Table

| Concept | What It Is | Example | Impact on Migration |
|---------|-----------|---------|---------------------|
| **Namespace** | Virtual cluster partition | `production`, `staging`, `dev` | Each needs separate migration plan |
| **Controller Namespace** | Where ingress-nginx pods run | Usually `ingress-nginx` or `kube-system` | One controller to replace |
| **Ingress Namespace** | Where routing rules exist | Any application namespace | Multiple ingresses to update |
| **Cross-Namespace** | Ingress can't route across namespaces | ❌ prod ingress → staging service | Each namespace is isolated |
| **One Controller, Many Ingresses** | Controller watches all namespaces | 1 controller → 50+ ingresses | Single point of migration coordination |
| **Resource Quotas** | Limits per namespace | Max 10 pods in dev | May need quota adjustments |
| **RBAC** | Access control per namespace | Dev team → dev namespace only | May need permission escalation |

---

## Key Takeaways

1. **Namespaces provide isolation** - like apartments in a building, sharing infrastructure but separated
2. **Ingress-nginx controller runs in ONE namespace** but manages ingresses in ALL namespaces
3. **Each ingress must be in the SAME namespace** as the service it routes to
4. **You can have identically named resources** in different namespaces without conflict
5. **For migration:** You need to find and update ingresses in EVERY namespace, not just one
6. **The controller location matters** because that's what you'll replace with AWS Load Balancer Controller
7. **Plan migration by namespace** to reduce risk and manage complexity

---

## Next Steps

Now that you understand namespaces and ingress-nginx:

1. **Run discovery across ALL namespaces** using the commands above
2. **Document which namespaces have ingresses** and their owners
3. **Create namespace-specific migration plans**
4. **Test in development namespace first**
5. **Progress through namespaces** based on risk and priority
6. **Monitor each namespace** after migration

---

## Additional Resources

- [Kubernetes Namespaces Documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress-NGINX Documentation](https://kubernetes.github.io/ingress-nginx/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

---

**Document Version:** 1.0  
**Last Updated:** January 27, 2026  
**Status:** Complete
