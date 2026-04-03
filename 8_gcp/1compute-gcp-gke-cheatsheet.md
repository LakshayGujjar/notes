# GCP Google Kubernetes Engine (GKE) Cheatsheet

> **TL;DR:** GKE is GCP's fully managed Kubernetes service — choose Autopilot for hands-off, per-Pod serverless Kubernetes, or Standard for full node-level control; both give you Google-managed control planes, deep GCP integrations, and enterprise-grade security out of the box.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [GKE Standard vs. Autopilot Deep Dive](#2-gke-standard-vs-autopilot-deep-dive)
3. [Cluster Architecture](#3-cluster-architecture)
4. [Node Pools & Compute](#4-node-pools--compute)
5. [Networking](#5-networking)
6. [Workload Management](#6-workload-management)
7. [Storage](#7-storage)
8. [Security](#8-security)
9. [Autoscaling](#9-autoscaling)
10. [CI/CD & GitOps](#10-cicd--gitops)
11. [Observability](#11-observability)
12. [IAM & Access Control](#12-iam--access-control)
13. [Multi-Cluster & Fleet Management](#13-multi-cluster--fleet-management)
14. [Cost Optimization](#14-cost-optimization)
15. [kubectl & gcloud CLI Quick Reference](#15-kubectl--gcloud-cli-quick-reference)
16. [Terraform Snippet](#16-terraform-snippet)
17. [Common Patterns & Best Practices](#17-common-patterns--best-practices)
18. [Troubleshooting Quick Reference](#18-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

**Google Kubernetes Engine (GKE)** is GCP's fully managed Kubernetes service. Google manages the control plane (API server, etcd, scheduler, controller manager) for free or at a fixed fee, while you manage workloads. GKE provides deep integrations with GCP services: Cloud Load Balancing, Persistent Disk, IAM, Cloud Logging, Cloud Monitoring, Artifact Registry, and more.

### Key Terminology

| Term | Description |
|---|---|
| **Cluster** | A set of nodes running containerized workloads, managed by a Kubernetes control plane |
| **Node** | A worker VM (GCE instance) that runs Pods |
| **Node Pool** | A group of nodes within a cluster that share the same configuration (machine type, OS, labels) |
| **Pod** | The smallest deployable unit — one or more containers sharing network and storage |
| **Deployment** | A controller managing a ReplicaSet of identical Pods with rolling update support |
| **Service** | A stable network endpoint (ClusterIP/NodePort/LoadBalancer) routing traffic to a set of Pods |
| **Namespace** | A virtual cluster within a cluster — used for resource isolation and multi-tenancy |
| **Control Plane** | The Kubernetes brain (API server, scheduler, etcd, controller manager) — Google-managed in GKE |
| **Data Plane** | The worker nodes where your Pods actually run |
| **kubelet** | Agent running on each node; ensures containers run as specified in PodSpecs |
| **kube-proxy** | Maintains network rules on nodes for Service routing |
| **etcd** | Distributed key-value store holding all cluster state |
| **kubectl** | CLI tool for interacting with the Kubernetes API server |
| **Ingress** | An API object managing external HTTP(S) access to Services, typically via a Load Balancer |
| **PersistentVolume (PV)** | A storage resource provisioned in the cluster |
| **PersistentVolumeClaim (PVC)** | A request for storage by a Pod |
| **ConfigMap** | Non-sensitive key-value configuration data injected into Pods |
| **Secret** | Sensitive key-value data (passwords, tokens) — stored encrypted in etcd in GKE |
| **Workload Identity** | GKE feature linking Kubernetes Service Accounts to GCP Service Accounts for IAM-based GCP API access |
| **Fleet** | A group of GKE clusters managed together (formerly Anthos) |
| **Release Channel** | A stream of GKE version updates (Rapid, Regular, Stable) |

### GKE vs. Other GCP Compute Options

| Feature | GKE | Cloud Run | App Engine | Cloud Functions | GCE |
|---|---|---|---|---|---|
| **Deploy unit** | Pod (container) | Container | App/Service | Function | VM |
| **Orchestration** | Kubernetes | Managed | Managed | Managed | Manual |
| **Scale to zero** | ❌ (Autopilot: partial) | ✅ | ✅ (Standard) | ✅ | ❌ |
| **Concurrency/instance** | Configurable | Up to 1000 | Configurable | 1–1000 | N/A |
| **Stateful workloads** | ✅ (StatefulSet) | ❌ | ❌ | ❌ | ✅ |
| **Custom networking** | ✅ Full | Limited | Limited | Limited | ✅ Full |
| **GPU/TPU support** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Infra management** | Node-level (Standard) / None (Autopilot) | None | None | None | Full |
| **Max container runtime** | Unlimited | 60 min | 60 min | 60 min | Unlimited |
| **Best for** | Microservices, complex orchestration | Simple APIs, serverless | Web apps | Event handlers | Full VM control |

### Self-Managed Kubernetes vs. GKE

| Aspect | Self-Managed K8s | GKE |
|---|---|---|
| Control plane management | Your responsibility | Google-managed |
| Upgrades | Manual, error-prone | Automated (release channels) |
| etcd backup | Your responsibility | Google-managed |
| Cloud integrations | Manual configuration | Native, built-in |
| SLA | None (DIY) | 99.95% (regional) |
| Cost | Infrastructure + ops time | Control plane fee + nodes |

---

## 2. GKE Standard vs. Autopilot Deep Dive

### Side-by-Side Comparison

| Attribute | GKE Standard | GKE Autopilot |
|---|---|---|
| **Node management** | Customer manages nodes and pools | Google manages all nodes |
| **Pricing model** | Per node (GCE VM cost) + $0.10/hr control plane | Per Pod (CPU + memory + ephemeral storage) |
| **Minimum cost** | ~$73/month (1 node) + control plane | ~$0 (no idle cost) |
| **Scale to zero** | ❌ Nodes always running | ✅ Pods scale to zero |
| **Node pool customization** | ✅ Full (machine type, disk, GPU, taints) | ❌ Google chooses node hardware |
| **GPU/TPU support** | ✅ | ✅ (A100, L4, TPU v4/v5) |
| **Node SSH access** | ✅ | ❌ |
| **DaemonSets** | ✅ | ⚠️ Limited (system only) |
| **Privileged containers** | ✅ | ❌ |
| **HostPath volumes** | ✅ | ❌ |
| **Security posture** | Customer-configured | Hardened by default (Shielded nodes, no SSH) |
| **Resource requests required** | Optional | ✅ Mandatory (billing basis) |
| **Cluster Autoscaler** | Optional (enable manually) | Always on |
| **Node Auto-Provisioning** | Optional | N/A (Google provisions) |
| **Spot/preemptible nodes** | ✅ | ✅ (Spot Pods) |
| **Binary Authorization** | ✅ | ✅ |
| **Max Pods per node** | 110 (configurable) | 32 per node (Google-managed) |
| **Best for** | Complex workloads, HPC, GPU, databases | Stateless microservices, batch, dev/test |

### When to Choose Standard

- You need GPUs, TPUs, or specific machine types for AI/ML workloads.
- You run stateful databases (PostgreSQL, MySQL) with specific disk requirements.
- You need DaemonSets for node-level agents (custom monitoring, security).
- You have legacy workloads requiring privileged containers or `hostPath` volumes.
- You want full control over node configuration, OS tuning, or custom runtime.
- Cost predictability is critical and you run sustained, consistent workloads.

### When to Choose Autopilot

- You run stateless microservices, APIs, or batch workloads.
- You want zero node management — no patching, no capacity planning.
- Your traffic is variable (Autopilot scales to zero → zero idle cost).
- You prioritize security — Autopilot enforces hardened defaults you can't accidentally disable.
- You're building a new application and want the simplest operational experience.

> **Note:** Autopilot charges are based on **requested** CPU/memory/storage — not actual usage. Always set accurate resource requests. Over-requesting wastes money; under-requesting causes OOMKills.

---

## 3. Cluster Architecture

### ASCII Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GKE Cluster (Regional)                           │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Control Plane  (Google-Managed)                │   │
│  │  ┌──────────────┐ ┌───────────┐ ┌──────────────────────┐   │   │
│  │  │  API Server  │ │ Scheduler │ │  Controller Manager  │   │   │
│  │  └──────┬───────┘ └─────┬─────┘ └──────────┬───────────┘   │   │
│  │         │               │                  │               │   │
│  │  ┌──────▼───────────────▼──────────────────▼───────────┐   │   │
│  │  │                    etcd                             │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │           Cloud Controller Manager                  │   │   │
│  │  │  (LB, PD, Routes — integrates with GCP APIs)        │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │ Kubernetes API                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Data Plane (Nodes)                        │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │  │
│  │  │  Node (Zone A) │  │  Node (Zone B) │  │  Node (Zone C) │ │  │
│  │  │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │ │  │
│  │  │  │  kubelet │  │  │  │  kubelet │  │  │  │  kubelet │  │ │  │
│  │  │  │kube-proxy│  │  │  │kube-proxy│  │  │  │kube-proxy│  │ │  │
│  │  │  │containerd│  │  │  │containerd│  │  │  │containerd│  │ │  │
│  │  │  │ Pod Pod  │  │  │  │ Pod Pod  │  │  │  │ Pod Pod  │  │ │  │
│  │  │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │ │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|---|---|
| **API Server** | The front-end for the K8s control plane; validates and processes all API requests |
| **Scheduler** | Assigns Pods to nodes based on resource availability, affinity, taints, and tolerations |
| **Controller Manager** | Runs control loops (Deployment, ReplicaSet, Node, Job controllers) |
| **etcd** | Distributed key-value store — source of truth for all cluster state |
| **Cloud Controller Manager** | Bridges K8s with GCP APIs: provisions Load Balancers, PersistentDisks, and routes |

### Node Components

| Component | Role |
|---|---|
| **kubelet** | Ensures containers described in PodSpecs are running and healthy |
| **kube-proxy** | Maintains iptables/ipvs rules for Service routing; implements ClusterIP/NodePort |
| **containerd** | Container runtime (replaced Docker in GKE 1.24+) |

### Zonal vs. Regional Clusters

| Attribute | Zonal Cluster | Regional Cluster |
|---|---|---|
| **Control plane** | Single zone | Replicated across 3 zones |
| **Node pool zones** | Single zone (default) or multi-zone | 3 zones by default |
| **SLA** | No control plane SLA | 99.95% uptime SLA |
| **Cost** | Lower (1 control plane) | Slightly higher (replicated CP) |
| **Resilience** | Zone failure = cluster outage | Zone failure = reduced capacity, no outage |
| **Best for** | Dev/test | Production |

### Release Channels

| Channel | GKE Version Lag | Auto-upgrade | Best for |
|---|---|---|---|
| **Rapid** | Latest (1–2 weeks after GA) | Yes | Early adopters, feature testing |
| **Regular** | 2–3 months behind latest | Yes | Most production workloads (recommended) |
| **Stable** | 4–6 months behind latest | Yes | Risk-averse production, compliance |
| **None** | Manual | No | Full version control (not recommended) |

```bash
# Create a regional cluster on the Regular release channel
gcloud container clusters create my-cluster \
  --region=us-central1 \
  --release-channel=regular \
  --num-nodes=1 \
  --enable-ip-alias \
  --workload-pool=my-project.svc.id.goog
```

### Cluster Upgrade Strategies

```bash
# Surge upgrade: replace nodes in waves (default)
gcloud container node-pools update my-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --max-surge-upgrade=1 \
  --max-unavailable-upgrade=0   # Zero-downtime: add before removing

# Blue/green node pool upgrade: create new pool, drain old, delete old
gcloud container node-pools create my-pool-v2 \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=n2-standard-4

kubectl cordon <old-node>      # Prevent new Pods
kubectl drain <old-node> \     # Evict existing Pods
  --ignore-daemonsets \
  --delete-emptydir-data

gcloud container node-pools delete my-pool-v1 \
  --cluster=my-cluster \
  --region=us-central1
```

---

## 4. Node Pools & Compute

### Create and Manage Node Pools

```bash
# Create a standard node pool
gcloud container node-pools create standard-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=n2-standard-4 \
  --num-nodes=3 \
  --disk-type=pd-ssd \
  --disk-size=100GB \
  --image-type=COS_CONTAINERD \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10

# List node pools
gcloud container node-pools list \
  --cluster=my-cluster \
  --region=us-central1

# Delete a node pool
gcloud container node-pools delete old-pool \
  --cluster=my-cluster \
  --region=us-central1
```

### Machine Type Selection Guide

| Workload | Recommended Series | Notes |
|---|---|---|
| General purpose APIs | `n2-standard-*` | Best price/performance for most |
| Memory-intensive (Java, DBs) | `n2-highmem-*` | 8 GB RAM per vCPU |
| CPU-intensive (encoding, math) | `c2-standard-*` | High-frequency Intel |
| Cost-sensitive batch | `e2-standard-*` | Cheapest; variable performance |
| ML inference (GPU) | `n1-standard-*` + GPU | Attach A100/L4/T4 |
| ARM workloads | `t2a-standard-*` | Ampere Altra; cheaper for scale-out |

### Spot / Preemptible Node Pools

```bash
# Spot node pool (recommended over preemptible — better availability)
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=n2-standard-4 \
  --spot \
  --num-nodes=0 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=20 \
  --node-taints=cloud.google.com/gke-spot=true:NoSchedule
```

```yaml
# Toleration to run Pods on Spot nodes
spec:
  tolerations:
    - key: cloud.google.com/gke-spot
      operator: Equal
      value: "true"
      effect: NoSchedule
  nodeSelector:
    cloud.google.com/gke-spot: "true"
```

### GPU Node Pool

```bash
# Create a GPU node pool (NVIDIA L4)
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=g2-standard-8 \
  --accelerator=type=nvidia-l4,count=1 \
  --num-nodes=0 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=4 \
  --node-taints=nvidia.com/gpu=present:NoSchedule
```

```yaml
# Pod requesting a GPU
spec:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  containers:
    - name: ml-job
      image: tensorflow/tensorflow:latest-gpu
      resources:
        limits:
          nvidia.com/gpu: "1"
```

### Node Taints and Labels

```bash
# Add a taint to a node pool (prevents general workloads from landing here)
gcloud container node-pools update dedicated-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --node-taints=dedicated=ml:NoSchedule

# Add labels to a node pool
gcloud container node-pools update dedicated-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --node-labels=team=ml,environment=prod
```

### Node Auto-Provisioning (NAP)

NAP automatically creates and deletes node pools to meet Pod scheduling demands.

```bash
# Enable NAP on a cluster
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --enable-autoprovisioning \
  --max-cpu=100 \
  --max-memory=400 \
  --autoprovisioning-scopes=https://www.googleapis.com/auth/cloud-platform
```

### Confidential GKE Nodes

```bash
# Create node pool with AMD SEV (memory encryption)
gcloud container node-pools create confidential-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=n2d-standard-4 \
  --enable-confidential-nodes
```

> **Note:** Confidential nodes use AMD SEV to encrypt VM memory in hardware. They require N2D machine types and cannot be mixed with standard nodes in the same pool.

---

## 5. Networking

### VPC-Native Networking (Alias IP)

```bash
# Create a VPC-native cluster with custom CIDR ranges
gcloud container clusters create my-cluster \
  --region=us-central1 \
  --enable-ip-alias \
  --network=my-vpc \
  --subnetwork=my-subnet \
  --cluster-secondary-range-name=pods \      # Pod CIDR from subnet secondary range
  --services-secondary-range-name=services   # Service CIDR from subnet secondary range
```

| CIDR | Default Range | Notes |
|---|---|---|
| Pod CIDR | `/14` (~250k IPs) | Allocated to nodes in `/24` blocks (110 Pods/node) |
| Service CIDR | `/20` (~4k IPs) | Virtual IPs for Services |
| Node CIDR | From subnet primary range | Actual node IPs |

### Services

```yaml
# ClusterIP — internal only
apiVersion: v1
kind: Service
metadata:
  name: my-app-clusterip
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP   # Default

---
# LoadBalancer — provisions a GCP External Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  annotations:
    cloud.google.com/load-balancer-type: "External"  # or "Internal"
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer

---
# Internal LoadBalancer (only accessible within VPC)
apiVersion: v1
kind: Service
metadata:
  name: my-app-internal-lb
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
spec:
  selector:
    app: my-app
  ports:
    - port: 80
  type: LoadBalancer
```

### GKE Ingress (HTTP(S) Load Balancer)

```yaml
# GKE Ingress — provisions a GCP HTTP(S) Load Balancer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
    networking.gke.io/managed-certificates: "my-cert"
spec:
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /v1/*
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-v1-service
                port:
                  number: 80
          - path: /v2/*
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-v2-service
                port:
                  number: 80
```

```yaml
# Managed Certificate for Ingress TLS
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: my-cert
spec:
  domains:
    - api.mycompany.com
```

### Gateway API (Modern Replacement for Ingress)

```yaml
# Gateway (provisions a GCP Load Balancer)
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-cert

---
# HTTPRoute — routes requests to backends
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - api.mycompany.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
          weight: 100
```

### Network Policies

```yaml
# Deny all ingress by default, then allow selectively
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # Applies to ALL pods in namespace
  policyTypes:
    - Ingress

---
# Allow ingress from pods with label app=frontend to app=backend on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
```

```bash
# Enable Dataplane V2 (eBPF-based, required for Network Policy in new clusters)
gcloud container clusters create my-cluster \
  --region=us-central1 \
  --enable-dataplane-v2
```

### Private Clusters

```bash
# Create a private cluster (nodes have no public IPs)
gcloud container clusters create private-cluster \
  --region=us-central1 \
  --enable-private-nodes \
  --enable-private-endpoint \              # Also make control plane private
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias \
  --enable-master-authorized-networks \
  --master-authorized-networks=10.0.0.0/8  # Only VPC-internal access to API server
```

### Cloud NAT for Egress

```bash
# Allow private nodes to reach the internet via Cloud NAT
gcloud compute routers create my-nat-router \
  --network=my-vpc \
  --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-nat-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1
```

---

## 6. Workload Management

### Pod (Fundamental Unit)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: us-central1-docker.pkg.dev/my-project/repo/app:v1.0.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"       # 0.25 vCPU guaranteed
          memory: "256Mi"
        limits:
          cpu: "500m"       # 0.5 vCPU max burst
          memory: "512Mi"
      env:
        - name: ENV
          value: "production"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 20
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Create 1 extra pod during update
      maxUnavailable: 0     # Zero-downtime: never remove before adding
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
    spec:
      serviceAccountName: my-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: app
          image: us-central1-docker.pkg.dev/my-project/repo/app:v1.0.0
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
```

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # Headless Service required for stable DNS
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
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:             # Each Pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: premium-rwo
        resources:
          requests:
            storage: 100Gi
```

### DaemonSet

```yaml
# Run one Pod per node — ideal for log agents, monitoring, security scanners
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentbit
          image: fluent/fluent-bit:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-export
spec:
  schedule: "0 2 * * *"         # 2 AM UTC daily
  concurrencyPolicy: Forbid      # Don't run overlapping jobs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: export-sa
          containers:
            - name: exporter
              image: us-central1-docker.pkg.dev/my-project/repo/exporter:v1
              resources:
                requests:
                  cpu: "500m"
                  memory: "512Mi"
```

### Quality of Service (QoS) Classes

| QoS Class | Condition | Eviction Priority |
|---|---|---|
| **Guaranteed** | `requests == limits` for all containers | Last to be evicted |
| **Burstable** | `requests < limits` (partial) | Middle priority |
| **BestEffort** | No requests or limits set | First to be evicted |

> **Note:** Always set resource `requests` and `limits` for production workloads. BestEffort pods are the first to be killed during node memory pressure, causing unexpected outages.

### Pod Disruption Budget

```yaml
# Ensure at least 2 replicas always available during voluntary disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2     # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

### Topology Spread Constraints

```yaml
# Spread Pods evenly across zones
spec:
  topologySpreadConstraints:
    - maxSkew: 1                          # Max difference between zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-app
```

### Affinity and Anti-Affinity

```yaml
spec:
  affinity:
    # Prefer same zone as other my-app pods (soft rule)
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: topology.kubernetes.io/zone

    # NEVER schedule two my-app pods on the same node (hard rule)
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

---

## 7. Storage

### StorageClasses in GKE

| StorageClass | Disk Type | Access Mode | Use Case |
|---|---|---|---|
| `standard` | HDD (pd-standard) | RWO | Dev/test, low-cost bulk storage |
| `standard-rwo` | Balanced SSD (pd-balanced) | RWO | Default for most workloads |
| `premium-rwo` | SSD (pd-ssd) | RWO | Production databases, low-latency |
| `standard-rwx` | Filestore | RWX | Shared storage (NFS) |

### Dynamic PVC Provisioning

```yaml
# PersistentVolumeClaim — auto-provisions a GCE Persistent Disk
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: premium-rwo
  resources:
    requests:
      storage: 50Gi
---
# Reference in a Pod
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
  containers:
    - name: app
      volumeMounts:
        - mountPath: /data
          name: data
```

### GCS FUSE CSI Driver (Mount GCS Bucket as Volume)

```yaml
# Enable GCS FUSE CSI on the cluster
# gcloud container clusters update my-cluster --update-addons=GcsFuseCsiDriver=ENABLED

spec:
  serviceAccountName: my-app-sa   # Must have storage.objectViewer on the bucket
  containers:
    - name: app
      volumeMounts:
        - name: gcs-data
          mountPath: /data
          readOnly: true
  volumes:
    - name: gcs-data
      csi:
        driver: gcsfuse.csi.storage.gke.io
        readOnly: true
        volumeAttributes:
          bucketName: my-gcs-bucket
          mountOptions: "implicit-dirs"
```

### Volume Snapshots

```yaml
# Create a VolumeSnapshot of a PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-pvc-snapshot
spec:
  volumeSnapshotClassName: pd-csi-driver-vsc
  source:
    persistentVolumeClaimName: app-data

---
# Restore from snapshot into a new PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-restored
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: premium-rwo
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: my-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### ConfigMap and Secret as Volumes

```yaml
# Mount a ConfigMap as a file
spec:
  volumes:
    - name: config
      configMap:
        name: app-config
    - name: secrets
      secret:
        secretName: app-secrets
        defaultMode: 0400     # Read-only for owner
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
```

---

## 8. Security

### Workload Identity

Workload Identity binds a Kubernetes Service Account to a GCP Service Account, eliminating the need for key files.

```bash
# Step 1: Enable Workload Identity on cluster
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --workload-pool=my-project.svc.id.goog

# Step 2: Create a GCP Service Account
gcloud iam service-accounts create gke-app-sa \
  --display-name="GKE App Service Account"

# Step 3: Grant GCP SA the IAM role it needs (e.g., Cloud Storage)
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:gke-app-sa@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Step 4: Allow the K8s SA to impersonate the GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  gke-app-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[production/my-app-ksa]"
```

```yaml
# Step 5: Annotate the Kubernetes Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-ksa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: gke-app-sa@my-project.iam.gserviceaccount.com
```

```yaml
# Step 6: Reference the K8s SA in the Pod
spec:
  serviceAccountName: my-app-ksa
```

### Pod Security Standards (PSA)

```yaml
# Apply Pod Security Standards at the namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce=hard block | warn=emit warning | audit=log only
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

| PSS Level | Key Restrictions |
|---|---|
| **Privileged** | No restrictions (for system pods only) |
| **Baseline** | Disallows privileged containers, hostPID, hostNetwork, dangerous capabilities |
| **Restricted** | All baseline + no root, read-only filesystem required, seccompProfile required |

### RBAC

```yaml
# Role — namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding — grant Role to a user or SA
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-cd-sa
    namespace: production
  - kind: User
    name: developer@mycompany.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Binary Authorization

```bash
# Enable Binary Authorization on cluster
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE

# Create a policy requiring attestation before deployment
cat > binauthz-policy.yaml << 'EOF'
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/my-project/attestors/build-verified
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
clusterAdmissionRules:
  us-central1.my-cluster:
    evaluationMode: ALWAYS_ALLOW   # Override for this cluster if needed
EOF

gcloud container binauthz policy import binauthz-policy.yaml
```

### GKE Sandbox (gVisor)

```bash
# Create a node pool with gVisor sandbox
gcloud container node-pools create sandbox-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --machine-type=n2-standard-4 \
  --sandbox=type=gvisor
```

```yaml
# Run a Pod in the gVisor sandbox
spec:
  runtimeClassName: gvisor
  containers:
    - name: untrusted-workload
      image: user-provided-image:latest
```

### Secret Encryption (CMEK)

```bash
# Enable application-layer secret encryption with a Cloud KMS key
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --database-encryption-key=projects/my-project/locations/us-central1/keyRings/my-ring/cryptoKeys/my-key
```

### Cluster Hardening Checklist

```
✅ Enable Workload Identity (disable node service account)
✅ Enable Shielded GKE Nodes (Secure Boot, vTPM, Integrity Monitoring)
✅ Use private clusters (no public node IPs)
✅ Enable authorized networks for control plane access
✅ Enable Binary Authorization
✅ Apply Pod Security Standards (at least 'baseline' for all namespaces)
✅ Enable network policies (Dataplane V2)
✅ Enable GKE Sandbox (gVisor) for untrusted workloads
✅ Enable CMEK for etcd encryption
✅ Disable legacy metadata APIs (--no-enable-legacy-apis)
✅ Enable audit logging
✅ Set resource requests/limits on all containers
✅ Use non-root containers (runAsNonRoot: true)
```

---

## 9. Autoscaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 50
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # Scale up when avg CPU > 60%
    # Scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60        # Double replicas every 60s max
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
```

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"     # Off | Initial | Recreate | Auto
    # Off      = recommendations only, no changes
    # Initial  = set on Pod creation only
    # Recreate = evict and recreate Pods with new requests
    # Auto     = same as Recreate (in-place update in future)
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "4Gi"
        controlledResources: ["cpu", "memory"]
```

> **Note:** Do **not** use HPA and VPA in `Auto`/`Recreate` mode on the same Deployment for CPU/memory — they conflict. Use VPA for `Off` (recommendations) + HPA for scaling, or use VPA for memory only + HPA for CPU.

### Cluster Autoscaler (CA)

```bash
# Enable Cluster Autoscaler on a node pool
gcloud container node-pools update my-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=20

# Configure CA profile
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --autoscaling-profile=optimize-utilization  # aggressive scale-down
  # or: balanced (default)
```

```yaml
# Cluster Autoscaler configuration via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  # CA settings (managed by GKE, shown for reference)
  scale-down-delay-after-add: "10m"
  scale-down-unneeded-time: "10m"
  scale-down-utilization-threshold: "0.5"
```

### KEDA (Kubernetes Event-Driven Autoscaling)

```yaml
# Scale on Pub/Sub subscription backlog
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-scaledobject
spec:
  scaleTargetRef:
    name: my-worker-deployment
  minReplicaCount: 0         # Scale to zero when no messages
  maxReplicaCount: 50
  cooldownPeriod: 300
  triggers:
    - type: gcp-pubsub
      authenticationRef:
        name: keda-trigger-auth-gcp
      metadata:
        subscriptionName: my-subscription
        projectID: my-project
        activationThreshold: "5"     # min messages before scaling from 0
        value: "10"                  # target messages per replica
```

---

## 10. CI/CD & GitOps

### Cloud Build for GKE

```yaml
# cloudbuild.yaml — build, push, deploy to GKE
steps:
  # Step 1: Run tests
  - name: python:3.12-slim
    entrypoint: pytest
    args: [tests/]

  # Step 2: Build container image
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA
      - .

  # Step 3: Push to Artifact Registry
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA

  # Step 4: Update image in deployment (kustomize or sed)
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: bash
    args:
      - -c
      - |
        gcloud container clusters get-credentials my-cluster \
          --region=us-central1 --project=$PROJECT_ID
        kubectl set image deployment/my-app \
          app=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA \
          --namespace=production

images:
  - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA

options:
  logging: CLOUD_LOGGING_ONLY
```

### Cloud Deploy (Progressive Delivery)

```yaml
# clouddeploy.yaml — pipeline definition
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
  location: us-central1
description: "Deploy my-app to staging then production"
serialPipeline:
  stages:
    - targetId: staging
      profiles: [staging]
    - targetId: production
      profiles: [production]
      strategy:
        canary:
          runtimeConfig:
            kubernetes:
              gatewayServiceMesh:
                httpRoute: my-app-route
                service: my-app-service
                deployment: my-app
          canaryDeployment:
            percentages: [25, 50, 75]
            verify: true
```

### Helm on GKE

```bash
# Install Helm (if not installed)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a chart into the cluster
helm install my-postgres bitnami/postgresql \
  --namespace=databases \
  --create-namespace \
  --set auth.postgresPassword=secretpassword \
  --set primary.persistence.storageClass=premium-rwo \
  --set primary.persistence.size=50Gi

# Upgrade a release
helm upgrade my-postgres bitnami/postgresql \
  --namespace=databases \
  --set image.tag=15.4.0

# List all Helm releases
helm list --all-namespaces
```

### Skaffold (Local Dev Workflow)

```yaml
# skaffold.yaml — local development with hot reload
apiVersion: skaffold/v4beta6
kind: Config
build:
  artifacts:
    - image: us-central1-docker.pkg.dev/my-project/repo/my-app
      docker:
        dockerfile: Dockerfile
deploy:
  kubectl:
    manifests:
      - k8s/*.yaml
portForward:
  - resourceType: deployment
    resourceName: my-app
    port: 8080
    localPort: 8080
```

```bash
# Run Skaffold in development mode (watch + rebuild + redeploy)
skaffold dev

# Run a single build-and-deploy cycle
skaffold run
```

### Config Sync (GitOps)

```bash
# Install Config Sync via GKE Hub
gcloud beta container fleet config-management enable \
  --project=my-project

# Apply Config Sync configuration
cat > config-sync.yaml << 'EOF'
applySpecVersion: 1
spec:
  configSync:
    enabled: true
    syncRepo: https://github.com/my-org/k8s-config
    syncBranch: main
    secretType: none
    policyDir: /clusters/production
EOF

gcloud beta container fleet config-management apply \
  --membership=my-cluster \
  --config=config-sync.yaml \
  --project=my-project
```

---

## 11. Observability

### GKE Metrics (Cloud Monitoring)

```bash
# Enable managed Prometheus (Google Cloud Managed Service for Prometheus)
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --enable-managed-prometheus

# Verify PodMonitoring CRD is available
kubectl get podmonitorings -A
```

```yaml
# PodMonitoring — scrape metrics from your app
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: my-app-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Structured Logging

```python
# Python app — emit structured logs that GKE/Cloud Logging parses
import json, sys, os

def log(severity: str, message: str, **kwargs):
    entry = {
        "severity": severity,
        "message": message,
        "pod": os.environ.get("POD_NAME", "unknown"),
        "namespace": os.environ.get("POD_NAMESPACE", "unknown"),
        **kwargs
    }
    print(json.dumps(entry), flush=True)

log("INFO", "Request processed", latency_ms=42, user_id="u-123")
```

### Useful kubectl Debug Commands

```bash
# Get all resources in a namespace
kubectl get all -n production

# Describe a failing Pod (events, status, conditions)
kubectl describe pod my-pod-xyz -n production

# Stream Pod logs
kubectl logs -f deployment/my-app -n production --all-containers

# Previous container logs (after CrashLoopBackOff)
kubectl logs my-pod-xyz -n production --previous

# Execute a shell in a running container
kubectl exec -it my-pod-xyz -n production -- /bin/bash

# Port-forward to a Service (local debugging)
kubectl port-forward svc/my-app 8080:80 -n production

# View resource usage (requires Metrics Server)
kubectl top nodes
kubectl top pods -n production --sort-by=memory

# Check events sorted by time
kubectl get events -n production --sort-by='.lastTimestamp'

# Rollout status and history
kubectl rollout status deployment/my-app -n production
kubectl rollout history deployment/my-app -n production
kubectl rollout undo deployment/my-app -n production    # Rollback
```

### Cloud Logging Filters for GKE

```bash
# All logs from a specific GKE workload
resource.type="k8s_container"
resource.labels.cluster_name="my-cluster"
resource.labels.namespace_name="production"
resource.labels.container_name="my-app"

# Error logs from all containers in a namespace
resource.type="k8s_container"
resource.labels.namespace_name="production"
severity>=ERROR

# Pod OOMKilled events
resource.type="k8s_node"
jsonPayload.reason="OOMKilling"

# Cluster autoscaler decisions
resource.type="k8s_cluster"
logName:"container.googleapis.com/cluster-autoscaler"

# Failed scheduling events
resource.type="k8s_pod"
jsonPayload.reason="FailedScheduling"

# Kubernetes audit log — kubectl exec sessions
logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"
protoPayload.methodName="io.k8s.core.v1.pods.exec"
```

---

## 12. IAM & Access Control

### GKE IAM Roles

| Role | Description |
|---|---|
| `roles/container.admin` | Full control of GKE clusters and Kubernetes objects |
| `roles/container.developer` | Deploy and manage Kubernetes workloads (no cluster create/delete) |
| `roles/container.viewer` | Read-only view of clusters |
| `roles/container.clusterAdmin` | Create, update, delete clusters |
| `roles/container.nodeServiceAccount` | Required for GKE node VMs (auto-assigned) |

```bash
# Grant GKE developer access
gcloud projects add-iam-policy-binding my-project \
  --member=user:developer@mycompany.com \
  --role=roles/container.developer

# Get kubeconfig credentials for a cluster
gcloud container clusters get-credentials my-cluster \
  --region=us-central1 \
  --project=my-project

# Get credentials for a private cluster (via internal IP)
gcloud container clusters get-credentials private-cluster \
  --region=us-central1 \
  --internal-ip \
  --project=my-project
```

### RBAC vs. IAM

| Aspect | IAM | RBAC |
|---|---|---|
| **Scope** | GCP project level (cluster create, node management) | Kubernetes namespace or cluster level |
| **Subject types** | GCP users, groups, service accounts | K8s users, groups, service accounts |
| **Controls** | GCP API actions (gcloud commands) | Kubernetes API actions (kubectl commands) |
| **Best practice** | Use for cluster lifecycle management | Use for workload-level access control |

> **Note:** GKE uses both IAM and RBAC. IAM controls who can interact with the cluster via GCP APIs (e.g., `gcloud container clusters create`). RBAC controls what Kubernetes operations a user can perform once they have cluster credentials.

### Fleet Membership IAM

```bash
# Register a cluster to a fleet (GKE Hub)
gcloud container fleet memberships register my-cluster \
  --gke-cluster=us-central1/my-cluster \
  --enable-workload-identity \
  --project=my-fleet-project

# Grant fleet-level connect gateway access
gcloud projects add-iam-policy-binding my-fleet-project \
  --member=user:operator@mycompany.com \
  --role=roles/gkehub.gatewayEditor
```

### Audit Logging

```bash
# View GKE audit logs (API calls to the cluster)
gcloud logging read \
  'logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"
   AND resource.type="k8s_cluster"
   AND resource.labels.cluster_name="my-cluster"' \
  --project=my-project \
  --limit=50
```

---

## 13. Multi-Cluster & Fleet Management

### Fleet Concepts

```
Fleet Host Project
├── Member Cluster A (us-central1) — production
├── Member Cluster B (europe-west1) — production-eu
└── Member Cluster C (us-central1) — staging

Fleet features:
- Config Sync (GitOps across all clusters)
- Policy Controller (OPA Gatekeeper at fleet level)
- Multi-Cluster Ingress (global load balancing)
- Multi-Cluster Services (cross-cluster service discovery)
```

### Fleet Registration

```bash
# Enable GKE Hub in the fleet host project
gcloud services enable gkehub.googleapis.com --project=my-fleet-project

# Register clusters to the fleet
gcloud container fleet memberships register prod-us \
  --gke-cluster=us-central1/my-cluster-us \
  --project=my-fleet-project

gcloud container fleet memberships register prod-eu \
  --gke-cluster=europe-west1/my-cluster-eu \
  --project=my-fleet-project

# List fleet members
gcloud container fleet memberships list --project=my-fleet-project
```

### Multi-Cluster Ingress (MCI)

```yaml
# MultiClusterIngress — routes traffic to the nearest healthy backend
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: global-ingress
  namespace: production
  annotations:
    networking.gke.io/static-ip: my-global-ip
spec:
  template:
    spec:
      backend:
        serviceName: global-backend
        servicePort: 80
---
# MultiClusterService — the backend referenced by MCI
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: global-backend
  namespace: production
spec:
  template:
    spec:
      selector:
        app: my-app
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 8080
```

### Multi-Cluster Services (MCS)

```yaml
# Export a Service to all clusters in the fleet
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
  name: my-api-service
  namespace: production

# In other clusters, import and use via DNS:
# my-api-service.production.svc.clusterset.local
```

### Use Cases for Multi-Cluster

| Use Case | Pattern |
|---|---|
| **Global HA** | Active-active clusters in multiple regions via MCI |
| **Geo-distribution** | Route users to nearest cluster; Multi-Cluster Ingress |
| **Environment isolation** | Separate clusters for dev/staging/prod with Config Sync |
| **Team isolation** | Separate clusters per team with fleet-level policy |
| **Blue/green cluster upgrade** | Create new cluster, migrate traffic, delete old |

---

## 14. Cost Optimization

### GKE Pricing Model

| Component | GKE Standard | GKE Autopilot |
|---|---|---|
| **Control plane** | $0.10/hr per cluster (free for first zonal cluster/project) | $0.10/hr per cluster |
| **Nodes** | GCE VM price (per node) | Not billed — Pods billed instead |
| **Pods (Autopilot)** | N/A | Per vCPU-second + GB-second + storage |
| **Spot nodes** | Up to 91% discount | Spot Pods available |
| **System overhead (Autopilot)** | N/A | 0.25 vCPU + 100 MB minimum per Pod |

### Spot Node Pool (Cost Saving)

```bash
# Spot nodes: up to 91% cheaper but can be preempted
gcloud container node-pools create spot-workloads \
  --cluster=my-cluster \
  --region=us-central1 \
  --spot \
  --machine-type=n2-standard-4 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=30

# ALWAYS set a toleration + priorityClass for Spot workloads
```

```yaml
# PriorityClass for batch workloads (lower priority = first evicted)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low-priority
value: 100            # vs 1000 for normal, 10000 for critical
globalDefault: false
preemptionPolicy: Never
```

### Resource Quotas (Prevent Resource Waste)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    count/pods: "100"
    count/services: "20"
    persistentvolumeclaims: "10"
---
# LimitRange — set default requests/limits for containers that don't specify
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
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
```

### GKE Usage Metering (Cost Attribution)

```bash
# Enable GKE usage metering — exports resource usage to BigQuery
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --resource-usage-bigquery-dataset=gke_usage_metering \
  --enable-network-egress-metering \
  --enable-resource-consumption-metering
```

### FinOps Best Practices

- Label all workloads with `team`, `env`, `cost-center` for chargeback reporting.
- Use **VPA in `Off` mode** to get right-sizing recommendations without disruption.
- **Delete idle namespaces** — unscheduled Pods in "empty" namespaces still consume cluster resources.
- **Use Autopilot for dev/test** — pay only for actual Pod usage; no idle node cost.
- **Schedule non-prod clusters to shut down** overnight with Cloud Scheduler + `gcloud container clusters resize --num-nodes=0`.
- Enable **Committed Use Discounts** (CUDs) for baseline node count in production Standard clusters.
- Use `kubectl-cost` or **Cloud Billing reports** filtered by GKE labels.

---

## 15. `kubectl` & `gcloud` CLI Quick Reference

### kubectl — Core Commands

| Command | Description |
|---|---|
| `kubectl get pods -n NS` | List all Pods in a namespace |
| `kubectl get all -n NS` | List all resources in a namespace |
| `kubectl describe pod NAME -n NS` | Full details + events for a Pod |
| `kubectl apply -f FILE.yaml` | Create or update resources from YAML |
| `kubectl delete -f FILE.yaml` | Delete resources defined in YAML |
| `kubectl delete pod NAME -n NS` | Force-delete a single Pod |
| `kubectl exec -it POD -n NS -- bash` | Open a shell in a running container |
| `kubectl logs -f POD -n NS` | Stream container logs |
| `kubectl logs POD -n NS --previous` | Logs from crashed/restarted container |
| `kubectl port-forward svc/SVC 8080:80` | Tunnel a Service to localhost |
| `kubectl scale deployment NAME --replicas=5` | Manually scale a Deployment |
| `kubectl rollout status deployment/NAME` | Watch a rolling update |
| `kubectl rollout undo deployment/NAME` | Roll back to previous revision |
| `kubectl top nodes` | Node CPU/memory usage |
| `kubectl top pods -n NS` | Pod CPU/memory usage |
| `kubectl get events --sort-by='.lastTimestamp'` | Recent cluster events |
| `kubectl config get-contexts` | List kubeconfig contexts |
| `kubectl config use-context CONTEXT` | Switch active cluster context |

```bash
# Practical examples

# Watch Pods update in real time
kubectl get pods -n production -w

# Get Pod YAML with current state
kubectl get pod my-pod -n production -o yaml

# Apply a kustomize directory
kubectl apply -k overlays/production/

# Dry run before applying (validate without creating)
kubectl apply -f deploy.yaml --dry-run=client

# Force restart all Pods in a Deployment
kubectl rollout restart deployment/my-app -n production

# Copy files from/to a Pod
kubectl cp my-pod:/app/logs/app.log ./app.log -n production

# Run a temporary debug pod
kubectl run debug --image=busybox:latest --rm -it --restart=Never -- sh
```

### gcloud container — GKE Commands

| Command | Description |
|---|---|
| `gcloud container clusters create NAME` | Create a GKE cluster |
| `gcloud container clusters delete NAME` | Delete a cluster |
| `gcloud container clusters upgrade NAME` | Upgrade cluster or node pool |
| `gcloud container clusters get-credentials NAME` | Configure kubectl for the cluster |
| `gcloud container clusters describe NAME` | Show cluster configuration |
| `gcloud container clusters list` | List all clusters in the project |
| `gcloud container clusters resize NAME --num-nodes=N` | Resize node count |
| `gcloud container node-pools create NAME` | Create a new node pool |
| `gcloud container node-pools delete NAME` | Delete a node pool |
| `gcloud container node-pools list` | List node pools in a cluster |
| `gcloud container images list` | List images in Artifact Registry |

```bash
# Create a production-ready regional cluster
gcloud container clusters create prod-cluster \
  --region=us-central1 \
  --release-channel=regular \
  --enable-ip-alias \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-master-authorized-networks \
  --master-authorized-networks=10.0.0.0/8 \
  --workload-pool=my-project.svc.id.goog \
  --enable-dataplane-v2 \
  --num-nodes=1 \
  --machine-type=n2-standard-4

# Upgrade only the control plane
gcloud container clusters upgrade prod-cluster \
  --region=us-central1 \
  --master \
  --cluster-version=1.28.3-gke.100

# Upgrade a specific node pool
gcloud container clusters upgrade prod-cluster \
  --region=us-central1 \
  --node-pool=default-pool \
  --cluster-version=1.28.3-gke.100
```

---

## 16. Terraform Snippet

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

# ── VPC & Subnet ──────────────────────────────────────────
resource "google_compute_network" "gke_vpc" {
  name                    = "gke-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gke_subnet" {
  name          = "gke-subnet"
  ip_cidr_range = "10.0.0.0/20"
  region        = "us-central1"
  network       = google_compute_network.gke_vpc.self_link

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.4.0.0/14"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.8.0.0/20"
  }
}

# ── Service Account for GKE nodes ────────────────────────
resource "google_service_account" "gke_node_sa" {
  account_id   = "gke-node-sa"
  display_name = "GKE Node Service Account"
}

resource "google_project_iam_member" "node_sa_log_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.gke_node_sa.email}"
}

resource "google_project_iam_member" "node_sa_metric_writer" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.gke_node_sa.email}"
}

resource "google_project_iam_member" "node_sa_artifact_reader" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.gke_node_sa.email}"
}

# ── GKE Regional Cluster ──────────────────────────────────
resource "google_container_cluster" "primary" {
  name     = "prod-cluster"
  location = "us-central1"

  # Remove the default node pool and manage separately
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.gke_vpc.self_link
  subnetwork = google_compute_subnetwork.gke_subnet.self_link

  # VPC-native (alias IP) networking
  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # Private cluster — nodes have no public IPs
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # Restrict control plane access to VPC only
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "VPC internal"
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Release channel
  release_channel {
    channel = "REGULAR"
  }

  # eBPF Dataplane V2 + Network Policy
  datapath_provider = "ADVANCED_DATAPATH"

  # Binary Authorization
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # Managed Prometheus
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
    managed_prometheus {
      enabled = true
    }
  }

  # Audit logging
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  # Shielded nodes
  node_config {
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }
}

# ── System Node Pool ──────────────────────────────────────
resource "google_container_node_pool" "system" {
  name       = "system-pool"
  location   = "us-central1"
  cluster    = google_container_cluster.primary.name
  node_count = 1

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }

  node_config {
    machine_type = "n2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"
    image_type   = "COS_CONTAINERD"

    service_account = google_service_account.gke_node_sa.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"   # Required for Workload Identity
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    labels = {
      pool = "system"
      env  = "production"
    }

    taint {
      key    = "CriticalAddonsOnly"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
  }
}

# ── Application Node Pool (Spot) ──────────────────────────
resource "google_container_node_pool" "app_spot" {
  name     = "app-spot-pool"
  location = "us-central1"
  cluster  = google_container_cluster.primary.name

  autoscaling {
    min_node_count = 0
    max_node_count = 20
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    max_surge       = 2
    max_unavailable = 1
  }

  node_config {
    machine_type = "n2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-balanced"
    image_type   = "COS_CONTAINERD"
    spot         = true          # Spot instances

    service_account = google_service_account.gke_node_sa.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    labels = {
      pool = "app"
      env  = "production"
    }

    taint {
      key    = "cloud.google.com/gke-spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
  }
}

# ── Kubernetes Provider (after cluster creation) ──────────
provider "kubernetes" {
  host                   = "https://${google_container_cluster.primary.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(google_container_cluster.primary.master_auth[0].cluster_ca_certificate)
}

data "google_client_config" "default" {}

# ── Kubernetes Namespace ──────────────────────────────────
resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      "pod-security.kubernetes.io/enforce" = "restricted"
      env                                  = "production"
    }
  }
}

# ── Kubernetes Service Account (Workload Identity) ────────
resource "kubernetes_service_account" "app_sa" {
  metadata {
    name      = "my-app-ksa"
    namespace = kubernetes_namespace.production.metadata[0].name
    annotations = {
      "iam.gke.io/gcp-service-account" = google_service_account.gke_node_sa.email
    }
  }
}

# ── Kubernetes Deployment ────────────────────────────────
resource "kubernetes_deployment" "app" {
  metadata {
    name      = "my-app"
    namespace = kubernetes_namespace.production.metadata[0].name
    labels = {
      app = "my-app"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "my-app"
      }
    }

    strategy {
      type = "RollingUpdate"
      rolling_update {
        max_surge       = "1"
        max_unavailable = "0"
      }
    }

    template {
      metadata {
        labels = {
          app = "my-app"
        }
      }

      spec {
        service_account_name = kubernetes_service_account.app_sa.metadata[0].name

        toleration {
          key      = "cloud.google.com/gke-spot"
          operator = "Equal"
          value    = "true"
          effect   = "NoSchedule"
        }

        container {
          name  = "app"
          image = "us-central1-docker.pkg.dev/${var.project_id}/my-repo/my-app:v1.0.0"

          port {
            container_port = 8080
          }

          resources {
            requests = {
              cpu    = "250m"
              memory = "256Mi"
            }
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
          }

          liveness_probe {
            http_get {
              path = "/healthz"
              port = 8080
            }
            initial_delay_seconds = 15
            period_seconds        = 20
          }

          readiness_probe {
            http_get {
              path = "/ready"
              port = 8080
            }
            initial_delay_seconds = 5
            period_seconds        = 10
          }
        }
      }
    }
  }
}

# ── Kubernetes Service ────────────────────────────────────
resource "kubernetes_service" "app" {
  metadata {
    name      = "my-app-service"
    namespace = kubernetes_namespace.production.metadata[0].name
  }

  spec {
    selector = {
      app = "my-app"
    }
    port {
      port        = 80
      target_port = 8080
    }
    type = "ClusterIP"
  }
}

# ── Outputs ───────────────────────────────────────────────
output "cluster_endpoint" {
  value       = google_container_cluster.primary.endpoint
  description = "GKE cluster API server endpoint"
  sensitive   = true
}

output "cluster_name" {
  value = google_container_cluster.primary.name
}

output "get_credentials_command" {
  value = "gcloud container clusters get-credentials ${google_container_cluster.primary.name} --region=us-central1 --project=${var.project_id}"
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project"
terraform apply -var="project_id=my-project"
```

---

## 17. Common Patterns & Best Practices

### Blue/Green Deployment

```yaml
# Two Deployments: blue (current) and green (new)
# Switch traffic by updating the Service selector

# Service — currently pointing to blue
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue      # ← Change to 'green' to switch traffic
  ports:
    - port: 80
      targetPort: 8080
---
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: app
          image: my-app:v1.0.0
---
# Green deployment (new version — deployed but not receiving traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: app
          image: my-app:v2.0.0
```

```bash
# Validate green, then switch traffic
kubectl patch service my-app \
  -p '{"spec":{"selector":{"version":"green"}}}'
```

### Canary Deployment with Gateway API

```yaml
# Split traffic: 90% stable, 10% canary
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: my-app-canary
spec:
  rules:
    - backendRefs:
        - name: my-app-stable
          port: 80
          weight: 90
        - name: my-app-canary
          port: 80
          weight: 10
```

### Namespace-per-Team Multi-Tenancy

```yaml
# Each team gets a namespace with ResourceQuota + LimitRange + RBAC
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    team: payments
    pod-security.kubernetes.io/enforce: baseline
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    count/pods: "50"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-payments-admin
  namespace: team-payments
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - kind: Group
    name: team-payments-developers
    apiGroup: rbac.authorization.k8s.io
```

### Hardened Pod Security Template

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: my-app:v1.0.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp          # Allow writes to /tmp only
  volumes:
    - name: tmp
      emptyDir: {}
```

### Sidecar Pattern (Istio / Cloud Service Mesh)

```bash
# Enable Cloud Service Mesh (Istio-based) on a cluster
gcloud container fleet mesh enable --project=my-project

# Label namespace for sidecar injection
kubectl label namespace production istio-injection=enabled
```

```yaml
# With injection enabled, every Pod in the namespace gets an Envoy sidecar
# No changes to your application YAML required
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production       # ← namespace has istio-injection=enabled
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:v1
          # Istio injects 'istio-proxy' sidecar automatically
```

### Init Container Pattern

```yaml
# Init container runs before app containers — useful for DB migrations, config prep
spec:
  initContainers:
    - name: db-migrate
      image: my-app:v1.0.0
      command: ["python", "manage.py", "migrate"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
  containers:
    - name: app
      image: my-app:v1.0.0
```

---

## 18. Troubleshooting Quick Reference

| Issue | Likely Cause | Diagnostic Command | Fix |
|---|---|---|---|
| **Pod stuck in `Pending`** | Insufficient resources; taint mismatch; PVC not bound | `kubectl describe pod NAME` | Add nodes (CA should trigger); check node taints; fix PVC |
| **`CrashLoopBackOff`** | App crashes on startup; missing env var; misconfigured probe | `kubectl logs NAME --previous` | Fix app startup; check probe paths; verify env/secrets |
| **`OOMKilled`** | Container exceeded memory limit | `kubectl describe pod NAME` → `OOMKilled` | Increase memory limit; fix memory leak; use VPA |
| **`ImagePullBackOff`** | Image not found; no pull credentials; wrong registry | `kubectl describe pod NAME` | Fix image tag; grant `artifactregistry.reader` to node SA; verify registry path |
| **Node `NotReady`** | Node disk pressure; memory pressure; kubelet crash | `kubectl describe node NAME` | Drain and delete node; CA will replace; check disk/memory usage |
| **PVC stuck in `Pending`** | No matching StorageClass; quota exceeded; CSI driver issue | `kubectl describe pvc NAME` | Verify StorageClass name; check ResourceQuota; check CSI driver pods |
| **HPA not scaling** | Metrics server not available; resource requests not set; wrong metric | `kubectl describe hpa NAME` | Install Metrics Server; set CPU/memory requests; verify HPA metric config |
| **Workload Identity token error** | K8s SA not annotated; IAM binding missing; wrong pool | Check app logs for `403` | Add annotation to K8s SA; add IAM binding; verify `workload-pool` cluster config |
| **Network policy blocking traffic** | Overly restrictive NetworkPolicy | `kubectl get networkpolicy -A` | Add explicit allow rule for the blocked traffic |
| **Cluster upgrade fails** | PodDisruptionBudget blocking drain; unhealthy Pods | `kubectl get pdb -A` | Fix underlying unhealthy Pods; adjust PDB `minAvailable` |
| **Ingress 502/503** | Pods not ready; health check failing; wrong targetPort | `kubectl describe ingress NAME` | Fix readiness probe; verify `targetPort` matches container port |
| **Ingress SSL pending** | ManagedCertificate provisioning (takes up to 60 min) | `kubectl describe managedcertificate NAME` | Wait; ensure DNS points to Ingress IP; check domain ownership |
| **DNS resolution fails in Pod** | CoreDNS down; wrong search domain; network policy blocking port 53 | `kubectl exec -it POD -- nslookup kubernetes` | Restart CoreDNS pods; add NetworkPolicy allowing DNS egress on port 53 |
| **Deployment rollout stuck** | `maxUnavailable=0` with unhealthy Pods blocking the rollout | `kubectl rollout status deployment/NAME` | Fix failing Pods; adjust rollout strategy; `kubectl rollout undo` |
| **Spot node eviction causing outage** | All replicas on Spot nodes; no PDB | `kubectl get events` | Spread Pods across Spot + on-demand; add PodDisruptionBudget |
| **etcd disk quota exceeded** | Too many old ConfigMaps/Secrets/Events | `gcloud container clusters describe` | Enable event/object TTL; run `kubectl delete events --all -n NS`; contact Google Support |
| **`kubectl` 403 error** | Missing RBAC permissions; stale kubeconfig | `kubectl auth can-i VERB RESOURCE` | Add RBAC RoleBinding; refresh credentials with `gcloud container clusters get-credentials` |

---

*Last updated: 2025 | Covers GKE GA features as of this date.*
*Refer to [cloud.google.com/kubernetes-engine/docs](https://cloud.google.com/kubernetes-engine/docs) for the latest information.*
