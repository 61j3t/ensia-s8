# Week 5 — Orchestration: Introduction to Kubernetes (K8s)

> This file merges two source documents: the main 29-slide Kubernetes deck and the
> 5-page Minikube hands-on supplement ("Deploy Your First Service with a Persistent
> Database"). Minikube material is integrated in Section 6.

---

## Bird's eye view

- **Kubernetes (K8s)** is the open-source platform (Greek for "Helmsman") that automates
  the deployment, scaling, and management of containerised applications. Born from
  Google's internal "Borg" system, it is now the industry standard for cloud-native
  orchestration.
- **The problem it solves**: Docker solved packaging (portable images), but at scale you
  need something to ask "Where should this container run? Is it alive? Do we need more?"
  Kubernetes is that "construction manager."
- **Cluster anatomy**: one **Control Plane** (the brain — decisions, scheduling, state)
  plus N **Worker Nodes** (the muscle — workload execution).
- **Declarative model**: you describe the *desired state* in YAML; K8s continuously
  reconciles *actual state* toward it. If a Pod dies, the controller spins up a
  replacement automatically.
- **Core capabilities**: automated bin-packing, self-healing, horizontal scaling, rolling
  updates with rollback.
- **Docker vs Kubernetes**: not either/or. Docker (the brick) builds and runs images on a
  single host; Kubernetes (the building) manages those images across a cluster.
- **Minikube** is a tool that runs a single-node Kubernetes cluster locally on your
  laptop for development and learning — both control-plane and worker node in one VM/
  container.

---

## Detailed notes

### 1. Why orchestration? The motivation

#### 1.1 The monolith-to-microservices shift

| Monolith | Microservices |
|---|---|
| Single codebase | Independent deployable features |
| High-risk updates ("one bug crashes everything") | High scalability |
| Easy to operate | Exponential management complexity |

#### 1.2 The container gap

Docker solved *packaging*:
- Portable artifacts (images)
- Dependencies bundled inside
- Works on any machine

But it created a *management problem* at scale:
- How do you track 1,000 containers?
- How do you enable multi-cloud?
- How do you ensure zero-downtime?

**"Containers built the bricks. We are missing the construction manager."**

#### 1.3 The role of orchestration

Three questions orchestration answers:
1. **Scheduling** — Where should this workload run?
2. **Lifecycle** — Is it still alive?
3. **Scaling** — Do we need more instances?

#### 1.4 Kubernetes as the orchestrator

Analogy: K8s is the conductor; containers are the musicians; YAML files are the sheet
music. K8s ensures the symphony plays correctly.

Key functions:
- **Coordinates**: runs containers at scale across many nodes
- **Maintains**: auto-replaces failed instances (self-healing)
- **Directs**: schedules workloads to specific nodes (bin packing)

#### 1.5 When to use Kubernetes vs Docker/Swarm

| Use Docker / Swarm if... | Use Kubernetes if... |
|---|---|
| Simple setup is priority | High availability is critical |
| Development / local testing | Complex microservices |
| Lightweight workloads | Auto-scaling is required |
| Single host operation | Multi-cloud strategy |

Note: AWS (EKS), Azure (AKS), and Google (GKE) offer managed Kubernetes to reduce setup
complexity.

---

### 2. Cluster architecture

#### 2.1 The 10,000-ft view

The diagram shows a rectangular cluster boundary. On the left sits the **Control Plane**
(a tall server rack labelled "The Brain / Decisions, Scheduling & State"). The client /
user arrow points into the control plane. On the right are three worker-node boxes
(smaller cubes) labelled "Workload Execution." A dashed vertical line separates the two
halves.

#### 2.2 Control plane components (the brain)

The control-plane diagram shows the **API Server** in the centre. Arrows connect it to
three surrounding components:

| Component | Role |
|---|---|
| **API Server** | The single entry point for all requests (kubectl, other components). Everything talks through it. |
| **etcd** | Distributed key-value store — the source of truth for all cluster state. "The cluster brain." |
| **Scheduler** | Watches for unscheduled Pods and assigns them to a suitable node (considers resources, affinity rules, bin-packing). |
| **Controller Manager** | Runs control loops that continuously compare desired state vs actual state and take corrective action. |

Flow: `kubectl` sends a request → **API Server** → stores intent in **etcd** → **Scheduler**
picks a node → **Controller Manager** ensures pods match desired count.

#### 2.3 Worker node components (the muscle)

Each worker node runs three critical processes:

| Component | Role |
|---|---|
| **Kubelet** | The agent. Receives pod specs from the control plane and ensures the containers described in those specs are running and healthy. |
| **Container Runtime** | The engine that actually pulls images and runs containers (e.g., `containerd`, Docker). |
| **Kube-proxy** | The networker. Maintains iptables/network rules on the node to implement Services (load balancing, routing). |

The worker node diagram shows the Kubelet (top) pointing down to the Container Runtime
layer which hosts multiple Pods, with Kube-proxy at the bottom managing the network
layer.

#### 2.4 Node health

- **Self-Registration**: Kubelets automatically register with the API server on startup.
- Node states: `Ready`, `NotReady`, `Unknown`.
- **Graceful Shutdown** flow: Node Shutdown Signal → Kubelet Notifies Pods → Pods Save
  State (30-second window) → Termination. Ensures no data loss.

---

### 3. Pods — the atomic unit

#### 3.1 What a Pod is

- The **smallest deployable unit** in Kubernetes. You do not deploy containers directly;
  you deploy Pods.
- A Pod is a **logical wrapper** (logical host) around one or more containers.
- All containers inside a Pod share:
  - The same **IP address** and port space
  - The same **storage volumes**
  - A `localhost` network (they talk to each other via localhost)

#### 3.2 Pod characteristics

- **Ephemeral**: Pods are created, destroyed, and never repaired — they are cattle, not
  pets.
- **Co-located**: all containers in a Pod always run on the same Node.
- **Shared context**: a sidecar container (e.g., a log shipper) sits alongside the main
  app container within the same Pod and communicates via shared memory/volume.

#### 3.3 Pod lifecycle

```
Pending  →  Running  →  Succeeded / Failed
```

#### 3.4 Multi-container patterns

| Pattern | How it works |
|---|---|
| **Init Container** | Runs to completion *before* the main app container starts (e.g., pre-seed config, wait for DB). |
| **Sidecar** | Runs *alongside* the main container for the full lifetime of the Pod (e.g., log shipper, proxy). |

#### 3.5 Minimal Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: main-app
    image: nginx:latest
    ports:
    - containerPort: 80
```

---

### 4. Workload controllers

#### 4.1 The desired-state reconciliation loop

You never manage Pods manually in production. Instead you give a **Deployment** (or
another controller) a blueprint; K8s continuously reconciles:

```
Desired State (YAML)  →  Reconciliation (Controller)  →  Actual State
         ^                    (if Diff != 0,                  |
         |_____________________ action taken) ________________|
```

#### 4.2 ReplicaSet

- Ensures a specified number of Pod replicas are running at all times.
- If one crashes, a replacement is created instantly.
- Rarely used directly; managed by Deployments.

#### 4.3 Deployment

- **Declarative updates**: you say *what* you want, K8s figures out *how* to get there.
- Wraps a ReplicaSet; adds update and rollback capabilities.
- Supports **rolling updates** — gradually replaces old Pods with new ones to prevent
  downtime (v1 Pods replaced one by one with v2 Pods).
- Supports **rollbacks** — instant undo if a bad version is pushed.

```bash
# Scale manually
kubectl scale deployment web-app --replicas=5

# Trigger a rollback
kubectl rollout undo deployment/web-app

# Check rollout status
kubectl rollout status deployment/web-app
```

Full Deployment YAML example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

#### 4.4 Advanced deployment strategy: Canary releases

Route a small fraction (e.g., 10%) of real user traffic to a new version while 90% stays
on the stable version. If metrics look good, gradually shift more traffic. Kubernetes
enables this via label selectors and multiple Deployments sharing one Service.

#### 4.5 Horizontal Pod Autoscaler (HPA)

Automatically adjusts the number of replicas based on CPU or memory utilisation:
```bash
kubectl autoscale deployment web-app --cpu-percent=50 --min=2 --max=10
```

#### 4.6 Other workload types (overview)

| Controller | Use case |
|---|---|
| **DaemonSet** | Runs exactly one Pod on every node (e.g., log collector, monitoring agent). |
| **StatefulSet** | For stateful apps that need stable network identity and persistent storage (e.g., databases). Pods get predictable names (pod-0, pod-1). |
| **Job** | Runs a Pod to completion once. |
| **CronJob** | Runs a Job on a schedule (like cron). |

---

### 5. Networking — Services

#### 5.1 The core problem

Pods are ephemeral; when they die and are recreated they get a new IP address. You cannot
hardcode Pod IPs. **Services** provide a stable virtual IP (ClusterIP) that always points
to the current healthy Pods matching a label selector.

Networking stack (from outside to inside):
```
Internet → Ingress (HTTP/S routing) → Service (stable address) → Pods (ephemeral)
```

#### 5.2 Service types

| Type | Scope | Use case |
|---|---|---|
| **ClusterIP** | Internal only (default) | Microservice-to-microservice communication |
| **NodePort** | External via a static port (30000–32767) on each node | Dev/testing, internal tools |
| **LoadBalancer** | External via a cloud load balancer | Production public-facing apps |
| **Ingress** | HTTP/HTTPS layer-7 routing (hostname/path-based) | Single entry point for multiple services |

The concentric-circle diagram shows ClusterIP at the centre (internal only), NodePort as
the next ring (exposed on a static port), and LoadBalancer as the outer ring reaching the
public internet/cloud.

#### 5.3 Service discovery

Applications do not need to know *where* a Pod is. They ask the **Service** by name. K8s
DNS resolves `<service-name>.<namespace>.svc.cluster.local` to the ClusterIP.

#### 5.4 Service YAML example (NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

---

### 6. Storage — Persistent Volumes

#### 6.1 The problem with ephemeral storage

By default, container storage dies with the Pod. For databases or any stateful data, you
need storage that **survives Pod restarts**.

#### 6.2 Storage objects

| Object | Role |
|---|---|
| **PersistentVolume (PV)** | Cluster-level storage resource (actual disk). Provisioned by admin or dynamically via StorageClass. |
| **PersistentVolumeClaim (PVC)** | A Pod's request for storage. Kubernetes binds it to a matching PV automatically. |
| **StorageClass** | Defines the "class" of storage (SSD, HDD, hostPath) and enables dynamic provisioning. |

Pods do not reference PVs directly — they reference PVCs.

#### 6.3 PV YAML (Minikube hostPath example)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/postgres
```

#### 6.4 PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### 7. Configuration — ConfigMaps and Secrets

**Separating code from configuration** allows updating env vars or DB URLs without
rebuilding container images.

| Object | Stores | Example content |
|---|---|---|
| **ConfigMap** | Non-sensitive data | DB URLs, feature flags, env variables |
| **Secret** | Sensitive data (base64-encoded) | Passwords, API tokens, TLS certificates |

Both are injected into Pods as environment variables or mounted as files.

ConfigMap example:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://db:5432/mydb"
  LOG_LEVEL: "info"
```

Secret example:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: bXlwYXNzd29yZA==   # base64("mypassword")
```

---

### 8. Labels, selectors, and namespaces

#### 8.1 Labels

Key/value pairs attached to any K8s object. They are how controllers and Services find
their Pods:
```yaml
labels:
  app: nginx
  env: production
  version: "1.25"
```

#### 8.2 Selectors

A selector filters objects by label:
```yaml
selector:
  matchLabels:
    app: nginx
```

Services use `selector` to determine which Pods receive traffic. Deployments use it to
identify the Pods they own.

#### 8.3 Namespaces

Logical isolation boundaries within a single cluster. Used to separate teams,
environments (dev/staging/prod), or projects. Resource names must be unique within a
namespace but can repeat across namespaces.

```bash
kubectl create namespace demo
kubectl config set-context --current --namespace=demo
kubectl get pods -n demo
```

---

### 9. Resource management

The Scheduler uses resource declarations to place Pods efficiently (bin packing):

```yaml
resources:
  requests:
    cpu: "100m"       # minimum guaranteed
    memory: "128Mi"
  limits:
    cpu: "250m"       # maximum allowed (noisy-neighbour protection)
    memory: "256Mi"
```

- **Requests**: minimum guaranteed resources. Used by the Scheduler.
- **Limits**: hard cap. Container is throttled (CPU) or OOM-killed (memory) if exceeded.
- **Affinity**: rules to co-locate or spread Pods across nodes.

---

### 10. Security overview

Security is defence-in-depth:

| Layer | Mechanism |
|---|---|
| **Image Security** | Scan images for CVEs; use minimal base images to reduce attack surface |
| **Network Policies** | Firewall rules between Pods (deny-all by default, allow only what's needed) |
| **Node Hardening** | Reduce OS-level vulnerabilities on the worker nodes |

---

### 11. The Kubernetes ecosystem pipeline

```
Code File → Container Image (Docker) → Pod (atomic unit) →
Deployment (manager/desired state) → Service (bridge/stable address) → User
```

Standard command-line interface for all interactions: `kubectl`.

---

### 12. Minikube — local Kubernetes hands-on

> Source: 05_week5_minikube.pdf — "Deploy Your First Service with a Persistent Database"

#### 12.1 What Minikube is

Minikube runs a **single-node Kubernetes cluster** locally. The single node acts as both
the control plane and a worker node. It is the standard tool for learning K8s without
cloud infrastructure.

Prerequisites: virtualisation enabled in BIOS, Docker running, `kubectl` installed,
`minikube` installed.

#### 12.2 Core Minikube commands

```bash
minikube start            # create and start the local cluster
minikube stop             # stop the cluster (preserves state)
minikube status           # show current cluster state
minikube dashboard        # open the Kubernetes web dashboard in browser
minikube service <name>   # open a NodePort service URL in browser
```

#### 12.3 End-to-end deployment walkthrough

The tutorial deploys a PostgreSQL database (with persistent storage) and an NGINX web
server, demonstrating PV/PVC, Deployments, and Services together.

**Step 1 — Start cluster and verify**

```bash
minikube start
kubectl get nodes          # should show STATUS = Ready
```

**Step 2 — Create a namespace (optional but recommended)**

```bash
kubectl create namespace demo
kubectl config set-context --current --namespace=demo
```

**Step 3 — Create the Persistent Volume**

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/postgres
```

```bash
kubectl apply -f pv.yaml
```

**Step 4 — Create the Persistent Volume Claim**

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc.yaml
```

**Step 5 — Deploy PostgreSQL**

The volume mount connects `/var/lib/postgresql/data` inside the container to the
persistent volume — so data survives Pod deletion.

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
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
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: mypassword
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-storage
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

```bash
kubectl apply -f postgres-deployment.yaml
```

**Step 6 — Expose PostgreSQL internally**

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: LoadBalancer
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres-service.yaml
```

**Step 7 — Deploy NGINX web app**

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f web-deployment.yaml
```

**Step 8 — Expose web app externally**

```yaml
# web-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

```bash
kubectl apply -f web-service.yaml
minikube service web      # opens browser at the NodePort URL
```

**Step 9 — Verify persistence**

Delete the database Pod. K8s recreates it via the Deployment controller. Data is
preserved because it lives on the PV, not inside the container.

```bash
kubectl delete pod -l app=postgres
```

**Step 10 — Troubleshooting commands**

```bash
kubectl get pods                    # list all pods and their state
kubectl describe pod <name>         # detailed info + events
kubectl logs <name>                 # container stdout/stderr
kubectl get pv,pvc                  # storage status and binding
kubectl get all                     # overview of all resources
```

---

### 13. Essential kubectl reference

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Pods
kubectl get pods
kubectl get pods -o wide             # show which node each pod runs on
kubectl describe pod <name>
kubectl logs <name>
kubectl exec -it <name> -- /bin/bash # interactive shell into a pod

# Apply / delete manifests
kubectl apply -f <file.yaml>
kubectl delete -f <file.yaml>
kubectl delete pod <name>

# Deployments
kubectl get deployments
kubectl scale deployment <name> --replicas=5
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>

# Services
kubectl get services
kubectl describe service <name>

# Storage
kubectl get pv,pvc

# Namespaces
kubectl get namespaces
kubectl create namespace <name>
```

---

## Key terms

| Term | Definition |
|---|---|
| **Kubernetes (K8s)** | Open-source container orchestration platform; automates deployment, scaling, and management. |
| **Cluster** | A set of machines (nodes) managed by Kubernetes as a single unit. |
| **Control Plane** | The "brain" — API Server, etcd, Scheduler, Controller Manager. |
| **API Server** | The single entry point for all Kubernetes requests; validates and forwards them. |
| **etcd** | Distributed key-value store holding all cluster state. |
| **Scheduler** | Assigns Pods to nodes based on resource availability and constraints. |
| **Controller Manager** | Runs reconciliation loops to enforce desired state. |
| **Worker Node** | A machine that runs Pods; contains Kubelet, Container Runtime, and Kube-proxy. |
| **Kubelet** | Agent on each worker node; receives pod specs and manages containers. |
| **Kube-proxy** | Maintains network rules on each node to implement Service routing. |
| **Container Runtime** | Engine that pulls images and runs containers (e.g., containerd). |
| **Pod** | Smallest deployable unit; one or more containers sharing IP and volumes. |
| **ReplicaSet** | Ensures N replicas of a Pod are always running. |
| **Deployment** | Manages ReplicaSets; enables declarative updates, rolling updates, and rollbacks. |
| **DaemonSet** | Runs one Pod per node (e.g., log agents). |
| **StatefulSet** | Manages stateful Pods with stable network identities and ordered scaling. |
| **Job / CronJob** | Run-to-completion workloads; CronJob adds a schedule. |
| **Service** | Stable virtual IP + DNS name that routes traffic to matching Pods. |
| **ClusterIP** | Default Service type; internal only. |
| **NodePort** | Exposes a Service on a static port on each node. |
| **LoadBalancer** | Cloud load balancer in front of a Service for production external access. |
| **Ingress** | HTTP/HTTPS layer-7 routing rules (host/path-based) directing to Services. |
| **PersistentVolume (PV)** | Cluster-level storage resource. |
| **PersistentVolumeClaim (PVC)** | A Pod's request for storage; bound to a PV. |
| **StorageClass** | Defines storage type and enables dynamic PV provisioning. |
| **ConfigMap** | Stores non-sensitive configuration as key-value pairs. |
| **Secret** | Stores sensitive data (base64-encoded); passwords, tokens. |
| **Namespace** | Logical isolation boundary within a cluster. |
| **Label** | Key/value metadata tag on a K8s object. |
| **Selector** | Filter that matches objects by label; used by Services and controllers. |
| **Desired state** | What you declare in YAML — K8s continuously works to match it. |
| **Reconciliation loop** | Controller pattern: compare desired vs actual state, take action if different. |
| **Rolling update** | Gradual Pod replacement during deployment to avoid downtime. |
| **HPA** | Horizontal Pod Autoscaler — scales replicas based on CPU/memory metrics. |
| **Bin packing** | Scheduler strategy to maximise hardware utilisation by fitting Pods efficiently. |
| **Minikube** | Tool for running a single-node K8s cluster locally for development. |
| **kubectl** | Command-line tool for interacting with a Kubernetes cluster. |

---

## Exam targets

1. **Explain the "container gap"** — why Docker alone is insufficient at scale and what
   three problems orchestration solves (scheduling, lifecycle, scaling).

2. **Draw and label the full cluster architecture**: Control Plane (API Server, etcd,
   Scheduler, Controller Manager) + Worker Nodes (Kubelet, Container Runtime, Kube-proxy)
   + Pods. Show where `kubectl` connects.

3. **Describe the Pod**: why it is the atomic unit, what "shared context" means (shared
   IP, shared volumes, localhost), and why Pods are ephemeral (never repaired, only
   replaced).

4. **Explain the desired-state reconciliation loop** — how a Deployment + Controller
   Manager enforces replicas. Walk through: YAML applied → API Server → etcd → Scheduler
   → Kubelet.

5. **Distinguish Service types** (ClusterIP, NodePort, LoadBalancer, Ingress) and choose
   the right one for a scenario. Explain why Services are needed (Pod IPs are ephemeral).

6. **Explain PV vs PVC**: who creates each, how binding works, why it matters for
   stateful applications like databases.

7. **Rolling updates vs rollbacks**: describe the rolling-update flow (v1 Pods replaced
   one-by-one with v2) and the command to undo.

8. **Minikube end-to-end**: list the steps to deploy a service with persistent storage
   (start cluster, PV, PVC, Deployment with volumeMount, Service, verify persistence by
   deleting and recreating Pod).

9. **Compare Docker and Kubernetes**: Docker = brick (builds/runs on single host,
   dev-focus); Kubernetes = building (manages across cluster, ops-focus). Not either/or.

10. **ConfigMap vs Secret**: when to use each; how they decouple config from image.

---

## Pitfalls

- **You deploy Pods, not containers.** The unit of deployment is the Pod. Containers are
  inside Pods. A common mistake is thinking of `docker run` as equivalent to `kubectl run
  container`.
- **Pods are ephemeral — never repaired, only replaced.** If a Pod dies, a new one is
  created with a new name and a new IP. Any data stored inside the container filesystem
  is lost unless a PV/PVC is used.
- **Services do not load-balance to Pods by IP — they use label selectors.** If you
  change a label on a Pod without updating the selector, the Service stops routing to it.
- **ClusterIP is internal only.** You cannot access a ClusterIP Service from outside the
  cluster without a NodePort, LoadBalancer, or Ingress. A very common beginner mistake.
- **etcd is the SPOF for cluster state.** In production, etcd must be run as an HA
  cluster (3 or 5 nodes). Loss of etcd = loss of the cluster.
- **ConfigMaps are not encrypted.** Never store passwords in a ConfigMap — use Secrets.
  Note: Secrets are only base64-encoded by default, not encrypted; production clusters
  require Secrets encryption at rest or an external vault.
- **Rolling updates do not guarantee zero downtime automatically.** You must configure
  `readinessProbes` so K8s knows when a new Pod is actually ready before removing an old
  one.
- **Minikube's `hostPath` PVs are node-local.** Data is stored on the Minikube VM; it
  does not survive `minikube delete`. This is fine for learning but never for production.
- **`kubectl delete pod` on a Deployment-managed Pod does not delete the Deployment.**
  The controller immediately recreates the Pod. To permanently remove it, delete the
  Deployment.
- **Resource `requests` vs `limits`**: forgetting `requests` means the Scheduler has no
  data to make placement decisions. Forgetting `limits` means a misbehaving container can
  starve its neighbours (noisy-neighbour problem).
