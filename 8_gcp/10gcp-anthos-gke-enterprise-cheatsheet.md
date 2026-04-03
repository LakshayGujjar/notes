# GCP Anthos / GKE Enterprise — Production Cheatsheet

---

## ANTHOS / GKE ENTERPRISE — OVERVIEW & ARCHITECTURE

Google rebranded **Anthos** to **GKE Enterprise** in 2023, consolidating multi-cloud Kubernetes management, policy enforcement, service mesh, and GitOps-driven configuration under a single enterprise SKU. The core value proposition is a unified control plane that manages GKE clusters alongside attached clusters (EKS, AKS, on-premises, bare-metal) through a Fleet abstraction, enabling consistent policy, security, and observability across hybrid and multi-cloud environments.

```text
┌─────────────────────────────────────────────────────────────────────┐
│                     FLEET HOST PROJECT                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    GKE HUB (Fleet API)                       │  │
│  │   ┌─────────────┐  ┌──────────────────┐  ┌───────────────┐  │  │
│  │   │Config Mgmt  │  │ Policy Controller│  │  ASM Feature  │  │  │
│  │   │  Feature    │  │    Feature       │  │    Feature    │  │  │
│  │   └──────┬──────┘  └────────┬─────────┘  └──────┬────────┘  │  │
│  │          │                  │                    │            │  │
│  │   ┌──────▼──────────────────▼────────────────────▼────────┐  │  │
│  │   │              Connect Gateway (API proxy)               │  │  │
│  │   └───────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────┬─────────────────────────────────┘  │
└───────────────────────────────┼─────────────────────────────────────┘
                                │  Fleet Memberships
          ┌─────────────────────┼─────────────────────────┐
          │                     │                         │
  ┌───────▼───────┐    ┌────────▼──────┐    ┌────────────▼──────────┐
  │  GKE Cluster  │    │ Attached EKS  │    │  Anthos Bare Metal    │
  │ (GCP-native)  │    │  /AKS Cluster │    │  / VMware Cluster     │
  │               │    │               │    │                       │
  │ Config Sync   │    │  Config Sync  │    │  Config Sync          │
  │ Policy Ctrl   │    │  Policy Ctrl  │    │  Policy Ctrl          │
  │ ASM Sidecar   │    │  ASM Sidecar  │    │  ASM Sidecar          │
  │ Connect Agent │    │ Connect Agent │    │  Connect Agent        │
  └───────────────┘    └───────────────┘    └───────────────────────┘
```

### Key Differentiators vs Standalone GKE

| Capability | GKE Standard | GKE Autopilot | GKE Enterprise |
|---|---|---|---|
| Fleet Management (Hub) | ❌ | ❌ | ✅ |
| Config Sync (ACM) | ❌ | ❌ | ✅ |
| Policy Controller | ❌ | ❌ | ✅ |
| Service Mesh (ASM) | Add-on only | Add-on only | ✅ Managed |
| Multi-cluster Ingress | ❌ | ❌ | ✅ |
| Binary Authorization | ✅ (add-on) | ✅ (add-on) | ✅ Integrated |
| Workload Identity Federation | ✅ | ✅ | ✅ |
| Config Controller | ❌ | ❌ | ✅ |
| Connect Gateway | ❌ | ❌ | ✅ |
| GKE Security Posture | Basic | Basic | ✅ Advanced |
| Dataplane V2 (eBPF) | ✅ Opt-in | ✅ Default | ✅ Default |

### Licensing Model

- **GKE Standard** — cluster management fee per cluster + node VM costs
- **GKE Autopilot** — per pod resource (vCPU + memory) billing; no node management
- **GKE Enterprise** — per **vCPU/hour** of registered cluster node capacity across **all** fleet-registered clusters (GKE + attached), regardless of cloud provider. Control plane nodes are not billed.

> ⚠️ **NOTE:** The per-vCPU Enterprise fee is additive on top of your existing GKE cluster management and Compute Engine VM costs for GKE clusters, and on top of your AWS/Azure node costs for attached clusters.

> 💡 **TIP:** GKE Autopilot clusters enrolled in a GKE Enterprise fleet are **not** subject to the per-vCPU Enterprise surcharge — Autopilot billing applies instead. Prefer Autopilot for workloads that can tolerate its constraints.

---

## FLEET MANAGEMENT (GKE HUB)

A **Fleet** is a logical grouping of Kubernetes clusters that share a common identity, policy boundary, and management surface. GKE Hub acts as the Fleet control plane — it stores membership registrations, orchestrates feature controllers (ACM, ASM, MCI), and provides the Connect Gateway for unified API server access. Think of it as a "super-cluster" that federates many clusters into a single operational unit.

### Resource Hierarchy

```text
┌──────────────────────────────────────────────────────────┐
│              Fleet Host Project                          │
│  (owns Hub API, Fleet memberships, Feature configs)      │
│                                                          │
│   GKE Hub  ──────── Fleet Memberships (1..1000)          │
│       │                     │                            │
│       ▼                     ▼                            │
│  Feature APIs        ┌──────────────┐                   │
│  (ACM, ASM,          │ Membership   │                   │
│   MCI, BinAuthz)     │  Resource    │                   │
│                      └──────┬───────┘                   │
└─────────────────────────────┼────────────────────────────┘
                              │ references
          ┌───────────────────┼──────────────────────┐
          │                   │                      │
┌─────────▼──────┐  ┌─────────▼──────┐  ┌───────────▼─────┐
│ GKE Cluster    │  │  GKE Cluster   │  │ Attached Cluster │
│ Service Proj A │  │ Service Proj B │  │ (EKS/AKS/ABM)   │
└────────────────┘  └────────────────┘  └─────────────────┘
```

> ⚠️ **NOTE:** The Fleet Host Project and cluster Service Projects can be the same project (simple setups) or different (enterprise multi-project org). The Hub API must be enabled in the Fleet Host Project: `gcloud services enable gkehub.googleapis.com`.

### Registering Clusters

```bash
# ── Auto-register at cluster creation time ──────────────────────────────────
gcloud container clusters create prod-cluster \
  --location=us-central1 \
  --fleet-project=FLEET_PROJECT_ID \
  --workload-pool=FLEET_PROJECT_ID.svc.id.goog

# ── Register an existing GKE cluster ────────────────────────────────────────
gcloud container fleet memberships register MEMBERSHIP_NAME \
  --gke-cluster=LOCATION/CLUSTER_NAME \
  --enable-workload-identity \
  --project=FLEET_PROJECT_ID

# ── Register an attached cluster (non-GKE) ──────────────────────────────────
gcloud container fleet memberships register MEMBERSHIP_NAME \
  --context=KUBECONFIG_CONTEXT \
  --kubeconfig=/path/to/kubeconfig \
  --enable-workload-identity \
  --project=FLEET_PROJECT_ID

# ── List all fleet memberships ───────────────────────────────────────────────
gcloud container fleet memberships list --project=FLEET_PROJECT_ID

# ── Describe a membership ────────────────────────────────────────────────────
gcloud container fleet memberships describe MEMBERSHIP_NAME \
  --project=FLEET_PROJECT_ID

# ── Unregister a membership ──────────────────────────────────────────────────
gcloud container fleet memberships unregister MEMBERSHIP_NAME \
  --gke-cluster=LOCATION/CLUSTER_NAME \
  --project=FLEET_PROJECT_ID
```

### Fleet Namespaces & Namespace Sameness

Fleet namespaces create a **namespace sameness** semantic — a namespace with the same name across all fleet clusters is treated as the same logical namespace for policy targeting, RBAC inheritance, and multi-cluster service discovery. Define them as `Namespace` objects in your Config Sync repo and ACM enforces their existence on all registered clusters.

### Connect Gateway

Connect Gateway lets you run `kubectl` commands against any fleet member cluster **without VPN or direct network access** — traffic is proxied through the Hub control plane via the Connect Agent running inside each cluster.

```bash
# Fetch kubeconfig for a fleet member via Connect Gateway
gcloud container fleet memberships get-credentials MEMBERSHIP_NAME \
  --project=FLEET_PROJECT_ID

# Now kubectl talks through Hub, not directly to the API server
kubectl get pods --all-namespaces
kubectl get nodes
```

### Fleet-Level IAM for RBAC

