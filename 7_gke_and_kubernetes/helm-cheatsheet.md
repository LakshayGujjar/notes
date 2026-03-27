# ⎈ Helm Comprehensive Cheatsheet

> A complete reference for Helm — the Kubernetes package manager — covering theory, CLI usage, chart development, templating, and best practices.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
   - [What is Helm?](#what-is-helm)
   - [Key Components](#key-components)
   - [Helm 3 vs Helm 2](#helm-3-vs-helm-2)
   - [Chart Structure](#chart-structure)
   - [The Rendering Pipeline](#the-rendering-pipeline)
   - [Release Lifecycle](#release-lifecycle)
2. [Installation & Setup](#2-installation--setup)
   - [Installing Helm](#installing-helm)
   - [Configuring kubeconfig](#configuring-kubeconfig)
   - [Managing Repositories](#managing-repositories)
   - [Searching for Charts](#searching-for-charts)
3. [Essential CLI Commands](#3-essential-cli-commands)
   - [Install / Upgrade / Uninstall](#install--upgrade--uninstall)
   - [Inspect & Introspect](#inspect--introspect)
   - [Lint, Template & Package](#lint-template--package)
   - [Repo Management](#repo-management)
   - [Plugins](#plugins)
   - [Key Flags Reference](#key-flags-reference)
4. [Chart Development](#4-chart-development)
   - [Creating a Chart](#creating-a-chart)
   - [Chart.yaml Explained](#chartyaml-explained)
   - [values.yaml Best Practices](#valuesyaml-best-practices)
   - [Overriding Values](#overriding-values)
   - [Values File Layering](#values-file-layering)
5. [Templating — Go Templates + Sprig](#5-templating--go-templates--sprig)
   - [Template Syntax Basics](#template-syntax-basics)
   - [Built-in Objects](#built-in-objects)
   - [Control Flow](#control-flow)
   - [Named Templates](#named-templates)
   - [Common Sprig Functions](#common-sprig-functions)
   - [_helpers.tpl](#_helperstpl)
   - [The tpl Function](#the-tpl-function)
6. [Hooks](#6-hooks)
   - [What Are Hooks?](#what-are-hooks)
   - [Hook Types](#hook-types)
   - [Hook Weights & Deletion Policies](#hook-weights--deletion-policies)
   - [Full Hook Example](#full-hook-example)
7. [Chart Dependencies (Subcharts)](#7-chart-dependencies-subcharts)
   - [Declaring Dependencies](#declaring-dependencies)
   - [Dependency Commands](#dependency-commands)
   - [Passing Values to Subcharts](#passing-values-to-subcharts)
   - [Global Values](#global-values)
   - [Conditions & Tags](#conditions--tags)
8. [Tests](#8-tests)
   - [Writing Helm Tests](#writing-helm-tests)
   - [Example: Connection Test Pod](#example-connection-test-pod)
9. [Library Charts](#9-library-charts)
   - [What is a Library Chart?](#what-is-a-library-chart)
   - [Creating & Consuming a Library Chart](#creating--consuming-a-library-chart)
10. [OCI Registries & Chart Distribution](#10-oci-registries--chart-distribution)
    - [OCI vs Traditional Repos](#oci-vs-traditional-repos)
    - [Push / Pull via OCI](#push--pull-via-oci)
11. [Security & Best Practices](#11-security--best-practices)
    - [required & Schema Validation](#required--schema-validation)
    - [RBAC Considerations](#rbac-considerations)
    - [Secrets Management](#secrets-management)
    - [Linting & CI Integration](#linting--ci-integration)
12. [Debugging & Troubleshooting](#12-debugging--troubleshooting)
    - [Local Rendering](#local-rendering)
    - [Common Errors & Fixes](#common-errors--fixes)
13. [Quick Reference Tables](#13-quick-reference-tables)

---

## 1. Core Concepts & Theory

### What is Helm?

Helm is the **package manager for Kubernetes** — analogous to `apt` for Debian or `brew` for macOS. It lets you define, install, upgrade, and share Kubernetes applications as versioned, reusable packages called **Charts**.

**Why use Helm?**
- Packages complex multi-resource Kubernetes apps into a single deployable unit
- Supports parameterised configuration via `values.yaml`
- Tracks release history, enabling rollbacks
- Enables sharing and reuse via public/private chart repositories
- Reduces copy-paste sprawl across environments (dev/staging/prod)

---

### Key Components

| Component | Description |
|-----------|-------------|
| **Chart** | A Helm package — a directory (or `.tgz`) containing Kubernetes manifest templates + metadata |
| **Release** | A running instance of a Chart deployed to a cluster with a specific name and config |
| **Repository** | An HTTP server (or OCI registry) hosting a collection of Charts |
| **Revision** | A versioned snapshot of a Release — each `install` or `upgrade` creates a new revision |
| **Values** | Configuration inputs that customise a Chart at deploy time |

> 💡 **Tip:** One Chart can produce many Releases. E.g. you can install `nginx` chart as both `nginx-frontend` and `nginx-backend` in the same namespace with different values.

---

### Helm 3 vs Helm 2

Helm 3 (released Nov 2019) was a major architectural overhaul. **Helm 2 is EOL — always use Helm 3.**

| Feature | Helm 2 | Helm 3 |
|---------|--------|--------|
| **Server component** | Required `Tiller` in-cluster | No server — client-only |
| **Security** | Tiller had cluster-admin by default (risky) | Uses kubeconfig RBAC directly |
| **Release storage** | Stored in ConfigMaps in `kube-system` | Stored as Secrets in the release namespace |
| **Namespaces** | Release was global | Release is namespace-scoped |
| **CRDs** | Managed manually | Managed in `crds/` directory |
| **Chart schema** | No built-in validation | `values.schema.json` support |
| **3-way merge** | 2-way strategic merge | 3-way strategic merge patch |

> 💡 **Tip:** Because Helm 3 stores release data as Secrets in the target namespace, you can scope RBAC per-namespace rather than granting cluster-wide permissions.

---

### Chart Structure

Running `helm create mychart` produces this layout:

```
mychart/
├── Chart.yaml            # Chart metadata (name, version, description, dependencies)
├── values.yaml           # Default configuration values
├── values.schema.json    # (Optional) JSON Schema to validate values
├── .helmignore           # Files to exclude when packaging (like .gitignore)
├── charts/               # Directory for subchart dependencies (auto-populated)
├── crds/                 # CustomResourceDefinition YAMLs — applied before templates
├── templates/            # Go template files rendered into Kubernetes manifests
│   ├── _helpers.tpl      # Named template definitions (NOT rendered directly)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt         # Printed to stdout after install/upgrade
│   └── tests/
│       └── test-connection.yaml
└── README.md             # (Optional) Human-readable documentation
```

**Key file roles:**

- **`Chart.yaml`** — Required. Defines the chart's identity (name, version, appVersion, type, dependencies).
- **`values.yaml`** — Default values. Users override these at install time.
- **`templates/`** — Go template files. Helm renders these using values to produce final YAML.
- **`_helpers.tpl`** — Starts with `_` so it is NOT rendered as a manifest. Contains reusable `define` blocks.
- **`NOTES.txt`** — Template rendered and printed to the user after install/upgrade. Use for usage instructions.
- **`.helmignore`** — Patterns of files to exclude from `helm package` output.
- **`crds/`** — CRDs are applied first, before any templates, and are NOT managed as part of the release lifecycle (not deleted on uninstall by default).

---

### The Rendering Pipeline

```
values.yaml  ──┐
--set flags  ──┤         ┌─────────────────────┐       ┌──────────────────┐
--values     ──┼────────▶│  Helm Template       │──────▶│  kubectl apply   │
               │         │  Engine (Go + Sprig) │       │  (to Kubernetes) │
Chart.yaml   ──┤         └─────────────────────┘       └──────────────────┘
templates/   ──┘                   │
                                   ▼
                         Rendered Kubernetes YAML
                         (Deployment, Service, etc.)
```

1. Helm **merges** all value sources (defaults → `--values` files → `--set` flags)
2. Each file in `templates/` is rendered through the **Go template engine** enriched with Sprig functions
3. The rendered YAML is sent to the Kubernetes API server
4. Helm records the release metadata as a **Secret** in the namespace

---

### Release Lifecycle

```
helm install          →  Creates Release (Revision 1)
helm upgrade          →  Updates Release (Revision 2, 3, ...)
helm rollback         →  Reverts to a previous Revision
helm uninstall        →  Deletes all resources + release record
```

```bash
# Full lifecycle example
helm install my-app ./mychart --namespace prod --create-namespace
helm upgrade my-app ./mychart --namespace prod --set image.tag=v2.0
helm rollback my-app 1 --namespace prod   # Roll back to revision 1
helm uninstall my-app --namespace prod
```

---

## 2. Installation & Setup

### Installing Helm

**Linux (script):**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Linux (manual — recommended for CI):**
```bash
# Download a specific version for reproducibility
HELM_VERSION="v3.14.0"
curl -LO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"
tar -zxvf helm-${HELM_VERSION}-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version  # verify installation
```

**macOS (Homebrew):**
```bash
brew install helm
```

**Windows (Chocolatey):**
```powershell
choco install kubernetes-helm
```

**Windows (Scoop):**
```powershell
scoop install helm
```

> 💡 **Tip:** Pin your Helm version in CI/CD pipelines. Helm minor releases can change default behaviours.

---

### Configuring kubeconfig

Helm uses the same `kubeconfig` as `kubectl`. No separate configuration needed.

```bash
# Helm reads $KUBECONFIG or ~/.kube/config by default
echo $KUBECONFIG

# Override the kubeconfig file for a single command
helm list --kubeconfig /path/to/other/config

# Override the cluster context
helm list --kube-context my-production-cluster

# Override the namespace
helm list --namespace staging

# Check which context Helm will use
kubectl config current-context
kubectl config get-contexts  # list all available contexts
```

---

### Managing Repositories

```bash
# Add a repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# List configured repositories
helm repo list

# Fetch the latest chart index from all configured repos
helm repo update

# Remove a repository
helm repo remove stable

# Show repository cache location
helm env HELM_REPOSITORY_CACHE
```

> 💡 **Tip:** Always run `helm repo update` before installing from a repo to ensure you have the latest chart versions.

---

### Searching for Charts

```bash
# Search within locally configured repositories (fast, offline)
helm search repo nginx
helm search repo bitnami/postgresql --versions   # show all available versions

# Search Artifact Hub (https://artifacthub.io) — the public chart registry
helm search hub wordpress
helm search hub --max-col-width 80 prometheus    # wider output for long names
```

---

## 3. Essential CLI Commands

### Install / Upgrade / Uninstall

```bash
# --- INSTALL ---

# Basic install from a repo chart
helm install my-release bitnami/nginx

# Install from a local chart directory
helm install my-app ./mychart

# Install from a packaged chart archive
helm install my-app ./mychart-1.0.0.tgz

# Install into a specific namespace, creating it if it doesn't exist
helm install my-app ./mychart \
  --namespace production \
  --create-namespace

# Dry run — render templates without deploying (great for review)
helm install my-app ./mychart --dry-run --debug

# Install with inline value overrides
helm install my-app ./mychart \
  --set image.repository=myrepo/myapp \
  --set image.tag=v1.2.3 \
  --set replicaCount=3

# Install with a custom values file
helm install my-app ./mychart --values ./env/production.yaml

# Install and wait until all resources are ready (or fail)
helm install my-app ./mychart --wait --timeout 5m

# Atomic: roll back automatically if install fails
helm install my-app ./mychart --atomic --timeout 5m

# Install a specific chart version
helm install my-app bitnami/nginx --version 15.1.0

# Generate a release name automatically
helm install bitnami/nginx --generate-name
```

```bash
# --- UPGRADE ---

# Upgrade an existing release
helm upgrade my-app ./mychart

# Install if not exists, upgrade if it does (idempotent — great for CI/CD)
helm upgrade --install my-app ./mychart \
  --namespace production \
  --create-namespace

# Upgrade with new values, keeping existing overrides
helm upgrade my-app ./mychart \
  --reuse-values \                    # keep previously set values
  --set image.tag=v2.0.0             # override only what changed

# Force resource replacement (delete + recreate) — use with caution
helm upgrade my-app ./mychart --force

# Reset values to chart defaults on upgrade
helm upgrade my-app ./mychart --reset-values
```

```bash
# --- UNINSTALL ---

# Uninstall a release (removes all resources + release record)
helm uninstall my-app --namespace production

# Keep release history after uninstall (useful for audit trails)
helm uninstall my-app --keep-history

# Dry run uninstall (shows what would be deleted)
helm uninstall my-app --dry-run
```

---

### Inspect & Introspect

```bash
# List all releases in current namespace
helm list

# List releases in all namespaces
helm list --all-namespaces   # or -A

# List releases in a specific state
helm list --failed
helm list --pending
helm list --superseded   # old revisions

# Show status of a release (last deployment state, notes, resources)
helm status my-app --namespace production

# Show full release history (all revisions)
helm history my-app --namespace production

# Roll back to a previous revision
helm rollback my-app 2               # roll back to revision 2
helm rollback my-app 0               # roll back to previous revision
helm rollback my-app 1 --wait        # wait for rollback to complete

# Retrieve information about a deployed release
helm get all my-app                  # everything below combined
helm get values my-app               # only the user-supplied values
helm get values my-app --all        # all values including defaults
helm get manifest my-app             # rendered Kubernetes YAML
helm get hooks my-app                # hook manifests
helm get notes my-app                # the NOTES.txt output

# Inspect a chart before installing
helm show all bitnami/nginx          # everything below combined
helm show chart bitnami/nginx        # Chart.yaml contents
helm show values bitnami/nginx       # default values.yaml
helm show readme bitnami/nginx       # README.md
```

---

### Lint, Template & Package

```bash
# Lint a chart for errors and best practices
helm lint ./mychart
helm lint ./mychart --strict         # fail on warnings too
helm lint ./mychart --values ./env/prod.yaml  # lint with specific values

# Render templates locally without deploying (output to stdout)
helm template my-app ./mychart
helm template my-app ./mychart --values ./env/prod.yaml
helm template my-app ./mychart --set image.tag=v2 --debug
helm template my-app ./mychart --output-dir ./rendered/  # write files to disk

# Package a chart into a versioned .tgz archive
helm package ./mychart
helm package ./mychart --destination ./dist/  # specify output directory
helm package ./mychart --sign --key my-key    # cryptographically sign the chart

# Push a packaged chart to a repository
helm push mychart-1.0.0.tgz oci://registry.example.com/charts

# Pull a chart from a repo (download .tgz without installing)
helm pull bitnami/nginx
helm pull bitnami/nginx --version 15.1.0 --untar  # download and extract
```

---

### Repo Management

```bash
# Full repository workflow
helm repo add myrepo https://charts.example.com
helm repo update myrepo              # update only one repo
helm repo update                     # update all repos
helm repo list                       # show configured repos
helm repo remove myrepo

# Index a local chart directory (generates index.yaml for hosting)
helm repo index ./charts/
helm repo index ./charts/ --url https://charts.example.com  # with base URL
helm repo index ./charts/ --merge existing-index.yaml       # merge with existing
```

---

### Plugins

```bash
# List installed plugins
helm plugin list

# Install a plugin from a URL or GitHub repo
helm plugin install https://github.com/databus23/helm-diff
helm plugin install https://github.com/jkroepke/helm-secrets

# Update a plugin
helm plugin update diff

# Remove a plugin
helm plugin uninstall diff

# Useful community plugins:
# helm-diff    — preview changes before upgrade  (helm diff upgrade ...)
# helm-secrets — manage encrypted secrets        (helm secrets install ...)
# helm-git     — use git repos as chart sources
# helm-unittest — unit test chart templates
```

---

### Key Flags Reference

| Flag | Applies To | Description |
|------|-----------|-------------|
| `--set key=val` | install/upgrade | Override a single value inline |
| `--set-string key=val` | install/upgrade | Force value to be treated as string |
| `--set-json key=json` | install/upgrade | Set a JSON value |
| `--values / -f file` | install/upgrade | Load overrides from a YAML file |
| `--dry-run` | install/upgrade/uninstall | Simulate without applying |
| `--debug` | most commands | Verbose output including rendered templates |
| `--atomic` | install/upgrade | Auto-rollback on failure |
| `--wait` | install/upgrade/rollback | Wait until all resources are ready |
| `--timeout` | install/upgrade/rollback | Timeout for `--wait` (default: `5m0s`) |
| `--namespace / -n` | most commands | Target namespace |
| `--create-namespace` | install/upgrade | Create namespace if it doesn't exist |
| `--reuse-values` | upgrade | Keep previous release's values |
| `--reset-values` | upgrade | Reset to chart defaults |
| `--force` | upgrade | Force resource replacement |
| `--version` | install/pull | Specific chart version to use |
| `--generate-name` | install | Auto-generate release name |
| `--output / -o` | list/status | Output format: `table`, `json`, `yaml` |
| `--all-namespaces / -A` | list | Show releases across all namespaces |

```bash
# --set deep nesting and arrays:
--set "ingress.hosts[0].host=myapp.example.com"
--set "ingress.hosts[0].paths[0].path=/"
--set "env[0].name=ENV_VAR,env[0].value=hello"

# --set with commas in values (use backslash escape):
--set "annotations.description=hello\, world"
```

> 💡 **Tip:** For anything more than 2-3 `--set` flags, switch to a `--values` file. It's easier to read, review in PRs, and commit to version control.

---

## 4. Chart Development

### Creating a Chart

```bash
# Scaffold a new chart (generates the full directory structure)
helm create mychart

# Result: a working Nginx deployment chart you can immediately modify
```

---

### Chart.yaml Explained

```yaml
# Chart.yaml — required metadata file for every chart

apiVersion: v2           # Helm chart API version — always "v2" for Helm 3

name: mychart            # Chart name (must match the directory name when developing)

description: A Helm chart for deploying MyApp  # Short, human-readable description

# "application" (default) — a deployable app
# "library"               — a shared template library (not directly deployable)
type: application

version: 1.2.3           # Chart version (SemVer). Increment on EVERY chart change.
                         # This is the version of the chart packaging itself.

appVersion: "2.0.1"      # Version of the APPLICATION being packaged (informational)
                         # Quoted string — can be any format your app uses

# Optional: mark chart as deprecated in a repository
deprecated: false

# Optional: home page for the chart / application
home: https://example.com/mychart

# Optional: list of chart sources
sources:
  - https://github.com/myorg/mychart

# Optional: list of maintainers
maintainers:
  - name: Alice Smith
    email: alice@example.com
    url: https://alice.dev

# Optional: keywords for helm search
keywords:
  - myapp
  - web
  - api

# Optional: chart icon URL (shown in Artifact Hub / Helm UI)
icon: https://example.com/icon.png

# Optional: minimum Helm version required
kubeVersion: ">=1.25.0-0"   # semver constraint on Kubernetes version

# Dependencies (subcharts) — see Section 7
dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled          # only install if this value is true
```

> 💡 **Tip:** Always increment `version` when you change anything in the chart. Helm uses `version` for release tracking. `appVersion` is purely informational.

---

### values.yaml Best Practices

```yaml
# values.yaml — default configuration for the chart
# Every value here can be overridden by the user at install time

# --- Image configuration ---
image:
  repository: myorg/myapp   # image name without tag
  tag: ""                   # default to empty — override at deploy time
  pullPolicy: IfNotPresent  # Always | Never | IfNotPresent

# Use empty string defaults for things that MUST be overridden
# and mark them clearly in comments

# --- Replica count ---
replicaCount: 1

# --- Service configuration ---
service:
  type: ClusterIP           # ClusterIP | NodePort | LoadBalancer
  port: 80
  targetPort: 8080

# --- Ingress (disabled by default) ---
ingress:
  enabled: false            # boolean flag pattern — opt-in features
  className: ""
  annotations: {}           # map default — always use {} not null
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []                   # list default — always use [] not null

# --- Resource requests & limits ---
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

# --- Environment variables (list pattern) ---
env: []
# env:
#   - name: DATABASE_URL
#     value: postgres://localhost/mydb

# --- Extra labels applied to all resources ---
commonLabels: {}

# --- Pod annotations ---
podAnnotations: {}

# --- Node selector / tolerations / affinity ---
nodeSelector: {}
tolerations: []
affinity: {}

# --- Service Account ---
serviceAccount:
  create: true
  name: ""                  # if empty, generated from release name
  annotations: {}

# --- Application-specific config ---
config:
  logLevel: info
  debug: false
  databaseUrl: ""           # INTENTIONALLY EMPTY — must be provided
```

> 💡 **Tip:** Initialize maps as `{}` and lists as `[]` in `values.yaml`. Templates using `range` and `with` handle these gracefully. `null` values can cause unexpected behaviour.

---

### Overriding Values

**Precedence order (highest wins):**
```
--set flags  >  --values files (right-to-left)  >  values.yaml defaults
```

```bash
# 1. --set overrides a single key (dot notation for nesting)
helm install my-app ./mychart --set image.tag=v2.0

# 2. --set-string forces string type (useful for numeric-looking values)
helm install my-app ./mychart --set-string image.tag="1234"

# 3. --values loads a whole file of overrides
helm install my-app ./mychart --values ./prod-values.yaml

# 4. Multiple --values files: RIGHT-MOST wins for duplicate keys
helm install my-app ./mychart \
  --values ./base.yaml \          # loaded first (lower priority)
  --values ./prod.yaml \          # loaded second
  --values ./secrets.yaml         # loaded last (highest priority among files)

# 5. Combining --values and --set: --set always wins
helm install my-app ./mychart \
  --values ./prod.yaml \
  --set image.tag=hotfix          # this wins over image.tag in prod.yaml
```

```bash
# Setting nested values with --set
--set "a.b.c=value"
# Equivalent YAML: a: { b: { c: value } }

# Setting array elements
--set "ingress.hosts[0]=myapp.com"
--set "env[0].name=FOO,env[0].value=bar"

# Setting a value to null (deletes key)
--set "tolerations=null"
```

---

### Values File Layering

A common pattern for multi-environment deployments:

```bash
# Directory structure
config/
├── values.yaml        # Base defaults (committed to repo)
├── values-dev.yaml    # Dev overrides
├── values-staging.yaml
└── values-prod.yaml

# Install for dev
helm upgrade --install my-app ./mychart \
  --values ./config/values.yaml \
  --values ./config/values-dev.yaml \
  --namespace dev

# Install for prod
helm upgrade --install my-app ./mychart \
  --values ./config/values.yaml \
  --values ./config/values-prod.yaml \
  --namespace production
```

```yaml
# values-prod.yaml — only override what differs from base
replicaCount: 5

image:
  tag: "1.9.3"   # pinned version for prod

resources:
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
```

> 💡 **Tip:** Never put secrets in values files committed to source control. Use `helm-secrets`, sealed-secrets, external-secrets, or CI/CD secret injection via `--set` instead.

---

## 5. Templating — Go Templates + Sprig

### Template Syntax Basics

```yaml
# Everything between {{ }} is a template expression
# Helm uses Go's text/template engine + Sprig function library

# Basic value substitution
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-myapp     # renders: "my-release-myapp"
  namespace: {{ .Release.Namespace }}

# Whitespace control:
# {{- trims whitespace/newlines BEFORE the expression
# -}} trims whitespace/newlines AFTER the expression

# Without whitespace control (produces extra blank lines):
metadata:
  name: {{ .Values.name }}

# With left-trim (removes preceding newline):
metadata:
  name: {{- .Values.name }}

# With both trims (removes surrounding whitespace entirely):
  {{- .Values.name -}}

# Pipeline: chain functions with |
  name: {{ .Values.name | upper | quote }}  # "MY-APP"

# Variables: assign with :=, reference with $
{{- $fullName := printf "%s-%s" .Release.Name .Chart.Name -}}
metadata:
  name: {{ $fullName }}
```

---

### Built-in Objects

```yaml
# .Values — access values from values.yaml (and overrides)
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}

# .Release — information about the current release
name: {{ .Release.Name }}             # release name
namespace: {{ .Release.Namespace }}   # target namespace
isInstall: {{ .Release.IsInstall }}   # true on first install
isUpgrade: {{ .Release.IsUpgrade }}   # true on upgrade
revision: {{ .Release.Revision }}     # revision number (int)
service: {{ .Release.Service }}       # always "Helm"

# .Chart — contents of Chart.yaml
name: {{ .Chart.Name }}               # chart name
version: {{ .Chart.Version }}         # chart version
appVersion: {{ .Chart.AppVersion }}   # app version
description: {{ .Chart.Description }}

# .Capabilities — Kubernetes cluster information
kubeVersion: {{ .Capabilities.KubeVersion.Version }}
# Check if a specific API is available:
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: extensions/v1beta1
{{- end }}

# .Files — access non-template files in the chart directory
# (files in templates/ are excluded)
config: |
  {{ .Files.Get "config/app.conf" | nindent 2 }}

# Iterate over files matching a glob pattern
{{- range $path, $content := .Files.Glob "config/*.conf" }}
  {{ $path }}: |
    {{ $content | nindent 4 }}
{{- end }}

# .Template — information about the current template file
file: {{ .Template.Name }}            # e.g. "mychart/templates/deployment.yaml"
basePath: {{ .Template.BasePath }}    # e.g. "mychart/templates"
```

---

### Control Flow

```yaml
# --- if / else if / else ---
{{- if .Values.ingress.enabled }}
# Ingress block rendered only when ingress.enabled is true
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- else if .Values.ingress.useNodePort }}
# Alternative block
{{- else }}
# Default block
{{- end }}

# Falsy values in Helm templates: false, 0, nil/null, empty string "", empty list [], empty map {}
# Everything else is truthy

# --- with (scopes context and checks for non-empty) ---
# Without "with":
{{- if .Values.podAnnotations }}
  annotations:
    {{- range $key, $val := .Values.podAnnotations }}
    {{ $key }}: {{ $val | quote }}
    {{- end }}
{{- end }}

# With "with" (cleaner — changes . to the scoped value):
{{- with .Values.podAnnotations }}
  annotations:
    {{- toYaml . | nindent 4 }}   # . is now .Values.podAnnotations
{{- end }}

# Access parent scope inside "with" using $:
{{- with .Values.podAnnotations }}
  name: {{ $.Release.Name }}        # $ always refers to root scope
{{- end }}

# --- range (iterate over lists and maps) ---

# Iterate over a list:
{{- range .Values.tolerations }}
- key: {{ .key }}
  operator: {{ .operator }}
{{- end }}

# Iterate over a list with index:
{{- range $i, $val := .Values.hostAliases }}
- ip: {{ $val.ip }}
  hostnames: {{ $val.hostnames | toYaml | nindent 4 }}
{{- end }}

# Iterate over a map:
{{- range $key, $value := .Values.labels }}
{{ $key }}: {{ $value | quote }}
{{- end }}

# Most common pattern — dump a map as YAML (no range needed):
labels:
  {{- toYaml .Values.commonLabels | nindent 2 }}
```

---

### Named Templates

Named templates let you define reusable template snippets — primarily in `_helpers.tpl`.

```yaml
# --- define a named template ---
# In _helpers.tpl:

{{/*
Generate standard labels applied to all resources.
Usage: include "mychart.labels" . 
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Generate the full app name (release + chart, max 63 chars).
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end }}
```

```yaml
# --- include vs template ---

# "template" — inserts rendered content directly (cannot be piped)
metadata:
  labels:
    {{ template "mychart.labels" . }}  # BAD: cannot pipe for indentation

# "include" — returns as string so it can be piped (ALWAYS prefer this)
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}  # GOOD: piped to nindent

# Passing a different context (scope) to a named template:
{{- include "mychart.labels" (dict "Release" .Release "Chart" .Chart "Values" .Values) }}
```

> 💡 **Tip:** Always use `include` (not `template`) in production templates so you can pipe the result to `nindent` for proper YAML indentation.

---

### Common Sprig Functions

```yaml
# --- String functions ---
{{ "hello" | upper }}                  # HELLO
{{ "HELLO" | lower }}                  # hello
{{ "hello world" | title }}            # Hello World
{{ "  hello  " | trim }}              # hello
{{ "hello" | quote }}                  # "hello"
{{ "hello" | squote }}                 # 'hello'
{{ "hello" | b64enc }}                 # aGVsbG8=
{{ "aGVsbG8=" | b64dec }}             # hello
{{ "hello world" | replace " " "-" }} # hello-world
{{ list "a" "b" "c" | join "," }}      # a,b,c
{{ "hello" | contains "ell" }}         # true
{{ "hello" | hasPrefix "hel" }}        # true
{{ "hello" | hasSuffix "llo" }}        # true
{{ printf "App: %s v%s" .Chart.Name .Chart.AppVersion }}

# --- Default & required ---
{{ .Values.image.tag | default "latest" }}      # use default if empty/nil
{{ required "image.tag is required" .Values.image.tag }}  # fail if missing

# --- Type conversion ---
{{ "42" | int }}                        # 42 (integer)
{{ 42 | toString }}                    # "42" (string)
{{ "true" | toBool }}                  # true (boolean)

# --- YAML/JSON serialization ---
{{ .Values.labels | toYaml }}          # serialize map/list to YAML string
{{ .Values.config | toJson }}          # serialize to JSON
{{ .Values.jsonString | fromJson }}    # parse JSON string to object
{{ .Values.yamlString | fromYaml }}    # parse YAML string to object

# --- Indentation (critical for YAML) ---
{{ .Values.labels | toYaml | indent 4 }}    # indent every line by 4 spaces
{{ .Values.labels | toYaml | nindent 4 }}   # like indent but adds leading newline
                                             # nindent is almost always what you want

# --- Math ---
{{ add 1 2 }}       # 3
{{ sub 5 2 }}       # 3
{{ mul 2 3 }}       # 6
{{ div 10 2 }}      # 5
{{ mod 10 3 }}      # 1
{{ max 1 5 3 }}     # 5
{{ min 1 5 3 }}     # 1

# --- Lists ---
{{ list "a" "b" "c" }}                 # creates a list
{{ list "a" "b" | append "c" }}        # append to list
{{ list "a" "b" "c" | first }}         # a
{{ list "a" "b" "c" | last }}          # c
{{ list "a" "b" "c" | len }}           # 3
{{ list "a" "b" "c" | has "b" }}       # true

# --- Dicts ---
{{ dict "key" "value" "foo" "bar" }}                   # create a map
{{ merge (dict "a" 1) (dict "b" 2) }}                  # merge two maps
{{ .Values.labels | keys | sortAlpha | join "," }}     # get + sort map keys

# --- Date ---
{{ now | date "2006-01-02" }}          # current date (Go reference time format)
{{ now | dateModify "+48h" | date "2006-01-02" }}

# --- UUID ---
{{ uuidv4 }}                           # generate a random UUID (use sparingly)

# --- Cryptographic ---
{{ randAlphaNum 32 }}                  # 32-char random alphanumeric string
{{ "mysecret" | sha256sum }}           # SHA-256 hash
```

> 💡 **Tip:** `nindent` is your best friend. It adds a leading newline before indenting, which means you can safely write `{{- include "mytemplate" . | nindent 4 }}` inline without worrying about blank lines.

---

### _helpers.tpl

`_helpers.tpl` is the conventional home for all named template definitions in a chart. The leading `_` tells Helm NOT to render it as a Kubernetes manifest.

```yaml
# templates/_helpers.tpl
# All definitions in this file are available to all other templates in the chart.

{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
Truncated at 63 chars — Kubernetes DNS label limit.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := .Values.nameOverride | default .Chart.Name }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Create chart label — used in the "helm.sh/chart" label.
*/}}
{{- define "mychart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels — applied to ALL resources for consistency.
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — used in matchLabels (must be stable across upgrades).
Do NOT add mutable labels here (like version) — they break rolling updates.
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name — uses override if set, otherwise generates one.
*/}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- .Values.serviceAccount.name | default (include "mychart.fullname" .) }}
{{- else }}
{{- .Values.serviceAccount.name | default "default" }}
{{- end }}
{{- end }}
```

---

### The tpl Function

`tpl` renders a string value **as a Go template**, enabling dynamic configuration strings in `values.yaml`.

```yaml
# values.yaml
appName: "myapp"
fullDomain: "{{ .Values.appName }}.{{ .Release.Namespace }}.svc.cluster.local"
configSnippet: |
  server_name {{ .Values.appName }};
  listen 80;
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}
data:
  # tpl renders the value string as a template, passing current context
  domain: {{ tpl .Values.fullDomain . }}
  # Result: "myapp.production.svc.cluster.local"

  nginx.conf: |
    {{- tpl .Values.configSnippet . | nindent 4 }}
```

> 💡 **Tip:** `tpl` is powerful but expensive — it re-runs the template engine. Use it only when you genuinely need dynamic value composition. Also beware: if a user supplies a malicious `tpl` value, it executes as a template (template injection risk in multi-tenant contexts).

---

## 6. Hooks

### What Are Hooks?

Hooks let you run Kubernetes Jobs (or other resources) at specific points in a release lifecycle — for example, database migrations before an upgrade, or cache warming after an install.

Hooks are regular Kubernetes manifest templates annotated with `helm.sh/hook`.

---

### Hook Types

| Annotation Value | Runs When |
|-----------------|-----------|
| `pre-install` | Before any templates are rendered/applied on first install |
| `post-install` | After all resources are successfully installed |
| `pre-upgrade` | Before upgrade templates are applied |
| `post-upgrade` | After all resources are successfully upgraded |
| `pre-rollback` | Before rollback templates are applied |
| `post-rollback` | After rollback completes |
| `pre-delete` | Before any resources are deleted on uninstall |
| `post-delete` | After all resources are deleted |
| `test` | Only when `helm test` is run |

---

### Hook Weights & Deletion Policies

```yaml
annotations:
  # The hook type (can be multiple, comma-separated)
  helm.sh/hook: pre-upgrade,pre-install

  # Execution order when multiple hooks of same type exist
  # Lower weight runs first. Default: "0". Can be negative.
  helm.sh/hook-weight: "-5"

  # When to delete the hook resource:
  # before-hook-creation  — delete old hook resource before creating new (default)
  # hook-succeeded        — delete after hook completes successfully
  # hook-failed           — delete even if hook fails (for cleanup)
  helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
```

---

### Full Hook Example

```yaml
# templates/hooks/db-migrate.yaml
# A pre-upgrade Job that runs database migrations before the new app version starts

apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-db-migrate
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    # This Job runs before upgrade AND before initial install
    helm.sh/hook: pre-install,pre-upgrade

    # Run this hook before other pre-upgrade hooks (lower number = earlier)
    helm.sh/hook-weight: "-1"

    # Clean up: delete the old Job before creating a new one,
    # and delete it again once it succeeds
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  # Retry up to 3 times on failure before the hook (and release) fails
  backoffLimit: 3

  template:
    metadata:
      name: db-migrate
    spec:
      restartPolicy: Never        # Required for Jobs used as hooks

      containers:
        - name: db-migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}

          command: ["python", "manage.py", "migrate", "--noinput"]

          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "mychart.fullname" . }}-secrets
                  key: database-url
```

> 💡 **Tip:** Always set `helm.sh/hook-delete-policy: before-hook-creation` — otherwise Helm will error if a hook Job from a previous release still exists.

---

## 7. Chart Dependencies (Subcharts)

### Declaring Dependencies

```yaml
# Chart.yaml — declare subcharts your chart depends on

dependencies:
  # PostgreSQL from Bitnami
  - name: postgresql           # must match the chart name exactly
    version: "13.x.x"         # semver constraint — "x" acts as wildcard
    repository: https://charts.bitnami.com/bitnami

  # Redis from Bitnami (conditionally enabled)
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled   # only install if .Values.redis.enabled is true
    alias: cache               # use "cache" instead of "redis" as the key name

  # OCI-based dependency
  - name: common
    version: "2.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts

  # Local subchart (path instead of repository)
  - name: mysubchart
    version: "0.1.0"
    repository: "file://../mysubchart"  # relative path to local chart

  # Tagged dependencies (grouped enable/disable)
  - name: monitoring
    version: "1.x.x"
    repository: https://prometheus-community.github.io/helm-charts
    tags:
      - observability           # enable/disable all "observability"-tagged deps at once
```

---

### Dependency Commands

```bash
# Download declared dependencies into charts/ directory
helm dependency update ./mychart
# Creates: charts/postgresql-13.x.x.tgz, charts/redis-18.x.x.tgz
# Also creates/updates: Chart.lock (pinned versions)

# Install from existing Chart.lock (without resolving — fast, reproducible)
helm dependency build ./mychart

# List current dependency status
helm dependency list ./mychart
```

> 💡 **Tip:** Commit `Chart.lock` to source control (like `package-lock.json`) for reproducible builds. Commit `charts/*.tgz` only if you need fully offline builds.

---

### Passing Values to Subcharts

```yaml
# values.yaml — pass values to subcharts using the chart name (or alias) as key

# Configure the "postgresql" subchart:
postgresql:
  auth:
    username: myapp
    password: ""              # use secretKeyRef or helm-secrets in production
    database: myapp_prod
  primary:
    persistence:
      enabled: true
      size: 20Gi

# Configure the "cache" subchart (alias for redis):
cache:
  architecture: standalone
  auth:
    enabled: false
```

---

### Global Values

Global values are accessible by ALL charts and subcharts using `.Values.global.*`:

```yaml
# values.yaml
global:
  imageRegistry: "registry.example.com"   # shared across all subcharts
  storageClass: "fast-ssd"
  environment: "production"

# In a subchart template — accessing global values:
image: {{ .Values.global.imageRegistry }}/mysubimage:latest
```

---

### Conditions & Tags

```yaml
# values.yaml — control which subcharts are installed

# Condition: per-chart boolean flag
redis:
  enabled: false          # disables the redis subchart

postgresql:
  enabled: true           # enables the postgresql subchart

# Tags: group-level enable/disable
tags:
  observability: false    # disables ALL charts tagged with "observability"
```

> 💡 **Tip:** Condition (per-chart flag) takes precedence over tags. If both are set, condition wins.

---

## 8. Tests

### Writing Helm Tests

Helm tests are Kubernetes Pods annotated with `helm.sh/hook: test`. They run to completion — if the Pod exits 0, the test passes; non-zero means failure.

```bash
# Run all tests for a release
helm test my-app --namespace production

# Run with logs printed on failure
helm test my-app --logs

# Run with timeout
helm test my-app --timeout 5m
```

---

### Example: Connection Test Pod

```yaml
# templates/tests/test-connection.yaml

apiVersion: v1
kind: Pod
metadata:
  name: {{ include "mychart.fullname" . }}-test-connection
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    # Marks this Pod as a test — only created when "helm test" is run
    helm.sh/hook: test

    # Delete the Pod after it succeeds to keep the cluster clean
    helm.sh/hook-delete-policy: hook-succeeded
spec:
  restartPolicy: Never      # Pod must not restart — exit code is the result

  containers:
    - name: wget
      image: busybox
      # Test that the service is reachable and returns HTTP 200
      command:
        - wget
        - "--spider"        # spider mode: check reachability without saving output
        - "--timeout=10"
        - "{{ include "mychart.fullname" . }}:{{ .Values.service.port }}"
```

> 💡 **Tip:** Put test files in `templates/tests/` and ensure they are ONLY annotated with `helm.sh/hook: test`. Without the annotation, they would be deployed as regular resources.

---

## 9. Library Charts

### What is a Library Chart?

A **library chart** (`type: library` in `Chart.yaml`) contains only named template definitions (`define` blocks). It cannot be installed directly — it has no deployable resources. Other charts depend on it to share common template logic.

**Use library charts when:**
- Multiple application charts share the same label schema
- You want a single place to maintain helper templates across an org
- You want to enforce naming conventions consistently

---

### Creating & Consuming a Library Chart

**1. Create the library chart:**
```bash
helm create common-lib
# Edit Chart.yaml: set type: library
# Delete everything in templates/ except _helpers.tpl
# Add your shared define blocks to templates/_helpers.tpl
```

```yaml
# common-lib/Chart.yaml
apiVersion: v2
name: common-lib
description: Shared Helm template library for MyOrg
type: library             # KEY: marks this as a library (not deployable)
version: 1.0.0
```

```yaml
# common-lib/templates/_helpers.tpl

{{/* Shared org-wide labels */}}
{{- define "common-lib.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
org.example.com/team: {{ .Values.global.team | default "platform" }}
{{- end }}

{{/* Standard resource annotations */}}
{{- define "common-lib.annotations" -}}
org.example.com/chart-version: {{ .Chart.Version | quote }}
{{- end }}
```

**2. Consume the library chart:**
```yaml
# myapp/Chart.yaml
dependencies:
  - name: common-lib
    version: "1.x.x"
    repository: "file://../common-lib"   # local, or use a real repo URL
```

```yaml
# myapp/templates/deployment.yaml
metadata:
  labels:
    # Call the shared template from the library chart
    {{- include "common-lib.labels" . | nindent 4 }}
  annotations:
    {{- include "common-lib.annotations" . | nindent 4 }}
```

```bash
helm dependency update ./myapp    # pulls common-lib into myapp/charts/
helm install my-app ./myapp
```

---

## 10. OCI Registries & Chart Distribution

### OCI vs Traditional Repos

| Feature | Traditional Repo (`index.yaml`) | OCI Registry |
|---------|--------------------------------|--------------|
| **Protocol** | HTTP/HTTPS with custom index | OCI Distribution Spec (same as container images) |
| **Auth** | Repo-specific (basic auth, tokens) | Standard container registry auth (docker login) |
| **Discovery** | `helm search repo` | Registry-native (e.g. docker hub browse) |
| **Storage** | Separate chart hosting server needed | Any OCI-compliant registry (ECR, GCR, GHCR, Harbor) |
| **Versioning** | `index.yaml` managed separately | OCI tags |
| **Helm support** | Helm 2+ | Helm 3.8+ (GA) |

---

### Push / Pull via OCI

```bash
# --- Login to an OCI registry ---
# Docker Hub
helm registry login registry-1.docker.io -u myuser

# GitHub Container Registry
helm registry login ghcr.io -u myuser --password-stdin < token.txt

# AWS ECR (uses AWS CLI for token)
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Logout
helm registry logout registry-1.docker.io

# --- Package the chart ---
helm package ./mychart
# Creates: mychart-1.0.0.tgz

# --- Push to OCI registry ---
helm push mychart-1.0.0.tgz oci://ghcr.io/myorg/charts
# Result: ghcr.io/myorg/charts/mychart:1.0.0

# --- Pull a chart from OCI ---
helm pull oci://ghcr.io/myorg/charts/mychart --version 1.0.0
helm pull oci://ghcr.io/myorg/charts/mychart --version 1.0.0 --untar

# --- Install directly from OCI ---
helm install my-app oci://ghcr.io/myorg/charts/mychart --version 1.0.0

# --- Upgrade from OCI ---
helm upgrade my-app oci://ghcr.io/myorg/charts/mychart --version 1.1.0

# --- Show info about an OCI chart (no install) ---
helm show chart oci://ghcr.io/myorg/charts/mychart --version 1.0.0
helm show values oci://ghcr.io/myorg/charts/mychart --version 1.0.0
```

> 💡 **Tip:** OCI-based charts do NOT use `helm repo add`. You reference them directly by their full `oci://` URL. There is no `helm search` support for OCI registries — use the registry's native UI or API.

---

## 11. Security & Best Practices

### required & Schema Validation

**Using `required` to enforce mandatory values:**
```yaml
# templates/deployment.yaml
# Helm will fail with a clear error if image.repository is not set
image: {{ required "A valid .Values.image.repository is required!" .Values.image.repository }}

# Multi-line error message trick:
{{- $dbUrl := required (printf "values.databaseUrl is required in namespace %s" .Release.Namespace) .Values.databaseUrl -}}
```

**Schema validation with `values.schema.json`:**
```json
// values.schema.json — validated automatically on install/upgrade/lint
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "description": "Number of replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Container image repository"
        },
        "tag": {
          "type": "string"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  }
}
```

> 💡 **Tip:** `values.schema.json` is one of the most underused Helm features. It catches typos and type errors BEFORE they hit the cluster, giving users clear error messages.

---

### RBAC Considerations

```yaml
# templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mychart.serviceAccountName" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: false  # disable auto-mount — only enable if needed
{{- end }}
```

```bash
# Helm itself uses kubeconfig RBAC — limit what Helm can do per namespace:

# Create a namespace-scoped role for Helm operations
kubectl create role helm-manager \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=deployments,services,configmaps,secrets \
  --namespace=production

kubectl create rolebinding helm-manager-binding \
  --role=helm-manager \
  --serviceaccount=production:helm-sa \
  --namespace=production
```

---

### Secrets Management

**Never store secrets in `values.yaml`** — they end up in Helm's release history (Secrets) and version control.

```yaml
# BAD: hardcoded secret in values.yaml
database:
  password: "supersecretpassword123"  # NEVER DO THIS
```

```bash
# Pattern 1: helm-secrets plugin (encrypts values files with SOPS/GPG)
helm plugin install https://github.com/jkroepke/helm-secrets
helm secrets install my-app ./mychart -f secrets.enc.yaml

# Pattern 2: CI/CD injects secrets via --set at deploy time
helm upgrade --install my-app ./mychart \
  --set database.password="${DB_PASSWORD}"  # injected from CI secret store

# Pattern 3: Reference existing Secrets (don't create them in Helm)
# In values.yaml, just store the secret name:
database:
  existingSecret: "my-db-credentials"    # name of pre-existing k8s Secret
  existingSecretKey: "password"

# In templates/deployment.yaml:
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.database.existingSecret }}
        key: {{ .Values.database.existingSecretKey }}

# Pattern 4: External Secrets Operator (ESO)
# Sync secrets from Vault/AWS Secrets Manager/GCP Secret Manager into k8s Secrets
# externalsecrets.io — configure ExternalSecret CRD to pull from your secret store
```

---

### Linting & CI Integration

```bash
# Full pre-commit / CI lint pipeline:

# 1. Lint the chart
helm lint ./mychart --strict --values ./ci/test-values.yaml

# 2. Render templates to catch runtime errors
helm template my-app ./mychart --values ./ci/test-values.yaml > /dev/null

# 3. Validate rendered YAML against live cluster schemas (requires cluster access)
helm template my-app ./mychart | kubectl apply --dry-run=server -f -

# 4. Validate without cluster (uses kubeval / kubeconform)
helm template my-app ./mychart | kubeconform -strict -kubernetes-version 1.28.0

# 5. Security scanning with Trivy or checkov
helm template my-app ./mychart | trivy config -
helm template my-app ./mychart | checkov -d - --framework kubernetes

# GitHub Actions snippet:
# - name: Lint Helm Chart
#   run: helm lint ./mychart --strict
# - name: Template Helm Chart
#   run: helm template my-app ./mychart --values ./ci/values.yaml
```

> 💡 **Tip:** Use `ct` (chart-testing) from the helm/chart-testing project for a complete CI workflow: it handles linting, schema validation, and test installs on kind/minikube clusters.

---

## 12. Debugging & Troubleshooting

### Local Rendering

```bash
# Render templates locally and print to stdout — no cluster needed
helm template my-app ./mychart

# With a specific values file
helm template my-app ./mychart --values ./env/prod.yaml

# With inline overrides
helm template my-app ./mychart --set image.tag=debug

# Debug mode: shows computed values + template sources before each manifest
helm template my-app ./mychart --debug

# Write rendered files to disk for inspection (one file per resource)
helm template my-app ./mychart --output-dir ./rendered/

# Dry run (sends manifests to API server for validation, but doesn't apply)
helm install my-app ./mychart --dry-run --debug

# Server-side dry run (more accurate validation)
helm install my-app ./mychart --dry-run=server
```

---

### Common Errors & Fixes

**1. Indentation errors (most common)**
```yaml
# WRONG — toYaml without nindent causes misaligned YAML
labels:
  {{ toYaml .Values.labels }}    # BAD: no leading newline, no indentation

# CORRECT — nindent adds a leading newline + indents all lines
labels:
  {{- toYaml .Values.labels | nindent 2 }}  # GOOD

# CORRECT — for nested blocks inside a resource (4-space indent)
spec:
  template:
    metadata:
      labels:
        {{- toYaml .Values.labels | nindent 8 }}
```

**2. Missing key / nil pointer**
```bash
# Error: template: mychart/templates/deployment.yaml:15:22:
#   executing "..." at <.Values.database.host>: nil pointer evaluating interface {}.host

# Fix: use "default" to handle nil values
{{ .Values.database.host | default "localhost" }}

# Or check existence first
{{- if .Values.database }}
host: {{ .Values.database.host }}
{{- end }}
```

**3. Wrong type (number vs string)**
```yaml
# WRONG — YAML interprets bare numbers as integers, but k8s expects string
tag: 1.0     # parsed as float 1.0, not string "1.0"

# FIX 1 — quote in values.yaml
tag: "1.0"

# FIX 2 — quote in template
image: "myapp:{{ .Values.image.tag | toString }}"

# FIX 3 — use --set-string at install time
--set-string image.tag=1.0
```

**4. Release already exists**
```bash
# Error: cannot re-use a name that is still in use

# Fix: use --upgrade-install (idempotent — install or upgrade)
helm upgrade --install my-app ./mychart

# Or uninstall first
helm uninstall my-app
```

**5. Hook resource already exists**
```bash
# Error: rendered manifests contain a resource that already exists (hook)

# Fix: add hook-delete-policy to clean up before re-creating
annotations:
  helm.sh/hook-delete-policy: before-hook-creation
```

**6. Lookup function for live cluster data**
```yaml
# Use lookup to read existing cluster resources during templating
# (only works on live clusters, not with --dry-run or helm template)
{{- $existingSecret := lookup "v1" "Secret" .Release.Namespace "my-secret" -}}
{{- if $existingSecret }}
# Secret exists — reuse its password
password: {{ $existingSecret.data.password }}
{{- else }}
# Secret doesn't exist — generate new password
password: {{ randAlphaNum 32 | b64enc }}
{{- end }}
```

**7. Debugging a running release**
```bash
# Print all info about a deployed release
helm get all my-app -n production

# Show what values were used for the last deploy
helm get values my-app -n production --all

# See the actual rendered manifests that were applied
helm get manifest my-app -n production

# Compare what's deployed vs what would be deployed (requires helm-diff plugin)
helm diff upgrade my-app ./mychart --values ./prod.yaml
```

---

## 13. Quick Reference Tables

### All Major CLI Commands

| Command | Description | Example |
|---------|-------------|---------|
| `helm install` | Install a chart as a new release | `helm install my-app ./mychart -n prod` |
| `helm upgrade` | Upgrade an existing release | `helm upgrade my-app ./mychart --set tag=v2` |
| `helm upgrade --install` | Install or upgrade (idempotent) | `helm upgrade --install my-app ./mychart` |
| `helm uninstall` | Delete a release | `helm uninstall my-app -n prod` |
| `helm list` | List releases | `helm list -A` |
| `helm status` | Show release status | `helm status my-app -n prod` |
| `helm history` | Show revision history | `helm history my-app` |
| `helm rollback` | Roll back to previous revision | `helm rollback my-app 1` |
| `helm get values` | Show release values | `helm get values my-app --all` |
| `helm get manifest` | Show rendered manifests | `helm get manifest my-app` |
| `helm get notes` | Show NOTES.txt output | `helm get notes my-app` |
| `helm template` | Render templates locally | `helm template my-app ./mychart` |
| `helm lint` | Lint a chart | `helm lint ./mychart --strict` |
| `helm package` | Package chart into .tgz | `helm package ./mychart` |
| `helm push` | Push chart to OCI registry | `helm push mychart-1.0.tgz oci://ghcr.io/org` |
| `helm pull` | Download chart from repo | `helm pull bitnami/nginx --untar` |
| `helm repo add` | Add a chart repository | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| `helm repo update` | Update local repo cache | `helm repo update` |
| `helm repo list` | List configured repos | `helm repo list` |
| `helm repo remove` | Remove a repo | `helm repo remove bitnami` |
| `helm search repo` | Search local repos | `helm search repo nginx --versions` |
| `helm search hub` | Search Artifact Hub | `helm search hub prometheus` |
| `helm show chart` | Show Chart.yaml | `helm show chart bitnami/nginx` |
| `helm show values` | Show default values | `helm show values bitnami/nginx` |
| `helm dependency update` | Download subchart deps | `helm dependency update ./mychart` |
| `helm dependency build` | Build from Chart.lock | `helm dependency build ./mychart` |
| `helm test` | Run chart tests | `helm test my-app --logs` |
| `helm create` | Scaffold a new chart | `helm create mychart` |
| `helm env` | Show Helm environment | `helm env` |
| `helm version` | Show Helm version | `helm version` |
| `helm plugin install` | Install a plugin | `helm plugin install https://github.com/databus23/helm-diff` |
| `helm registry login` | Log into OCI registry | `helm registry login ghcr.io -u user` |

---

### Template Built-in Objects

| Object | Field | Type | Description |
|--------|-------|------|-------------|
| `.Values` | _(dynamic)_ | map | All merged values from values.yaml + overrides |
| `.Release` | `.Name` | string | Release name |
| `.Release` | `.Namespace` | string | Target namespace |
| `.Release` | `.IsInstall` | bool | True on first install |
| `.Release` | `.IsUpgrade` | bool | True on upgrade |
| `.Release` | `.Revision` | int | Release revision number |
| `.Release` | `.Service` | string | Always `"Helm"` |
| `.Chart` | `.Name` | string | Chart name |
| `.Chart` | `.Version` | string | Chart version |
| `.Chart` | `.AppVersion` | string | App version |
| `.Chart` | `.Description` | string | Chart description |
| `.Capabilities` | `.KubeVersion.Version` | string | Kubernetes version string |
| `.Capabilities` | `.KubeVersion.Major` | string | K8s major version |
| `.Capabilities` | `.KubeVersion.Minor` | string | K8s minor version |
| `.Capabilities` | `.APIVersions.Has "..."` | bool | Check if API version exists |
| `.Files` | `.Get "path"` | string | Get file contents as string |
| `.Files` | `.Glob "pattern"` | map | Get files matching glob |
| `.Files` | `.AsConfig` | string | Render files as ConfigMap data |
| `.Files` | `.AsSecrets` | string | Render files as Secret data (b64) |
| `.Template` | `.Name` | string | Current template file path |
| `.Template` | `.BasePath` | string | Chart's templates directory path |

---

### Hook Types Summary

| Hook | Runs | Typical Use Case |
|------|------|-----------------|
| `pre-install` | Before first install resources applied | DB schema creation |
| `post-install` | After first install succeeds | Send webhook / notification |
| `pre-upgrade` | Before upgrade resources applied | DB migrations, backup |
| `post-upgrade` | After upgrade succeeds | Cache invalidation |
| `pre-rollback` | Before rollback applied | Backup current state |
| `post-rollback` | After rollback succeeds | Notification / audit |
| `pre-delete` | Before uninstall | Export data, create backup |
| `post-delete` | After uninstall completes | Cleanup external resources |
| `test` | Only on `helm test` | Connectivity / smoke tests |

---

### Commonly Used Sprig Functions

| Function | Example | Output |
|----------|---------|--------|
| `default` | `"" \| default "latest"` | `latest` |
| `required` | `required "msg" .Values.x` | error if `.Values.x` is empty |
| `quote` | `"hello" \| quote` | `"hello"` |
| `squote` | `"hello" \| squote` | `'hello'` |
| `upper` | `"hello" \| upper` | `HELLO` |
| `lower` | `"HELLO" \| lower` | `hello` |
| `title` | `"hello world" \| title` | `Hello World` |
| `trim` | `"  hi  " \| trim` | `hi` |
| `trimSuffix` | `"app-" \| trimSuffix "-"` | `app` |
| `trunc` | `"alongname" \| trunc 5` | `along` |
| `replace` | `"a b" \| replace " " "-"` | `a-b` |
| `contains` | `"hello" \| contains "ell"` | `true` |
| `hasPrefix` | `"hello" \| hasPrefix "hel"` | `true` |
| `hasSuffix` | `"hello" \| hasSuffix "llo"` | `true` |
| `printf` | `printf "%s-%s" "a" "b"` | `a-b` |
| `toYaml` | `.Values.map \| toYaml` | YAML string |
| `fromYaml` | `"key: val" \| fromYaml` | map object |
| `toJson` | `.Values.map \| toJson` | JSON string |
| `fromJson` | `"{}" \| fromJson` | map object |
| `indent` | `"text" \| indent 4` | `    text` |
| `nindent` | `"text" \| nindent 4` | `\n    text` |
| `b64enc` | `"secret" \| b64enc` | `c2VjcmV0` |
| `b64dec` | `"c2VjcmV0" \| b64dec` | `secret` |
| `sha256sum` | `"data" \| sha256sum` | SHA-256 hex |
| `randAlphaNum` | `randAlphaNum 16` | random 16-char string |
| `uuidv4` | `uuidv4` | random UUID |
| `now` | `now \| date "2006-01-02"` | `2024-03-15` |
| `int` | `"42" \| int` | `42` |
| `toString` | `42 \| toString` | `"42"` |
| `list` | `list "a" "b" "c"` | `[a b c]` |
| `dict` | `dict "k" "v"` | `{k: v}` |
| `merge` | `merge $a $b` | merged map |
| `keys` | `.Values.map \| keys` | list of keys |
| `sortAlpha` | `list "b" "a" \| sortAlpha` | `[a b]` |
| `join` | `list "a" "b" \| join ","` | `a,b` |
| `split` | `"a,b" \| split ","` | map with `_0`,`_1` |
| `splitList` | `"a,b" \| splitList ","` | `[a b]` |
| `has` | `list "a" "b" \| has "a"` | `true` |
| `uniq` | `list "a" "a" "b" \| uniq` | `[a b]` |
| `add` | `add 1 2` | `3` |
| `sub` | `sub 5 3` | `2` |
| `mul` | `mul 2 4` | `8` |
| `div` | `div 10 2` | `5` |
| `mod` | `mod 7 3` | `1` |
| `max` | `max 1 5 3` | `5` |
| `min` | `min 1 5 3` | `1` |

---

*Generated with ❤️ — Last updated for Helm 3.14+*
*Official docs: [helm.sh/docs](https://helm.sh/docs) | Sprig functions: [masterminds.github.io/sprig](http://masterminds.github.io/sprig)*
