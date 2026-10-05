# Kubernetes Deep Dive — Hands-On Learning Notes

## Goal

Learn Kubernetes from **basic to intermediate** by implementing a real application rather than memorizing definitions.

Running example: a **FastAPI backend** deployed locally using Docker Desktop Kubernetes.

---

# 1. Kubernetes Roadmap

1. Why Kubernetes
2. Kubernetes Architecture
3. Cluster, Node, Control Plane
4. Pods
5. Containers inside Pods
6. Deployments
7. ReplicaSets
8. Services
9. Service Types
10. Labels & Selectors
11. Namespaces
12. ConfigMaps & Secrets
13. Volumes & Persistent Storage
14. Requests & Limits
15. Probes
16. Scheduling
17. Taints & Tolerations
18. Affinity
19. Scaling
20. Rolling Updates & Rollbacks
21. Ingress
22. Networking
23. RBAC
24. Jobs & CronJobs
25. StatefulSets & DaemonSets
26. Real application architecture
27. kubectl commands
28. Interview questions

---

# 2. What Exactly Is Kubernetes?

Kubernetes is a **container orchestration platform**.

The important word is **orchestration**.

Docker solves the problem of packaging and running applications in containers. Kubernetes manages those containerized applications at scale.

Typical Kubernetes responsibilities:

- Scheduling workloads
- Maintaining the desired number of replicas
- Restarting failed workloads
- Service discovery
- Load balancing
- Scaling
- Rolling updates
- Rollbacks
- Configuration management
- Secret management
- Storage management
- Health checks

---

# 3. Kubernetes Cluster

A Kubernetes cluster is a collection of machines managed by Kubernetes.

```text
                 Kubernetes Cluster
                        |
          +-------------+-------------+
          |                           |
     Control Plane               Worker Nodes
          |                    +------+------+------+
          |                    |      |      |      |
          |                   Node   Node   Node   ...
```

There are two major parts:

### Control Plane

The brain of Kubernetes. It makes decisions and maintains cluster state.

### Worker Nodes

Machines that actually run application workloads.

---

# 4. Control Plane Components

```text
                 Control Plane
                      |
        +-------------+-------------+
        |             |             |
   API Server     Scheduler    Controllers
        |
       etcd
```

## 4.1 kube-apiserver

The Kubernetes API Server is the main entry point to the cluster.

When you run:

```bash
kubectl get pods
```

conceptually:

```text
kubectl -> API Server -> Kubernetes resources
```

Kubernetes has an API-driven architecture.

---

## 4.2 etcd

`etcd` is Kubernetes' distributed key-value store.

It stores cluster state and configuration information.

Conceptually:

```text
API Server
    |
   etcd
    |
Cluster State
```

Important: **etcd is not your application database.**

---

## 4.3 kube-scheduler

The scheduler decides which worker node should run a newly created Pod.

It considers factors such as:

- CPU availability
- Memory availability
- Resource requests
- Node selectors
- Affinity
- Taints and tolerations
- Other scheduling constraints

Example:

```text
New Pod
   |
Scheduler
   |
+--+-----------+
|              |
Node 1       Node 2
CPU low      CPU available
   X              OK
                  |
                  Pod
```

---

## 4.4 Controller Manager

Controllers continuously compare:

```text
Desired State
      vs
Current State
```

Example:

```text
Desired = 3 Pods
Current = 2 Pods

Controller notices the difference
        |
        v
Creates another Pod
        |
        v
Current = 3 Pods
```

This continuous reconciliation is a fundamental Kubernetes concept.

---

# 5. Worker Node

A worker node runs application workloads.

Important components:

```text
Worker Node
|
+-- kubelet
+-- Container Runtime
+-- kube-proxy
+-- Pods
```

## kubelet

The kubelet is the agent running on each worker node.

Its job is to make sure Pods assigned to that node are running correctly.

Conceptually:

```text
Control Plane
      |
      | Run Pod
      v
   kubelet
      |
      v
Container Runtime
      |
      v
Container
```

## Container Runtime

The runtime actually runs containers.

Common runtimes include:

- containerd
- CRI-O

## kube-proxy

Helps implement Kubernetes Service networking and traffic forwarding on nodes.

---

# 6. Pod — The Smallest Deployable Unit

Kubernetes primarily manages **Pods**, not individual containers directly.

A Pod usually contains one container:

```text
Pod
└── Container
```

A Pod can also contain multiple tightly coupled containers:

```text
Pod
├── Main Container
└── Sidecar Container
```