```bash
# Grant a user read-only Connect Gateway access to all fleet clusters
gcloud projects add-iam-policy-binding FLEET_PROJECT_ID \
  --member="user:engineer@example.com" \
  --role="roles/gkehub.gatewayReader"

# Grant RBAC impersonation for kubectl commands (needed alongside Gateway IAM)
# Apply this ClusterRoleBinding on each member cluster:
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gateway-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: User
    name: engineer@example.com
EOF
```

### Terraform — Fleet Registration

```hcl
resource "google_gke_hub_membership" "membership" {
  membership_id = "my-cluster-membership"
  project       = var.fleet_project_id

  endpoint {
    gke_cluster {
      resource_link = "//container.googleapis.com/${google_container_cluster.primary.id}"
    }
  }

  authority {
    issuer = "https://container.googleapis.com/v1/${google_container_cluster.primary.id}"
  }
}
```

> ⚠️ **NOTE:** Hub project has a **soft limit of 1,000 memberships per project**. Connect Agent version must be kept current — version drift causes Connect Gateway authentication failures. Workload Identity pools are scoped per Fleet project (`PROJECT.svc.id.goog`); avoid running multiple Fleet host projects that share member clusters.

---

## CONFIG MANAGEMENT (ANTHOS CONFIG MANAGEMENT / ACM)

Anthos Config Management (ACM) provides GitOps-native configuration synchronization for fleet clusters. It continuously reconciles the desired state defined in a Git repository (or OCI image) with the actual state of all registered clusters, preventing drift and enabling consistent policy enforcement at scale. The two primary components are **Config Sync** (the reconciler daemon) and **Policy Controller** (the admission webhook).

### Architecture

```text
┌────────────────────────────────────────────────────────────────┐
│                    Git / OCI Registry                          │
│          (fleet-config repo / Artifact Registry image)         │
└───────────────────────────────┬────────────────────────────────┘
                                │  pull (every syncPeriod, default 15s)
                ┌───────────────▼──────────────────┐
                │      Config Sync Controller       │
                │  (reconciler-manager + reconciler) │
                │   runs in config-management-system │
                └───────────────┬──────────────────┘
                                │  apply / watch
          ┌─────────────────────▼──────────────────────┐
          │          Kubernetes API Server              │
          │  ┌──────────────────────────────────────┐  │
          │  │ Policy Controller (OPA Gatekeeper)   │  │
          │  │   ValidatingWebhookConfiguration     │  │
          │  └──────────────────────────────────────┘  │
          └─────────────────────────────────────────────┘
```

### Enabling ACM via Fleet

```bash
# Enable the Config Management feature on the fleet
gcloud container fleet config-management enable \
  --project=FLEET_PROJECT_ID

# Apply configuration to a specific cluster membership
gcloud container fleet config-management apply \
  --membership=MEMBERSHIP_NAME \
  --config=apply-spec.yaml \
  --project=FLEET_PROJECT_ID

# Check sync status across all fleet members
gcloud container fleet config-management status \
  --project=FLEET_PROJECT_ID
```

### apply-spec.yaml Reference

```yaml
applySpecVersion: 1
spec:
  configSync:
    enabled: true
    sourceFormat: hierarchy        # hierarchy | unstructured
    syncRepo: https://github.com/org/fleet-config
    syncBranch: main
    syncRev: HEAD
    secretType: token              # none | token | ssh | gcpserviceaccount | gcenode
    policyDir: clusters/prod
    preventDrift: true             # re-applies if manual changes detected
    syncPeriod: 15s
  policyController:
    enabled: true
    referentialRulesEnabled: true  # cross-object policies
    logDeniesEnabled: true
    templateLibraryInstalled: true
    auditIntervalSeconds: 60
    exemptableNamespaces:
      - kube-system
```

### RootSync vs RepoSync

**RootSync** manages cluster-scoped and all namespace-scoped resources — one per cluster, operated by the reconciler running in `config-management-system`. **RepoSync** manages namespace-scoped resources for a single namespace — multiple can exist, one per namespace, enabling namespace-level GitOps delegation.

```yaml
# RootSync — cluster-wide, git source
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured
  git:
    repo: https://github.com/org/fleet-config
    branch: main
    dir: clusters/prod
    auth: gcpserviceaccount
    gcpServiceAccountEmail: config-sync-sa@PROJECT.iam.gserviceaccount.com
---
# RepoSync — namespace-scoped, delegated to a team
apiVersion: configsync.gke.io/v1beta1
kind: RepoSync
metadata:
  name: repo-sync
  namespace: my-app
spec:
  sourceFormat: unstructured
  git:
    repo: https://github.com/org/my-app-config
    branch: main
    dir: k8s/
    auth: token
    secretRef:
      name: git-credentials
```

### Repo Format Comparison

| Aspect | Hierarchy Format | Unstructured Format |
|---|---|---|
| Requires `system/repo.yaml` | ✅ Yes | ❌ No |
| Directory structure enforced | ✅ Strict (cluster/, namespaces/, etc.) | ❌ Free-form |
| Abstract namespaces / inheritance | ✅ Yes | ❌ No |
| ClusterSelector / NamespaceSelector | ✅ Yes | ✅ Yes |
| Kustomize / Helm support | ❌ Limited | ✅ Full |
| Recommended for | Large fleet with inheritance | App-centric GitOps |

### nomos CLI Reference

```bash
# Check sync status across all clusters (requires kubeconfig for each)
nomos status

# Validate repo structure before pushing
nomos vet --path=./config-root

# Render the hydrated manifests as they would appear on clusters
nomos hydrate --path=./config-root

# Initialize a new hierarchy repo
nomos init --path=./config-root
```

### ConfigSync Status Inspection

```bash
# Check RootSync status (errors, commit hash, conditions)
kubectl get rootsync -n config-management-system -o yaml
kubectl describe rootsync root-sync -n config-management-system

# Check RepoSync for a specific namespace
kubectl get reposync -n my-app -o yaml
kubectl describe reposync repo-sync -n my-app

# Watch reconciler logs
kubectl logs -n config-management-system \
  -l app=reconciler-manager -f

# Check reconciler pod per RootSync
kubectl logs -n config-management-system \
  -l configsync.gke.io/reconciler=root-reconciler -f
```

> ⚠️ **NOTE:** `sourceFormat: hierarchy` requires a `system/repo.yaml` file at the repo root — missing it causes a parse error on sync. OCI image sources require images hosted in Artifact Registry (not Docker Hub). Secrets of type `token` or `ssh` must live in the `config-management-system` namespace with specific key names (`username`/`token` or `ssh`).

> 💡 **TIP:** Enable `preventDrift: true` in production — without it, manual `kubectl apply` changes persist until the next sync cycle, creating invisible drift. With it enabled, Config Sync will immediately revert any out-of-band change.

---

## POLICY CONTROLLER (OPA GATEKEEPER)

Policy Controller is a managed, hardened deployment of OPA Gatekeeper that runs as a validating and mutating admission webhook on each GKE Enterprise cluster. It evaluates Kubernetes API requests against **Constraints** — instances of policy rules defined as Rego code in **ConstraintTemplates**. The audit controller periodically scans existing resources for violations, enabling policy-as-code enforcement without disrupting currently running workloads.

### ConstraintTemplate — Rego Policy Definition

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
  annotations:
    description: "Requires all containers to define CPU and memory limits"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresourcelimits

        import future.keywords.contains
        import future.keywords.if

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' is missing a CPU limit", [container.name])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' is missing a memory limit", [container.name])
        }
```

### Constraint Instance

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-resource-limits
spec:
  enforcementAction: deny          # deny | warn | dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
      matchExpressions:
        - key: policy-enforcement
          operator: In
          values: ["enabled"]
    excludedNamespaces:
      - kube-system
      - gke-system
      - istio-system
  parameters:
    exemptImages:
      - "gcr.io/google-containers/pause:*"
```

### Built-in Policy Template Library

| Category | Example Templates |
|---|---|
| General | `k8srequiredlabels`, `k8sallowedrepos`, `k8sblockwildcardingress`, `k8suniqueserviceselector` |
| Pod Security | `k8spspallowprivilegeescalation`, `k8spspreadonlyrootfilesystem`, `k8spsphostnamespace`, `k8spspprivilegedcontainer` |
| Networking | `k8snoexternalservices`, `k8srequirednodeaffinity`, `k8sblock-loadbalancer` |
| RBAC | `k8srestrictrolebindings`, `k8sblockclusteradminbinding`, `k8srequiredannotations` |
| Workloads | `k8srequiredprobeselector`, `k8scontainerratiolicpumemory`, `k8shorizontalpodautoscaler` |

