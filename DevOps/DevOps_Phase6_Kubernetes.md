# DevOps Phase 6 — Kubernetes

> **Learning Goal:** Master container orchestration — understand how Kubernetes manages thousands of containers across clusters, handles scaling, self-healing, networking, configuration, and production deployments.

---

## Table of Contents

1. [Why Kubernetes Exists](#1-why-kubernetes-exists)
2. [Kubernetes Architecture](#2-kubernetes-architecture)
3. [Pods — The Atomic Unit](#3-pods--the-atomic-unit)
4. [Deployments — Managing Application Lifecycle](#4-deployments--managing-application-lifecycle)
5. [Services — Networking & Discovery](#5-services--networking--discovery)
6. [Ingress — External Traffic Routing](#6-ingress--external-traffic-routing)
7. [ConfigMaps & Secrets](#7-configmaps--secrets)
8. [Storage — PersistentVolumes & PersistentVolumeClaims](#8-storage--persistentvolumes--persistentvolumeclaims)
9. [StatefulSets — Stateful Workloads](#9-statefulsets--stateful-workloads)
10. [Namespaces — Multi-Tenancy & Organization](#10-namespaces--multi-tenancy--organization)
11. [Autoscaling — HPA, VPA, Cluster Autoscaler](#11-autoscaling--hpa-vpa-cluster-autoscaler)
12. [Helm — Package Manager for Kubernetes](#12-helm--package-manager-for-kubernetes)
13. [RBAC — Role-Based Access Control](#13-rbac--role-based-access-control)
14. [Production Patterns & Best Practices](#14-production-patterns--best-practices)
15. [Debugging Kubernetes](#15-debugging-kubernetes)
16. [Interview Mastery](#16-interview-mastery)

---

## 1. Why Kubernetes Exists

### Beginner Explanation

Docker gave us containers — a way to package and run applications consistently. But in production, you don't run one container. You run hundreds or thousands, across many servers. Questions arise:

- Which server should this container run on? (The one with free memory?)
- What if a container crashes at 3am? (Who restarts it?)
- What if traffic spikes? (Who starts more containers?)
- How do containers find each other? (Service A needs to talk to Service B)
- How do I update my app without downtime? (Rolling out new versions)
- What if an entire server dies? (Reschedule its containers elsewhere)

**Kubernetes answers all of these questions automatically.** You tell Kubernetes "I want 5 copies of my app running" and it figures out where to place them, keeps them running, handles networking, and scales up or down as needed.

### Real-World Analogy

Think of Kubernetes as an **airport control tower:**

- **Planes (containers)** need to land at **gates (nodes/servers)**
- The **control tower (Kubernetes control plane)** decides which gate each plane goes to
- If a gate breaks, the control tower reroutes planes to working gates
- If too many planes arrive, the airport opens more gates (autoscaling)
- Each plane has a flight number (service name) that passengers use — they don't care which physical gate it's at

### What Kubernetes Manages

```
WITHOUT KUBERNETES (manual orchestration):

You (the human) must:
  1. SSH into servers to start containers
  2. Monitor if containers crash → manually restart
  3. Monitor server load → manually move containers
  4. Update containers one by one → risk downtime
  5. Manage networking between containers manually
  6. Handle secrets, configs, storage manually
  
  At scale (50+ services, 100+ containers): IMPOSSIBLE manually

WITH KUBERNETES:

You declare: "I want 5 replicas of my API, with 512MB RAM each"
Kubernetes handles:
  ✅ Scheduling containers to healthy nodes
  ✅ Restarting crashed containers (self-healing)
  ✅ Scaling up/down based on load
  ✅ Rolling updates with zero downtime
  ✅ Service discovery and load balancing
  ✅ Secret and config management
  ✅ Storage orchestration
  ✅ Automatic bin-packing (efficient resource usage)
```

### Kubernetes vs Docker Compose vs Docker Swarm

| Feature | Docker Compose | Docker Swarm | Kubernetes |
|---------|---------------|-------------|------------|
| Scale | Single host | Multi-host | Multi-host, multi-region |
| Use case | Development | Simple production | Enterprise production |
| Self-healing | No | Basic | Advanced (probes, PDB) |
| Scaling | Manual | Basic auto-scale | HPA, VPA, cluster autoscaler |
| Networking | Simple bridge | Overlay network | CNI plugins (Calico, Cilium) |
| Storage | Docker volumes | Docker volumes | PV/PVC, CSI drivers |
| Ecosystem | Minimal | Minimal | Massive (Helm, Istio, ArgoCD) |
| Learning curve | Low | Medium | High |
| Industry adoption | Dev only | Declining | Industry standard |

---

## 2. Kubernetes Architecture

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          KUBERNETES CLUSTER                                  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      CONTROL PLANE (Master)                           │  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────────────────┐   │  │
│  │  │  API Server  │  │  etcd         │  │  Controller Manager      │   │  │
│  │  │  (kube-api)  │  │  (data store) │  │  (control loops)         │   │  │
│  │  │              │  │              │  │                          │   │  │
│  │  │ ─ Front door │  │ ─ Key-value  │  │  ─ ReplicaSet controller │   │  │
│  │  │ ─ REST API   │  │ ─ Cluster    │  │  ─ Deployment controller │   │  │
│  │  │ ─ Auth/Authz │  │   state      │  │  ─ Node controller       │   │  │
│  │  │ ─ Validation │  │ ─ Source of  │  │  ─ Job controller        │   │  │
│  │  │              │  │   truth      │  │                          │   │  │
│  │  └──────┬───────┘  └───────────────┘  └──────────────────────────┘   │  │
│  │         │                                                             │  │
│  │  ┌──────┴───────────────────────────────────────────────────────┐     │  │
│  │  │                    Scheduler (kube-scheduler)                 │     │  │
│  │  │  ─ Watches for unscheduled pods                              │     │  │
│  │  │  ─ Picks best node based on resources, affinity, taints      │     │  │
│  │  └──────────────────────────────────────────────────────────────┘     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌────────────────────────┐  ┌────────────────────────┐  ┌──────────────┐  │
│  │     WORKER NODE 1      │  │     WORKER NODE 2      │  │  NODE 3      │  │
│  │                        │  │                        │  │              │  │
│  │  ┌─────────────────┐   │  │  ┌─────────────────┐   │  │  ┌────────┐  │  │
│  │  │    kubelet      │   │  │  │    kubelet      │   │  │  │kubelet │  │  │
│  │  │ (node agent)    │   │  │  │ (node agent)    │   │  │  │        │  │  │
│  │  └─────────────────┘   │  │  └─────────────────┘   │  │  └────────┘  │  │
│  │  ┌─────────────────┐   │  │  ┌─────────────────┐   │  │  ┌────────┐  │  │
│  │  │   kube-proxy    │   │  │  │   kube-proxy    │   │  │  │kube-   │  │  │
│  │  │ (networking)    │   │  │  │ (networking)    │   │  │  │proxy   │  │  │
│  │  └─────────────────┘   │  │  └─────────────────┘   │  │  └────────┘  │  │
│  │  ┌─────────────────┐   │  │  ┌─────────────────┐   │  │  ┌────────┐  │  │
│  │  │ Container       │   │  │  │ Container       │   │  │  │contain-│  │  │
│  │  │ Runtime         │   │  │  │ Runtime         │   │  │  │erd     │  │  │
│  │  │ (containerd)    │   │  │  │ (containerd)    │   │  │  │        │  │  │
│  │  └─────────────────┘   │  │  └─────────────────┘   │  │  └────────┘  │  │
│  │                        │  │                        │  │              │  │
│  │  ┌──────┐ ┌──────┐    │  │  ┌──────┐ ┌──────┐    │  │  ┌──────┐   │  │
│  │  │Pod A │ │Pod B │    │  │  │Pod C │ │Pod D │    │  │  │Pod E │   │  │
│  │  └──────┘ └──────┘    │  │  └──────┘ └──────┘    │  │  └──────┘   │  │
│  └────────────────────────┘  └────────────────────────┘  └──────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role | Analogy |
|-----------|------|---------|
| **API Server** | Single entry point for ALL cluster operations. Every kubectl command, every internal component talks through it. | Reception desk — everything goes through here |
| **etcd** | Distributed key-value store. Holds entire cluster state (what's running, desired state, configs). | The filing cabinet — source of truth |
| **Scheduler** | Watches for new pods without a node assignment. Picks the best node based on resources, constraints, affinity. | Air traffic controller — decides where pods land |
| **Controller Manager** | Runs control loops. Each controller watches state and makes reality match desired state. | The thermostat — detects drift and corrects it |

### Worker Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Agent on every node. Receives pod specs from API server, ensures containers are running and healthy. Reports status back. |
| **kube-proxy** | Handles networking rules on each node. Implements Services (load-balances traffic to pods). Uses iptables or IPVS. |
| **Container Runtime** | Actually runs containers (containerd, CRI-O). Kubelet talks to it via CRI (Container Runtime Interface). |

### The Declarative Model — The Most Important Concept

```
IMPERATIVE (how Docker works):
  "Start 3 containers"
  "Stop container #2"
  "Start a replacement"
  YOU manage the state manually.

DECLARATIVE (how Kubernetes works):
  "I want 3 replicas running at all times"
  Kubernetes CONTINUOUSLY ensures reality matches your declaration.
  If one dies → Kubernetes starts a new one automatically.
  If a node fails → Kubernetes reschedules pods to healthy nodes.
  You declare WHAT you want. Kubernetes figures out HOW.

This is the RECONCILIATION LOOP:
  1. You submit desired state (YAML manifest)
  2. Kubernetes stores it in etcd
  3. Controllers detect: actual state ≠ desired state
  4. Controllers take action to reconcile
  5. Repeat forever
```

---

## 3. Pods — The Atomic Unit

### What is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It's a wrapper around one or more containers that:
- Share the same network namespace (same IP address)
- Share the same storage volumes
- Are always co-scheduled on the same node
- Share the same lifecycle (created and destroyed together)

### Why Pods, Not Individual Containers?

```
SINGLE-CONTAINER POD (99% of cases):
┌────────────────────────────┐
│  Pod                       │
│  IP: 10.244.1.5            │
│  ┌──────────────────────┐  │
│  │  Container: nginx     │  │
│  │  Port: 80             │  │
│  └──────────────────────┘  │
└────────────────────────────┘

MULTI-CONTAINER POD (sidecar pattern):
┌────────────────────────────────────────────┐
│  Pod                                       │
│  IP: 10.244.1.6 (shared by both)           │
│  ┌──────────────────┐  ┌────────────────┐  │
│  │  Container: app   │  │  Container:    │  │
│  │  Port: 8080       │  │  log-shipper   │  │
│  │                   │  │  (fluentbit)   │  │
│  │  Writes logs to   │  │  Reads from    │  │
│  │  /var/log/app     │  │  /var/log/app  │  │
│  └──────────────────┘  └────────────────┘  │
│          │                     │            │
│          └─────── shared volume ──────┘     │
└────────────────────────────────────────────┘

Multi-container pods: containers that MUST be co-located.
Examples: app + log shipper, app + proxy sidecar, app + config reloader
```

### Pod Manifest (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    version: v1
    environment: production
spec:
  containers:
    - name: app
      image: registry.example.com/myapp:sha-abc1234
      ports:
        - containerPort: 8080
      
      # Resource requests (scheduler uses these for placement)
      # Limits (container gets killed if it exceeds these)
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"       # 250 millicores = 0.25 CPU
        limits:
          memory: "512Mi"
          cpu: "500m"
      
      # Health checks
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
        failureThreshold: 3
      
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
      
      startupProbe:
        httpGet:
          path: /health/started
          port: 8080
        failureThreshold: 30
        periodSeconds: 2
      
      # Environment variables
      env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
      
      # Volume mounts
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true

  # Volumes available to all containers in this pod
  volumes:
    - name: config-volume
      configMap:
        name: app-config

  # Restart policy
  restartPolicy: Always     # Always, OnFailure, Never
```

### Probes — Keeping Pods Healthy

```
THREE TYPES OF PROBES:

1. STARTUP PROBE (checked first)
   "Has the application finished starting up?"
   - Runs until it succeeds, then hands off to liveness/readiness
   - Protects slow-starting apps from being killed prematurely
   - If it fails repeatedly → pod is killed and restarted

2. LIVENESS PROBE (checked continuously)
   "Is the application still alive / not deadlocked?"
   - If it fails → Kubernetes RESTARTS the container
   - Use case: detect deadlocks, infinite loops, corrupted state
   - WARNING: Don't make liveness check dependencies (DB, cache)
     If DB is down, your app isn't "dead" — restarting won't fix it

3. READINESS PROBE (checked continuously)
   "Can this pod handle traffic RIGHT NOW?"
   - If it fails → pod is REMOVED from Service load balancer (no traffic)
   - Container keeps running (not killed)
   - Use case: app is alive but can't serve (warming cache, waiting for DB)
   - When it passes again → pod is added back to the load balancer

PROBE TYPES:
  httpGet:   GET request to a path/port (200-399 = success)
  tcpSocket: TCP connection attempt (success if connection opens)
  exec:      Run a command inside container (exit 0 = success)
```

```
Timeline of probe behavior:

Container starts
   │
   ├── startupProbe runs ─────────────── succeeds after 10s
   │   (liveness/readiness paused)
   │
   ├── readinessProbe starts ──────────── passes → added to Service
   │   livenessProbe starts               (receives traffic)
   │
   ├── ... normal operation ...
   │
   ├── readinessProbe FAILS ───────────── removed from Service
   │   (pod still running, just no traffic)
   │
   ├── readinessProbe passes again ────── added back to Service
   │
   ├── livenessProbe FAILS 3x ────────── container RESTARTED
   │
   └── (cycle repeats)
```

### Pod Lifecycle

```bash
# Create a pod
kubectl apply -f pod.yaml

# Get pod status
kubectl get pods
kubectl get pods -o wide    # Shows node, IP

# Describe (detailed info, events, probe status)
kubectl describe pod my-app

# Logs
kubectl logs my-app
kubectl logs my-app -f                    # Follow
kubectl logs my-app -c sidecar           # Specific container in multi-container pod
kubectl logs my-app --previous            # Logs from previous crashed container

# Exec into a pod
kubectl exec -it my-app -- /bin/bash
kubectl exec my-app -- cat /app/config.json

# Port forward (access pod from localhost)
kubectl port-forward pod/my-app 8080:8080

# Delete
kubectl delete pod my-app
```

---

## 4. Deployments — Managing Application Lifecycle

### What is a Deployment?

You almost never create pods directly. A **Deployment** manages pods for you:

- Ensures the desired number of pod replicas are running
- Handles rolling updates (zero-downtime deploys)
- Supports rollback to previous versions
- Manages the underlying ReplicaSet

### Deployment → ReplicaSet → Pod Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT (my-app)                                            │
│  "I want 3 replicas of my-app:v2"                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  REPLICASET (my-app-7d6f9f5c4b) — current                 │  │
│  │  "Maintain exactly 3 pods matching this template"           │  │
│  │                                                             │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │  │
│  │  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │                    │  │
│  │  │ my-app  │  │ my-app  │  │ my-app  │                    │  │
│  │  │ :v2     │  │ :v2     │  │ :v2     │                    │  │
│  │  └─────────┘  └─────────┘  └─────────┘                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  REPLICASET (my-app-5c8d4e2a1f) — old (scaled to 0)       │  │
│  │  (kept for rollback history)                                │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Complete Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  # Number of pod replicas
  replicas: 3

  # How to find pods that belong to this deployment
  selector:
    matchLabels:
      app: my-app

  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during rollout (4 total briefly)
      maxUnavailable: 0    # Never take a pod down until new one is Ready

  # Keep this many old ReplicaSets for rollback
  revisionHistoryLimit: 10

  # Pod template — what each pod looks like
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      # Graceful shutdown
      terminationGracePeriodSeconds: 30

      containers:
        - name: app
          image: registry.example.com/myapp:sha-abc1234
          ports:
            - containerPort: 8080
              protocol: TCP
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          
          envFrom:
            - configMapRef:
                name: my-app-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password

      # Pod anti-affinity — spread replicas across nodes
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
                        - my-app
                topologyKey: kubernetes.io/hostname
```

### Rolling Update — How It Works

```
ROLLING UPDATE (maxSurge: 1, maxUnavailable: 0):

Step 1: Current state — 3 pods running v1
  [Pod1:v1 ✅] [Pod2:v1 ✅] [Pod3:v1 ✅]

Step 2: New pod created with v2 (maxSurge = 1 allows 4 total)
  [Pod1:v1 ✅] [Pod2:v1 ✅] [Pod3:v1 ✅] [Pod4:v2 ⏳ starting]

Step 3: New pod passes readiness probe
  [Pod1:v1 ✅] [Pod2:v1 ✅] [Pod3:v1 ✅] [Pod4:v2 ✅]

Step 4: Old pod terminated (maxUnavailable = 0: only after new is ready)
  [Pod2:v1 ✅] [Pod3:v1 ✅] [Pod4:v2 ✅] [Pod1:v1 terminating...]

Step 5: Next new pod created
  [Pod2:v1 ✅] [Pod3:v1 ✅] [Pod4:v2 ✅] [Pod5:v2 ⏳ starting]

... continues until all pods are v2 ...

Final: [Pod4:v2 ✅] [Pod5:v2 ✅] [Pod6:v2 ✅]

At NO POINT were fewer than 3 ready pods available = zero downtime
```

### Deployment Commands

```bash
# Create/update deployment
kubectl apply -f deployment.yaml

# Watch rollout progress
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=3

# Scale manually
kubectl scale deployment/my-app --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/my-app app=registry.example.com/myapp:sha-def5678

# Pause/resume rollout (for canary-like behavior)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Restart all pods (rolling restart)
kubectl rollout restart deployment/my-app
```

---

## 5. Services — Networking & Discovery

### The Problem Services Solve

Pods are ephemeral — they get new IP addresses every time they're recreated. You can't hardcode pod IPs. **Services** provide a stable endpoint (DNS name + IP) that load-balances across matching pods.

```
WITHOUT SERVICES:
  App A needs to call App B
  App B has 3 pods: 10.244.1.5, 10.244.2.3, 10.244.1.8
  Pod crashes → new IP: 10.244.3.2
  App A is now calling a dead IP → BROKEN

WITH SERVICES:
  Service "app-b" → stable IP: 10.96.45.12
  DNS: app-b.default.svc.cluster.local
  Automatically routes to healthy pods
  Pod dies and restarts → Service updates its endpoints automatically
  App A calls "http://app-b:8080" → always works
```

### Service Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. ClusterIP (default) — internal only                            │
│                                                                     │
│     ┌─────────────────────────────────┐                             │
│     │   Service: my-app               │                             │
│     │   ClusterIP: 10.96.45.12        │                             │
│     │   Port: 80                      │                             │
│     └────────────┬────────────────────┘                             │
│                  │ load balances                                    │
│           ┌──────┼──────┐                                          │
│           ▼      ▼      ▼                                          │
│       [Pod1]  [Pod2]  [Pod3]                                       │
│                                                                     │
│     Only accessible from INSIDE the cluster                        │
│                                                                     │
│  2. NodePort — accessible from outside via node IP                 │
│                                                                     │
│     External: http://node-ip:30080 → Service → Pods               │
│     Port range: 30000-32767                                        │
│                                                                     │
│  3. LoadBalancer — provisions cloud LB (AWS ALB, GCP LB)           │
│                                                                     │
│     External: http://lb-ip:80 → Service → Pods                    │
│     Cloud provider creates a real load balancer                    │
│     Most common for production external access                     │
│                                                                     │
│  4. ExternalName — DNS alias to external service                   │
│                                                                     │
│     Maps service name to an external DNS (e.g., RDS endpoint)      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Manifests

```yaml
# ClusterIP Service (internal)
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: my-app      # Routes to pods with label app=my-app
  ports:
    - port: 80            # Port the service listens on
      targetPort: 8080    # Port the pods actually listen on
      protocol: TCP

---
# NodePort Service (external via node ports)
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # Accessible at <any-node-ip>:30080

---
# LoadBalancer Service (cloud provider LB)
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### DNS in Kubernetes

```
Every Service gets a DNS entry automatically:

  <service-name>.<namespace>.svc.cluster.local

Examples:
  my-app.production.svc.cluster.local     → full FQDN
  my-app.production                       → short form (cross-namespace)
  my-app                                  → shortest (same namespace)

From a pod in "production" namespace:
  curl http://my-app:80          ← resolves to ClusterIP
  curl http://redis:6379         ← resolves to redis service

From a pod in a DIFFERENT namespace:
  curl http://my-app.production:80   ← must include namespace
```

### How Services Work (Internals)

```
1. You create a Service with selector: app=my-app
2. Kubernetes creates an Endpoints object listing all pod IPs matching that selector
3. kube-proxy on every node watches Endpoints
4. kube-proxy configures iptables/IPVS rules on its node:
   "Any traffic to 10.96.45.12:80 → round-robin to [pod1-ip:8080, pod2-ip:8080, pod3-ip:8080]"
5. When a pod is created/deleted, Endpoints are updated → kube-proxy updates rules

Result: Any pod on any node can reach the Service IP,
and traffic is distributed across healthy pods.
```

---

## 6. Ingress — External Traffic Routing

### What is Ingress?

A Service of type LoadBalancer creates one cloud LB per service — expensive. With 50 services, that's 50 load balancers.

**Ingress** is a single entry point that routes external HTTP/HTTPS traffic to different services based on hostname or URL path. One load balancer, many services.

```
WITHOUT INGRESS (expensive):
  api.example.com    → LoadBalancer 1 ($20/mo) → API Service
  web.example.com    → LoadBalancer 2 ($20/mo) → Web Service
  admin.example.com  → LoadBalancer 3 ($20/mo) → Admin Service
  Total: $60/month for 3 LBs

WITH INGRESS (efficient):
  ┌───────────────────────────────────────────┐
  │  Single Load Balancer ($20/mo)            │
  │           │                               │
  │           ▼                               │
  │  Ingress Controller (nginx/traefik)       │
  │           │                               │
  │     ┌─────┼──────────────┐                │
  │     │     │              │                │
  │     ▼     ▼              ▼                │
  │   /api  /web          /admin              │
  │     │     │              │                │
  │     ▼     ▼              ▼                │
  │  API Svc  Web Svc    Admin Svc           │
  └───────────────────────────────────────────┘
```

### Ingress Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    # NGINX Ingress Controller specific
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  
  # TLS certificate
  tls:
    - hosts:
        - api.example.com
        - www.example.com
      secretName: example-tls-cert
  
  rules:
    # Host-based routing
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
    
    - host: www.example.com
      http:
        paths:
          # Path-based routing
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: cdn-service
                port:
                  number: 80
```

### Ingress Controllers

The Ingress resource is just configuration. An **Ingress Controller** implements it (reads Ingress resources and configures actual routing).

| Controller | Best For |
|-----------|---------|
| NGINX Ingress | General purpose, most popular |
| Traefik | Auto-discovery, simple config, built-in dashboard |
| AWS ALB Ingress | AWS-native, integrates with ALB |
| Istio Gateway | Service mesh environments |
| Contour (Envoy) | High-performance, advanced routing |

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml
```

---

## 7. ConfigMaps & Secrets

### ConfigMaps — Non-Sensitive Configuration

```yaml
# ConfigMap definition
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Simple key-value pairs
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  
  # Entire config file as a value
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://app:8080;
        }
    }
  
  app.json: |
    {
      "feature_flags": {
        "new_dashboard": true,
        "beta_api": false
      }
    }
```

### Using ConfigMaps in Pods

```yaml
spec:
  containers:
    - name: app
      
      # Method 1: Load ALL keys as environment variables
      envFrom:
        - configMapRef:
            name: app-config
      
      # Method 2: Load specific keys as env vars
      env:
        - name: MY_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
      
      # Method 3: Mount as files in a volume
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        # Each key becomes a file:
        # /etc/config/DATABASE_HOST (content: "postgres.production...")
        # /etc/config/nginx.conf (content: the nginx config)
```

### Secrets — Sensitive Configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=              # base64("admin")
  password: c3VwZXJzZWNyZXQ=     # base64("supersecret")
  connection-string: cG9zdGdyZXNxbDovL2FkbWluOnN1cGVyc2VjcmV0QGRiOjU0MzIvYXBw

---
# Using stringData (plain text — Kubernetes encodes it for you)
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
stringData:
  STRIPE_SECRET_KEY: "sk_live_abc123..."
  JWT_SECRET: "my-jwt-signing-secret"
```

```bash
# Create secret from command line
kubectl create secret generic db-secret \
    --from-literal=username=admin \
    --from-literal=password=supersecret

# Create from file
kubectl create secret generic tls-cert \
    --from-file=cert.pem \
    --from-file=key.pem
```

### Important: Kubernetes Secrets Are NOT Encrypted

```
⚠️ Kubernetes Secrets are base64 ENCODED, not ENCRYPTED.
Anyone with kubectl access can decode them:
  kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

For real security:
  1. Enable etcd encryption at rest (encrypts secrets in etcd)
  2. Use RBAC to limit who can read secrets
  3. Use external secrets managers:
     - HashiCorp Vault (+ External Secrets Operator)
     - AWS Secrets Manager
     - GCP Secret Manager
     - Azure Key Vault
  4. Use Sealed Secrets (Bitnami) for GitOps
```

---

## 8. Storage — PersistentVolumes & PersistentVolumeClaims

### The Storage Problem

Pods are ephemeral. When a pod is rescheduled to a different node, local storage is lost. Databases, file uploads, and stateful workloads need storage that persists across pod restarts and rescheduling.

### Storage Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  POD                                                                 │
│  ┌──────────────────┐                                                │
│  │  Container        │                                                │
│  │  mount: /data ────────────────────┐                                │
│  └──────────────────┘                │                                │
│                                      ▼                                │
│                          PersistentVolumeClaim (PVC)                  │
│                          "I need 10Gi of SSD storage"                 │
│                                      │                                │
│                                      │ binds to                       │
│                                      ▼                                │
│                          PersistentVolume (PV)                        │
│                          "Here is 10Gi on AWS EBS gp3"               │
│                                      │                                │
│                                      │ backed by                      │
│                                      ▼                                │
│                          Actual Storage                               │
│                          (AWS EBS, GCP PD, NFS, Ceph)                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

ANALOGY:
  PVC = "I need a 2-bedroom apartment" (the request)
  PV  = "Here's apartment 4B in Building 3" (the actual unit)
  StorageClass = "Luxury apartments" vs "Budget apartments" (the type)
```

### StorageClass (Dynamic Provisioning)

```yaml
# StorageClass — defines what TYPE of storage to create
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com    # AWS EBS CSI driver
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain    # Keep the volume after PVC is deleted
volumeBindingMode: WaitForFirstConsumer   # Don't provision until pod is scheduled
allowVolumeExpansion: true
```

### PersistentVolumeClaim

```yaml
# PVC — request for storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce      # One pod at a time can mount RW
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

### Using PVC in a Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
```

### Access Modes

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | One node can mount read-write (most common) |
| ReadOnlyMany | ROX | Multiple nodes can mount read-only |
| ReadWriteMany | RWX | Multiple nodes can mount read-write (NFS, EFS) |

---

## 9. StatefulSets — Stateful Workloads

### When to Use StatefulSets vs Deployments

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod naming | Random (my-app-7d6f9-abc12) | Ordered (my-app-0, my-app-1, my-app-2) |
| Startup order | All at once | Sequential (0 first, then 1, then 2) |
| Storage | Shared or none | Each pod gets its own PVC |
| Network identity | Random | Stable hostname per pod |
| Use case | Stateless apps (APIs, web servers) | Databases, message queues, distributed systems |

### StatefulSet Guarantees

```
STATEFULSET "my-db" with 3 replicas:

Pods: my-db-0, my-db-1, my-db-2 (stable names, NEVER random)

Storage: Each pod gets its OWN PersistentVolume
  my-db-0 → pvc-my-db-data-my-db-0 → EBS vol-abc
  my-db-1 → pvc-my-db-data-my-db-1 → EBS vol-def
  my-db-2 → pvc-my-db-data-my-db-2 → EBS vol-ghi

Network: Each pod gets a stable DNS name via headless service
  my-db-0.my-db-headless.production.svc.cluster.local
  my-db-1.my-db-headless.production.svc.cluster.local
  my-db-2.my-db-headless.production.svc.cluster.local

Ordering:
  Scale up:   0 → 1 → 2 (sequential)
  Scale down: 2 → 1 → 0 (reverse order)
  Updates:    2 → 1 → 0 (reverse, one at a time)

If my-db-1 dies and is recreated:
  - It keeps the name "my-db-1"
  - It re-attaches to the SAME PVC (same data)
  - It gets the SAME DNS name
```

### StatefulSet Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless    # Required — headless service name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  
  # Each replica gets its own PVC (not shared)
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi

---
# Headless Service (required for StatefulSet DNS)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None      # "None" = headless — DNS returns pod IPs directly
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## 10. Namespaces — Multi-Tenancy & Organization

### What Are Namespaces?

Namespaces are **virtual clusters** within a physical cluster. They provide isolation for:
- Resource naming (two teams can both have a "frontend" deployment)
- Access control (RBAC per namespace)
- Resource quotas (limit CPU/memory per team)
- Network policies (restrict cross-namespace traffic)

```
KUBERNETES CLUSTER
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Namespace: team-a                                      │
│  ┌─────────────────────────────────────────────┐        │
│  │  Deployment: frontend                       │        │
│  │  Deployment: backend                        │        │
│  │  Service: frontend, backend                 │        │
│  │  ResourceQuota: 8 CPU, 16Gi memory          │        │
│  └─────────────────────────────────────────────┘        │
│                                                         │
│  Namespace: team-b                                      │
│  ┌─────────────────────────────────────────────┐        │
│  │  Deployment: frontend  (same name, no conflict!)│    │
│  │  Deployment: ml-service                      │       │
│  │  ResourceQuota: 16 CPU, 64Gi memory          │       │
│  └─────────────────────────────────────────────┘        │
│                                                         │
│  Namespace: monitoring  (cluster-wide tools)            │
│  ┌─────────────────────────────────────────────┐        │
│  │  Prometheus, Grafana, Loki                  │        │
│  └─────────────────────────────────────────────┘        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"

---
# LimitRange — default resource limits for pods in this namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

---

## 11. Autoscaling — HPA, VPA, Cluster Autoscaler

### Three Levels of Autoscaling

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Level 1: HPA (Horizontal Pod Autoscaler)                      │
│  "Add more pods when CPU/memory/custom metric is high"          │
│                                                                 │
│  Level 2: VPA (Vertical Pod Autoscaler)                        │
│  "Give each pod more CPU/memory when it needs it"               │
│                                                                 │
│  Level 3: Cluster Autoscaler                                   │
│  "Add more NODES when pods can't be scheduled"                  │
│                                                                 │
│  Traffic spike:                                                 │
│    1. HPA adds more pods                                        │
│    2. No node has capacity → pods are Pending                   │
│    3. Cluster Autoscaler provisions a new node                  │
│    4. Pending pods are scheduled on new node                    │
│    5. Traffic subsides → HPA removes pods                       │
│    6. Node is underutilized → Cluster Autoscaler removes node   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  
  minReplicas: 3
  maxReplicas: 50
  
  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Add pods when avg CPU > 70%
    
    # Scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Scale on custom metric (requests per second from Prometheus)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"    # 1000 RPS per pod
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up again
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60    # Add at most 4 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60    # Remove at most 10% pods per minute
```

### Cluster Autoscaler

```yaml
# Cluster Autoscaler configuration (EKS example)
# Typically deployed as a Deployment in kube-system namespace

# Key settings:
# --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled
# --scale-down-delay-after-add=10m        # Don't scale down within 10min of scale up
# --scale-down-unneeded-time=10m          # Node must be underutilized for 10min
# --scale-down-utilization-threshold=0.5  # Scale down if < 50% utilized
```

---

## 12. Helm — Package Manager for Kubernetes

### What is Helm?

Helm is the **package manager for Kubernetes** — like apt for Ubuntu or npm for Node.js. It packages multiple Kubernetes manifests into a single, configurable, versioned unit called a **chart**.

### Why Helm?

```
WITHOUT HELM:
  Deploy an app = apply 10+ YAML files manually:
    deployment.yaml, service.yaml, ingress.yaml,
    configmap.yaml, secret.yaml, hpa.yaml,
    serviceaccount.yaml, pdb.yaml, networkpolicy.yaml...
  
  Different environments? Copy all files, change values manually.
  Update? Edit files, track what changed.
  Rollback? Good luck remembering what was applied.

WITH HELM:
  helm install my-app ./chart --values production-values.yaml
  helm upgrade my-app ./chart --values production-values.yaml
  helm rollback my-app 3    # Instant rollback to revision 3
  helm uninstall my-app     # Clean removal of ALL resources
```

### Helm Chart Structure

```
my-app-chart/
├── Chart.yaml              # Chart metadata (name, version, description)
├── values.yaml             # Default configuration values
├── templates/              # Kubernetes manifests with Go templating
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl        # Reusable template snippets
│   └── NOTES.txt           # Post-install message
├── charts/                 # Sub-charts (dependencies)
└── .helmignore             # Files to exclude from package
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.2.0           # Chart version
appVersion: "2.5.1"      # Application version

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
# Default values — overridable per environment
replicaCount: 3

image:
  repository: registry.example.com/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hostname: myapp.example.com
  tls: true

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 70

postgresql:
  enabled: true
  auth:
    database: myapp
```

### Templated Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Helm Commands

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
helm search hub prometheus    # Search Artifact Hub

# Install a chart
helm install my-release bitnami/nginx
helm install my-app ./my-app-chart --values production.yaml --namespace production

# List installed releases
helm list -A    # All namespaces

# Upgrade (update existing release)
helm upgrade my-app ./my-app-chart --values production.yaml

# Rollback
helm rollback my-app 2    # Roll back to revision 2
helm history my-app       # See revision history

# Uninstall
helm uninstall my-app --namespace production

# Template locally (see what YAML would be generated without applying)
helm template my-app ./my-app-chart --values production.yaml

# Diff (see what would change on upgrade)
helm diff upgrade my-app ./my-app-chart --values production.yaml
```

### Environment-Specific Values

```bash
# Different values per environment
helm upgrade my-app ./chart \
    --values values.yaml \
    --values values-production.yaml \
    --set image.tag=sha-abc1234
```

```yaml
# values-production.yaml (overrides defaults)
replicaCount: 5
image:
  tag: "sha-abc1234"
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
ingress:
  hostname: api.production.example.com
```

---

## 13. RBAC — Role-Based Access Control

### RBAC Concepts

```
WHO           can do         WHAT              on WHICH RESOURCES?
(Subject)                    (Verbs)           (Resources)

ServiceAccount  ──────────►  get, list, watch  ──────► pods, services
User                         create, update             deployments
Group                        delete, patch              secrets
                             exec                       configmaps

Connected by: RoleBinding (namespace) or ClusterRoleBinding (cluster-wide)
```

### RBAC Manifests

```yaml
# ServiceAccount (identity for pods)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production

---
# Role (what permissions, within a namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]          # Can read secrets but not list/create/delete

---
# RoleBinding (connect ServiceAccount to Role)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole (cluster-wide permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-nodes
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

---

## 14. Production Patterns & Best Practices

### Pod Disruption Budget (PDB)

Prevents Kubernetes from taking down too many pods during voluntary disruptions (node drain, cluster upgrade):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2        # Always keep at least 2 pods running
  # OR
  # maxUnavailable: 1    # At most 1 pod can be down at a time
  selector:
    matchLabels:
      app: my-app
```

### Network Policies

```yaml
# Only allow traffic from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend     # Only frontend pods can reach API
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres     # API can only talk to postgres
      ports:
        - port: 5432
```

### Resource Management Best Practices

```yaml
# ALWAYS set resource requests and limits
resources:
  requests:
    memory: "256Mi"    # Scheduler uses this for placement
    cpu: "250m"        # Guaranteed minimum
  limits:
    memory: "512Mi"    # OOM killed if exceeded
    cpu: "1000m"       # Throttled if exceeded (not killed)

# RULES:
# 1. ALWAYS set memory limits (prevents one pod from starving the node)
# 2. requests.memory ≈ actual average usage
# 3. limits.memory = request × 1.5-2x (headroom for spikes)
# 4. CPU limits are controversial — some teams set them, some don't
#    (CPU throttling can cause latency spikes)
```

### Production Deployment Checklist

```
✅ Resource requests and limits set
✅ Liveness, readiness, and startup probes configured
✅ PodDisruptionBudget configured (minAvailable ≥ 2)
✅ Pod anti-affinity (spread across nodes/zones)
✅ HPA configured with sensible min/max
✅ Secrets from external manager (not plain K8s secrets)
✅ Network policies restricting traffic
✅ ServiceAccount with minimal RBAC
✅ Rolling update strategy (maxUnavailable: 0)
✅ terminationGracePeriodSeconds matches app shutdown time
✅ Container runs as non-root
✅ Read-only root filesystem where possible
✅ Image uses specific tag (not :latest)
✅ Resource quotas per namespace
```

---

## 15. Debugging Kubernetes

### Systematic Debugging Flow

```
Pod not running? Follow this flow:

kubectl get pods
  │
  ├── Status: Pending
  │     → kubectl describe pod <name>
  │     → Look at Events section
  │     → Common: insufficient resources, node selector mismatch, PVC not bound
  │
  ├── Status: CrashLoopBackOff
  │     → kubectl logs <pod> --previous
  │     → App is crashing on startup
  │     → Common: missing env var, wrong config, DB unreachable
  │
  ├── Status: ImagePullBackOff
  │     → kubectl describe pod <name>
  │     → Image doesn't exist or registry auth failed
  │     → Check: image name/tag, imagePullSecrets
  │
  ├── Status: Running but not working
  │     → kubectl logs <pod> -f
  │     → kubectl exec -it <pod> -- /bin/sh
  │     → kubectl port-forward pod/<pod> 8080:8080
  │     → Common: wrong port, misconfigured service selector
  │
  └── Status: Terminating (stuck)
      → kubectl delete pod <name> --grace-period=0 --force
      → (last resort — investigate why it's stuck first)
```

### Essential Debug Commands

```bash
# Get overview
kubectl get all -n production
kubectl get pods -o wide     # Shows node and IP
kubectl get events --sort-by='.lastTimestamp' -n production

# Deep dive on a resource
kubectl describe pod my-app-7d6f9-abc12
kubectl describe node worker-1

# Logs
kubectl logs my-app-7d6f9-abc12
kubectl logs my-app-7d6f9-abc12 --previous    # Crashed container's logs
kubectl logs -l app=my-app --all-containers   # All pods with a label
kubectl logs my-app-7d6f9-abc12 -c sidecar    # Specific container

# Interactive debugging
kubectl exec -it my-app-7d6f9-abc12 -- /bin/sh
kubectl port-forward svc/my-app 8080:80

# Debug networking
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
# From debug pod: nslookup my-service, curl http://my-service:80

# Resource usage
kubectl top pods -n production
kubectl top nodes

# Check RBAC
kubectl auth can-i get pods --as system:serviceaccount:production:my-app

# Dry run (validate without applying)
kubectl apply -f deployment.yaml --dry-run=server
```

---

## 16. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: What is Kubernetes and why do organizations use it?**

**Perfect Answer:**

"Kubernetes is a container orchestration platform originally developed by Google, now maintained by the CNCF. It automates the deployment, scaling, and management of containerized applications across clusters of machines.

Organizations use it because:

1. **Self-healing:** If a container crashes, Kubernetes automatically restarts it. If a node dies, pods are rescheduled to healthy nodes.

2. **Horizontal scaling:** Based on CPU, memory, or custom metrics, Kubernetes adds or removes pod replicas automatically.

3. **Zero-downtime deployments:** Rolling updates replace pods gradually — users never experience downtime.

4. **Service discovery and load balancing:** Containers find each other via DNS. Traffic is automatically distributed across healthy instances.

5. **Declarative management:** You declare the desired state ('run 5 replicas') and Kubernetes continuously reconciles reality to match.

6. **Portability:** Runs on AWS, GCP, Azure, on-premise, or bare metal. Avoids cloud vendor lock-in at the compute layer.

The core value proposition is that Kubernetes lets a small team reliably operate hundreds of microservices with the same effort it would take to operate a handful manually."

---

**Q2: Explain the difference between a Pod, Deployment, and Service.**

**Perfect Answer:**

"These are three fundamental abstractions that work together:

**Pod** — the smallest deployable unit. It's one or more containers that share networking and storage. You rarely create pods directly because they're not resilient — if a pod dies, nothing recreates it.

**Deployment** — manages pods. You declare 'I want 3 replicas of my app' and the Deployment ensures 3 pods are always running. It handles rolling updates, rollbacks, and scaling. Internally, it creates a ReplicaSet that maintains the pod count.

**Service** — provides stable networking. Pods get random IPs that change on restart. A Service gives a stable IP and DNS name that load-balances traffic to all pods matching its selector. Other services find it via DNS: `http://my-service:80`.

The relationship:
```
Deployment → creates ReplicaSet → creates Pods
Service → selects Pods by label → routes traffic to them
```

You define what runs (Deployment) and how it's accessed (Service) separately. This separation is key — you can update how the app runs without changing how it's accessed."

---

**Q3: What are the three types of probes in Kubernetes?**

**Perfect Answer:**

"Kubernetes has three probes that monitor container health:

**Startup Probe:** Checked first. Determines if the application has finished starting. Until it succeeds, liveness and readiness probes are disabled. This protects slow-starting applications (like Java apps that take 30 seconds to initialize) from being killed before they're ready.

**Liveness Probe:** Checked continuously after startup. Answers: 'Is this container still functioning, or is it in a broken state?' If it fails repeatedly, Kubernetes *restarts* the container. Use case: detecting deadlocks or corrupted internal state that only a restart can fix.

**Readiness Probe:** Checked continuously after startup. Answers: 'Can this container handle traffic right now?' If it fails, the pod is *removed from the Service's load balancer* — it stops receiving traffic but keeps running. When it passes again, traffic resumes. Use case: container is alive but temporarily can't serve (warming up cache, waiting for a dependency).

Critical best practice: Never make your liveness probe depend on external systems. If your liveness checks the database and the database goes down, Kubernetes will restart ALL your pods — making things worse. Readiness is the right probe for dependency checks."

---

### Intermediate Interview Questions

---

**Q4: How does a rolling update work in Kubernetes? How do you achieve zero-downtime deployments?**

**Perfect Answer:**

"When you update a Deployment's pod template (typically the image tag), Kubernetes performs a rolling update:

1. A new ReplicaSet is created for the new version
2. New pods are created one at a time (controlled by `maxSurge`)
3. Each new pod must pass its readiness probe before old pods are terminated
4. Old pods are terminated one at a time (controlled by `maxUnavailable`)
5. This continues until all pods are running the new version

For true zero-downtime, several things must be in place:

- **`maxUnavailable: 0`** — never reduce capacity below desired replicas
- **Readiness probes** — new pods only get traffic after they're ready
- **Graceful shutdown** — when old pods receive SIGTERM, they must finish in-flight requests before exiting. Set `terminationGracePeriodSeconds` appropriately.
- **Connection draining** — the Service removes the pod from its endpoints, but in-flight connections need time to complete. The app should stop accepting NEW connections but finish existing ones.
- **Pre-stop hook** — add a small delay (5 seconds) to allow kube-proxy to update iptables rules before the pod starts shutting down. This prevents traffic being sent to a terminating pod:
```yaml
lifecycle:
  preStop:
    exec:
      command: ['sh', '-c', 'sleep 5']
```

The `maxSurge` and `maxUnavailable` settings let you tune speed vs. resource usage:
- Fast rollout: `maxSurge: 50%, maxUnavailable: 50%` (but uses more resources)
- Safe rollout: `maxSurge: 1, maxUnavailable: 0` (slower but always at full capacity)"

---

**Q5: Explain how Kubernetes networking works. How does a request reach a pod?**

**Perfect Answer:**

"Kubernetes networking operates on three levels:

**1. Pod-to-Pod networking:**
Every pod gets its own IP address. Any pod can reach any other pod by IP without NAT. This is implemented by a CNI (Container Network Interface) plugin like Calico or Cilium. The CNI creates a flat network — pods on different nodes can still reach each other directly through routes or VXLAN tunnels.

**2. Service networking (internal):**
A ClusterIP Service gets a virtual IP (e.g., 10.96.45.12). kube-proxy on each node programs iptables rules that intercept traffic to this virtual IP and load-balance it to the backend pod IPs. When pods are added/removed, the Endpoints object is updated and kube-proxy reconfigures the rules.

DNS resolution: CoreDNS resolves `my-service.my-namespace.svc.cluster.local` to the Service's ClusterIP.

**3. External traffic:**
An Ingress Controller (like NGINX) runs as a pod with a LoadBalancer Service. The cloud LB routes external traffic to the Ingress Controller, which reads Ingress resources to decide which backend Service to route each request to based on host/path.

Full flow for an external request:
```
Client → DNS → Cloud LB → Ingress Controller pod → 
  reads Ingress rules → forwards to ClusterIP Service → 
    iptables/IPVS → selected Pod
```

Key principle: Pods communicate directly by IP. Services add a stable DNS name + load balancing layer on top. Ingress adds external routing on top of that."

---

**Q6: What is a StatefulSet and when would you use it instead of a Deployment?**

**Perfect Answer:**

"A StatefulSet is a workload controller for applications that need stable identity and persistent storage per instance.

The guarantees it provides that Deployments don't:

1. **Stable network identity:** Pods are named sequentially (postgres-0, postgres-1, postgres-2) with stable DNS names via a headless Service. This is critical for distributed systems where nodes need to know each other's addresses.

2. **Ordered operations:** Pods are created, updated, and deleted in order. Pod N+1 doesn't start until Pod N is Ready. This matters for databases that require initialization order (e.g., the first node bootstraps the cluster).

3. **Persistent storage per pod:** Each pod gets its own PersistentVolumeClaim that survives pod rescheduling. If postgres-1 is moved to a different node, it reattaches to the same volume with all its data.

Use cases:
- Databases (PostgreSQL, MySQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Distributed caches (Redis Cluster)
- Search engines (Elasticsearch)
- Any system where 'node identity' matters

I would NOT use a StatefulSet for:
- Stateless web servers, APIs — use Deployment
- Workers that process from a queue — use Deployment
- Anything that can tolerate random pod names and shared/no storage

A common interview follow-up: 'Should you run databases in Kubernetes?' My answer: it depends on team expertise. Managed services (RDS, Cloud SQL) are simpler to operate. But organizations with strong platform teams do run databases on Kubernetes successfully (using operators like CloudNativePG or Zalando's postgres-operator)."

---

### Advanced Interview Questions

---

**Q7: Explain the Kubernetes scheduler. How does it decide where to place a pod?**

**Perfect Answer:**

"The scheduler watches for newly created pods that have no node assigned (nodeName is empty). For each unscheduled pod, it runs a two-phase algorithm:

**Phase 1: Filtering (which nodes CAN run this pod?)**
Eliminates nodes that don't satisfy hard constraints:
- Insufficient resources (not enough CPU/memory for the pod's requests)
- Node selectors don't match
- Taints that the pod doesn't tolerate
- Pod affinity/anti-affinity rules (hard rules)
- PV/PVC topology constraints (storage is zone-specific)

**Phase 2: Scoring (which node is BEST?)**
Remaining nodes are scored 0-100 based on:
- Resource balance (prefer nodes that balance utilization evenly)
- Affinity preferences (soft rules — prefer co-location or spreading)
- Data locality (prefer nodes where required PVs already exist)
- Pod topology spread (distribute across zones/nodes)

The highest-scoring node wins. If there's a tie, one is chosen randomly.

**Advanced scheduling features:**

- **Taints and tolerations:** Nodes can be 'tainted' to repel pods. Only pods that explicitly tolerate the taint get scheduled there. Use case: GPU nodes should only run ML workloads.
```yaml
# Taint a node
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule

# Pod that tolerates it
tolerations:
  - key: gpu
    operator: Equal
    value: 'true'
    effect: NoSchedule
```

- **Pod topology spread:** Ensure pods are evenly distributed across failure domains:
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
```

- **Priority and preemption:** High-priority pods can evict lower-priority pods when resources are scarce.

In production, I ensure scheduling works correctly by: setting accurate resource requests, using topology spread for HA, using taints for specialized hardware, and monitoring for Pending pods (which indicate scheduling failures)."

---

**Q8: How would you handle a production outage where pods are in CrashLoopBackOff?**

**Perfect Answer:**

"CrashLoopBackOff means the container is starting, crashing, Kubernetes restarts it, it crashes again — in a loop with exponential backoff between restarts.

**Immediate response (under 5 minutes):**

1. **Check logs from the crashed container:**
```bash
kubectl logs <pod> --previous
```
The `--previous` flag shows logs from the last crashed instance. This usually reveals the error — missing environment variable, database connection refused, file not found, etc.

2. **Check recent changes:**
```bash
kubectl rollout history deployment/my-app
```
Was there a recent deployment? If yes, rollback immediately:
```bash
kubectl rollout undo deployment/my-app
```
Restore service first, investigate after.

3. **Describe the pod for more context:**
```bash
kubectl describe pod <pod>
```
Check Events — they might show OOMKilled (memory limit), failed mounts (PVC issues), or failed probes.

**Common root causes and fixes:**

| Symptom in logs | Root cause | Fix |
|-----------------|------------|-----|
| Connection refused to DB | Database unreachable | Check DB pod/service, check network policies |
| Missing env var | Secret/ConfigMap deleted or misconfigured | Restore the missing config |
| OOMKilled (exit code 137) | App exceeds memory limit | Increase limits or fix memory leak |
| Permission denied | Wrong user/filesystem permissions | Fix SecurityContext or volume permissions |
| No such file | Missing ConfigMap mount or code regression | Rollback or fix the mount |
| Exec format error | Wrong CPU architecture (arm vs amd64) | Rebuild image for correct platform |

**If I can't determine the cause from logs:**
```bash
# Override the command to get a shell (skip the crashing entrypoint)
kubectl run debug --image=registry.example.com/myapp:sha-abc --restart=Never \
    --command -- /bin/sh -c 'sleep 3600'
kubectl exec -it debug -- /bin/sh
# Now I can inspect the filesystem, test connections, etc.
```

**Post-incident:** Add monitoring for CrashLoopBackOff count, improve readiness probes, and ensure the deployment pipeline includes smoke tests in staging."

---

**Q9: Design a Kubernetes deployment architecture for a high-availability application serving 50,000 requests per second.**

**Perfect Answer:**

"For 50k RPS with high availability, I'd design this architecture:

**Cluster topology:**
- Multi-AZ cluster (minimum 3 availability zones)
- Control plane: managed (EKS/GKE/AKS) for HA without maintenance overhead
- Node pools: separate pools for different workload types (general, memory-optimized, GPU)

**Application layer:**
```yaml
Deployment:
  replicas: calculated based on load testing
  # If each pod handles 2000 RPS → need 25 pods minimum
  # Add 50% headroom → 38 pods
  # Spread across 3 AZs → ~13 per AZ
  
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0    # Never reduce capacity during deploy
  
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
  
  podAntiAffinity:
    # Don't put all pods on one node
    preferredDuringSchedulingIgnoredDuringExecution: [...]
```

**Autoscaling:**
```yaml
HPA:
  minReplicas: 38       # Enough for normal traffic
  maxReplicas: 150      # Enough for 3x spike
  metrics:
    - cpu: 60%          # Scale before saturation
    - custom: requests_per_second per pod
  behavior:
    scaleUp: aggressive (respond to spikes quickly)
    scaleDown: conservative (don't thrash on traffic dips)
```

**Networking:**
- Ingress: NGINX or Envoy-based Ingress Controller with multiple replicas
- Consider a service mesh (Istio/Linkerd) for mTLS, circuit breaking, retries
- Network policies: microsegmentation between services

**Reliability:**
- Pod Disruption Budgets: `minAvailable: 80%` (survive node maintenance)
- Circuit breakers: prevent cascade failures if downstream is slow
- Rate limiting at ingress level
- Readiness probes with dependency checks (remove pod from LB if DB unreachable)

**Data layer:**
- Application is stateless (state in Redis/PostgreSQL)
- Redis: Sentinel or Cluster mode with anti-affinity across AZs
- PostgreSQL: managed (RDS/Cloud SQL) with read replicas for read-heavy paths

**Observability (critical at this scale):**
- Prometheus + Grafana for metrics (per-pod, per-service, per-endpoint)
- Distributed tracing (Jaeger/Tempo) for latency debugging
- Centralized logging (Loki/ELK)
- Alerts on: error rate, latency p99, pod restarts, HPA at max, node resource exhaustion

**Deployment safety:**
- Canary deployments (route 5% traffic to new version first)
- Automated rollback if error rate increases
- Feature flags for gradual rollout of new features

This architecture handles single-AZ failures (pods redistributed), single-node failures (PDB + topology spread), and traffic spikes (HPA + cluster autoscaler)."

---

### Scenario-Based Questions

---

**Q10: Pods are stuck in Pending status. Walk me through debugging.**

**Perfect Answer:**

"Pending means the scheduler cannot find a suitable node. I debug systematically:

**Step 1: Get the reason from pod events:**
```bash
kubectl describe pod <pending-pod> | grep -A 20 Events
```

The Events section tells you exactly why:

**'Insufficient cpu' or 'Insufficient memory':**
- Pods are requesting more resources than available on any node
- Fix: reduce resource requests, add nodes, or check if other pods are over-requesting
```bash
kubectl top nodes    # Check actual node utilization
kubectl describe node <node> | grep -A 5 'Allocated resources'
```

**'0/5 nodes are available: 5 node(s) had taint that the pod didn't tolerate':**
- All nodes have taints the pod doesn't tolerate
- Fix: add a toleration to the pod, or taint fewer nodes

**'no persistent volumes available' or 'unbound PersistentVolumeClaims':**
- The PVC hasn't been bound (no matching PV, storage class can't provision)
- Fix: check StorageClass exists, check cloud provider quotas, check volumeBindingMode

**'0/5 nodes are available: 5 pod has unresolvable anti-affinity':**
- Pod anti-affinity rules can't be satisfied (e.g., requires each pod on a different node, but all nodes already have one)
- Fix: add more nodes, relax anti-affinity from required to preferred

**'FailedScheduling: no nodes matched node selector':**
- Pod has `nodeSelector` or `nodeAffinity` that no node matches
- Fix: label the appropriate nodes, or adjust the selector

**If there's no event yet:**
- Scheduler might be overwhelmed (very rare) or the pod is waiting for a PVC (WaitForFirstConsumer)

**Prevention:**
- Monitor for Pending pods (Prometheus alert on `kube_pod_status_phase{phase='Pending'} > 0` for more than 5 minutes)
- Use Cluster Autoscaler to add nodes when resources are exhausted
- Set sensible resource requests (not inflated)"

---

**Q11: You need to perform a Kubernetes cluster upgrade. How do you do it safely?**

**Perfect Answer:**

"Cluster upgrades are high-risk operations. I follow a structured process:

**Pre-upgrade:**
1. Read the release notes for breaking changes and deprecated APIs
2. Check if any manifests use deprecated API versions (`kubectl deprecations`)
3. Test the upgrade in a non-production cluster first
4. Ensure all PodDisruptionBudgets are configured
5. Verify backup of etcd (for self-managed clusters)
6. Ensure cluster autoscaler won't interfere (disable scale-down during upgrade)

**Upgrade order (always):**
1. Control plane first (API server, scheduler, controller manager)
2. Worker nodes second (one at a time or one node pool at a time)

**Control plane upgrade:**
For managed Kubernetes (EKS/GKE/AKS):
```bash
# EKS example
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.29
```
This is handled by the cloud provider — non-disruptive for running workloads.

**Worker node upgrade (the risky part):**
Use a rolling strategy — drain one node at a time:

```bash
# 1. Cordon the node (prevent new pods from being scheduled)
kubectl cordon worker-1

# 2. Drain the node (evict all pods, respecting PDBs)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data --grace-period=60

# 3. Upgrade the node (depends on method):
#    - Managed node groups: replace the node with a new one at the new version
#    - Self-managed: apt upgrade kubelet kubectl, restart kubelet

# 4. Uncordon (mark as schedulable again)
kubectl uncordon worker-1

# 5. Verify pods are rescheduled and healthy
kubectl get pods -o wide | grep worker-1
```

**Key safety measures:**
- PodDisruptionBudgets prevent draining from evicting too many pods at once
- The drain respects graceful termination periods
- Do it during low-traffic periods
- Monitor error rates during the process
- Have a rollback plan (for self-managed: keep old node around until verified)

**Post-upgrade:**
- Run the full test suite against the upgraded cluster
- Monitor for any new crash loops or errors
- Update deprecated API versions in manifests"

---

### Production Debugging Questions

---

**Q12: An application's response time has increased from 50ms to 2 seconds after a Kubernetes deployment. The pod is running and healthy. What do you check?**

**Perfect Answer:**

"The pod is healthy (probes pass) but slow. This is a performance issue, not a crash. I investigate:

**1. Check if the new code is the problem:**
```bash
# Compare revision history
kubectl rollout history deployment/my-app
# If recent deploy correlates with latency increase: rollback
kubectl rollout undo deployment/my-app
# If latency recovers → it's a code issue. Investigate the diff.
```

**2. Check resource throttling:**
```bash
kubectl top pods -l app=my-app
```
- CPU near limit → the pod is being **throttled**. CPU throttling adds latency without killing the pod or failing probes. Solution: increase CPU limit or fix the CPU-intensive code path.
- Memory near limit → possible GC pressure (JVM/Go garbage collection under memory pressure is slow)

**3. Check if the pod count dropped:**
```bash
kubectl get hpa my-app
```
Did HPA scale down? Fewer pods = more load per pod = higher latency.

**4. Check downstream dependencies:**
```bash
kubectl exec -it <pod> -- curl -w "@curl-format.txt" http://database:5432
kubectl exec -it <pod> -- curl -w "@curl-format.txt" http://redis:6379
```
If the database or cache is slow, the app will be slow even though it's 'healthy'. Check:
- Database query latency (check connection pool exhaustion)
- Redis latency
- External API latency

**5. Check for noisy neighbor / node issues:**
```bash
kubectl top nodes
kubectl get pods -o wide | grep <same-node>
```
Is another pod on the same node consuming excessive CPU/IO? Node-level resource contention can slow all pods on that node.

**6. Check network:**
- DNS resolution slowness (check CoreDNS pods and latency)
- Network policy changes that add latency
- Service mesh sidecar adding overhead

**7. Check application metrics (if available):**
- Request queue depth (are requests waiting in a queue?)
- Thread pool exhaustion (all threads busy?)
- Connection pool exhaustion (waiting for DB connections?)

In my experience, the most common causes are: CPU throttling (increase limits), connection pool exhaustion (increase pool size), or a regression in a code path that hit a slow query."

---

### Key Interview Terms

| Term | When to Use |
|------|------------|
| **Reconciliation loop** | Explaining Kubernetes's declarative model |
| **Desired state vs actual state** | Core Kubernetes philosophy |
| **Control plane** | Discussing architecture |
| **CNI plugin** | Networking (Calico, Cilium) |
| **CSI driver** | Storage (EBS CSI, NFS CSI) |
| **Operator pattern** | Managing complex stateful applications |
| **CRD (Custom Resource Definition)** | Extending Kubernetes |
| **Pod Disruption Budget** | High availability during maintenance |
| **Topology spread** | Multi-AZ resilience |
| **Admission controller** | Policy enforcement (OPA/Gatekeeper) |
| **Service mesh** | Advanced networking (Istio, Linkerd) |
| **GitOps (ArgoCD/Flux)** | Declarative deployment automation |

---

[⬇️ Download This File](#)

---

*Phase 6 Complete. Ready for Phase 7 — Cloud Platforms.*