Containers in the same Pod share:

- Network namespace
- Pod IP
- localhost
- Mounted volumes

---

# 7. Why Pods Exist

Suppose an application and a logging sidecar need to live together:

```text
Pod
├── Application
└── Logging Sidecar
```

They can communicate using localhost because they share the Pod network namespace.

---

# 8. Pod IPs Are Ephemeral

Example:

```text
Pod A -> 10.244.1.10
Pod B -> 10.244.1.11
Pod C -> 10.244.1.12
```

If Pod A dies, Kubernetes may create a replacement with a different IP:

```text
New Pod -> 10.244.1.20
```

Therefore applications should not depend directly on Pod IPs.

This leads to the **Service** concept.

---

# 9. Deployment

You normally should not manually manage individual Pods for an application.

Instead use a Deployment:

```text
Deployment
    |
    v
ReplicaSet
    |
    +-- Pod
    +-- Pod
    +-- Pod
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: backend

spec:
  replicas: 3

  selector:
    matchLabels:
      app: backend

  template:
    metadata:
      labels:
        app: backend

    spec:
      containers:
        - name: backend
          image: mybackend:1.0
          ports:
            - containerPort: 8000
```

This means Kubernetes should maintain three instances of the backend.

---

# 10. ReplicaSet

A ReplicaSet ensures that the desired number of Pods exists.

Example:

```text
Desired = 3
Current = 3
```

If one Pod dies:

```text
Desired = 3
Current = 2
```

ReplicaSet creates another Pod.

```text
Pod 1
Pod 3
   |
   v
ReplicaSet
   |
   v
Create Pod 4
```

The important hierarchy is:

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
     |
     v
Containers
```

---

# 11. Service

Pods are ephemeral and their IP addresses can change.

A Service provides a stable network endpoint.

```text
Service
   |
   +-- Pod 1
   +-- Pod 2
   +-- Pod 3
```

For example:

```text
backend-service:8000
```

The Service selects Pods using labels and selectors.

---

# 12. Labels and Selectors

Pod:

```yaml
metadata:
  labels:
    app: backend
```

Service:

```yaml
selector:
  app: backend
```

Conceptually:

```text
Service
 selector: app=backend
       |
       +-- Pod app=backend  <- selected
       +-- Pod app=backend  <- selected
       +-- Pod app=frontend <- ignored
```

Labels and selectors are fundamental to Kubernetes.

---

# 13. Service Types

## ClusterIP

Default Service type. Accessible inside the cluster.

```text
Frontend
   |
   v
Backend Service
   |
   v
Backend Pods
```

## NodePort

Exposes a Service through a port on each node.

```text
Client
  |
NodeIP:30080
  |
Service
  |
Pods
```

## LoadBalancer

Usually used with cloud environments.

```text
Internet
   |
Cloud Load Balancer
   |
Kubernetes Service
   |
Pods
```

---

# 14. Namespace

Namespaces logically separate resources within a cluster.

```text
Cluster
|
+-- development
|   +-- backend
|   +-- frontend
|
+-- testing
|   +-- backend
|   +-- frontend
|
+-- production
    +-- backend
    +-- frontend
```

The same resource name can exist in different namespaces.

Example:

```bash
kubectl get pods -n production
```

---

# 15. ConfigMap

A ConfigMap stores non-sensitive configuration.

Examples:

```text
API_URL
LOG_LEVEL
ENVIRONMENT
```

Instead of hard-coding configuration into the image, inject it into the Pod.

Example:

```yaml
env:
  - name: API_URL
    valueFrom:
      configMapKeyRef:
        name: backend-config
        key: API_URL
```

---

# 16. Secret

Secrets are intended for sensitive configuration.

Examples:

```text
DB_PASSWORD
API_KEY
JWT_SECRET
```

Important security point: Kubernetes Secrets should not be treated as automatically secure simply because the object is called a Secret. Storage encryption, access control, and cluster security must be configured appropriately.

---

# 17. Resource Requests and Limits

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

### Request

Resources Kubernetes uses when scheduling the Pod.

### Limit

Configured upper bound on resource consumption for the container.

Think:

```text
Request = What I need for scheduling
Limit   = Maximum configured consumption
```

---

# 18. Health Probes

Kubernetes needs to know whether an application is healthy.

## Liveness Probe

Question:

> Is the application alive?

Failure can cause the container to be restarted.

## Readiness Probe

Question:

> Is the application ready to receive traffic?

If it fails, the Pod can be removed from Service endpoints without necessarily restarting the container.

## Startup Probe

Useful for slow-starting applications, such as applications loading large ML models.

Example:

```text
Container starts
     |