### Audit Mode — Checking Violations

```bash
# List all constraint kinds and their violation counts
kubectl get constraints

# Describe a specific constraint — shows violations under .status.violations
kubectl describe K8sRequiredResourceLimits require-resource-limits

# Extract violation details via jsonpath
kubectl get K8sRequiredResourceLimits require-resource-limits \
  -o jsonpath='{.status.violations[*]}'

# Watch gatekeeper audit logs
kubectl logs -n gatekeeper-system \
  -l control-plane=audit-controller -f
```

### Mutation Policies

```yaml
# AssignMetadata — add/replace labels or annotations
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: add-team-label
spec:
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
  location: "metadata.labels.team"
  parameters:
    assign:
      value: "platform"
---
# Assign — set a field deep in the spec
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: set-imagepullpolicy
spec:
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
  location: "spec.containers[name:*].imagePullPolicy"
  parameters:
    assign:
      value: "Always"
```

### Referential Constraints

Referential constraints allow Rego policies to query the state of **other** Kubernetes objects during admission evaluation — for example, validating that a Pod's `serviceAccountName` references a ServiceAccount that already exists, or that an Ingress uses a valid TLS Secret.

```yaml
# In ConstraintTemplate spec, enable external data / inventory sync:
spec:
  crd:
    spec:
      names:
        kind: K8sServiceAccountExists
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sserviceaccountexists
        violation[{"msg": msg}] {
          sa_name := input.review.object.spec.serviceAccountName
          sa_name != "default"
          # inventory is populated by the Config Sync cache
          count([sa | sa := data.inventory.namespace[_]["v1"]["ServiceAccount"][sa_name]]) == 0
          msg := sprintf("ServiceAccount '%v' does not exist", [sa_name])
        }
```

> ⚠️ **NOTE:** ConstraintTemplates must use API version `templates.gatekeeper.sh/v1` (stable) — v1alpha1 is deprecated. Import `future.keywords` for newer Rego syntax (`if`, `contains`). Referential constraints require `referentialRulesEnabled: true` in the ACM spec and the Policy Controller cache to be populated. Audit runs every 60 seconds by default — high constraint counts (> 500) degrade audit performance.

---

## ANTHOS SERVICE MESH (ASM)

Anthos Service Mesh is Google's managed distribution of Istio, providing mutual TLS, fine-grained traffic management, L7 authorization, and deep observability for microservices running inside a GKE Enterprise fleet. The **managed** variant delegates the Istio control plane lifecycle entirely to Google, eliminating operational overhead of running and upgrading `istiod` while retaining full Istio API compatibility.

### ASM Variants

| Variant | Control Plane | Data Plane | Upgrades | Best For |
|---|---|---|---|---|
| Managed ASM — ISTIOD | Google-managed `istiod` | Managed sidecar injection | Automatic (channel-based) | Default production choice |
| Managed ASM — TRAFFIC_DIRECTOR | Google-managed (xDS via Traffic Director) | Managed proxy injection | Automatic | High-scale, no istiod pod |
| In-cluster ASM | Self-managed `istiod` in `istio-system` | Self-managed injection | Manual | Custom Istio configuration |

### Enabling Managed ASM via Fleet

```bash
# Enable the Service Mesh feature on the fleet
gcloud container fleet mesh enable --project=FLEET_PROJECT_ID

# Enable automatic managed ASM on a specific cluster membership
gcloud container fleet mesh update \
  --management automatic \
  --memberships MEMBERSHIP_NAME \
  --project=FLEET_PROJECT_ID \
  --location=global

# Verify ASM provisioning status
gcloud container fleet mesh describe --project=FLEET_PROJECT_ID
```

### Sidecar Injection

```bash
# Enable sidecar injection for a namespace (in-cluster or managed ISTIOD)
kubectl label namespace my-app istio-injection=enabled

# For managed ASM — use the managed injection label instead
kubectl label namespace my-app \
  mesh.cloud.google.com/proxy='{"managed":"true"}'

# Revision-based injection (for canary upgrades — in-cluster only)
kubectl label namespace my-app istio.io/rev=asm-1-20 --overwrite

# Restart all pods to trigger injection
kubectl rollout restart deployment -n my-app
```

### Core Istio Resources

```yaml
# ── VirtualService — traffic routing, canary, header-based routing ──────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service-vs
  namespace: my-app
spec:
  hosts: ["my-service"]
  http:
    - match:
        - headers:
            x-env:
              exact: canary
      route:
        - destination:
            host: my-service
            subset: v2
    - route:
        - destination:
            host: my-service
            subset: v1
          weight: 90
        - destination:
            host: my-service
            subset: v2
          weight: 10
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx,gateway-error,reset"
---
# ── DestinationRule — subsets, load balancing, circuit breaking, mTLS ───────
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service-dr
  namespace: my-app
spec:
  host: my-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 3s
      http:
        h2UpgradePolicy: UPGRADE
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_CONN
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
# ── PeerAuthentication — enforce mTLS per namespace ──────────────────────────
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT          # STRICT | PERMISSIVE | DISABLE
---
# ── AuthorizationPolicy — L7 RBAC ────────────────────────────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/my-app/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
      when:
        - key: source.ip
          notValues: ["10.0.0.0/8"]
---
# ── Gateway — expose services externally via Istio ingress gateway ───────────
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: my-app
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: my-tls-secret
      hosts:
        - "app.example.com"
```

### ASM Telemetry

ASM automatically exports Istio metrics to Cloud Monitoring and traces to Cloud Trace without any additional instrumentation. Key metrics:

- `istio.io/service/server/request_count` — total inbound RPS per service
- `istio.io/service/server/response_latencies` — P50/P99/P999 latency distributions
- `istio.io/service/client/request_count` — outbound call counts (per source+destination)
- `istio.io/service/server/response_code` — HTTP response code distribution

### Multi-cluster Service Mesh (East-West Gateway)

```bash
# For cross-cluster mTLS service discovery, deploy east-west gateways
# on each cluster and expose them via an internal LB:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-eastwestgateway
  namespace: istio-system
  labels:
    app: istio-eastwestgateway
    istio: eastwestgateway
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: istio-eastwestgateway
  ports:
    - name: tls
      port: 15443
      targetPort: 15443
EOF
```

### istioctl CLI Reference

```bash
# Validate mesh configuration and detect common misconfigurations
istioctl analyze -n my-app

# Show xDS sync status of all sidecar proxies
istioctl proxy-status

# Inspect Envoy cluster config for a specific pod
istioctl proxy-config cluster my-pod-xxx -n my-app

# Inspect active routes for a pod
istioctl proxy-config route my-pod-xxx -n my-app

# Inspect listeners
istioctl proxy-config listener my-pod-xxx -n my-app

# Human-readable traffic policy explanation for a pod
istioctl x describe pod my-pod-xxx -n my-app

# Check if a source pod can reach a destination
istioctl x authz check my-pod-xxx -n my-app
```

> ⚠️ **NOTE:** Managed ASM `TRAFFIC_DIRECTOR` variant has **no `istiod` pod** — do not look for it in `istio-system`. Debug via `trafficdirector.googleapis.com` API and check proxy-status for xDS sync errors. Managed ASM uses release **channels** (`stable`, `regular`, `rapid`) — do not mix revision labels (`istio.io/rev`) with the managed injection label (`mesh.cloud.google.com/proxy`) on the same namespace.

---

## MULTI-CLUSTER INGRESS & MULTI-CLUSTER SERVICES

Multi-cluster Ingress (MCI) provides a single anycast Global External HTTPS Load Balancer VIP that distributes traffic across backend pods running in multiple GKE clusters across regions. It is backed by Cloud Load Balancing with NEG (Network Endpoint Group) backends auto-provisioned per cluster. Multi-cluster Services (MCS) complements MCI by enabling cross-cluster service discovery through a DNS-based API.

### Architecture

