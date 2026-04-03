# 🚀 GCP CI/CD & DevSecOps — Comprehensive Cheatsheet
## Cloud Build · Artifact Registry · Cloud Deploy · Secure Source Manager

> **Audience:** DevOps engineers, platform engineers, SREs, and developers building production CI/CD pipelines on GCP.
> **Last updated:** March 2026 | Covers Cloud Build, Artifact Registry, Cloud Deploy, and Secure Source Manager with full YAML, CLI, and code examples.

---

## Table of Contents

1. [Overview & Service Comparison](#1-overview--service-comparison)
2. [Cloud Build](#2-cloud-build)
3. [Artifact Registry](#3-artifact-registry)
4. [Cloud Deploy](#4-cloud-deploy)
5. [Secure Source Manager](#5-secure-source-manager)
6. [End-to-End CI/CD Pipeline: Complete Example](#6-end-to-end-cicd-pipeline-complete-example)
7. [Security & Compliance Across All Four Services](#7-security--compliance-across-all-four-services)
8. [Monitoring, Logging & Alerting](#8-monitoring-logging--alerting)
9. [gcloud CLI Quick Reference](#9-gcloud-cli-quick-reference)
10. [Pricing Summary](#10-pricing-summary)
11. [Quick Reference & Comparison Tables](#11-quick-reference--comparison-tables)

---

## 1. Overview & Service Comparison

### The GCP CI/CD Pipeline

```
🔐 Secure Source Manager          📦 Cloud Build              📦 Artifact Registry         🚀 Cloud Deploy
  (Source Code Hosting)    →        (Build & Test)      →      (Store Artifacts)      →     (Continuous Delivery)
  ├── Git repositories              ├── Build triggers           ├── Docker images            ├── Delivery pipelines
  ├── Branch protection             ├── Build steps              ├── npm/Maven/PyPI            ├── Releases & rollouts
  ├── PR workflows                  ├── Unit tests               ├── Helm charts               ├── Dev→Staging→Prod
  └── Audit logs                    └── Image builds             └── Vulnerability scans       └── Canary/Blue-Green
```

---

### Side-by-Side Service Comparison

| Service | Purpose | Key Resources | Pricing Unit | Primary GCP Integrations |
|---|---|---|---|---|
| **Cloud Build** | Managed CI — execute builds, tests, deploys | Builds, Triggers, Worker Pools | Per build-minute by machine | Cloud Run, GKE, AR, Secret Manager |
| **Artifact Registry** | Universal artifact store | Repositories, Images, Packages | Per GB-month + egress | Cloud Build, GKE, Cloud Run, Binary Auth |
| **Cloud Deploy** | Managed CD — promote releases across envs | Pipelines, Targets, Releases, Rollouts | Per successful deployment | GKE, Cloud Run, Cloud Build, Pub/Sub |
| **Secure Source Manager** | Managed Git hosting with compliance controls | Instances, Repositories, Branches | Per instance-hour + user-month | Cloud Build, Cloud Audit Logs, Secret Manager |

---

### Decision Guide: GCP Services vs. Alternatives

| Requirement | GCP Native | Alternative | Choose GCP When... |
|---|---|---|---|
| **Build automation** | Cloud Build | GitHub Actions, Jenkins | Deep GCP integration, no infra management |
| **Artifact storage** | Artifact Registry | Artifactory, Nexus | GCP-native auth, vulnerability scanning |
| **Continuous delivery** | Cloud Deploy | Argo CD, Spinnaker, Flux | GKE/Cloud Run targets, managed approvals |
| **Source hosting** | Secure Source Manager | GitHub Enterprise, GitLab | Compliance (FedRAMP, HIPAA), GCP-native IAM |

---

### Required APIs & Common IAM Roles

```bash
# Enable all four service APIs
gcloud services enable \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  clouddeploy.googleapis.com \
  securesourcemanager.googleapis.com \
  secretmanager.googleapis.com \
  container.googleapis.com \
  run.googleapis.com \
  --project=MY_PROJECT
```

| Service | Role | Grants |
|---|---|---|
| Cloud Build | `roles/cloudbuild.builds.builder` | Submit and manage builds |
| Cloud Build | `roles/cloudbuild.builds.viewer` | View build history |
| Artifact Registry | `roles/artifactregistry.admin` | Full repo management |
| Artifact Registry | `roles/artifactregistry.writer` | Push artifacts |
| Artifact Registry | `roles/artifactregistry.reader` | Pull artifacts |
| Cloud Deploy | `roles/clouddeploy.admin` | Manage pipelines, targets |
| Cloud Deploy | `roles/clouddeploy.releaser` | Create releases, promote |
| Cloud Deploy | `roles/clouddeploy.approver` | Approve/reject promotions |
| Cloud Deploy | `roles/clouddeploy.viewer` | Read pipelines, rollouts |
| Secure Source Manager | `roles/securesourcemanager.admin` | Manage instances |
| Secure Source Manager | `roles/securesourcemanager.repoAdmin` | Manage specific repos |
| Secure Source Manager | `roles/securesourcemanager.repoWriter` | Push code to repos |
| Secure Source Manager | `roles/securesourcemanager.repoReader` | Clone/read repos |

---

## 2. 🔄 Cloud Build

### What is Cloud Build?

Cloud Build is a **fully managed, serverless CI service** that executes builds as a series of Docker container steps on ephemeral VMs. Each build is isolated, runs in its own environment, and is billed per minute.

---

### Build Config Anatomy (`cloudbuild.yaml`)

```yaml
# cloudbuild.yaml — annotated reference
steps:
  - id: 'run-tests'                          # Unique step identifier
    name: 'python:3.11-slim'                 # Docker image to run this step in
    dir: 'src'                               # Working directory within workspace
    entrypoint: 'pip'                        # Override container entrypoint
    args: ['install', '-r', 'requirements.txt']
    env:                                     # Environment variables
      - 'ENV=test'
      - 'LOG_LEVEL=debug'
    secretEnv: ['DB_PASSWORD']               # Secrets from Secret Manager
    timeout: '300s'                          # Step timeout (default 600s)
    waitFor: ['-']                           # '-' = start immediately (parallel)

  - id: 'run-unit-tests'
    name: 'python:3.11-slim'
    args: ['python', '-m', 'pytest', 'tests/']
    waitFor: ['run-tests']                   # Wait for specific step by ID

  - id: 'build-image'
    name: 'gcr.io/cloud-builders/docker'
    args:
      - build
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '.'
    waitFor: ['run-unit-tests']

substitutions:
  _MY_ENV: 'staging'                         # Custom substitution (prefix _)
  _IMAGE_TAG: 'latest'

options:
  machineType: 'E2_HIGHCPU_8'               # Machine type for build workers
  diskSizeGb: 100                            # Disk size for workspace
  logging: CLOUD_LOGGING_ONLY               # CLOUD_LOGGING_ONLY | GCS_ONLY | LEGACY
  env:                                       # Global env for all steps
    - 'DOCKER_BUILDKIT=1'

timeout: '1200s'                             # Overall build timeout (default 60 min)

artifacts:                                   # Upload non-image artifacts to GCS
  objects:
    location: 'gs://my-bucket/artifacts/$BUILD_ID'
    paths: ['*.jar', 'dist/**']

availableSecrets:                            # Secret Manager integration
  secretManager:
    - versionName: 'projects/$PROJECT_ID/secrets/db-password/versions/latest'
      env: 'DB_PASSWORD'

images:                                      # Images to push at end of build
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest'
```

---

### Built-in Substitution Variables

| Variable | Description | Example Value |
|---|---|---|
| `$PROJECT_ID` | GCP project ID | `my-project-123` |
| `$BUILD_ID` | Unique build UUID | `abc123-...` |
| `$SHORT_SHA` | First 7 chars of commit SHA | `a1b2c3d` |
| `$COMMIT_SHA` | Full commit SHA | `a1b2c3d4e5f6...` |
| `$REPO_NAME` | Repository name | `my-app` |
| `$BRANCH_NAME` | Git branch name | `main` |
| `$TAG_NAME` | Git tag (if trigger is tag push) | `v1.2.3` |
| `$TRIGGER_NAME` | Name of the build trigger | `push-to-main` |
| `$_SUBSTITUTION` | Custom user-defined substitution | `staging` |

---

### Complete cloudbuild.yaml Examples

```yaml
# ── Example 1: Docker Build → Push to Artifact Registry ──────────────
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - build
      - '--cache-from'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags',
           'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app']

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest'

options:
  logging: CLOUD_LOGGING_ONLY
```

```yaml
# ── Example 2: Full Pipeline (lint → test → build → push → deploy) ───
steps:
  # Step 1: Install dependencies
  - id: 'install'
    name: 'python:3.11-slim'
    entrypoint: 'pip'
    args: ['install', '-r', 'requirements.txt', '--user']

  # Step 2: Lint (parallel with tests)
  - id: 'lint'
    name: 'python:3.11-slim'
    args: ['python', '-m', 'flake8', 'src/']
    waitFor: ['install']

  # Step 3: Unit tests (parallel with lint)
  - id: 'test'
    name: 'python:3.11-slim'
    args: ['python', '-m', 'pytest', 'tests/', '--tb=short', '-q']
    waitFor: ['install']

  # Step 4: Build Docker image (after lint AND tests pass)
  - id: 'build'
    name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t',
           'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA', '.']
    waitFor: ['lint', 'test']

  # Step 5: Push image
  - id: 'push'
    name: 'gcr.io/cloud-builders/docker'
    args: ['push',
           'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA']
    waitFor: ['build']

  # Step 6: Create Cloud Deploy release
  - id: 'deploy'
    name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - deploy
      - releases
      - create
      - 'release-$SHORT_SHA'
      - '--delivery-pipeline=my-app-pipeline'
      - '--region=us-central1'
      - '--images=my-app=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
    waitFor: ['push']

options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
```

```yaml
# ── Example 3: Parallel Steps with waitFor ────────────────────────────
steps:
  - id: 'checkout-deps'
    name: 'node:20-alpine'
    entrypoint: 'npm'
    args: ['ci']

  # These two run in parallel after checkout-deps
  - id: 'lint-js'
    name: 'node:20-alpine'
    args: ['npm', 'run', 'lint']
    waitFor: ['checkout-deps']

  - id: 'type-check'
    name: 'node:20-alpine'
    args: ['npm', 'run', 'type-check']
    waitFor: ['checkout-deps']

  - id: 'unit-tests'
    name: 'node:20-alpine'
    args: ['npm', 'test', '--', '--runInBand']
    waitFor: ['checkout-deps']

  # Only runs after ALL three parallel steps complete
  - id: 'build-app'
    name: 'node:20-alpine'
    args: ['npm', 'run', 'build']
    waitFor: ['lint-js', 'type-check', 'unit-tests']
```

```yaml
# ── Example 4: Using Secret Manager Secrets ───────────────────────────
steps:
  - id: 'integration-test'
    name: 'python:3.11-slim'
    entrypoint: 'python'
    args: ['tests/integration/run_tests.py']
    secretEnv:
      - 'DB_CONNECTION_STRING'
      - 'API_KEY'

availableSecrets:
  secretManager:
    - versionName: 'projects/$PROJECT_ID/secrets/db-connection/versions/latest'
      env: 'DB_CONNECTION_STRING'
    - versionName: 'projects/$PROJECT_ID/secrets/api-key/versions/latest'
      env: 'API_KEY'
```

```yaml
# ── Example 5: Custom Substitutions ──────────────────────────────────
substitutions:
  _DEPLOY_ENV: 'dev'                         # Override at trigger or runtime
  _IMAGE_TAG: '$SHORT_SHA'
  _REGION: 'us-central1'

steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t',
           '${_REGION}-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:${_IMAGE_TAG}', '.']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args: ['run', 'deploy', 'my-app',
           '--image=${_REGION}-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:${_IMAGE_TAG}',
           '--region=${_REGION}',
           '--set-env-vars=ENV=${_DEPLOY_ENV}']
```

---

### Build Triggers

```bash
# Create trigger for push to main branch
gcloud builds triggers create github \
  --name="push-to-main" \
  --repo-owner="my-org" \
  --repo-name="my-app" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --substitutions="_DEPLOY_ENV=prod" \
  --region=us-central1

# Create trigger for push to any branch (PR trigger)
gcloud builds triggers create github \
  --name="pr-validation" \
  --repo-owner="my-org" \
  --repo-name="my-app" \
  --pull-request-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --comment-control="COMMENTS_ENABLED_FOR_EXTERNAL_CONTRIBUTORS_ONLY"

# Create trigger from SSM repository
gcloud builds triggers create cloud-source-repositories \
  --name="ssm-push-main" \
  --repo="projects/MY_PROJECT/locations/us-central1/repositories/my-app" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml"

# Run a trigger manually
gcloud builds triggers run push-to-main \
  --region=us-central1 \
  --branch=main

# List triggers
gcloud builds triggers list --region=us-central1

# gcloud builds submit (manual build submission)
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --substitutions="_DEPLOY_ENV=dev,_IMAGE_TAG=test" \
  --region=us-central1 \
  --machine-type=E2_HIGHCPU_8
```

---

### Trigger Configuration YAML

```yaml
# trigger.yaml — declarative trigger definition
name: push-to-main
filename: cloudbuild.yaml
github:
  owner: my-org
  name: my-app
  push:
    branch: ^main$
substitutions:
  _DEPLOY_ENV: prod
  _REGION: us-central1
includedFiles:
  - 'src/**'
  - 'cloudbuild.yaml'
  - 'Dockerfile'
ignoredFiles:
  - '**.md'
  - 'docs/**'
serviceAccount: 'projects/MY_PROJECT/serviceAccounts/cloudbuild-sa@MY_PROJECT.iam.gserviceaccount.com'
```

---

### Python: Trigger a Build Programmatically

```python
from google.cloud.devtools import cloudbuild_v1
from google.protobuf import duration_pb2

client = cloudbuild_v1.CloudBuildClient()

# Trigger a build programmatically
build = cloudbuild_v1.Build(
    steps = [
        cloudbuild_v1.BuildStep(
            name = "gcr.io/cloud-builders/docker",
            args = ["build", "-t",
                    "us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest", "."],
        ),
        cloudbuild_v1.BuildStep(
            name = "gcr.io/cloud-builders/docker",
            args = ["push",
                    "us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest"],
        ),
    ],
    images         = ["us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest"],
    timeout        = duration_pb2.Duration(seconds=600),
    substitutions  = {"_DEPLOY_ENV": "staging"},
)

operation = client.create_build(project_id="my-project", build=build)
result = operation.result()
print(f"Build ID:    {result.id}")
print(f"Build state: {result.status.name}")
print(f"Logs URL:    {result.log_url}")
```

---

### Private Worker Pools

```bash
# Create a private pool (isolated workers with VPC access)
gcloud builds worker-pools create my-private-pool \
  --region=us-central1 \
  --peered-network=projects/MY_PROJECT/global/networks/my-vpc \
  --worker-machine-type=e2-standard-4 \
  --worker-disk-size=100

# Use private pool in build
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --worker-pool=projects/MY_PROJECT/locations/us-central1/workerPools/my-private-pool
```

> 💡 **When to use private pools:** Use when builds need access to private GCP resources (Cloud SQL, internal APIs, private Artifact Registry), when you need larger or custom machine types, or when network isolation is required for compliance.

---

## 3. 📦 Artifact Registry

### Repository Formats & Modes

| Format | Description | Auth Method | Use Case |
|---|---|---|---|
| **Docker** | Container images | `gcloud auth configure-docker` | App images, build caches |
| **Maven** | Java artifacts (JARs, WARs) | `settings.xml` with token | Java/Kotlin projects |
| **npm** | Node.js packages | `.npmrc` with token | Node.js projects |
| **Python (pip)** | Python packages (wheels) | `pip.conf` / `--index-url` | Python libraries |
| **Go** | Go modules | GOPROXY env var | Go projects |
| **Apt** | Debian packages | `apt-transport-artifact-registry` | OS packages |
| **Yum/RPM** | RPM packages | `yum.repos.d` config | RHEL/CentOS packages |
| **Helm** | Kubernetes Helm charts | `helm registry login` | K8s app packaging |
| **Generic** | Any file type | `gcloud artifacts generic upload` | Binaries, assets |

---

### Repository Modes

| Mode | Description | Use Case |
|---|---|---|
| **Standard** | Push/pull your own artifacts | Primary artifact storage |
| **Remote** | Proxy and cache external registries | Reduce external dependencies, air-gap support |
| **Virtual** | Aggregate multiple repos with priority | Unified pull endpoint across team repos |

---

### gcloud: Artifact Registry Setup

```bash
# ── Create repositories ────────────────────────────────────────────────
# Standard Docker repository
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production Docker images" \
  --immutable-tags \
  --labels=env=prod,team=platform

# Remote repository (proxies Docker Hub)
gcloud artifacts repositories create docker-hub-proxy \
  --repository-format=docker \
  --location=us-central1 \
  --mode=REMOTE_REPOSITORY \
  --remote-repo-config-desc="Docker Hub proxy" \
  --remote-docker-repo=DOCKER_HUB

# Remote repository (proxies PyPI)
gcloud artifacts repositories create pypi-proxy \
  --repository-format=python \
  --location=us-central1 \
  --mode=REMOTE_REPOSITORY \
  --remote-python-repo=PYPI

# Virtual repository (aggregate)
gcloud artifacts repositories create my-virtual-repo \
  --repository-format=docker \
  --location=us-central1 \
  --mode=VIRTUAL_REPOSITORY \
  --upstream-policy-file=upstream_policy.json

# ── Authentication ────────────────────────────────────────────────────
# Configure Docker credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# ── Docker push/pull ─────────────────────────────────────────────────
docker build -t us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app:v1.0.0 .
docker push us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app:v1.0.0
docker pull us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app:v1.0.0

# ── List artifacts ────────────────────────────────────────────────────
# List repositories
gcloud artifacts repositories list --location=us-central1

# List Docker images in a repo
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo \
  --include-tags \
  --format="table(IMAGE,TAGS,CREATE_TIME,UPDATE_TIME)"

# List all packages (any format)
gcloud artifacts packages list \
  --repository=my-docker-repo \
  --location=us-central1

# List versions of a package
gcloud artifacts versions list \
  --package=my-app \
  --repository=my-docker-repo \
  --location=us-central1

# List tags
gcloud artifacts tags list \
  --package=my-app \
  --version=sha256:abc123 \
  --repository=my-docker-repo \
  --location=us-central1

# ── Delete operations ─────────────────────────────────────────────────
# Delete a specific tag
gcloud artifacts tags delete v1.0.0 \
  --package=my-app \
  --version=sha256:abc123 \
  --repository=my-docker-repo \
  --location=us-central1

# Delete all untagged images (be careful!)
gcloud artifacts docker images delete \
  us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app \
  --delete-tags --quiet

# ── IAM per-repository ────────────────────────────────────────────────
# Grant GKE node SA read access to pull images
gcloud artifacts repositories add-iam-policy-binding my-docker-repo \
  --location=us-central1 \
  --member="serviceAccount:gke-node-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# Grant Cloud Build SA write access to push images
gcloud artifacts repositories add-iam-policy-binding my-docker-repo \
  --location=us-central1 \
  --member="serviceAccount:cloudbuild-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

---

### Naming Convention

```
LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG

Examples:
  us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.2.3
  us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest
  europe-west1-docker.pkg.dev/my-project/prod-repo/api-server:$SHORT_SHA
  asia-northeast1-docker.pkg.dev/my-project/services/worker:${_BUILD_ENV}-${SHORT_SHA}
```

---

### Package Repository Setup (npm, Maven, pip)

```bash
# ── npm ──────────────────────────────────────────────────────────────
# Get auth token
gcloud auth print-access-token

# .npmrc configuration
cat > .npmrc << 'EOF'
@my-org:registry=https://us-central1-npm.pkg.dev/MY_PROJECT/my-npm-repo/
//us-central1-npm.pkg.dev/MY_PROJECT/my-npm-repo/:always-auth=true
//us-central1-npm.pkg.dev/MY_PROJECT/my-npm-repo/:_authToken=${NPM_TOKEN}
EOF

# Publish package
npm publish --registry=https://us-central1-npm.pkg.dev/MY_PROJECT/my-npm-repo/

# ── Maven ─────────────────────────────────────────────────────────────
# settings.xml server configuration
# <server>
#   <id>artifact-registry</id>
#   <configuration>
#     <httpConfiguration>
#       <get><usePreemptive>true</usePreemptive></get>
#     </httpConfiguration>
#   </configuration>
# </server>

# ── pip ──────────────────────────────────────────────────────────────
# Install from Artifact Registry
pip install my-package \
  --index-url "https://us-central1-python.pkg.dev/MY_PROJECT/my-python-repo/simple/"

# Upload a package
gcloud artifacts packages import \
  --location=us-central1 \
  --repository=my-python-repo \
  --source=dist/my_package-1.0.0.tar.gz

# ── Helm ─────────────────────────────────────────────────────────────
gcloud auth print-access-token | helm registry login \
  -u oauth2accesstoken \
  --password-stdin \
  us-central1-docker.pkg.dev

# Push Helm chart
helm push my-chart-1.0.0.tgz oci://us-central1-docker.pkg.dev/MY_PROJECT/my-helm-repo

# Pull Helm chart
helm pull oci://us-central1-docker.pkg.dev/MY_PROJECT/my-helm-repo/my-chart --version 1.0.0
```

---

### Cleanup Policies

```json
// cleanup-policy.json — keep 5 most recent + delete old untagged
[
  {
    "name": "keep-recent-versions",
    "action": {"type": "Keep"},
    "mostRecentVersions": {
      "keepCount": 5
    }
  },
  {
    "name": "delete-old-untagged",
    "action": {"type": "Delete"},
    "condition": {
      "olderThan":     "604800s",
      "tagState":      "UNTAGGED"
    }
  },
  {
    "name": "delete-old-dev-images",
    "action": {"type": "Delete"},
    "condition": {
      "olderThan":     "2592000s",
      "tagPrefixes":   ["dev-", "test-"],
      "packageNamePrefixes": ["my-app"]
    }
  }
]
```

```bash
# Apply cleanup policy
gcloud artifacts repositories set-cleanup-policies my-docker-repo \
  --location=us-central1 \
  --policy=cleanup-policy.json \
  --no-dry-run
```

---

### Vulnerability Scanning

```bash
# Enable automatic on-push scanning (enabled by default for Docker repos)
gcloud artifacts repositories update my-docker-repo \
  --location=us-central1 \
  --enable-vulnerability-scanning

# List vulnerabilities for an image
gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app:v1.0.0 \
  --show-package-vulnerability

# Get detailed CVE info
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/MY_PROJECT/my-docker-repo/my-app \
  --format="table(vulnerability.vulnerability,vulnerability.severity,vulnerability.packageName)"
```

> ⚠️ **Vulnerability scanning** requires the Container Scanning API (`containerscanning.googleapis.com`) to be enabled. Scans run automatically on push for Docker repositories with scanning enabled.

---

### Python: Artifact Registry Client

```python
from google.cloud import artifactregistry_v1

client = artifactregistry_v1.ArtifactRegistryClient()

# List repositories
parent = "projects/my-project/locations/us-central1"
repos  = client.list_repositories(parent=parent)
for repo in repos:
    print(f"Repo: {repo.name:60s} Format: {repo.format_.name}")

# List packages in a repo
repo_name = f"{parent}/repositories/my-docker-repo"
packages  = client.list_packages(parent=repo_name)
for pkg in packages:
    print(f"Package: {pkg.name}, Display: {pkg.display_name}")

# List versions of a package
pkg_name  = f"{repo_name}/packages/my-app"
versions  = client.list_versions(parent=pkg_name)
for v in versions:
    print(f"Version: {v.name}, Tags: {[t.name for t in v.tags]}")
```

---

## 4. 🚀 Cloud Deploy

### Core Concepts

```
Delivery Pipeline
  ├── Stage 1: dev     → Target: dev-cluster   (auto-promote)
  ├── Stage 2: staging → Target: staging-cluster (auto-promote on success)
  └── Stage 3: prod    → Target: prod-cluster   (requires approval)

Release: snapshot of manifests + images at $COMMIT_SHA
  └── Rollout: execution of deploying release to a specific target
         ├── Status: PENDING_APPROVAL → IN_PROGRESS → SUCCEEDED / FAILED
         └── Strategy: STANDARD | CANARY | BLUE_GREEN
```

---

### Delivery Pipeline YAML

```yaml
# clouddeploy.yaml — delivery pipeline + targets in one file

# ── Delivery Pipeline ─────────────────────────────────────────────────
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
  annotations:
    description: "my-app delivery pipeline"
  labels:
    app: my-app
    team: platform
spec:
  description: "Deploy my-app through dev → staging → prod"
  serialPipeline:
    stages:
      - targetId: dev
        profiles: [dev]            # Skaffold profile to use for this stage
        strategy:
          standard:
            verify: true           # Run post-deploy verification job

      - targetId: staging
        profiles: [staging]
        strategy:
          canary:
            canaryDeployment:
              percentages: [25, 50, 75]  # Traffic split percentages
              verify: true

      - targetId: prod
        profiles: [prod]
        strategy:
          blueGreen:
            serviceNetworking:
              service: my-app-service
              deployment: my-app
            verify: true

---
# ── GKE Target ────────────────────────────────────────────────────────
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
spec:
  description: "Development GKE cluster"
  deployParameters:
    - values:
        REPLICAS: "1"
        MEMORY_LIMIT: "256Mi"
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/dev-cluster
  requireApproval: false

---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
spec:
  description: "Staging GKE cluster"
  deployParameters:
    - values:
        REPLICAS: "2"
        MEMORY_LIMIT: "512Mi"
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/staging-cluster
  requireApproval: false

---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
spec:
  description: "Production GKE cluster — requires approval"
  deployParameters:
    - values:
        REPLICAS: "5"
        MEMORY_LIMIT: "1Gi"
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/prod-cluster
  requireApproval: true                # Human approval required before deploy

---
# ── Cloud Run Target ──────────────────────────────────────────────────
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev-run
spec:
  cloudRun:
    location: projects/MY_PROJECT/locations/us-central1
  requireApproval: false
```

---

### Canary Deployment YAML

```yaml
# canary-pipeline.yaml — 25% initial canary
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: canary-pipeline
spec:
  serialPipeline:
    stages:
      - targetId: prod
        profiles: [prod]
        strategy:
          canary:
            runtimeConfig:
              kubernetes:
                serviceNetworking:
                  service: my-app-svc
                  deployment: my-app
            canaryDeployment:
              percentages: [25]          # Start at 25%, then manual advance to 100%
              verify: true               # Run verify job at each step
```

---

### Blue/Green Deployment YAML

```yaml
# blue-green-pipeline.yaml
spec:
  serialPipeline:
    stages:
      - targetId: prod
        profiles: [prod]
        strategy:
          blueGreen:
            serviceNetworking:
              service: my-app-svc
              deployment: my-app
            autoUpdateReferences: true
            podOverlayDisabled: false
            verify: true
```

---

### Automation Rules

```yaml
# automation.yaml — auto-promote dev → staging on success
apiVersion: deploy.cloud.google.com/v1
kind: Automation
metadata:
  name: auto-promote-dev-to-staging
spec:
  serviceAccount: 'deploy-sa@MY_PROJECT.iam.gserviceaccount.com'
  selector:
    - target:
        id: dev
  suspended: false
  rules:
    - promoteReleaseRule:
        id: promote-after-success
        wait: 0s                         # Promote immediately after dev succeeds
        destinationTargetId: staging
        destinationPhase: stablePhase
```

---

### Deploy Hooks (Pre/Post Deploy Jobs)

```yaml
# skaffold.yaml with verify hooks
apiVersion: skaffold/v4beta7
kind: Config
deploy:
  kubectl:
    manifests:
      - k8s/base/**
profiles:
  - name: prod
    patches:
      - op: add
        path: /deploy/kubectl/hooks
        value:
          after:
            - name: integration-tests
              container:
                name: tests
                image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/test-runner:latest
                command: ["./run_integration_tests.sh"]
```

---

### gcloud: Cloud Deploy Operations

```bash
# ── Setup ─────────────────────────────────────────────────────────────
# Apply pipeline and targets (YAML files)
gcloud deploy apply \
  --file=clouddeploy.yaml \
  --region=us-central1 \
  --project=MY_PROJECT

# ── Releases ──────────────────────────────────────────────────────────
# Create a release (the snapshot to deploy)
gcloud deploy releases create release-$(date +%Y%m%d-%H%M%S) \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --images=my-app=us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app:$COMMIT_SHA

# Create release from tag
gcloud deploy releases create release-v1-2-3 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --images=my-app=us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app:v1.2.3 \
  --deploy-parameters=ENV=prod,REGION=us-central1

# List releases
gcloud deploy releases list \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# ── Rollouts ──────────────────────────────────────────────────────────
# List rollouts for a release
gcloud deploy rollouts list \
  --release=release-20260316-120000 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Describe a rollout (see status, phase, logs)
gcloud deploy rollouts describe ROLLOUT_ID \
  --release=RELEASE_ID \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Approve a rollout (for targets requiring approval)
gcloud deploy rollouts approve ROLLOUT_ID \
  --release=RELEASE_ID \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Reject a rollout
gcloud deploy rollouts reject ROLLOUT_ID \
  --release=RELEASE_ID \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --reason="Failed staging load test"

# Advance canary to next phase
gcloud deploy rollouts advance ROLLOUT_ID \
  --release=RELEASE_ID \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# ── Promotion ─────────────────────────────────────────────────────────
# Manually promote a release to the next stage
gcloud deploy releases promote \
  --release=RELEASE_ID \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --to-target=staging

# ── Rollback ──────────────────────────────────────────────────────────
# Roll back a target to the previous release
gcloud deploy rollbacks create \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --target-id=prod \
  --release=PREVIOUS_RELEASE_ID

# ── Targets ───────────────────────────────────────────────────────────
gcloud deploy targets list --region=us-central1
gcloud deploy targets describe prod --region=us-central1

# ── Delivery pipelines ────────────────────────────────────────────────
gcloud deploy delivery-pipelines list --region=us-central1
gcloud deploy delivery-pipelines describe my-app-pipeline --region=us-central1
```

---

### Python: Cloud Deploy Operations

```python
from google.cloud import deploy_v1
import time

client = deploy_v1.CloudDeployClient()
REGION  = "us-central1"
PROJECT = "my-project"
parent  = f"projects/{PROJECT}/locations/{REGION}"

# ── Create a release ──────────────────────────────────────────────────
release = deploy_v1.Release(
    skaffold_version = "skaffold/v2.8",
    build_artifacts  = [
        deploy_v1.BuildArtifact(
            image = "my-app",
            tag   = f"us-central1-docker.pkg.dev/{PROJECT}/my-repo/my-app:abc1234",
        )
    ],
)

operation = client.create_release(
    parent      = f"{parent}/deliveryPipelines/my-app-pipeline",
    release_id  = f"release-{int(time.time())}",
    release     = release,
)
created_release = operation.result(timeout=120)
print(f"Release: {created_release.name}")

# ── Get rollout status ────────────────────────────────────────────────
rollouts = client.list_rollouts(
    parent = f"{created_release.name}/targets/dev/rollouts"
    # In practice: parent = created_release.name
)
for rollout in rollouts:
    print(f"Rollout: {rollout.name}")
    print(f"  State:  {deploy_v1.Rollout.State(rollout.state).name}")
    print(f"  Phase:  {rollout.phase_id}")

# ── Approve a promotion ───────────────────────────────────────────────
client.approve_rollout(
    name    = f"{parent}/deliveryPipelines/my-app-pipeline/releases/RELEASE_ID/rollouts/ROLLOUT_ID",
    approved = True,
)
```

---

## 5. 🔐 Secure Source Manager

### What is Secure Source Manager?

Secure Source Manager (SSM) is Google Cloud's **fully managed, enterprise-grade Git hosting service** with built-in compliance controls, GCP-native IAM, CMEK encryption, and complete Cloud Audit Log coverage. It replaces Cloud Source Repositories for compliance-sensitive environments.

---

### SSM vs. Alternatives

| Feature | Secure Source Manager | Cloud Source Repos (legacy) | GitHub Enterprise | GitLab Self-Managed |
|---|---|---|---|---|
| **Managed by Google** | ✅ Fully | ✅ Fully | ❌ | ❌ |
| **GCP-native IAM** | ✅ Fine-grained | ✅ | ❌ | ❌ |
| **CMEK encryption** | ✅ | ❌ | ❌ | ❌ (EE) |
| **Private VPC instance** | ✅ | ❌ | ✅ (self-host) | ✅ (self-host) |
| **FedRAMP / HIPAA** | ✅ | ❌ | ❌ (GovCloud) | ❌ |
| **Cloud Audit Logs** | ✅ All ops | ✅ | ❌ | ❌ |
| **Branch protection** | ✅ | ❌ | ✅ | ✅ |
| **PR/code review** | ✅ | ❌ | ✅ | ✅ |
| **CI integration** | Cloud Build native | Cloud Build native | GitHub Actions | GitLab CI |
| **Best for** | Regulated industries, GCP-native teams | Simple GCP teams | Existing GitHub orgs | Existing GitLab orgs |

---

### gcloud: Secure Source Manager

```bash
# ── Instance Management ───────────────────────────────────────────────
# Create a standard instance
gcloud secure-source-manager instances create my-ssm-instance \
  --location=us-central1 \
  --description="Production source code hosting"

# Create a private instance with CMEK
gcloud secure-source-manager instances create my-private-ssm \
  --location=us-central1 \
  --is-private \
  --private-config-ca-pool=projects/MY_PROJECT/locations/us-central1/caPools/my-ca-pool \
  --kms-key=projects/MY_PROJECT/locations/us-central1/keyRings/ssm-ring/cryptoKeys/ssm-key

# List instances
gcloud secure-source-manager instances list --location=us-central1

# Describe instance (get API endpoint URL)
gcloud secure-source-manager instances describe my-ssm-instance \
  --location=us-central1 \
  --format="json(name,state,hostConfig)"

# ── Repository Management ─────────────────────────────────────────────
# Create a repository
gcloud secure-source-manager repositories create my-app \
  --instance=my-ssm-instance \
  --location=us-central1 \
  --description="Main application repository" \
  --default-branch=main

# List repositories
gcloud secure-source-manager repositories list \
  --instance=my-ssm-instance \
  --location=us-central1 \
  --format="table(name,description,createTime)"

# Describe a repository (get clone URLs)
gcloud secure-source-manager repositories describe my-app \
  --instance=my-ssm-instance \
  --location=us-central1

# Delete a repository
gcloud secure-source-manager repositories delete my-app \
  --instance=my-ssm-instance \
  --location=us-central1 --quiet

# ── IAM Bindings ──────────────────────────────────────────────────────
# Grant write access to a developer
gcloud secure-source-manager repositories add-iam-policy-binding my-app \
  --instance=my-ssm-instance \
  --location=us-central1 \
  --member="user:developer@company.com" \
  --role="roles/securesourcemanager.repoWriter"

# Grant read access to CI service account
gcloud secure-source-manager repositories add-iam-policy-binding my-app \
  --instance=my-ssm-instance \
  --location=us-central1 \
  --member="serviceAccount:cloudbuild-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/securesourcemanager.repoReader"
```

---

### Clone & Push Code

```bash
# Get the repository URL from describe output
SSM_INSTANCE_URL=$(gcloud secure-source-manager instances describe my-ssm-instance \
  --location=us-central1 --format="value(hostConfig.html)")

# Clone via HTTPS (uses gcloud auth token)
gcloud secure-source-manager repositories clone my-app \
  --instance=my-ssm-instance \
  --location=us-central1

# Or clone using git directly with auth helper
git -c credential.helper='gcloud auth print-access-token' \
  clone https://${SSM_INSTANCE_URL}/v1/projects/MY_PROJECT/locations/us-central1/repositories/my-app

# Push code
cd my-app
git add .
git commit -m "Initial commit"
git push origin main
```

---

### Integrate with Cloud Build

```bash
# Create Cloud Build trigger on SSM repository push
gcloud builds triggers create cloud-source-repositories \
  --name="ssm-push-to-main" \
  --repo="projects/MY_PROJECT/locations/us-central1/repositories/my-app" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --region=us-central1

# Note: SSM triggers use cloud-source-repositories trigger type
# with the SSM repository resource path
```

---

### Python: Secure Source Manager Client

```python
from google.cloud import securesourcemanager_v1

client = securesourcemanager_v1.SecureSourceManagerClient()

PROJECT  = "my-project"
LOCATION = "us-central1"
INSTANCE = "my-ssm-instance"

# List instances
instances = client.list_instances(
    parent = f"projects/{PROJECT}/locations/{LOCATION}"
)
for inst in instances:
    print(f"Instance: {inst.name}, State: {inst.state.name}")

# List repositories
repos = client.list_repositories(
    parent = f"projects/{PROJECT}/locations/{LOCATION}/instances/{INSTANCE}"
)
for repo in repos:
    print(f"Repo: {repo.name}")
    print(f"  Branch: {repo.initial_config.default_branch}")
    print(f"  Created: {repo.create_time}")

# Get a specific repository
repo = client.get_repository(
    name = f"projects/{PROJECT}/locations/{LOCATION}/instances/{INSTANCE}/repositories/my-app"
)
print(f"Clone HTTPS: {repo.uris.html}")
print(f"Clone git:   {repo.uris.git_https}")

# Create a new repository
operation = client.create_repository(
    parent        = f"projects/{PROJECT}/locations/{LOCATION}/instances/{INSTANCE}",
    repository_id = "new-service",
    repository    = securesourcemanager_v1.Repository(
        description = "New microservice repository",
        initial_config = securesourcemanager_v1.Repository.InitialConfig(
            default_branch  = "main",
            gitignores      = ["Python"],
            license         = "Apache-2.0",
            readme          = "DEFAULT",
        )
    )
)
new_repo = operation.result()
print(f"Created: {new_repo.name}")
```

---

## 6. 🔄 End-to-End CI/CD Pipeline: Complete Example

### Architecture

```
Developer pushes to SSM
         │ Cloud Build trigger fires
         ▼
Cloud Build Pipeline (cloudbuild.yaml):
  1. Install dependencies
  2. Run linting
  3. Run unit tests
  4. Build Docker image
  5. Push to Artifact Registry
  6. Run vulnerability scan
  7. Sign image (Binary Authorization)
  8. Create Cloud Deploy release
         │
         ▼
Cloud Deploy Delivery Pipeline:
  Stage 1: dev     (auto-promote)
  Stage 2: staging (auto-promote after verify)
  Stage 3: prod    (requires approval + canary)
         │
         ▼
GKE Cluster / Cloud Run Service
```

---

### Step 1: Service Account Setup

```bash
# ── Create dedicated service accounts ─────────────────────────────────
# Cloud Build SA
gcloud iam service-accounts create cloudbuild-sa \
  --display-name="Cloud Build Service Account"

# Cloud Deploy SA
gcloud iam service-accounts create clouddeploy-sa \
  --display-name="Cloud Deploy Service Account"

# GKE node SA
gcloud iam service-accounts create gke-node-sa \
  --display-name="GKE Node Service Account"

PROJ="MY_PROJECT"
CB_SA="cloudbuild-sa@${PROJ}.iam.gserviceaccount.com"
CD_SA="clouddeploy-sa@${PROJ}.iam.gserviceaccount.com"
GKE_SA="gke-node-sa@${PROJ}.iam.gserviceaccount.com"

# Cloud Build SA permissions
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CB_SA}" --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CB_SA}" --role="roles/clouddeploy.releaser"
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CB_SA}" --role="roles/secretmanager.secretAccessor"
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CB_SA}" --role="roles/logging.logWriter"

# Cloud Deploy SA permissions
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CD_SA}" --role="roles/container.developer"
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CD_SA}" --role="roles/run.developer"
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${CD_SA}" --role="roles/storage.objectCreator"
gcloud iam service-accounts add-iam-policy-binding $CD_SA \
  --member="serviceAccount:service-${PROJ_NUM}@gcp-sa-clouddeploy.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# GKE node SA permissions (pull images from AR)
gcloud projects add-iam-policy-binding $PROJ \
  --member="serviceAccount:${GKE_SA}" --role="roles/artifactregistry.reader"
```

---

### Step 2: Artifact Registry Repository

```bash
gcloud artifacts repositories create my-app-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="my-app Docker images"

gcloud auth configure-docker us-central1-docker.pkg.dev
```

---

### Step 3: Complete cloudbuild.yaml

```yaml
# cloudbuild.yaml — full CI pipeline
substitutions:
  _REGION:   'us-central1'
  _REPO:     'my-app-repo'
  _APP_NAME: 'my-app'
  _PIPELINE: 'my-app-pipeline'
  _IMAGE:    '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_APP_NAME}'

steps:
  # Parallel: lint and security scan
  - id: 'lint'
    name: 'python:3.11-slim'
    entrypoint: 'sh'
    args: ['-c', 'pip install flake8 -q && flake8 src/']
    waitFor: ['-']

  - id: 'sast-scan'
    name: 'aquasec/trivy:latest'
    args: ['fs', '--exit-code', '0', '--severity', 'HIGH,CRITICAL', '.']
    waitFor: ['-']

  # Unit tests
  - id: 'unit-tests'
    name: 'python:3.11-slim'
    entrypoint: 'sh'
    args: ['-c', 'pip install -r requirements.txt -q && python -m pytest tests/unit/ -q']
    waitFor: ['lint']

  # Build Docker image with cache
  - id: 'build-image'
    name: 'gcr.io/cloud-builders/docker'
    args:
      - build
      - '--cache-from'
      - '${_IMAGE}:latest'
      - '--build-arg'
      - 'BUILD_DATE=$_DATE'
      - '-t'
      - '${_IMAGE}:$SHORT_SHA'
      - '-t'
      - '${_IMAGE}:latest'
      - '.'
    waitFor: ['unit-tests', 'sast-scan']

  # Push image
  - id: 'push-image'
    name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', '${_IMAGE}']
    waitFor: ['build-image']

  # Scan pushed image for vulnerabilities
  - id: 'vuln-scan'
    name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        gcloud artifacts docker images describe ${_IMAGE}:$SHORT_SHA \
          --show-package-vulnerability \
          --format="json(image_summary.vulnerability_summary)" \
          --quiet
    waitFor: ['push-image']

  # Create Cloud Deploy release (triggers deployment to dev)
  - id: 'create-release'
    name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - deploy
      - releases
      - create
      - 'release-$SHORT_SHA'
      - '--delivery-pipeline=${_PIPELINE}'
      - '--region=${_REGION}'
      - '--images=${_APP_NAME}=${_IMAGE}:$SHORT_SHA'
      - '--deploy-parameters=GIT_SHA=$SHORT_SHA,BUILD_ID=$BUILD_ID'
    waitFor: ['vuln-scan']

serviceAccount: 'projects/$PROJECT_ID/serviceAccounts/cloudbuild-sa@$PROJECT_ID.iam.gserviceaccount.com'
options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
  dynamicSubstitutions: true
```

---

### Step 4: Complete skaffold.yaml

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app

build:
  artifacts:
    - image: my-app
      docker:
        dockerfile: Dockerfile
  tagPolicy:
    gitCommit: {}

manifests:
  rawYaml:
    - k8s/base/**

deploy:
  kubectl:
    defaultNamespace: my-app

profiles:
  - name: dev
    patches:
      - op: replace
        path: /manifests/rawYaml
        value:
          - k8s/overlays/dev/**
    deploy:
      kubectl:
        defaultNamespace: my-app-dev

  - name: staging
    patches:
      - op: replace
        path: /manifests/rawYaml
        value:
          - k8s/overlays/staging/**
    deploy:
      kubectl:
        defaultNamespace: my-app-staging

  - name: prod
    patches:
      - op: replace
        path: /manifests/rawYaml
        value:
          - k8s/overlays/prod/**
    deploy:
      kubectl:
        defaultNamespace: my-app-prod
    verify:
      - name: smoke-test
        container:
          name: smoke-test
          image: us-central1-docker.pkg.dev/MY_PROJECT/my-app-repo/test-runner:latest
          command:
            - ./smoke_test.sh
```

---

### Step 5: Complete clouddeploy.yaml

```yaml
# clouddeploy.yaml — pipeline + targets

apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
spec:
  serialPipeline:
    stages:
      - targetId: dev
        profiles: [dev]
        strategy:
          standard:
            verify: false

      - targetId: staging
        profiles: [staging]
        strategy:
          standard:
            verify: true      # Run smoke tests after staging deploy

      - targetId: prod
        profiles: [prod]
        strategy:
          canary:
            runtimeConfig:
              kubernetes:
                serviceNetworking:
                  service: my-app-svc
                  deployment: my-app
            canaryDeployment:
              percentages: [25]   # 25% canary, manual advance to 100%
              verify: true

---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
spec:
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/dev-cluster
  requireApproval: false

---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
spec:
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/staging-cluster
  requireApproval: false

---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
spec:
  gke:
    cluster: projects/MY_PROJECT/locations/us-central1/clusters/prod-cluster
  requireApproval: true       # Requires approval before 25% canary starts
```

---

### Canary Deployment Walkthrough

```bash
# 1. Release is created by Cloud Build and deploys to dev automatically

# 2. Check dev rollout status
gcloud deploy rollouts list \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --filter="targetId=dev"

# 3. Promote to staging (auto-promote or manual)
gcloud deploy releases promote \
  --release=release-abc1234 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# 4. Staging verify job runs... wait for SUCCEEDED

# 5. Promote to prod (requires approval first)
# An approver receives Pub/Sub notification
gcloud deploy rollouts approve ROLLOUT_ID \
  --release=release-abc1234 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# 6. Canary starts at 25% traffic — monitor error rates

# 7. If metrics look good, advance to 100%
gcloud deploy rollouts advance ROLLOUT_ID \
  --release=release-abc1234 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --phase-id=stable

# If issues detected, roll back immediately
gcloud deploy rollbacks create \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --target-id=prod
```

---

## 7. 🛡️ Security & Compliance Across All Four Services

### Binary Authorization

```bash
# 1. Enable Binary Authorization
gcloud services enable binaryauthorization.googleapis.com

# 2. Create an attestor
gcloud container binauthz attestors create quality-gate-attestor \
  --attestation-authority-note=projects/MY_PROJECT/notes/quality-gate \
  --attestation-authority-note-public-keys="pem-file=/path/to/public.pem,comment=prod-signing-key"

# 3. Create a policy requiring attestation
cat > policy.yaml << 'EOF'
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/MY_PROJECT/attestors/quality-gate-attestor
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
clusterAdmissionRules:
  us-central1.prod-cluster:
    evaluationMode: REQUIRE_ATTESTATION
    requireAttestationsBy:
      - projects/MY_PROJECT/attestors/quality-gate-attestor
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
EOF

gcloud container binauthz policy import policy.yaml

# 4. Sign image in Cloud Build (using cosign)
# Add to cloudbuild.yaml:
# - id: 'sign-image'
#   name: 'gcr.io/cloud-builders/gcloud'
#   entrypoint: 'bash'
#   args:
#     - '-c'
#     - |
#       cosign sign --key gcpkms://projects/$PROJECT_ID/locations/us-central1/keyRings/signing-ring/cryptoKeys/signing-key \
#         us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA
```

---

### SLSA Compliance with Cloud Build

| SLSA Level | Requirements | Cloud Build Support |
|---|---|---|
| **Level 0** | No guarantees | N/A |
| **Level 1** | Build process documented | ✅ Build logs |
| **Level 2** | Hosted build, signed provenance | ✅ Provenance attestation |
| **Level 3** | Hardened build service, non-falsifiable | ✅ with Buildx / SLSA builder |

```bash
# Enable SLSA provenance generation in Cloud Build
# Add to cloudbuild.yaml options:
# options:
#   requestedVerifyOption: VERIFIED
#   logging: CLOUD_LOGGING_ONLY

# View generated provenance
gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app:$SHORT_SHA \
  --show-provenance
```

---

### Secret Management in Cloud Build

```yaml
# cloudbuild.yaml — accessing Secret Manager secrets
steps:
  - id: 'use-secret'
    name: 'python:3.11-slim'
    entrypoint: 'python'
    args: ['deploy.py', '--env', '$_DEPLOY_ENV']
    secretEnv:
      - 'DB_PASSWORD'
      - 'SLACK_WEBHOOK'

availableSecrets:
  secretManager:
    - versionName: 'projects/$PROJECT_ID/secrets/db-password/versions/latest'
      env: 'DB_PASSWORD'
    - versionName: 'projects/$PROJECT_ID/secrets/slack-webhook/versions/latest'
      env: 'SLACK_WEBHOOK'
```

---

### VPC Service Controls

```bash
# Create a service perimeter restricting all four services
gcloud access-context-manager perimeters create cicd-perimeter \
  --title="CI/CD Service Perimeter" \
  --resources="projects/MY_PROJECT_NUMBER" \
  --restricted-services=\
"cloudbuild.googleapis.com,\
artifactregistry.googleapis.com,\
clouddeploy.googleapis.com,\
securesourcemanager.googleapis.com" \
  --policy=MY_POLICY_ID
```

---

### CMEK Configuration

```bash
# Create KMS keys for each service
for SERVICE in cloudbuild artifactregistry clouddeploy ssm; do
  gcloud kms keys create ${SERVICE}-key \
    --location=us-central1 \
    --keyring=cicd-keyring \
    --purpose=encryption
done

# Grant service accounts access to their keys
gcloud kms keys add-iam-policy-binding cloudbuild-key \
  --location=us-central1 \
  --keyring=cicd-keyring \
  --member="serviceAccount:service-PROJ_NUM@gcp-sa-cloudbuild.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

## 8. 📊 Monitoring, Logging & Alerting

### Cloud Build Pub/Sub Notifications

```bash
# Cloud Build publishes build events to this topic automatically
# Topic: projects/$PROJECT_ID/topics/cloud-builds

# Subscribe to build events
gcloud pubsub subscriptions create build-notifications \
  --topic=cloud-builds \
  --project=MY_PROJECT

# Pull build events
gcloud pubsub subscriptions pull build-notifications \
  --auto-ack \
  --limit=10 \
  --format=json | jq '.[] | {build_id: .message.attributes.buildId, status: .message.attributes.status}'
```

---

### Cloud Logging Queries

```bash
# ── Cloud Build logs ──────────────────────────────────────────────────
# Failed builds in last 24h
gcloud logging read \
  'resource.type="build"
   logName:"projects/MY_PROJECT/logs/cloudbuild"
   jsonPayload.status="FAILURE"
   timestamp>="2026-03-15T00:00:00Z"' \
  --project=MY_PROJECT --limit=50 \
  --format="table(timestamp,jsonPayload.id,jsonPayload.status)"

# Build duration > 10 minutes
gcloud logging read \
  'resource.type="build"
   jsonPayload.finishTime>""' \
  --project=MY_PROJECT --limit=20

# ── Cloud Deploy logs ──────────────────────────────────────────────────
gcloud logging read \
  'resource.type="clouddeploy.googleapis.com/DeliveryPipeline"
   severity>=WARNING' \
  --project=MY_PROJECT --limit=50

# Failed rollouts
gcloud logging read \
  'resource.type="clouddeploy.googleapis.com/Rollout"
   jsonPayload.state="FAILED"' \
  --project=MY_PROJECT --limit=20

# ── Artifact Registry logs ────────────────────────────────────────────
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="artifactregistry.googleapis.com"
   protoPayload.methodName:"docker.images.push"' \
  --project=MY_PROJECT --limit=50

# ── Secure Source Manager audit logs ──────────────────────────────────
gcloud logging read \
  'resource.type="securesourcemanager.googleapis.com/Instance"
   protoPayload.serviceName="securesourcemanager.googleapis.com"' \
  --project=MY_PROJECT --limit=100
```

---

### Setting Up Build Failure Alerts

```bash
# Create Cloud Monitoring alert for build failures
gcloud alpha monitoring policies create \
  --display-name="Cloud Build Failure Alert" \
  --condition-filter='
    metric.type="cloudbuild.googleapis.com/build/count"
    metric.labels.status="FAILURE"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --notification-channels=NOTIFICATION_CHANNEL_ID \
  --project=MY_PROJECT

# Alert for rollout failures
gcloud alpha monitoring policies create \
  --display-name="Cloud Deploy Rollout Failure" \
  --condition-filter='
    metric.type="clouddeploy.googleapis.com/rollout/count"
    metric.labels.state="FAILED"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --notification-channels=NOTIFICATION_CHANNEL_ID
```

---

### DORA Metrics on GCP

| DORA Metric | How to Measure on GCP | Target (Elite) |
|---|---|---|
| **Deployment Frequency** | `clouddeploy.googleapis.com/rollout/count` where state=SUCCEEDED / time period | Multiple per day |
| **Lead Time for Changes** | Time from commit (SSM) to successful prod rollout | < 1 hour |
| **Change Failure Rate** | FAILED rollouts / total rollouts | < 5% |
| **MTTR** | Time from FAILED rollout detection to rollback SUCCEEDED | < 1 hour |

```sql
-- BigQuery query for deployment frequency (using Cloud Logging export)
SELECT
  DATE(timestamp) AS deploy_date,
  COUNT(*) AS deployments,
  COUNTIF(JSON_VALUE(jsonPayload, '$.state') = 'SUCCEEDED') AS successful
FROM `MY_PROJECT.cloud_logs.deploy_logs_*`
WHERE resource.type = 'clouddeploy.googleapis.com/Rollout'
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY 1 ORDER BY 1 DESC;
```

---

## 9. 📋 gcloud CLI Quick Reference

### Cloud Build

```bash
gcloud builds submit . --config=cloudbuild.yaml --region=R    # Submit a build
gcloud builds list --region=R                                  # List recent builds
gcloud builds describe BUILD_ID --region=R                     # Get build details
gcloud builds cancel BUILD_ID --region=R                       # Cancel a running build
gcloud builds log BUILD_ID --region=R --stream                 # Stream build logs

gcloud builds triggers list --region=R                         # List triggers
gcloud builds triggers create github [flags]                   # Create GitHub trigger
gcloud builds triggers describe TRIGGER --region=R             # Get trigger details
gcloud builds triggers run TRIGGER --region=R --branch=main    # Run trigger manually
gcloud builds triggers delete TRIGGER --region=R               # Delete trigger
gcloud builds triggers import --source=trigger.yaml --region=R # Import from YAML
gcloud builds triggers export TRIGGER --destination=out.yaml   # Export to YAML

gcloud builds worker-pools create POOL --region=R [flags]      # Create private pool
gcloud builds worker-pools list --region=R                     # List worker pools
```

### Artifact Registry

```bash
gcloud artifacts repositories create REPO --location=L --repository-format=F  # Create repo
gcloud artifacts repositories list --location=L                                 # List repos
gcloud artifacts repositories describe REPO --location=L                        # Describe repo
gcloud artifacts repositories delete REPO --location=L --quiet                  # Delete repo

gcloud artifacts docker images list REPO_PATH --include-tags                    # List images
gcloud artifacts docker images describe IMAGE_PATH --show-package-vulnerability # Describe + vulns
gcloud artifacts docker images delete IMAGE_PATH --delete-tags                  # Delete image

gcloud artifacts packages list --repository=R --location=L                      # List packages
gcloud artifacts versions list --package=P --repository=R --location=L         # List versions
gcloud artifacts tags list --package=P --version=V --repository=R --location=L # List tags
gcloud artifacts tags delete TAG --package=P --version=V --repository=R --location=L # Delete tag

gcloud artifacts repositories set-cleanup-policies R --policy=FILE --location=L # Set cleanup
gcloud artifacts repositories add-iam-policy-binding R --location=L \           # Add IAM binding
  --member=M --role=ROLE

gcloud auth configure-docker LOCATION-docker.pkg.dev                            # Configure Docker
```

### Cloud Deploy

```bash
gcloud deploy apply --file=FILE --region=R                             # Apply pipeline/target config
gcloud deploy delivery-pipelines list --region=R                       # List pipelines
gcloud deploy delivery-pipelines describe PIPELINE --region=R          # Describe pipeline
gcloud deploy delivery-pipelines delete PIPELINE --region=R --force    # Delete pipeline

gcloud deploy targets list --region=R                                  # List targets
gcloud deploy targets describe TARGET --region=R                       # Describe target

gcloud deploy releases create RELEASE --delivery-pipeline=P --region=R \  # Create release
  --images=IMAGE_MAP
gcloud deploy releases list --delivery-pipeline=P --region=R           # List releases
gcloud deploy releases promote --release=R --delivery-pipeline=P --region=R  # Promote release
gcloud deploy releases describe RELEASE --delivery-pipeline=P --region=R # Describe release

gcloud deploy rollouts list --release=R --delivery-pipeline=P --region=R   # List rollouts
gcloud deploy rollouts describe ROLLOUT --release=R --delivery-pipeline=P --region=R
gcloud deploy rollouts approve ROLLOUT --release=R --delivery-pipeline=P --region=R
gcloud deploy rollouts reject ROLLOUT --release=R --delivery-pipeline=P --region=R
gcloud deploy rollouts advance ROLLOUT --release=R --delivery-pipeline=P --region=R

gcloud deploy rollbacks create --delivery-pipeline=P --region=R --target-id=TARGET # Rollback
```

### Secure Source Manager

```bash
gcloud secure-source-manager instances create INST --location=L         # Create instance
gcloud secure-source-manager instances list --location=L                # List instances
gcloud secure-source-manager instances describe INST --location=L       # Describe instance
gcloud secure-source-manager instances delete INST --location=L         # Delete instance

gcloud secure-source-manager repositories create REPO \                 # Create repo
  --instance=INST --location=L
gcloud secure-source-manager repositories list \                         # List repos
  --instance=INST --location=L
gcloud secure-source-manager repositories describe REPO \               # Describe repo
  --instance=INST --location=L
gcloud secure-source-manager repositories delete REPO \                 # Delete repo
  --instance=INST --location=L
gcloud secure-source-manager repositories clone REPO \                  # Clone repo
  --instance=INST --location=L

# IAM bindings
gcloud secure-source-manager repositories add-iam-policy-binding REPO \
  --instance=INST --location=L --member=M --role=ROLE
```

---

### Output Formatting Tips

```bash
# JSON output (best for scripting)
gcloud builds list --region=us-central1 --format=json

# Custom table
gcloud artifacts docker images list us-central1-docker.pkg.dev/MY_PROJECT/my-repo \
  --format="table(IMAGE,TAGS,UPDATE_TIME.date('%Y-%m-%d'))"

# Filter builds by status
gcloud builds list --region=us-central1 \
  --filter="status=FAILURE" \
  --sort-by="~createTime" \
  --limit=10

# Extract single value
gcloud builds describe BUILD_ID --region=us-central1 --format="value(status)"
```

---

## 10. 💰 Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Verify at [cloud.google.com/pricing](https://cloud.google.com/pricing).

### Free Tier Summary

| Service | Free Monthly Quota | Paid Beyond Free |
|---|---|---|
| **Cloud Build** | 120 build-minutes/day (e2 machines) | See table below |
| **Artifact Registry** | 0.5 GB storage | $0.10/GB-month |
| **Cloud Deploy** | First 10 deployments/month | $0.03 per successful deployment |
| **Secure Source Manager** | None | Per instance-hour + user-month |

### Cloud Build Pricing

| Machine Type | vCPU | RAM | Price/Minute |
|---|---|---|---|
| **e2-medium** (default) | 2 | 4 GB | $0.003 |
| **e2-standard-2** | 2 | 8 GB | $0.006 |
| **e2-highcpu-8** | 8 | 8 GB | $0.016 |
| **e2-highcpu-32** | 32 | 32 GB | $0.064 |
| **n1-highcpu-8** | 8 | 7.2 GB | $0.040 |
| **n1-highcpu-32** | 32 | 28.8 GB | $0.160 |

### Artifact Registry Pricing

| Resource | Price |
|---|---|
| Storage (Standard) | $0.10/GB-month |
| Storage (Multi-region) | $0.20/GB-month |
| Network egress (same region) | Free |
| Network egress (different region) | Standard egress rates |
| Vulnerability scanning | Container Scanning API: $0.26/image |

### Cost Optimization Tips

| Service | Tip | Savings |
|---|---|---|
| Cloud Build | Use `--cache-from` for Docker layer caching | 30–70% build time |
| Cloud Build | Use parallel steps (`waitFor`) efficiently | Reduce wall-clock time |
| Cloud Build | Use `e2-highcpu` only for heavy builds | Right-size machine type |
| Artifact Registry | Set cleanup policies to delete old artifacts | 40–80% storage reduction |
| Artifact Registry | Use regional repos (avoid multi-regional if not needed) | 50% storage cost |
| Cloud Deploy | Use standard strategy for non-critical envs | Fewer deployments |
| SSM | Share one instance across teams (namespace repos) | Reduce instance-hours |

### Monthly Cost Example

```
Team of 10 engineers, 50 builds/day, 10 deployments/day:

Cloud Build (50 builds × 8 min avg × e2-highcpu-8 × 22 days):
  50 × 8 × $0.016 × 22 = $140.80/month

Artifact Registry (20 GB storage + 100 GB egress/month):
  20 GB × $0.10 + 100 GB × $0.08 = $10/month

Cloud Deploy (10 deploys/day × 22 days = 220 deployments):
  220 × $0.03 = $6.60/month

Secure Source Manager (1 instance, 10 active users):
  Estimated $50–100/month depending on usage

Total: ~$210–260/month for full CI/CD platform
```

---

## 11. Quick Reference & Comparison Tables

### CI/CD Pipeline Stage → GCP Service

| Pipeline Stage | GCP Service | Key Action |
|---|---|---|
| Source code hosting | Secure Source Manager | Push code, create PR, merge |
| Trigger CI | Cloud Build (trigger) | Detect push/PR → start build |
| Build & test | Cloud Build | Run tests, build image |
| Store artifacts | Artifact Registry | Push image, scan vulns |
| Sign artifacts | Cloud Build + Binary Auth | Attest image with key |
| Create release | Cloud Deploy | Snapshot images + manifests |
| Deploy to dev | Cloud Deploy (auto) | Standard deployment |
| Deploy to staging | Cloud Deploy | Standard + verify job |
| Deploy to prod | Cloud Deploy (canary) | 25% → 100% with approval |
| Monitor & rollback | Cloud Deploy + Monitoring | Alert on failure → rollback |

---

### Cloud Build vs. GitHub Actions vs. Jenkins

| Feature | Cloud Build | GitHub Actions | Jenkins |
|---|---|---|---|
| **Management** | Fully managed | Fully managed | Self-managed |
| **GCP integration** | ✅ Native | ✅ (with GCP actions) | ⚠️ Plugins |
| **Pricing** | Per build-minute | Per minute (hosted) | Infrastructure cost |
| **Private runners** | ✅ Worker pools | ✅ Self-hosted runners | ✅ Agents |
| **Parallel steps** | ✅ `waitFor` | ✅ Matrix/parallel | ✅ Parallel stages |
| **Secret Manager** | ✅ Native | ✅ (via action) | ✅ (plugin) |
| **Binary Auth** | ✅ Native | ⚠️ Manual | ⚠️ Manual |
| **VPC access** | ✅ Worker pools | ✅ Self-hosted | ✅ |
| **SLSA provenance** | ✅ Level 2/3 | ✅ (slsa-github-generator) | ⚠️ DIY |
| **Setup effort** | Low | Low | High |

---

### Artifact Registry Format Comparison

| Format | Push Command | Pull Command | Auth Method | File |
|---|---|---|---|---|
| Docker | `docker push` | `docker pull` | `configure-docker` | Dockerfile |
| npm | `npm publish` | `npm install` | `.npmrc` | `package.json` |
| Maven | `mvn deploy` | `mvn package` | `settings.xml` | `pom.xml` |
| PyPI (pip) | `twine upload` | `pip install` | `pip.conf` | `setup.py` |
| Helm | `helm push` | `helm pull` | `helm registry login` | `Chart.yaml` |
| Generic | `gcloud artifacts generic upload` | `gcloud artifacts generic download` | ADC | Any |

---

### Cloud Deploy Strategy Comparison

| Feature | Standard | Canary | Blue/Green |
|---|---|---|---|
| **Traffic shift** | All at once | Incremental % | Full parallel env |
| **Rollback speed** | Minutes | Seconds (shift back) | Instant (DNS/LB switch) |
| **Resource cost** | Normal | Normal + canary pods | 2× resources |
| **Risk** | High (all traffic) | Low (% controlled) | Medium (cutover) |
| **Verify support** | ✅ | ✅ at each % | ✅ |
| **GKE support** | ✅ | ✅ | ✅ |
| **Cloud Run support** | ✅ | ✅ | ✅ |
| **Best for** | Dev/staging | Production | Instant rollback needed |

---

### Cloud Build Step Field Reference

| Field | Required | Description |
|---|---|---|
| `name` | ✅ | Container image to run this step |
| `args` | | Arguments passed to the container entrypoint |
| `entrypoint` | | Override the container entrypoint |
| `dir` | | Working directory within the workspace |
| `env` | | List of `KEY=VALUE` environment variables |
| `secretEnv` | | Environment variable names populated from secrets |
| `id` | | Unique step identifier for `waitFor` references |
| `waitFor` | | Step IDs to wait for; `['-']` = start immediately |
| `timeout` | | Step timeout (default 600s, max 86400s) |
| `volumes` | | Named volumes to mount in the step |
| `allowFailure` | | If true, build continues even if step fails |
| `script` | | Inline shell script (alternative to `args`) |

---

### Common IAM Roles Summary

| Role | Service | Who Needs It |
|---|---|---|
| `cloudbuild.builds.builder` | Cloud Build | Cloud Build SA |
| `cloudbuild.builds.editor` | Cloud Build | DevOps team |
| `artifactregistry.writer` | Artifact Registry | Cloud Build SA |
| `artifactregistry.reader` | Artifact Registry | GKE node SA, Cloud Run SA |
| `clouddeploy.releaser` | Cloud Deploy | Cloud Build SA |
| `clouddeploy.approver` | Cloud Deploy | Team leads, release managers |
| `clouddeploy.viewer` | Cloud Deploy | All team members |
| `container.developer` | GKE | Cloud Deploy SA |
| `run.developer` | Cloud Run | Cloud Deploy SA |
| `securesourcemanager.repoWriter` | SSM | Developers |
| `securesourcemanager.repoReader` | SSM | Cloud Build SA, CI service accounts |
| `secretmanager.secretAccessor` | Secret Manager | Cloud Build SA, Cloud Deploy SA |

---

### Anti-Patterns & Fixes

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Using default Cloud Build SA | Over-permissive (project editor) | Create dedicated SA with minimum roles |
| Hardcoded secrets in `cloudbuild.yaml` | Secret exposed in logs and repo | Use `availableSecrets` with Secret Manager |
| No `--cache-from` in Docker builds | Rebuilding all layers every time | Always use `--cache-from` with AR image |
| Single delivery pipeline for all apps | Changes affect all apps | One pipeline per application/service |
| No approval gate on production | Accidental prod deployments | Set `requireApproval: true` on prod target |
| Pushing `latest` tag only | No image immutability, rollbacks impossible | Always tag with `$SHORT_SHA` + `latest` |
| No cleanup policies | AR storage grows unboundedly | Set cleanup policies to keep last N versions |
| Using Container Registry (legacy) | Deprecated, fewer features | Migrate to Artifact Registry |
| No vulnerability scanning gate | Deploying images with critical CVEs | Add vuln scan step; fail on CRITICAL |
| Force-pushing to main | Overwrites history, breaks audit trail | Enable branch protection in SSM |
| Sequential build steps | Slow builds (no parallelism) | Use `waitFor: ['-']` for independent steps |
| No Binary Authorization | Any image can be deployed to prod | Require attestation on prod GKE cluster |

---

*Generated for GCP CI/CD Services | Official Docs: [cloud.google.com/build/docs](https://cloud.google.com/build/docs) · [cloud.google.com/artifact-registry/docs](https://cloud.google.com/artifact-registry/docs) · [cloud.google.com/deploy/docs](https://cloud.google.com/deploy/docs) · [cloud.google.com/secure-source-manager/docs](https://cloud.google.com/secure-source-manager/docs)*