Loading ML model
     |
45 seconds
     |
Application ready
```

---

# 19. Rolling Updates

Suppose the current version is v1:

```text
v1  v1  v1
```

Deploy v2:

```text
v2  v1  v1
v2  v2  v1
v2  v2  v2
```

Kubernetes gradually replaces old Pods with new Pods.

This is a **rolling update**.

---

# 20. Rollback

If v2 is broken, you can roll back.

```bash
kubectl rollout undo deployment/backend
```

Conceptually:

```text
v1
 |
v2
 |
Bug
 |
Rollback
 |
v1
```

---

# 21. Scaling

Manual scaling:

```bash
kubectl scale deployment backend --replicas=10
```

From:

```text
3 Pods
```

to:

```text
10 Pods
```

Kubernetes can also use the **Horizontal Pod Autoscaler (HPA)**.

Conceptually:

```text
Load increases
     |
     v
HPA
     |
     v
Pods increase
```

When load decreases, Pods can scale down.

---

# 22. Ingress

Ingress provides HTTP/HTTPS routing into Kubernetes services.

Example:

```text
                 Internet
                    |
                 Ingress
                    |
        +-----------+-----------+
        |           |           |
       /api        /login   /dashboard
        |           |           |
     Backend       Auth      Frontend
```

Important distinction:

- **Ingress** = Kubernetes API/resource describing HTTP routing rules.
- **Ingress Controller** = component that actually implements those rules.

---

# 23. Persistent Storage

Containers are generally ephemeral. Kubernetes provides storage abstractions for persistent data.

Important concepts:

```text
StorageClass
     |
     v
PersistentVolumeClaim (PVC)
     |
     v
PersistentVolume (PV)
     |
     v
Actual Storage
```

Typical flow:

```text
Pod
 |
 PVC
 |
 PV
 |
 Disk / Cloud Storage
```

---

# 24. Jobs

A Job runs a task to completion.

Example:

```text
Job
 |
 v
Pod
 |
 v
Database migration
 |
 v
Completed
```

Unlike a Deployment, a Job is not intended to keep an application continuously running.

---

# 25. CronJob

A CronJob runs Jobs on a schedule.

Example:

```text
Every day at 2 AM
        |
     CronJob
        |
       Job
        |
       Pod
        |
     Backup
```

---

# 26. StatefulSet

Deployments are commonly used for stateless applications.

StatefulSet is useful for workloads that need stable identity and/or persistent storage.

Example:

```text
StatefulSet
   |
   +-- postgres-0
   +-- postgres-1
   +-- postgres-2
```

Stateful workloads often need stable Pod identities and storage relationships.

---

# 27. DaemonSet

A DaemonSet runs a Pod on each eligible node.

Example:

```text
Node 1 -> Logging Agent
Node 2 -> Logging Agent
Node 3 -> Logging Agent
Node 4 -> Logging Agent
```

Common uses:

- Log collection
- Monitoring agents
- Node-level networking components

---

# 28. Scheduling

Suppose your cluster has:

```text
Node 1 -> CPU optimized
Node 2 -> GPU
Node 3 -> Memory optimized
```

An ML workload may need to run on the GPU node.

Kubernetes provides:

- Node labels
- Node selectors
- Node affinity
- Taints
- Tolerations

---

# 29. Node Selector

Label a node:

```text
hardware=gpu
```

Then specify:

```yaml
nodeSelector:
  hardware: gpu
```

The Pod is scheduled onto a matching node.

---

# 30. Taints and Tolerations

Suppose GPU nodes are expensive and should only run GPU workloads.

Taint the node:

```text
gpu=true:NoSchedule
```

Normal Pods:

```text
Cannot schedule
```

A Pod with a matching toleration can schedule there.

---

# 31. RBAC

RBAC = **Role-Based Access Control**.

It controls what users and workloads are allowed to do.

Important objects:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
ServiceAccount
```

Example policy idea:

```text
Developer
  |
  +-- get pods
  +-- view deployments
  +-- cannot delete production resources
```

---

# 32. Complete Request Flow

Suppose you run:

```bash
kubectl apply -f deployment.yaml
```

Conceptually:

```text
kubectl
   |
   v
API Server
   |
   v
etcd
   |
   v
Controller Manager
   |
   v
ReplicaSet
   |
   v
Scheduler
   |
   v
Worker Node
   |
 kubelet
   |
Container Runtime
   |
   v
Container
```