```text
                           Internet
                               │
                    ┌──────────▼──────────┐
                    │  Global Anycast VIP  │   ← single IP, GeoDNS
                    │ External HTTPS LB    │
                    └──────────┬──────────┘
                               │
              ┌────────────────▼──────────────────┐
              │         Config Cluster             │
              │  (hosts MCI + MCS CRD reconciler)  │
              │  MultiClusterIngress resource       │
              │  MultiClusterService resource       │
              └──────┬────────────────────┬────────┘
                     │                    │
           ┌─────────▼──────┐  ┌──────────▼──────┐
           │  Cluster A NEG │  │  Cluster B NEG  │
           │  us-central1   │  │  europe-west1   │
           │  Pod: my-app   │  │  Pod: my-app    │
           └────────────────┘  └─────────────────┘
```

### Enabling Multi-cluster Ingress Feature

```bash
# Enable MCI — specify which cluster is the "config cluster"
gcloud container fleet ingress enable \
  --config-membership=projects/FLEET_PROJECT/locations/global/memberships/CONFIG_CLUSTER \
  --project=FLEET_PROJECT_ID

# Verify feature status
gcloud container fleet ingress describe --project=FLEET_PROJECT_ID
```

### MultiClusterIngress + MultiClusterService YAMLs

```yaml
# Reserve a global static IP first:
# gcloud compute addresses create my-global-ip --global

apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: my-ingress
  namespace: my-app
  annotations:
    networking.gke.io/static-ip: my-global-ip
    networking.gke.io/pre-shared-certs: my-ssl-cert
spec:
  template:
    spec:
      backend:
        serviceName: my-mcs
        servicePort: 80
      rules:
        - host: app.example.com
          http:
            paths:
              - path: /api
                pathType: Prefix
                backend:
                  serviceName: my-mcs-api
                  servicePort: 8080
              - path: /
                pathType: Prefix
                backend:
                  serviceName: my-mcs
                  servicePort: 80
---
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: my-mcs
  namespace: my-app
  annotations:
    # BackendConfig for health check customization
    cloud.google.com/backend-config: '{"default": "my-backend-config"}'
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

### Multi-cluster Services (MCS) — Cross-Cluster Discovery

```yaml
# Export a service from a cluster (apply in source cluster)
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
  name: my-backend
  namespace: my-app
---
# ServiceImport is auto-created in consuming clusters
# DNS record auto-provisioned:
# my-backend.my-app.svc.clusterset.local
```

```bash
# Enable MCS fleet feature (separate from MCI)
gcloud container fleet multi-cluster-services enable \
  --project=FLEET_PROJECT_ID

# Grant the Hub SA permission to create ServiceImports
gcloud projects add-iam-policy-binding FLEET_PROJECT_ID \
  --member="serviceAccount:FLEET_PROJECT_ID.hub.id.goog" \
  --role="roles/multiclusterservicediscovery.serviceAgent"
```

> ⚠️ **NOTE:** The Config Cluster must reside in the **same project as the Fleet Host**. MCI requires a `BackendConfig` for custom health check paths; default health check is `/` on the target port. NEGs are automatically provisioned per cluster but take 3–5 minutes to appear as healthy backends. MCS (`multiclusterservicediscovery`) is a separate fleet feature from MCI (`multiclusteringress`) — enable both if you need both.

---

## WORKLOAD IDENTITY & SECURITY

GKE Enterprise provides a layered security model combining **Workload Identity Federation** (eliminating static service account keys), **Binary Authorization** (supply chain integrity enforcement), and the **GKE Security Posture Dashboard** (continuous runtime risk detection). Together they enforce a zero-trust posture from image build through runtime.

### Workload Identity Federation

```bash
# Step 1: Ensure the cluster has a Workload Pool
gcloud container clusters describe CLUSTER_NAME \
  --format="value(workloadIdentityConfig.workloadPool)"
# Should return: FLEET_PROJECT_ID.svc.id.goog

# Step 2: Bind the Kubernetes Service Account (KSA) to a Google Service Account (GSA)
gcloud iam service-accounts add-iam-policy-binding GSA_EMAIL \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:FLEET_PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]" \
  --project=GCP_PROJECT_ID

# Step 3: Annotate the KSA
kubectl annotate serviceaccount KSA_NAME \
  --namespace NAMESPACE \
  iam.gke.io/gcp-service-account=GSA_EMAIL

# Verify token exchange works from inside a pod
kubectl run -it --rm debug --image=google/cloud-sdk:slim \
  --serviceaccount=KSA_NAME -n NAMESPACE \
  -- gcloud auth print-access-token
```

### Binary Authorization

```bash
# Export current policy for editing
gcloud container binauthz policy export > policy.yaml

# Import updated policy
gcloud container binauthz policy import policy.yaml

# Create an attestor backed by a Container Analysis Note
gcloud container binauthz attestors create built-by-cloud-build \
  --attestation-authority-note=projects/PROJECT/notes/cloudbuild-note \
  --attestation-authority-note-project=PROJECT

# Add an attestor public key (KMS asymmetric signing key)
gcloud container binauthz attestors public-keys add \
  --attestor=built-by-cloud-build \
  --keyversion-project=PROJECT \
  --keyversion-location=global \
  --keyversion-keyring=binauthz-keys \
  --keyversion-key=build-signing-key \
  --keyversion=1
```

```yaml
# Binary Authorization policy
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/PROJECT_ID/attestors/built-by-cloud-build

clusterAdmissionRules:
  us-central1.prod-cluster:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
      - projects/PROJECT_ID/attestors/built-by-cloud-build
      - projects/PROJECT_ID/attestors/security-team-attestor
  us-central1.dev-cluster:
    evaluationMode: ALWAYS_ALLOW
    enforcementMode: DRYRUN_AUDIT_LOG_ONLY

globalPolicyEvaluationMode: ENABLE
```

### GKE Security Posture Dashboard

Enable via cluster configuration:

```bash
gcloud container clusters update CLUSTER_NAME \
  --location=LOCATION \
  --enable-workload-vulnerability-scanning \
  --security-posture=standard    # standard | enterprise
```

Detection categories:
- **Vulnerability scanning** — OS-level CVEs in running container images
- **Workload misconfiguration** — privileged containers, host PID/network, writable root FS
- **RBAC analysis** — overpermissive ClusterRoleBindings, wildcard verbs
- **Runtime threat detection** — anomalous process execution, cryptomining patterns

### Fleet-Level IAM Roles

| Role | Scope | Purpose |
|---|---|---|
| `roles/gkehub.viewer` | Fleet project | Read memberships and features |
| `roles/gkehub.editor` | Fleet project | Create/update/delete fleet resources |
| `roles/gkehub.gatewayReader` | Fleet project | Read-only Connect Gateway (kubectl get) |
| `roles/gkehub.gatewayEditor` | Fleet project | Read-write Connect Gateway (full kubectl) |
| `roles/gkehub.gatewayAdmin` | Fleet project | Full Connect Gateway + IAM management |
| `roles/mesh.admin` | Fleet project | Manage ASM feature and configuration |
| `roles/meshconfig.admin` | Fleet project | Read/write mesh telemetry config |

### Secret Manager Integration — Comparison

| Aspect | External Secrets Operator (ESO) | Secrets Store CSI Driver |
|---|---|---|
| Sync mechanism | Kubernetes Secret (copy) | Volume mount (in-memory) |
| Secret rotation | Automatic (polling interval) | On pod restart |
| Access without pod restart | ✅ Yes (Secret updated) | ❌ No |
| GCP auth | Workload Identity | Workload Identity |
| Namespace isolation | Per SecretStore CRD | Per SecretProviderClass CRD |
| Env var support | ✅ Native (via Secret) | ✅ Via secretObjects |
| Recommended for | Broad rotation needs | Ephemeral secrets, no Secret copy |

---

## CONFIG CONTROLLER & INFRASTRUCTURE MANAGEMENT

Config Controller is a fully managed, hosted instance of Config Connector, Policy Controller, and Config Sync, providing a **GitOps-driven infrastructure management** plane for GCP resources. Rather than writing Terraform HCL, platform teams define GCP resources as Kubernetes CRDs and push them to a Git repository — Config Sync applies them to the Config Controller cluster, and Config Connector reconciles the resulting GCP API calls continuously.

### Config Controller vs Terraform

| Aspect | Config Controller | Terraform |
|---|---|---|
| API / DSL | Kubernetes CRDs (YAML) | HCL |
| State storage | etcd (Kubernetes) | `.tfstate` file (local/GCS) |
| Drift detection | Continuous (controller loop) | On explicit `terraform plan` |
| GitOps native | ✅ Built-in (Config Sync) | Via external CD pipeline |
| GCP resource support | Config Connector CRDs (300+) | `google` provider (400+) |
| Multi-cloud | GCP only | Multi-cloud |
| Secret handling | Secret Manager + ESO | Vault / Terraform Vault provider |
| RBAC model | Kubernetes RBAC | Workspace-level policies |

### Creating a Config Controller Instance

```bash
# Create the Config Controller instance (takes ~15 min)
gcloud anthos config controller create my-config-controller \
  --location=us-central1 \
  --project=MANAGEMENT_PROJECT_ID \
  --man-block=10.128.0.0/28

# Get credentials for the Config Controller cluster
gcloud anthos config controller get-credentials my-config-controller \
  --location=us-central1 \
  --project=MANAGEMENT_PROJECT_ID

# Grant the Config Controller SA org-level permissions
export SA_EMAIL=$(kubectl get ConfigConnectorContext -n config-control \
  -o jsonpath='{.items[0].spec.googleServiceAccount}')

gcloud organizations add-iam-policy-binding ORG_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/resourcemanager.projectCreator"
```

### Config Connector Resource Examples

```yaml
# ── GCS Bucket ───────────────────────────────────────────────────────────────
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: my-prod-bucket
  namespace: config-control
  annotations:
    cnrm.cloud.google.com/project-id: "my-gcp-project"
    cnrm.cloud.google.com/deletion-policy: "abandon"   # prevent accidental deletion
spec:
  location: US-CENTRAL1
  storageClass: STANDARD
  uniformBucketLevelAccess: true
  versioning:
    enabled: true
  lifecycleRule:
    - action:
        type: Delete
      condition:
        age: 90
---
# ── IAM Service Account ───────────────────────────────────────────────────────
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMServiceAccount
metadata:
  name: my-app-sa
  namespace: config-control
  annotations:
    cnrm.cloud.google.com/project-id: "my-gcp-project"
spec:
  displayName: "My Application Service Account"
---
# ── IAM Policy Member (grant roles to the SA above) ──────────────────────────
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: my-app-sa-secret-accessor
  namespace: config-control
spec:
  member: serviceAccount:my-app-sa@my-gcp-project.iam.gserviceaccount.com
  role: roles/secretmanager.secretAccessor
  resourceRef:
    apiVersion: resourcemanager.cnrm.cloud.google.com/v1beta1
    kind: Project
    external: projects/my-gcp-project
---
# ── Pub/Sub Topic ─────────────────────────────────────────────────────────────
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  name: my-events-topic
  namespace: config-control
  annotations:
    cnrm.cloud.google.com/project-id: "my-gcp-project"
spec:
  messageRetentionDuration: "86400s"
```

> ⚠️ **NOTE:** Config Controller is **single-region** — if the region goes down, your IaC control plane is unavailable (though existing GCP resources continue to run). By default, deleting a Config Connector CR **deletes the underlying GCP resource** — always annotate production resources with `cnrm.cloud.google.com/deletion-policy: abandon`. Config Connector's Google Service Account needs broad IAM permissions; scope them to specific projects rather than org-wide where possible.

---

## ANTHOS ON-PREMISES & ATTACHED CLUSTERS

GKE Enterprise extends its Fleet management capabilities beyond GCP to clusters running on-premises (bare metal, VMware) and in other clouds (EKS, AKS). These "attached clusters" register with the Fleet Hub, enabling the same Config Sync, Policy Controller, and ASM features as native GKE clusters, with some limitations for cross-cloud networking features like Multi-cluster Ingress.

### Anthos on Bare Metal (ABM) — Deployment Modes

| Mode | Description | Use Case |
|---|---|---|
| Standalone | Single cluster performs both admin and user workload roles | Small edge sites, single-node |
| Admin + User | Separate admin cluster manages multiple user clusters | Centralized enterprise ops |
| Hybrid | Admin plane runs on user cluster (co-located) | Medium deployments, resource-constrained |
| Multi-cluster | Multiple user clusters managed by one admin cluster | Large enterprise, multi-site |

### Anthos Clusters on VMware (Formerly Anthos on vSphere)

```text
┌──────────────────────────────────────────────────────────────┐
│                      vCenter / ESXi                          │
│                                                              │
│  ┌──────────────────┐      ┌────────────────────────────┐   │
│  │  Admin Workstation│      │      Admin Cluster         │   │
│  │  (gkectl tooling) │─────►│  (manages user clusters)   │   │
│  └──────────────────┘      └──────────┬─────────────────┘   │
│                                        │                     │
│          ┌─────────────────────────────┼──────────────────┐  │
│          │                             │                  │  │
│  ┌───────▼──────┐            ┌─────────▼──────┐           │  │
│  │ User Cluster │            │ User Cluster   │           │  │
│  │    Prod      │            │    Staging     │           │  │
│  └──────────────┘            └────────────────┘           │  │
│                                                            │  │
│  Load Balancer Options: F5 BIG-IP | Seesaw | MetalLB      │  │
└───────────────────────────────────────────────────────────-┘
```

### Attached Clusters — Feature Support Matrix

| Feature | GKE (native) | Attached EKS | Attached AKS | Bare Metal | VMware |
|---|---|---|---|---|---|
| Config Sync | ✅ | ✅ | ✅ | ✅ | ✅ |
| Policy Controller | ✅ | ✅ | ✅ | ✅ | ✅ |
| Anthos Service Mesh | ✅ | ✅ | ✅ | ✅ | ✅ |
| Connect Gateway | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multi-cluster Ingress | ✅ | ❌ | ❌ | ❌ | ❌ |
| Workload Identity | ✅ | ✅ (via WIF) | ✅ (via WIF) | ❌ | ❌ |
| GKE Security Posture | ✅ | ✅ (partial) | ✅ (partial) | ✅ (partial) | ✅ (partial) |
| Managed Prometheus | ✅ | ✅ | ✅ | ✅ | ✅ |

### Registering an EKS Cluster

```bash
# Prerequisites: kubectl context for EKS cluster, AWS credentials configured

# Register EKS cluster with Fleet
gcloud container fleet memberships register eks-prod \
  --context=arn:aws:eks:us-east-1:123456789012:cluster/prod \
  --kubeconfig=/path/to/kubeconfig \
  --enable-workload-identity \
  --has-private-issuer \
  --project=FLEET_PROJECT_ID

# Verify the Connect Agent is running in the EKS cluster
kubectl get pods -n gke-connect --context=arn:aws:eks:us-east-1:123456789012:cluster/prod

# Apply ACM config to the EKS membership
gcloud container fleet config-management apply \
  --membership=eks-prod \
  --config=apply-spec.yaml \
  --project=FLEET_PROJECT_ID
```

> 💡 **TIP:** For EKS and AKS attached clusters, Workload Identity Federation requires the cluster's OIDC issuer to be reachable by Google's STS. Use `--has-private-issuer` flag and register the issuer URL manually if the EKS OIDC endpoint is private.

---

## OBSERVABILITY & LOGGING FOR GKE ENTERPRISE

GKE Enterprise integrates natively with Cloud Operations Suite — providing Managed Prometheus for metrics, Cloud Logging for structured log aggregation, and Cloud Trace via ASM for distributed tracing, all without requiring custom exporters or agents. Fleet-level dashboards aggregate signals across all registered clusters into a unified observability plane.

### Managed Prometheus (GMP)

```bash
# Enable managed collection on a cluster
kubectl patch operatorconfig config -n gmp-system \
  --type=merge --patch '{"collection":{"enabled":true}}'

# Verify managed collector pods are running
kubectl get pods -n gmp-system -l app.kubernetes.io/name=collector
```

```yaml
# PodMonitoring — scrape metrics from a specific pod label selector
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: my-app-metrics
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      timeout: 10s
  targetLabels:
    metadata:
      - pod
      - namespace
---
# ClusterPodMonitoring — cluster-wide scraping (cluster-scoped resource)
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: kube-state-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  endpoints:
    - port: http-metrics
      interval: 60s
```

### Cloud Logging — Log Sinks & Exclusions

```bash
# Create a log sink to export GKE workload logs to BigQuery
gcloud logging sinks create gke-workload-sink \
  bigquery.googleapis.com/projects/PROJECT/datasets/gke_logs \
  --log-filter='resource.type="k8s_container" AND resource.labels.cluster_name="prod-cluster"' \
  --project=CLUSTER_PROJECT