Application traffic can flow like:

```text
User
 |
v
Ingress / LoadBalancer
 |
v
Service
 |
v
Pod
 |
v
Container
```

---

# 33. Most Important Kubernetes Hierarchy

```text
Kubernetes Cluster
|
+-- Control Plane
|   +-- API Server
|   +-- Scheduler
|   +-- Controller Manager
|   +-- etcd
|
+-- Worker Nodes
    |
    +-- kubelet
    +-- Container Runtime
    +-- kube-proxy
    |
    +-- Pods
        |
        +-- Containers
```

Application management:

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
     |
     v
Containers
```

Networking:

```text
Internet
   |
   v
Ingress / LoadBalancer
   |
   v
Service
   |
   v
Pods
   |
   v
Containers
```

Configuration:

```text
ConfigMap ----+
              |
              v
             Pod
              ^
              |
Secret -------+
```

Storage:

```text
Pod
 |
 v
PVC
 |
 v
PV
 |
 v
Storage
```

---

# 34. Interview Priority

## Must Know

- Kubernetes
- Cluster
- Control Plane
- Worker Node
- API Server
- etcd
- Scheduler
- Controller Manager
- kubelet
- Container Runtime
- Pod
- Deployment
- ReplicaSet
- Service
- ClusterIP
- NodePort
- LoadBalancer
- Labels and Selectors
- Namespace
- ConfigMap
- Secret
- Requests and Limits
- Liveness / Readiness / Startup probes
- Rolling Update
- Rollback
- HPA
- Ingress

## Intermediate

- PersistentVolume
- PersistentVolumeClaim
- StorageClass
- StatefulSet
- DaemonSet
- Job
- CronJob
- NodeSelector
- Affinity
- Taints and Tolerations
- RBAC
- ServiceAccount
- Kubernetes networking

## Later / Advanced

- Service Mesh
- Helm
- Operators
- CRDs
- CNI
- CSI
- Admission Controllers
- Network Policies
- Pod Security
- GitOps
- Argo CD
- Prometheus
- Grafana

---

# 35. The Mental Model

Do not memorize Kubernetes as isolated definitions.

Think:

```text
                 KUBERNETES
                     |
             "Desired State"
                     |
          +----------+----------+
          |                     |
          v                     v
     CONTROL PLANE          WORKER NODES
          |                     |
    Makes decisions          Runs Pods
          |                     |
    +-----+-----+               v
    |     |     |              Pod
    v     v     v               |
   API  Scheduler Controllers   v
  Server                       Container
    |
   etcd
```

The most important principle is:

> Kubernetes continuously tries to make the actual state of the cluster match the desired state you declare.

---

# 36. Hands-On Learning Strategy

For every Kubernetes concept, use this sequence:

```text
CONCEPT
   |
   v
WHY DOES IT EXIST?
   |
   v
WRITE YAML
   |
   v
kubectl apply
   |
   v
OBSERVE WITH kubectl
   |
   v
BREAK SOMETHING
   |
   v
WATCH KUBERNETES RESPOND
   |
   v
UNDERSTAND THE INTERNAL FLOW
```

This is more valuable than memorizing YAML.

---

# 37. Current Hands-On Project

We are using:

```text
Docker Desktop
        |
        v
Local Kubernetes Cluster
        |
        v
FastAPI Application
```

Current project structure:

```text
kubernetesK8s
|
+-- main
|   +-- app.py
|
+-- k8s
|   +-- pod.yaml
|
+-- Dockerfile
+-- requirements.txt
```

The Docker image is:

```text
kubernetesk8s:1.0
```

Current learning path:

```text
FastAPI
   |
   v
Dockerfile
   |
   v
Docker Image
   |
   v
Kubernetes Pod
   |
   v
Service
   |
   v
Frontend / Postman
```

---

# 38. Next Hands-On Session

Next, connect the frontend and API and test everything using Postman.

Planned flow:

```text
React Frontend
      |
      | HTTP Request
      v
Kubernetes Service
      |
      v
FastAPI Pods
      |
      v
FastAPI API
```

We will cover:

- GET requests
- POST requests
- JSON request bodies
- FastAPI endpoints
- Postman testing
- Kubernetes Service networking
- Frontend -> backend communication
- Port mapping
- Pod IP vs Service IP
- Environment variables
- CORS
- Eventually Ingress

The goal is to understand the entire request path, not just make it work.