# Create a log exclusion to reduce noise from health check endpoints
gcloud logging exclusions create suppress-health-checks \
  --filter='resource.type="k8s_container" AND textPayload:"GET /healthz"' \
  --project=CLUSTER_PROJECT

# Stream logs from a specific namespace to stdout
gcloud logging read \
  'resource.type="k8s_container" AND resource.labels.namespace_name="my-app"' \
  --project=CLUSTER_PROJECT \
  --freshness=10m \
  --format=json
```

### Key GKE Metrics Reference

| Metric | Description | Suggested Alert Threshold |
|---|---|---|
| `kubernetes.io/container/cpu/limit_utilization` | CPU usage vs CPU limit per container | `> 0.85` for 5 min |
| `kubernetes.io/container/memory/limit_utilization` | Memory usage vs memory limit | `> 0.90` for 5 min |
| `kubernetes.io/container/restart_count` | Container restart count delta | `> 5` in 10 min |
| `kubernetes.io/pod/volume/used_bytes` | PVC bytes used | `> 80%` of capacity |
| `kubernetes.io/node/cpu/allocatable_utilization` | Node-level CPU saturation | `> 0.80` for 10 min |
| `kubernetes.io/node/memory/allocatable_utilization` | Node-level memory pressure | `> 0.85` for 5 min |
| `istio.io/service/server/request_count` | Inbound RPS per service | Baseline × 2 spike |
| `istio.io/service/server/response_latencies` | P99 request latency | Exceeds SLO budget |
| `istio.io/service/server/response_code` | 5xx error rate | `> 1%` for 2 min |

> 💡 **TIP:** Define **SLOs in Cloud Monitoring** against Istio request metrics rather than synthetic monitors — this gives you automatic burn rate alerting. Use the `istio.io/service/server/response_latencies` metric with percentile aggregation to track P99/P999 latency budgets.

---

## GITOPS WORKFLOW PATTERNS

GitOps with GKE Enterprise centers on a **single source of truth** in Git: all Kubernetes manifests, Constraints, NetworkPolicies, RBAC, and even the ACM configuration itself are committed to a repository and continuously reconciled by Config Sync. The Fleet's ability to select subsets of clusters via ClusterSelectors enables environment-targeted rollouts without maintaining separate repositories per environment.

### Reference Fleet Repository Structure (Hierarchy Format)

```text
fleet-config/
├── system/
│   └── repo.yaml                    # required for hierarchy format
├── cluster/                         # cluster-scoped resources
│   ├── clusterrolebindings/
│   │   └── view-binding.yaml
│   └── namespaces/
│       ├── my-app.yaml
│       └── monitoring.yaml
├── clusterregistry/                 # ClusterSelector definitions
│   ├── prod-clusters.yaml
│   ├── staging-clusters.yaml
│   └── us-clusters.yaml
├── namespaces/                      # namespace-scoped resources
│   ├── my-app/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── networkpolicy.yaml
│   │   └── hpa.yaml
│   └── monitoring/
│       ├── podmonitoring.yaml
│       └── alertpolicies.yaml
└── abstract-namespaces/             # inherited namespace configs
    ├── base/
    │   ├── resourcequota.yaml
    │   └── limitrange.yaml
    └── prod/
        └── networkpolicy-strict.yaml
```

### ClusterSelector — Environment-Targeted Configs

```yaml
# Define a ClusterSelector in clusterregistry/
apiVersion: configmanagement.gke.io/v1
kind: ClusterSelector
metadata:
  name: selector-prod
spec:
  selector:
    matchLabels:
      environment: prod
---
# Label clusters at registration time or via Hub
# (labels applied in the Fleet membership resource)

# Annotate a resource to target only prod clusters:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
  annotations:
    configmanagement.gke.io/cluster-selector: selector-prod
spec:
  replicas: 3
  # ...
```

```bash
# Label a fleet membership for ClusterSelector targeting
gcloud container fleet memberships update MEMBERSHIP_NAME \
  --update-labels environment=prod \
  --project=FLEET_PROJECT_ID
```

### NamespaceSelector

```yaml
apiVersion: configmanagement.gke.io/v1
kind: NamespaceSelector
metadata:
  name: selector-team-platform
spec:
  selector:
    matchLabels:
      team: platform

# Annotate a policy to apply only to "team: platform" namespaces
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
  annotations:
    configmanagement.gke.io/namespace-selector: selector-team-platform
spec:
  enforcementAction: deny
  # ...
```

### OCI Image Source for Config Sync

Using OCI images removes Git network dependencies and enables content-addressable, immutable config snapshots:

```yaml
# In apply-spec.yaml or RootSync CRD
spec:
  configSync:
    enabled: true
    sourceType: oci
    oci:
      image: us-central1-docker.pkg.dev/PROJECT/fleet-config/config:latest
      dir: clusters/prod
      auth: gcpserviceaccount
      gcpServiceAccountEmail: config-sync-sa@PROJECT.iam.gserviceaccount.com
```

```bash
# Build and push config OCI image to Artifact Registry
cd ./config-root
tar -czf - . | docker import - fleet-config:latest
docker tag fleet-config:latest \
  us-central1-docker.pkg.dev/PROJECT/fleet-config/config:latest
docker push us-central1-docker.pkg.dev/PROJECT/fleet-config/config:latest

# Or use crane for OCI-native push (no Docker daemon required)
crane append \
  --base gcr.io/distroless/static \
  --new_layer <(tar -czf - -C ./config-root .) \
  --tag us-central1-docker.pkg.dev/PROJECT/fleet-config/config:latest
```

> 💡 **TIP:** Use OCI image digests (not tags) in production RootSync specs to make config immutable: `image: ...config@sha256:abc123`. Pin to a digest and update it via a CD pipeline after validation — this gives you atomic, auditable config rollouts identical to how you deploy application images.

---

## TROUBLESHOOTING DECISION TREES

### Cluster Not Syncing (Config Sync)

```text
RootSync status shows Error?
├── YES → kubectl describe rootsync root-sync -n config-management-system
│         Check .status.conditions[].reason:
│         ├── "GitAuth" / "FetchFailed"
│         │     → Check Secret in config-management-system namespace
│         │       kubectl get secret git-credentials -n config-management-system
│         │     → Verify secretType matches (token | ssh | gcpserviceaccount)
│         │     → For gcpserviceaccount: verify WIF annotation on reconciler SA
│         ├── "Parsing" / "InvalidSyntax"
│         │     → Run: nomos vet --path=./config-root
│         │     → Check for tabs vs spaces in YAML
│         │     → Verify repo.yaml exists (hierarchy format)
│         ├── "ResourceConflict"
│         │     → Duplicate resource names across RootSync + RepoSync
│         │     → kubectl get rootsync,reposync -A to find overlap
│         └── "KNV" error codes
│               → KNV1021: resource managed by different source
│               → KNV1047: illegal namespace for cluster-scoped resource
│
└── NO — sync appears Stale (commit hash not advancing)?
          ├── Check reconciler pod is running:
          │   kubectl get pods -n config-management-system
          │   -l configsync.gke.io/reconciler=root-reconciler
          ├── Check reconciler logs:
          │   kubectl logs -n config-management-system
          │   -l app=reconciler-manager --tail=50
          ├── preventDrift blocked a manual change?
          │   → Check Cloud Audit Logs for "deny" from Config Sync webhook
          └── Git repo permissions changed?
                → Re-validate token/SSH key has repo read access
```

### Policy Controller Constraint Not Enforcing

```text
Constraint created but admission not blocked?
├── Check enforcementAction:
│   kubectl get CONSTRAINT_KIND CONSTRAINT_NAME -o jsonpath='{.spec.enforcementAction}'
│   └── Must be "deny" — "dryrun" and "warn" do not block admission
│
├── Constraint status — is it enforced on this node?
│   kubectl describe CONSTRAINT_KIND CONSTRAINT_NAME
│   └── .status.byPod[].enforced must be true for the pod's node
│
├── Validating webhook registered and active?
│   kubectl get validatingwebhookconfiguration | grep gatekeeper
│   kubectl describe validatingwebhookconfiguration gatekeeper-validating-webhook-configuration
│   └── Check .webhooks[].rules[] matches the resource kind you're testing
│
├── Pod's namespace excluded?
│   Check spec.match.excludedNamespaces in the constraint
│
├── Pod's namespace missing the required label?
│   Check spec.match.namespaceSelector — does the namespace have required labels?
│   kubectl get namespace my-ns --show-labels
│
└── Gatekeeper controller logs showing errors?
    kubectl logs -n gatekeeper-system
    -l control-plane=controller-manager --tail=50
```

### ASM Sidecar Not Injected

```text
Pod missing istio-proxy container?
├── Namespace injection label present?
│   kubectl get namespace NAMESPACE --show-labels
│   ├── Missing "istio-injection=enabled"
│   │   → kubectl label namespace NAMESPACE istio-injection=enabled
│   │   → kubectl rollout restart deployment -n NAMESPACE
│   └── Has "istio.io/rev=X" (revision-based) but using managed injection?
│       → Pick one: either use istio.io/rev OR mesh.cloud.google.com/proxy
│
├── Pod-level injection opt-out?
│   kubectl get pod POD_NAME -o jsonpath='{.metadata.annotations}'
│   └── sidecar.istio.io/inject: "false" → remove the annotation
│
├── Managed ASM provisioning complete?
│   gcloud container fleet mesh describe --project=PROJECT
│   └── .membershipStates[MEMBERSHIP].servicemesh.controlPlaneManagement.state
│       must be "ACTIVE" (not "PROVISIONING")
│
├── In-cluster ASM: istiod running?
│   kubectl get pods -n istio-system -l app=istiod
│   └── Not running → check istiod logs, resource limits, PodDisruptionBudget
│
└── MutatingWebhookConfiguration registered?
    kubectl get mutatingwebhookconfiguration | grep istio
    └── Missing → istiod failed to start or register webhook
```

### Connect Gateway kubectl Auth Failure

```text
Error: failed to get credentials / 403 Forbidden?
├── IAM permissions:
│   gcloud projects get-iam-policy FLEET_PROJECT_ID \
│     --flatten="bindings[].members" \
│     --filter="bindings.members:user:YOU@example.com"
│   └── Needs: roles/gkehub.gatewayEditor (or gatewayAdmin)
│
├── RBAC on the member cluster:
│   kubectl auth can-i get pods \
│     --as=user:YOU@example.com -n default
│   └── Needs a ClusterRoleBinding on the member cluster granting access
│
├── Membership registered?
│   gcloud container fleet memberships list --project=FLEET_PROJECT_ID
│   └── Missing → register the cluster (see Fleet Management section)
│
├── Connect Agent healthy on member cluster?
│   kubectl get pods -n gke-connect
│   └── CrashLoopBackOff → check SA permissions, Hub connectivity
│   kubectl logs -n gke-connect -l app=gke-connect-agent
│
└── Correct fleet project in kubeconfig?
    gcloud container fleet memberships get-credentials MEMBERSHIP \
      --project=FLEET_PROJECT_ID
    kubectl config current-context
    → Context must reference connectgateway.googleapis.com endpoint
```

---

## LIMITS, QUOTAS & COST OPTIMIZATION

Understanding GKE Enterprise's resource limits is critical for capacity planning, and its per-vCPU billing model rewards right-sizing above all else. The limits below represent defaults — most soft limits can be increased by filing a quota request with Google Cloud support.

### Key Limits & Quotas

| Resource | Default Limit | Notes |
|---|---|---|
| Fleet memberships per project | 1,000 | Soft limit — file quota request to increase |
| Fleet features per project | 25 | Covers ACM, ASM, MCI, etc. |
| RootSync per cluster | 1 | One cluster-wide GitOps root |
| RepoSync per cluster | Up to 1 per namespace | No hard cap; performance degrades > 100 |
| Policy Controller ConstraintTemplates | No hard limit | Audit degrades noticeably > 500 constraints |
| ASM managed control plane revisions | 1 active per cluster | Can run alongside 1 in-cluster revision |
| MultiClusterIngress objects per fleet | 100 | Hard limit |
| Config Controller instances | 1 per project per region | Request increase for multi-region |
| Connect Gateway concurrent sessions | 10 per membership | Per-user API rate limits apply |
| Config Sync sync period | Minimum 10s | Default 15s; lower values increase API server load |

### GKE Enterprise Pricing Levers

- **Billing unit:** per registered vCPU-hour across all fleet cluster nodes (control plane excluded)
- **Node type matters:** only schedulable worker node vCPUs count — tainting nodes as unschedulable does not exclude them
- **Autopilot exception:** GKE Autopilot clusters in an Enterprise fleet bill at Autopilot pod rates, not the Enterprise per-vCPU rate
- **Attached clusters:** AWS/Azure node vCPUs are billed at the Enterprise rate despite running outside GCP

### Cost Optimization Checklist

- [ ] Enable **Cluster Autoscaler** on all user clusters — prevents idle node vCPUs from accumulating Enterprise fees
- [ ] Run stateless workloads on **Spot / Preemptible** node pools — 60–90% cheaper Compute, same Enterprise licensing
- [ ] Use **GKE Autopilot** for workloads compatible with its restrictions — no per-vCPU Enterprise surcharge
- [ ] Apply **Vertical Pod Autoscaler (VPA)** recommendations before registering clusters — right-sizing reduces total vCPU count
- [ ] Purchase **Committed Use Discounts (CUDs)** for predictable node pool capacity
- [ ] Audit **unused Fleet memberships** quarterly — deregistering saves per-vCPU fees on idle clusters
- [ ] Use **OCI images** for Config Sync instead of Git — eliminates Git egress costs for large repos across many clusters
- [ ] Enable **GKE usage metering** to attribute Enterprise costs to teams by namespace
- [ ] Use **Node Auto-Provisioning (NAP)** to bin-pack workloads onto fewer nodes

> 💡 **TIP:** Run `gcloud container clusters describe CLUSTER --format="value(currentNodeCount)"` across all fleet clusters combined with instance machine type vCPU counts to calculate your current Enterprise billing exposure. Integrate this into a monthly FinOps report.

---

## TERRAFORM REFERENCE FOR GKE ENTERPRISE

This end-to-end Terraform module provisions a production GKE Enterprise cluster, registers it to a Fleet, enables ACM with Config Sync + Policy Controller, and enables Managed ASM — representing a complete Enterprise baseline.

```hcl
locals {
  fleet_project   = "my-fleet-project"
  cluster_project = "my-cluster-project"
  location        = "us-central1"
  cluster_name    = "prod-cluster"
  sync_repo       = "https://github.com/org/fleet-config"
}

# ── APIs ──────────────────────────────────────────────────────────────────────
resource "google_project_service" "apis" {
  for_each = toset([
    "container.googleapis.com",
    "gkehub.googleapis.com",
    "anthos.googleapis.com",
    "anthosconfigmanagement.googleapis.com",
    "mesh.googleapis.com",
    "meshconfig.googleapis.com",
    "monitoring.googleapis.com",
    "binaryauthorization.googleapis.com",
  ])
  project            = local.cluster_project
  service            = each.value
  disable_on_destroy = false
}

# ── Node SA ───────────────────────────────────────────────────────────────────
resource "google_service_account" "node_sa" {
  account_id   = "gke-node-sa"
  display_name = "GKE Node Service Account"
  project      = local.cluster_project
}

resource "google_project_iam_member" "node_sa_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/artifactregistry.reader",
  ])
  project = local.cluster_project
  role    = each.value
  member  = "serviceAccount:${google_service_account.node_sa.email}"
}

# ── GKE Cluster ───────────────────────────────────────────────────────────────
resource "google_container_cluster" "prod" {
  name     = local.cluster_name
  project  = local.cluster_project
  location = local.location

  remove_default_node_pool = true
  initial_node_count       = 1

  # Fleet registration
  fleet {
    project = local.fleet_project
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${local.fleet_project}.svc.id.goog"
  }

  # Dataplane V2 (eBPF-based networking + NetworkPolicy)
  datapath_provider = "ADVANCED_DATAPATH"

  # Binary Authorization
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # Logging & Monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS", "API_SERVER", "SCHEDULER"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "APISERVER", "CONTROLLER_MANAGER", "SCHEDULER"]
    managed_prometheus {
      enabled = true
    }
  }

  # Security posture
  security_posture_config {
    mode               = "ENTERPRISE"
    vulnerability_mode = "VULNERABILITY_ENTERPRISE"
  }

  # Private cluster (recommended for production)
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  ip_allocation_policy {}

  release_channel {
    channel = "REGULAR"
  }

  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }

  depends_on = [google_project_service.apis]
}

# ── Node Pool ──────────────────────────────────────────────────────────────────
resource "google_container_node_pool" "prod_nodes" {
  name     = "prod-node-pool"
  project  = local.cluster_project
  cluster  = google_container_cluster.prod.name
  location = local.location

  autoscaling {
    min_node_count  = 2
    max_node_count  = 10
    location_policy = "BALANCED"
  }

  node_config {
    machine_type    = "e2-standard-4"
    disk_size_gb    = 100
    disk_type       = "pd-ssd"
    image_type      = "COS_CONTAINERD"
    service_account = google_service_account.node_sa.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"    # required for Workload Identity
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    labels = {
      environment = "prod"
      managed-by  = "terraform"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    strategy        = "SURGE"
    max_surge       = 1
    max_unavailable = 0
  }
}

# ── Fleet Features ─────────────────────────────────────────────────────────────
resource "google_gke_hub_feature" "acm" {
  name     = "configmanagement"
  project  = local.fleet_project
  location = "global"
}

resource "google_gke_hub_feature" "asm" {
  name     = "servicemesh"
  project  = local.fleet_project
  location = "global"
}

resource "google_gke_hub_feature" "mcs" {
  name     = "multiclusterservicediscovery"
  project  = local.fleet_project
  location = "global"
}

# ── ACM Feature Membership ────────────────────────────────────────────────────
resource "google_gke_hub_feature_membership" "acm_membership" {
  project    = local.fleet_project
  location   = "global"
  feature    = google_gke_hub_feature.acm.name
  membership = google_container_cluster.prod.fleet[0].membership

  configmanagement {
    version = "1.19.0"

    config_sync {
      source_format = "unstructured"
      git {
        sync_repo   = local.sync_repo
        sync_branch = "main"
        policy_dir  = "clusters/prod"
        secret_type = "gcpserviceaccount"
        gcp_service_account_email = google_service_account.config_sync_sa.email
      }
    }

    policy_controller {
      enabled                    = true
      template_library_installed = true
      referential_rules_enabled  = true
      log_denies_enabled         = true
      audit_interval_seconds     = 60
    }
  }
}

# ── Config Sync Service Account ───────────────────────────────────────────────
resource "google_service_account" "config_sync_sa" {
  account_id   = "config-sync-sa"
  display_name = "Config Sync Service Account"
  project      = local.cluster_project
}

resource "google_project_iam_member" "config_sync_ar_reader" {
  project = local.cluster_project
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.config_sync_sa.email}"
}

# Allow Config Sync KSA to impersonate the GSA via Workload Identity
resource "google_service_account_iam_member" "config_sync_wi" {
  service_account_id = google_service_account.config_sync_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member = "serviceAccount:${local.fleet_project}.svc.id.goog[config-management-system/root-reconciler]"
}
```

---

## QUICK REFERENCE COMMAND CHEATSHEET

```bash
# ─── FLEET ────────────────────────────────────────────────────────────────────
gcloud container fleet memberships list --project=PROJECT
gcloud container fleet memberships describe MEMBERSHIP --project=PROJECT
gcloud container fleet memberships get-credentials MEMBERSHIP --project=PROJECT
gcloud container fleet memberships update MEMBERSHIP --update-labels env=prod --project=PROJECT
gcloud container fleet features list --project=PROJECT

# ─── CONFIG MANAGEMENT ────────────────────────────────────────────────────────
gcloud container fleet config-management enable --project=PROJECT
gcloud container fleet config-management status --project=PROJECT
gcloud container fleet config-management apply \
  --membership=MEMBERSHIP --config=apply-spec.yaml --project=PROJECT
gcloud container fleet config-management fetch-for-apply \
  --membership=MEMBERSHIP --project=PROJECT
nomos status
nomos vet --path=./config-root
nomos hydrate --path=./config-root
kubectl get rootsync -n config-management-system
kubectl get reposync -A
kubectl describe rootsync root-sync -n config-management-system
kubectl logs -n config-management-system -l app=reconciler-manager -f

# ─── POLICY CONTROLLER ────────────────────────────────────────────────────────
kubectl get constrainttemplates
kubectl get constraints
kubectl describe constrainttemplate TEMPLATE_NAME
kubectl get CONSTRAINT_KIND CONSTRAINT_NAME -o yaml
kubectl get CONSTRAINT_KIND CONSTRAINT_NAME \
  -o jsonpath='{.status.violations[*].message}'
kubectl logs -n gatekeeper-system -l control-plane=controller-manager -f
kubectl logs -n gatekeeper-system -l control-plane=audit-controller -f

# ─── SERVICE MESH ─────────────────────────────────────────────────────────────
gcloud container fleet mesh enable --project=PROJECT
gcloud container fleet mesh describe --project=PROJECT
gcloud container fleet mesh update \
  --management automatic --memberships MEMBERSHIP --project=PROJECT --location=global
istioctl analyze -n NAMESPACE
istioctl proxy-status
istioctl proxy-config cluster POD_NAME -n NAMESPACE
istioctl proxy-config route POD_NAME -n NAMESPACE
istioctl proxy-config listener POD_NAME -n NAMESPACE
istioctl x describe pod POD_NAME -n NAMESPACE
istioctl x authz check POD_NAME -n NAMESPACE
kubectl get peerauthentication -A
kubectl get authorizationpolicy -A
kubectl get virtualservice -A
kubectl get destinationrule -A
kubectl get gateway -A
kubectl get envoyfilter -A

# ─── MULTI-CLUSTER INGRESS ────────────────────────────────────────────────────
gcloud container fleet ingress enable \
  --config-membership=projects/PROJECT/locations/global/memberships/CONFIG_CLUSTER \
  --project=PROJECT
kubectl get multiclusteringress -A
kubectl get multiclusterservice -A
kubectl get serviceexport -A
kubectl get serviceimport -A
kubectl describe multiclusteringress MY_MCI -n NAMESPACE

# ─── CONFIG CONTROLLER ────────────────────────────────────────────────────────
gcloud anthos config controller create NAME --location=REGION --project=PROJECT
gcloud anthos config controller list --location=REGION --project=PROJECT
gcloud anthos config controller get-credentials NAME --location=REGION --project=PROJECT
gcloud anthos config controller delete NAME --location=REGION --project=PROJECT
kubectl get gcp -A                         # list all Config Connector resources
kubectl get storagebucket -A
kubectl get iamserviceaccount -A
kubectl get iampolicymember -A
kubectl describe storagebucket BUCKET_NAME -n config-control

# ─── WORKLOAD IDENTITY ────────────────────────────────────────────────────────
gcloud iam service-accounts add-iam-policy-binding GSA_EMAIL \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:FLEET_PROJECT.svc.id.goog[NAMESPACE/KSA]"
kubectl annotate serviceaccount KSA \
  --namespace NAMESPACE \
  iam.gke.io/gcp-service-account=GSA_EMAIL

# ─── BINARY AUTHORIZATION ─────────────────────────────────────────────────────
gcloud container binauthz policy export
gcloud container binauthz policy import policy.yaml
gcloud container binauthz attestors list --project=PROJECT
gcloud container binauthz attestors describe ATTESTOR --project=PROJECT
gcloud container binauthz attestations list \
  --attestor=ATTESTOR \
  --attestor-project=PROJECT \
  --artifact-url=IMAGE_URL

# ─── SECURITY POSTURE ─────────────────────────────────────────────────────────
gcloud container clusters update CLUSTER \
  --location=LOCATION \
  --security-posture=enterprise \
  --enable-workload-vulnerability-scanning
gcloud container clusters describe CLUSTER --location=LOCATION \
  --format="value(securityPostureConfig)"

# ─── MANAGED PROMETHEUS ───────────────────────────────────────────────────────
kubectl get podmonitoring -A
kubectl get clusterpodmonitoring -A
kubectl describe podmonitoring MY_PODMONITORING -n NAMESPACE
kubectl get pods -n gmp-system
kubectl logs -n gmp-system -l app.kubernetes.io/name=collector -f
```
