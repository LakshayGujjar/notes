# GCP Developer Tooling — Production Cheatsheet
### Cloud Shell · Cloud Code · Deployment Manager

---

## PART 1 — CLOUD SHELL

Cloud Shell is a browser-based, fully managed Linux environment that gives every GCP user a persistent, pre-authenticated development machine with zero local setup. It runs as an ephemeral Debian VM with a 5 GB persistent home directory, a pre-installed toolchain covering everything from `gcloud` to `terraform`, and an optional web-based IDE — making it ideal for quick admin tasks, scripting prototypes, and working from any device.

---

### 1.1 OVERVIEW & ARCHITECTURE

```text
┌───────────────────────────────────────────────────────────────────────────┐
│                          USER BROWSER                                     │
│                                                                           │
│  ┌─────────────────────────┐    ┌──────────────────────────────────────┐ │
│  │  Cloud Shell Terminal   │    │  Cloud Shell Editor (Theia/OpenVSX)  │ │
│  │  (browser-based SSH)    │    │  Web-based VS Code equivalent        │ │
│  └────────────┬────────────┘    └──────────────────┬───────────────────┘ │
└───────────────┼──────────────────────────────────────┼────────────────────┘
                │ HTTPS (WebSocket)                    │ HTTPS
                └────────────────────┬─────────────────┘
                                     │
┌────────────────────────────────────▼──────────────────────────────────────┐
│               CLOUD SHELL INSTANCE (Ephemeral Debian VM)                  │
│               e2-small equivalent (0.2 vCPU / 1.7 GB RAM)                │
│               Boost Mode: e2-standard-2 (2 vCPU / 8 GB RAM)              │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  Pre-installed Toolchain                                            │ │
│  │  gcloud · kubectl · docker · terraform · helm · git · python3      │ │
│  │  node · go · java · vim · nano · tmux · screen · jq · curl        │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌──────────────────────────┐   ┌─────────────────────────────────────┐ │
│  │  /home/$USER  (5 GB)     │   │  Web Preview                        │ │
│  │  ✅ Persists across      │   │  Port → https://PORT-PROJ.          │ │
│  │  sessions (persistent    │   │        cloudshell.dev/              │ │
│  │  disk volume)            │   │  OAuth-gated; HTTPS only            │ │
│  └──────────────────────────┘   └─────────────────────────────────────┘ │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │  /tmp, /usr, /opt, apt cache — ❌ Ephemeral (reset on VM recycle)  ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                           │
│  Pre-authenticated as logged-in Google identity                           │
│  ADC (Application Default Credentials) auto-available                    │
└───────────────────────────────────────────────────────────────────────────┘
                                     │
                          GCP APIs / Cloud Console
```

#### Session Lifecycle

| Event | Behaviour |
|---|---|
| Idle for 20 minutes | Session terminated; VM may be recycled |
| Browser tab closed | Terminal disconnects; tmux sessions survive until VM recycle |
| VM recycled | All ephemeral paths reset; `/home/$USER` preserved |
| New session started | May get a fresh VM; home dir always remounted |
| Max session duration | 12 hours hard limit per session |
| No activity for 120 days | Persistent home directory **wiped** |
| Boost Mode | Upgraded VM (e2-standard-2); limited to 24 hours/week total |

> ⚠️ **NOTE:** tmux sessions do **not** survive Cloud Shell VM recycling — they survive browser disconnections within the same VM session only. If the idle timeout fires and the VM is recycled, tmux sessions are lost. Use Cloud Shell for long-running processes only within an active session window, or offload to Cloud Run Jobs / GCE for truly persistent workloads.

---

### 1.2 ENVIRONMENT & PRE-INSTALLED TOOLING

#### Pre-installed Tool Reference

| Tool | Purpose | Upgrade / Install Command |
|---|---|---|
| `gcloud` CLI | GCP API access, resource management | `gcloud components update` |
| `kubectl` | Kubernetes cluster management | `gcloud components install kubectl` |
| `docker` | Container build, push, run | Pre-installed; daemon running; no restart needed |
| `terraform` | HashiCorp IaC | `tfenv install latest` or manual binary install |
| `helm` | Kubernetes package manager | Pre-installed; `helm repo update` to refresh |
| `git` | Version control | Pre-installed; configure identity on first use |
| `python3` / `pip3` | Python scripting and data work | Pre-installed; `pip3 install --user PACKAGE` |
| `node` / `npm` | Node.js development | `nvm install --lts` for latest LTS |
| `java` / `mvn` | Java and Maven builds | `sdk install java` via pre-installed SDKMAN |
| `go` | Go development | Pre-installed; `go install` for additional tools |
| `vim` / `nano` / `emacs` | Terminal text editors | Pre-installed |
| `tmux` / `screen` | Terminal multiplexing for long ops | Pre-installed |
| `jq` | JSON processing in scripts | Pre-installed |
| `curl` / `wget` | HTTP client utilities | Pre-installed |

#### Persistent vs Ephemeral Storage

| Location | Persists Across Sessions | Size | Notes |
|---|---|---|---|
| `/home/$USER` | ✅ Yes | 5 GB | Persistent volume; the **only** path that survives VM recycling |
| `/tmp` | ❌ No | RAM-limited | Cleared on every VM recycle |
| `/usr`, `/opt`, `/var` | ❌ No | Ephemeral | All `apt install` packages reset |
| Docker image cache | ❌ No | Ephemeral | Images must be pulled again each session |
| `apt` packages | ❌ No | Ephemeral | Use `.customize_environment` to reinstall automatically |
| Environment variables set in terminal | ❌ No | — | Add to `~/.bashrc` or `~/.profile` to persist |

> 💡 **TIP:** Store scripts, configs, and credentials (references, not values) in `/home/$USER`. Put secrets in Secret Manager and reference them by name. For files > 5 GB, use a GCS bucket mounted with `gcsfuse` or transfer with `gcloud storage`.

#### `.customize_environment` — Auto-run on VM Recycle

```bash
#!/bin/bash
# ~/.customize_environment
# This script runs automatically each time Cloud Shell provisions a new VM.
# Use it to restore ephemeral state: apt packages, binary tools, env settings.
# It runs as your user via sudo; output logged to ~/.customize_environment.log

# ── System packages ────────────────────────────────────────────────────────
sudo apt-get update -qq
sudo apt-get install -y -qq \
  jq \
  httpie \
  fzf \
  ripgrep \
  tree \
  netcat-openbsd

# ── Pin a specific Terraform version ───────────────────────────────────────
TERRAFORM_VERSION="1.8.5"
if ! terraform version 2>/dev/null | grep -q "${TERRAFORM_VERSION}"; then
  wget -q "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip" \
    -O /tmp/terraform.zip
  unzip -q /tmp/terraform.zip -d /tmp/terraform-bin
  sudo mv /tmp/terraform-bin/terraform /usr/local/bin/terraform
  rm -rf /tmp/terraform.zip /tmp/terraform-bin
fi

# ── Shell aliases and environment ─────────────────────────────────────────
echo 'alias k=kubectl' >> ~/.bashrc
echo 'alias tf=terraform' >> ~/.bashrc
echo 'export EDITOR=vim' >> ~/.bashrc
echo 'export KUBE_EDITOR=vim' >> ~/.bashrc

# ── Configure git identity (only if not already set) ──────────────────────
git config --global user.email "you@example.com" 2>/dev/null || true
git config --global user.name  "Your Name"        2>/dev/null || true
```

---

### 1.3 GCLOUD AUTHENTICATION & CONFIGURATION

Cloud Shell is automatically authenticated as the Google identity that opened it — no `gcloud auth login` is required. Application Default Credentials (ADC) are also pre-configured, so any GCP client library or tool that uses ADC (Terraform, Python GCP libraries, etc.) inherits the Cloud Shell identity without additional configuration.

```bash
# ── View current auth and configuration state ─────────────────────────────
gcloud config list                       # Show all active config values
gcloud auth list                         # Show authenticated accounts
gcloud auth print-access-token           # Print current OAuth2 token (for curl testing)

# ── Set default project and region (avoids --project flag on every command) ──
gcloud config set project MY_PROJECT_ID
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# ── Named configurations for multi-project workflows ──────────────────────
gcloud config configurations create prod-config
gcloud config set project my-prod-project
gcloud config set compute/region us-east1

gcloud config configurations create dev-config
gcloud config set project my-dev-project
gcloud config set compute/region us-central1

gcloud config configurations list                       # List all configs
gcloud config configurations activate prod-config        # Switch to prod
gcloud config configurations describe prod-config        # View prod config values

# ── Override for a single command without switching config ─────────────────
gcloud compute instances list --project=other-project
gcloud storage ls --project=other-project gs://

# ── Service account impersonation (test restricted IAM policies) ───────────
gcloud config set auth/impersonate_service_account \
  restricted-sa@my-project.iam.gserviceaccount.com

# Run commands as the SA to verify its permissions:
gcloud storage ls gs://restricted-bucket

# Always unset after testing — Cloud Shell ops run as the SA while set
gcloud config unset auth/impersonate_service_account

# ── ADC for local application development ─────────────────────────────────
# Cloud Shell: ADC is auto-set to your Google identity
# Local machine: run this to set ADC to your personal credentials
gcloud auth application-default login

# Check which ADC credentials are active
gcloud auth application-default print-access-token
cat ~/.config/gcloud/application_default_credentials.json | jq .client_id
```

> ⚠️ **NOTE:** When you set `auth/impersonate_service_account`, **all subsequent gcloud commands** in that configuration run as the service account — including commands that modify resources. Always verify the impersonation is unset before continuing normal operations: `gcloud config get auth/impersonate_service_account`.

---

### 1.4 WEB PREVIEW & PORT FORWARDING

Web Preview proxies any port on the Cloud Shell VM through Google's infrastructure, exposing it as an HTTPS URL accessible only to the logged-in Google user. It eliminates the need to deploy an application to test its HTTP interface — just run it locally on Cloud Shell and preview in the browser.

```bash
# ── Basic HTTP server for static file preview ─────────────────────────────
python3 -m http.server 8080
# Then click "Web Preview" → "Preview on port 8080" in the toolbar

# ── Flask development server ──────────────────────────────────────────────
export FLASK_APP=app.py
export FLASK_ENV=development
flask run --host=0.0.0.0 --port=8080

# ── Node.js / Express server ──────────────────────────────────────────────
node server.js                          # Assumes app listens on 0.0.0.0:8080

# ── Get the Web Preview URL programmatically ──────────────────────────────
cloudshell get-web-preview-url -p 8080

# ── Use kubectl proxy to access Kubernetes services via Web Preview ────────
kubectl proxy --port=8080 &
# Access: https://8080-PROJECT.cloudshell.dev/api/v1/namespaces/default/services/
```

#### Port Reference

| Port | Common Use | Notes |
|---|---|---|
| `8080` | Default preview port | "Web Preview" button defaults here |
| `3000` | Node.js / React dev server | Select via "Change port" in Web Preview |
| `5000` | Flask / Python | Select via "Change port" |
| `4200` | Angular CLI dev server | Select via "Change port" |
| `8888` | Jupyter Notebook | Select via "Change port" |
| `1024–65535` | Any custom port | Selectable in Web Preview dropdown |

> ⚠️ **NOTE:** Web Preview is **OAuth-gated** — only the authenticated Google user who opened Cloud Shell can access the preview URL. It cannot be shared with others. For shared previews, deploy to Cloud Run or App Engine instead.

> 💡 **TIP:** Only **one port** can be previewed at a time in the browser, but you can run multiple servers simultaneously. Switch the active preview port via "Web Preview" → "Change port" without stopping the servers.

---

### 1.5 CLOUD SHELL EDITOR (THEIA / VS CODE IN BROWSER)

Cloud Shell Editor is a web-based IDE built on Theia (the open-source VS Code foundation) that runs on the same Cloud Shell VM as the terminal. It shares the filesystem, credentials, and network — so files edited in the IDE appear immediately in the terminal and vice versa.

```bash
# ── Open Cloud Shell Editor ───────────────────────────────────────────────
cloudshell edit myfile.py                # Open a specific file in the editor
cloudshell open-workspace .              # Open current directory as workspace
cloudshell edit ~/.customize_environment # Quick edit for the customize script
cloudshell edit ~/.bashrc                # Edit shell profile

# ── Open editor from a Git repository URL (shareable "Open in Cloud Shell") ─
# Share this URL to let anyone clone a repo and open it in Cloud Shell Editor:
# https://shell.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/org/repo&cloudshell_open_in_editor=README.md
```

#### Cloud Shell Editor vs Local VS Code

| Aspect | Cloud Shell Editor | Local VS Code |
|---|---|---|
| Extension source | OpenVSX registry (open-source) | VS Code Marketplace (Microsoft) |
| Microsoft extensions | ❌ Not available (Copilot, LiveShare) | ✅ Available |
| Authentication | Auto-inherited from Cloud Shell | Requires `gcloud auth` setup |
| File access | Cloud Shell VM filesystem only | Local + remote via SSH extension |
| Performance | Tied to Cloud Shell VM (e2-small) | Local machine resources |
| Cost | Free (included in Cloud Shell) | Free (local) |
| Best for | Quick edits, GCP-native workflows | Full-time development |

> 💡 **TIP:** Install the **Cloud Code** extension in Cloud Shell Editor (it's available on OpenVSX) to get Kubernetes Explorer, Cloud Run deployment, and Secret Manager browsing directly in the browser-based IDE — no local tooling required.

---

### 1.6 TMUX & PRODUCTIVITY IN CLOUD SHELL

tmux (terminal multiplexer) is pre-installed in Cloud Shell and essential for any operation that takes more than a few minutes. It keeps processes running through browser disconnections and the 20-minute idle timeout, and lets you manage multiple terminal panes in a single Cloud Shell session.

```bash
# ── Session management ────────────────────────────────────────────────────
tmux new-session -s main                 # Start a named session
tmux attach-session -t main              # Reattach after browser reconnect
tmux list-sessions                       # List all active sessions
tmux kill-session -t main               # Terminate a session
tmux new-session -d -s background        # Start a detached (background) session

# Detach from current session (session stays alive):
# Keyboard: Ctrl+B, then D

# ── Window and pane management ────────────────────────────────────────────
# Create a new window (tab) in the session:
# Keyboard: Ctrl+B, then C

# Split pane horizontally (side by side):
# Keyboard: Ctrl+B, then %

# Split pane vertically (top and bottom):
# Keyboard: Ctrl+B, then " (double-quote)

# Navigate between panes:
# Keyboard: Ctrl+B, then arrow keys

# Resize panes:
# Keyboard: Ctrl+B, then Ctrl+arrow keys (hold Ctrl)

# ── Running long operations safely ────────────────────────────────────────
# Start a background session and run a long command (won't be killed by idle timeout)
tmux new-session -d -s tf-deploy
tmux send-keys -t tf-deploy \
  'cd ~/infra && terraform apply -auto-approve 2>&1 | tee /tmp/tf.log' \
  Enter

# Reattach to monitor progress
tmux attach-session -t tf-deploy

# Check output of detached session without attaching
tmux capture-pane -pt tf-deploy

# ── Practical patterns for Cloud Shell ────────────────────────────────────
# Pane 1: editor (vim or watch file changes)
# Pane 2: apply commands / tail logs
# Pane 3: kubectl get pods -w (watch cluster state)

# Start a 3-pane layout:
tmux new-session -s workspace \; \
  split-window -h \; \
  split-window -v \; \
  select-pane -t 0
```

> 💡 **TIP:** Add `tmux attach-session -t main 2>/dev/null || tmux new-session -s main` to your `~/.bashrc` to automatically rejoin (or create) a named tmux session every time you open Cloud Shell. This simulates a persistent terminal experience.

---

### 1.7 CLOUD SHELL AS A SECURE BASTION / ADMIN CONSOLE

Cloud Shell can serve as a zero-setup, always-patched admin workstation for accessing private GCE VMs and GKE clusters through Identity-Aware Proxy (IAP) tunneling — no jump box, no VPN, no external IPs required on target resources.

```bash
# ── SSH to a private GCE VM (no external IP, no VPN needed) ──────────────
gcloud compute ssh my-private-vm \
  --zone=us-central1-a \
  --tunnel-through-iap
# IAP opens a WebSocket tunnel through Google's network; no port 22 exposure needed

# ── SCP file to/from a private VM via IAP ─────────────────────────────────
gcloud compute scp ./config.yaml my-private-vm:~/app/config.yaml \
  --zone=us-central1-a \
  --tunnel-through-iap

gcloud compute scp my-private-vm:~/logs/app.log ./downloaded-app.log \
  --zone=us-central1-a \
  --tunnel-through-iap

# ── SSH tunnel to a port on a private VM (e.g., database on port 5432) ────
gcloud compute ssh my-db-vm \
  --zone=us-central1-a \
  --tunnel-through-iap \
  -- -L 5432:localhost:5432 -N -f
# Now connect locally: psql -h localhost -p 5432 -U myuser

# ── Connect to a private GKE cluster ──────────────────────────────────────
gcloud container clusters get-credentials my-private-cluster \
  --zone=us-central1-a \
  --project=MY_PROJECT
# gcloud automatically uses IAP for private clusters when public endpoint is disabled

kubectl get nodes
kubectl get pods --all-namespaces

# ── Access Kubernetes Dashboard via Web Preview ────────────────────────────
kubectl proxy --port=8080 &
# Web Preview on port 8080 → access:
# /api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# ── Run GKE exec plugin commands ──────────────────────────────────────────
kubectl exec -it $(kubectl get pod -l app=my-app -o jsonpath='{.items[0].metadata.name}') \
  -- /bin/bash
```

#### Cloud Shell as Admin Workstation vs Traditional Jump Box

| Aspect | Cloud Shell | Persistent Jump Box (GCE VM) |
|---|---|---|
| Patching | Google-managed; always current | Manual OS patching required |
| Authentication | Google Identity (SSO) | SSH keys or passwords |
| Audit trail | Cloud Audit Logs (who accessed, when) | OS-level logs |
| Cost | Free | VM cost ($10–50+/month) |
| Availability | Browser-based; any device | SSH client required |
| Credential management | Auto-authenticated via IAP | Manual key rotation |

---

### 1.8 AUTOMATION, SCRIPTING & CLOUD SHELL API

Cloud Shell can be launched with specific startup commands, git repositories, and tutorial walkthroughs via URL parameters — enabling "Open in Cloud Shell" buttons in documentation, READMEs, and onboarding workflows.

```bash
# ── cloudshell CLI utility commands ───────────────────────────────────────

# Launch an interactive tutorial (Markdown-based walkthrough in Cloud Shell)
cloudshell launch-tutorial ~/tutorials/onboarding.md

# Get the current Web Preview URL for a port
cloudshell get-web-preview-url -p 8080

# Open a file in Cloud Shell Editor
cloudshell edit path/to/file.yaml

# Open a workspace directory in the editor
cloudshell open-workspace ~/my-project
```

#### Open-in-Cloud-Shell URL Parameters

| Parameter | Purpose | Example |
|---|---|---|
| `cloudshell_git_repo` | Clone a Git repo on launch | `cloudshell_git_repo=https://github.com/org/repo` |
| `cloudshell_open_in_editor` | Open a file in the editor | `cloudshell_open_in_editor=README.md` |
| `cloudshell_tutorial` | Launch a tutorial on open | `cloudshell_tutorial=tutorial.md` |
| `cloudshell_working_dir` | Set the working directory | `cloudshell_working_dir=examples/basic` |
| `cloudshell_image` | Use a custom Cloud Shell image | `cloudshell_image=gcr.io/myproject/custom-shell` |

```
# Full "Open in Cloud Shell" URL example:
https://shell.cloud.google.com/cloudshell/open?
  git_repo=https://github.com/GoogleCloudPlatform/cloud-shell-tutorials
  &cloudshell_tutorial=tutorials/storage/tutorial.md
  &cloudshell_open_in_editor=src/main.py
```

#### Cloud Shell Tutorials (Markdown Walkthrough Format)

```markdown
<!-- tutorial.md — Cloud Shell interactive tutorial -->

# Deploy Your First Cloud Run Service

<walkthrough-project-setup billing="true"></walkthrough-project-setup>

## Step 1: Enable APIs

Click the Cloud Shell button to enable required APIs:
<walkthrough-enable-apis apis="run.googleapis.com,cloudbuild.googleapis.com">
</walkthrough-enable-apis>

## Step 2: Open the service file

<walkthrough-editor-open-file filePath="src/main.py">
</walkthrough-editor-open-file>

## Step 3: Deploy

Run the following command:
```bash
gcloud run deploy my-service --source . --region us-central1
```
```

> 💡 **TIP:** Cloud Shell Tutorials are an excellent format for team runbooks — store the `.md` file in a Git repo and share an "Open in Cloud Shell" URL that launches the tutorial automatically. Anyone on the team can follow the runbook steps interactively without any local setup.

---

### 1.9 LIMITS, QUOTAS & BEST PRACTICES

#### Limits Reference

| Parameter | Value | Notes |
|---|---|---|
| Persistent home directory | 5 GB | Cannot be increased; use GCS FUSE for overflow |
| Idle timeout | 20 minutes | Session terminated; VM may be recycled |
| Maximum session duration | 12 hours | Hard limit; start a new session after |
| Boost Mode weekly cap | 24 hours per week | Resets weekly; e2-standard-2 class VM |
| Concurrent sessions | 2 per Google account | Additional sessions reuse existing VMs |
| Web Preview ports active | 1 at a time | Switchable without stopping servers |
| Custom Cloud Shell images | Supported | Host on GCR/Artifact Registry; set via URL param |
| Disk wipe on inactivity | After 120 days | Triggered by no Cloud Shell usage across all sessions |
| Cost | Free | No charge for Cloud Shell; GCP resources you create are billed |

#### Best Practices

- **Never store secrets as plaintext** in `/home/$USER` — use Secret Manager and reference by name in scripts
- **Use named `gcloud` configurations** (`gcloud config configurations`) for multi-project workflows instead of switching `--project` per command
- **Use `tmux` for any operation > 5 minutes** — browser disconnections won't kill the process
- **Commit scripts to Git** — Cloud Shell is not long-term storage; treat home dir as ephemeral beyond configuration dotfiles
- **Pin tool versions in `.customize_environment`** to ensure consistent behavior across VM recycles
- **Use Boost Mode sparingly** — it has a weekly hour cap; reserve it for heavy builds, large data processing, or Terraform runs against large state
- **For files > 5 GB**, stream directly to/from GCS (`gcloud storage cp`) rather than staging in Cloud Shell


---

## PART 2 — CLOUD CODE

Cloud Code is a suite of IDE extensions for VS Code and JetBrains IDEs that embeds GCP-native development workflows — Kubernetes deployment, Cloud Run service management, Secret Manager browsing, Cloud SQL connectivity, and real-time log streaming — directly into the developer's editor. It wraps **Skaffold** as its build-and-deploy engine, enabling a continuous inner development loop where file saves trigger automatic rebuilds and redeployments to local or remote Kubernetes clusters.

---

### 2.1 OVERVIEW & ARCHITECTURE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEVELOPER IDE (VS Code / IntelliJ)                       │
│                                                                             │
│  ┌──────────────┐ ┌───────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │  Kubernetes  │ │  Cloud Run    │ │   Secret     │ │  Cloud SQL       │ │
│  │  Explorer    │ │  Explorer     │ │   Manager    │ │  Explorer        │ │
│  └──────┬───────┘ └───────┬───────┘ └──────┬───────┘ └────────┬─────────┘ │
│         │                 │                │                  │            │
│  ┌──────▼─────────────────▼────────────────▼──────────────────▼──────────┐ │
│  │              Cloud Code Plugin Core                                   │ │
│  │   Skaffold integration · API Explorer · Log Viewer · Monitoring       │ │
│  └─────────────────────────────────┬───────────────────────────────────┘ │
└───────────────────────────────────┼─────────────────────────────────────-─┘
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │                         │                          │
┌─────────▼───────────┐   ┌─────────▼──────────┐   ┌──────────▼──────────┐
│  LOCAL KUBERNETES   │   │  REMOTE GKE        │   │  CLOUD RUN          │
│  minikube / kind    │   │  Private or public │   │  Fully managed      │
│  Docker Desktop     │   │  cluster           │   │  serverless         │
└─────────┬───────────┘   └──────────┬─────────┘   └──────────┬──────────┘
          │                          │                         │
          └──────────────────────────┼─────────────────────────┘
                                     │
                         ┌───────────▼────────────┐
                         │  Artifact Registry /   │
                         │  Container Registry    │
                         │  (image destination)   │
                         └────────────────────────┘

Supporting Services (sidebar integrations):
  Secret Manager API ──► ${secret:...} resolution in launch configs
  Cloud SQL Auth Proxy ──► automatic proxy start on "Connect" click
  Cloud Logging API ──► real-time log streaming in Output panel
  Cloud Monitoring ──► dashboard links from Explorer context menu
  Discovery API ──► API Explorer method browsing + code generation
```

#### Cloud Code Components

| Component | Function | Backed By |
|---|---|---|
| Kubernetes Explorer | Browse, describe, delete cluster resources; exec into pods | `kubectl` + kubeconfig |
| Cloud Run Explorer | List services, view revisions/traffic splits, deploy, stream logs | `gcloud run` |
| Secret Manager | Browse secrets and versions; inject into launch configs | Secret Manager API |
| Cloud SQL Explorer | Browse instances/databases; open integrated SQL editor | Cloud SQL Auth Proxy |
| API Explorer | Browse 200+ GCP APIs; run live requests; generate code | GCP Discovery API |
| Log Viewer | Stream Cloud Logging for GKE pods, Cloud Run, Cloud Functions | Cloud Logging API |
| Skaffold integration | Continuous build+deploy on file save; debug attach | Skaffold |

---

### 2.2 INSTALLATION & SETUP

#### VS Code

```bash
# Install via VS Code CLI (Extension ID: GoogleCloudTools.cloudcode)
code --install-extension GoogleCloudTools.cloudcode

# Or: VS Code → Extensions panel → search "Cloud Code" → Install

# Post-install: authenticate (opens browser OAuth flow)
# Status bar → "Cloud Code" → "Sign In"
# Or: Cmd/Ctrl+Shift+P → "Cloud Code: Sign In"

# Ensure gcloud is installed and ADC is configured for local development
gcloud version
gcloud auth application-default login        # Sets ADC to your Google identity
gcloud config set project MY_PROJECT_ID

# Verify Cloud Code can reach your GKE cluster
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a \
  --project=MY_PROJECT_ID
kubectl get nodes                             # Should list cluster nodes
```

#### IntelliJ / JetBrains IDEs

```
Settings (Cmd+,) → Plugins → Marketplace → search "Cloud Code" → Install
Supported IDEs: IntelliJ IDEA, GoLand, PyCharm, WebStorm, Rider, CLion
Post-install: Tools → Google Cloud Code → Sign In
```

#### Local Kubernetes Options

| Option | Install | Best For | RAM Usage |
|---|---|---|---|
| minikube | `brew install minikube` / winget | Full single-node K8s locally | ~4 GB |
| Docker Desktop K8s | Docker Desktop → Settings → Kubernetes | macOS/Windows integrated | ~4 GB |
| kind | `go install sigs.k8s.io/kind@latest` | CI/CD testing, lightweight | ~1 GB |
| Cloud Shell + GKE | None (Cloud Shell pre-configured) | Zero local setup | 0 local |

> 💡 **TIP:** For teams without powerful laptops, skip local Kubernetes entirely — connect Cloud Code directly to a shared dev GKE cluster. Use separate namespaces per developer to avoid conflicts. This keeps local machine requirements minimal and ensures parity with production networking.

---

### 2.3 KUBERNETES DEVELOPMENT WORKFLOW

The inner dev loop with Cloud Code: **save a file → Skaffold detects the change → rebuilds the container image → pushes to registry (or loads to minikube) → applies updated manifests → streams logs back to IDE Output panel** — all without leaving the editor.

#### skaffold.yaml — The Core Configuration

```yaml
# skaffold.yaml — committed to version control; shared between Cloud Code and CI/CD
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: my-app

build:
  artifacts:
    - image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app
      docker:
        dockerfile: Dockerfile
        buildArgs:
          APP_VERSION: "{{.VERSION}}"
  local:
    push: false          # false = load directly into minikube (faster for local dev)
    useBuildkit: true    # Enable BuildKit for faster, cached builds

deploy:
  kubectl:
    manifests:
      - k8s/base/deployment.yaml
      - k8s/base/service.yaml
      - k8s/base/configmap.yaml

portForward:
  - resourceType: service
    resourceName: my-app-service
    namespace: default
    port: 80
    localPort: 8080      # Access app at localhost:8080 during dev loop

# Environment-specific profiles (see section 2.9)
profiles:
  - name: local
    build:
      local:
        push: false
  - name: staging
    build:
      googleCloudBuild:
        projectId: MY_PROJECT
    deploy:
      kubectl:
        manifests: ["k8s/overlays/staging/**"]
```

#### Starting a Development Session

```bash
# Via Command Palette (Cmd/Ctrl+Shift+P):
# → "Cloud Code: Run on Kubernetes"   (select cluster; starts skaffold dev)
# → "Cloud Code: Debug on Kubernetes" (starts skaffold debug; attaches IDE debugger)

# Equivalent CLI commands (what Cloud Code runs under the hood):
skaffold dev --port-forward --tail          # Continuous dev loop + log streaming
skaffold debug --port-forward              # Dev loop with debugger port exposed

# Cloud Code handles debugger attachment automatically:
# - Go: uses Delve (dlv)
# - Java: uses JDWP (Java Debug Wire Protocol)
# - Python: uses debugpy
# - Node.js: uses --inspect flag
# No manual debugger configuration required — Cloud Code injects the agent
```

#### Kubernetes Explorer Sidebar Operations

| Operation | How | Equivalent CLI |
|---|---|---|
| Browse all resources | Expand cluster tree | `kubectl get all -A` |
| Stream pod logs | Click pod → "View Logs" | `kubectl logs -f POD -n NS` |
| Exec into pod | Right-click pod → "Get Terminal" | `kubectl exec -it POD -- /bin/bash` |
| Apply a manifest | Right-click file → "Apply to Cluster" | `kubectl apply -f FILE` |
| Delete a resource | Right-click resource → "Delete" | `kubectl delete TYPE NAME` |
| Port-forward to pod | Right-click pod → "Port Forward" | `kubectl port-forward POD LOCAL:REMOTE` |
| View resource YAML | Right-click → "Describe" | `kubectl describe TYPE NAME` |

> 💡 **TIP:** Cloud Code's YAML editor provides **IntelliSense and schema validation** for Kubernetes manifests — hover over any field for documentation, and incorrect values are underlined in red before you apply. Type `k8sdeploy`, `k8ssvc`, or `k8sconfigmap` and press Tab to expand full boilerplate snippets.

---

### 2.4 CLOUD RUN DEVELOPMENT WORKFLOW

Cloud Code integrates Cloud Run deployment directly into the IDE, handling Dockerfile detection, image building via Cloud Build or local Docker, and `gcloud run deploy` — exposed as a guided wizard rather than CLI flags.

#### Deploy to Cloud Run from VS Code

```bash
# Command Palette → "Cloud Code: Deploy to Cloud Run"
# Wizard prompts: project, region, service name, allow unauthenticated (yes/no)
# Cloud Code generates and manages skaffold.yaml automatically
```

```yaml
# Generated skaffold.yaml for Cloud Run deployment
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: my-cloud-run-service

build:
  artifacts:
    - image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-service
      docker:
        dockerfile: Dockerfile

deploy:
  cloudrun:
    projectid: MY_PROJECT
    region: us-central1
    # Cloud Code will prompt for service.yaml configuration,
    # or read from a service.yaml in the project root
```

```yaml
# service.yaml — Cloud Run service spec (committed alongside code)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "10"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 80
      timeoutSeconds: 300
      containers:
        - image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-service
          resources:
            limits:
              cpu: "2"
              memory: 1Gi
          env:
            - name: ENV
              value: production
```

#### Cloud Run Local Emulator

```bash
# Command Palette → "Cloud Code: Run on Cloud Run Emulator"
# Runs the container locally with Cloud Run environment variables injected:
# - PORT (automatically set to 8080)
# - K_SERVICE, K_REVISION, K_CONFIGURATION (Cloud Run metadata vars)
# - GOOGLE_CLOUD_PROJECT

# Access the emulated service via Cloud Shell Web Preview on port 8080
# Useful for testing Cloud Run-specific behaviour without deploying
```

#### Cloud Run Explorer Sidebar

- Lists all Cloud Run services across regions in one view
- Click a service to see: URL, active revision, traffic split percentages, container specs
- Right-click service → "View Logs" → streams logs in real time to Output panel
- Right-click service → "Open in Cloud Console" → opens directly in browser

---

### 2.5 SECRET MANAGER INTEGRATION

Cloud Code's Secret Manager sidebar lets developers browse project secrets, view versions, and inject secret values into local run configurations without ever storing values in files. Secret references in `launch.json` are resolved at runtime by calling Secret Manager API.

```json
// .vscode/launch.json — inject secrets as environment variables at run time
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch API Server",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/src/main.py",
      "env": {
        "DB_PASSWORD":    "${secret:projects/MY_PROJECT/secrets/db-password/versions/latest}",
        "API_KEY":        "${secret:projects/MY_PROJECT/secrets/api-key/versions/3}",
        "JWT_SECRET":     "${secret:projects/MY_PROJECT/secrets/jwt-secret/versions/latest}"
      }
    }
  ]
}
```

> ⚠️ **NOTE:** The `${secret:...}` syntax is a **Cloud Code-specific feature** in `launch.json`. It is resolved by the Cloud Code extension at launch time using your local ADC credentials. This syntax does NOT work in raw VS Code without Cloud Code, in CI/CD pipelines, or in Kubernetes manifests — use the Secret Manager client library or External Secrets Operator for those contexts.

#### Secret Manager Sidebar Operations

| Operation | How in Cloud Code |
|---|---|
| Browse all secrets | Cloud Code sidebar → "Secret Manager" → expand project |
| View secret versions | Click secret → see version list with creation dates and states |
| Create a new secret | Right-click "Secret Manager" → "Create Secret" |
| Access a secret version | Click version → copies value to clipboard |
| Open in Console | Right-click secret → "Open in Cloud Console" |

---

### 2.6 CLOUD SQL INTEGRATION

Cloud Code's Cloud SQL Explorer automatically manages a Cloud SQL Auth Proxy connection when you click "Connect" on an instance — eliminating the need to download, start, and configure the proxy binary manually.

```bash
# What Cloud Code does automatically on "Connect" click:
# 1. Downloads Cloud SQL Auth Proxy if not cached
# 2. Starts the proxy for the selected instance
# 3. Opens a SQL editor connected via localhost:{port}

# Manual equivalent (useful in Cloud Shell or CI/CD without Cloud Code):
curl -o cloud-sql-proxy \
  "https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.0/cloud-sql-proxy.linux.amd64"
chmod +x cloud-sql-proxy

# Start proxy for a PostgreSQL instance
./cloud-sql-proxy MY_PROJECT:us-central1:my-pg-instance \
  --port=5432 \
  --auto-iam-authn             # Use IAM for authentication (no password needed)

# Start proxy for a MySQL instance
./cloud-sql-proxy MY_PROJECT:us-central1:my-mysql-instance \
  --port=3306

# Connect your client to localhost after proxy is running
psql "host=127.0.0.1 port=5432 dbname=mydb user=myuser@myproject.iam"
mysql -h 127.0.0.1 -P 3306 -u myuser -p
```

#### Cloud SQL Explorer Features

- **Browse**: instances → databases → tables (schema tree)
- **Query editor**: write and execute SQL against Cloud SQL from inside VS Code
- **Connection string helper**: generates connection string for major frameworks (Spring, Django, etc.)
- **IAM auth**: connects using your ADC identity when `--auto-iam-authn` is configured

---

### 2.7 API EXPLORER & CODE GENERATION

Cloud Code API Explorer provides an interactive browser for all GCP REST APIs — search by service name, select a method, fill parameters, execute against live GCP, and generate ready-to-use client library code in the language of your choice.

```bash
# Access in VS Code:
# Cmd/Ctrl+Shift+P → "Cloud Code: GCP API Explorer"
# Or: Cloud Code sidebar → "APIs" → search for service name

# Workflow:
# 1. Search: "storage", "pubsub", "container", "bigquery"
# 2. Select method: e.g., storage.buckets.list
# 3. Fill parameters: project, maxResults, prefix
# 4. Click "Execute" → response shows in panel (live GCP call using your ADC)
# 5. Click "Generate Code" → choose Python/Java/Go/Node.js → snippet inserted at cursor
```

```python
# Example: Cloud Code-generated Python snippet for Cloud Storage bucket listing
from google.cloud import storage


def list_buckets(project_id: str) -> None:
    """Lists all GCS buckets in the specified project."""
    client = storage.Client(project=project_id)
    buckets = client.list_buckets()
    for bucket in buckets:
        print(f"  {bucket.name}")


if __name__ == "__main__":
    list_buckets("MY_PROJECT_ID")
```

> 💡 **TIP:** API Explorer is particularly useful when you know what you want to do but are unsure of the exact REST API parameters. Execute the request interactively to see the response schema, then generate the code snippet — faster than reading API docs and writing client code from scratch.

---

### 2.8 LOG STREAMING & MONITORING

Cloud Code Log Viewer streams Cloud Logging logs from GKE pods, Cloud Run services, and Cloud Functions directly into the VS Code Output panel, with real-time filtering using Cloud Logging query syntax — no need to open the Cloud Console log explorer.

```bash
# Access in VS Code:
# Cmd/Ctrl+Shift+P → "Cloud Code: View Logs"
# Or: Kubernetes Explorer → right-click pod → "View Logs"
# Or: Cloud Run Explorer → right-click service → "View Logs"

# Log filter syntax (same as Cloud Logging query language):
# resource.type="k8s_container" AND severity>=ERROR
# resource.labels.pod_name:"my-app" AND textPayload:"Exception"
# timestamp>="2025-01-01T00:00:00Z"

# Equivalent gcloud command for the same log stream:
gcloud logging read \
  'resource.type="k8s_container"
   AND resource.labels.cluster_name="my-cluster"
   AND resource.labels.pod_name="my-app-7d9f8b-xxx"
   AND severity>=ERROR' \
  --project=MY_PROJECT \
  --freshness=30m \
  --format="table(timestamp,severity,textPayload)"

# Tail logs in real time (Cloud Logging):
gcloud alpha logging tail \
  'resource.type="cloud_run_revision"
   AND resource.labels.service_name="my-service"' \
  --project=MY_PROJECT
```

#### Cloud Monitoring Integration

- **Kubernetes Explorer**: right-click a deployment → "View Monitoring Dashboard" → opens Cloud Monitoring in browser pre-filtered for that workload
- **Cloud Run Explorer**: right-click service → "View Metrics" → opens request count, latency, error rate dashboards
- Alerts configured in Cloud Monitoring appear as warnings in the Explorer tree when a service is in an alerting state

---

### 2.9 SKAFFOLD — THE ENGINE UNDER CLOUD CODE

Skaffold is an open-source, Google-created build-and-deploy automation tool that Cloud Code wraps for its Kubernetes and Cloud Run workflows. Understanding Skaffold directly is essential for CI/CD pipelines, custom workflows, and team standardisation — the same `skaffold.yaml` used in Cloud Code's dev loop works unmodified in Cloud Build, GitHub Actions, or any CI system.

```bash
# ── Installation ───────────────────────────────────────────────────────────
curl -Lo skaffold \
  "https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64"
chmod +x skaffold && sudo mv skaffold /usr/local/bin

# ── Core commands ──────────────────────────────────────────────────────────
skaffold dev                      # Continuous dev loop: watch → build → deploy → log
skaffold dev --port-forward       # Dev loop with automatic port forwarding
skaffold run                      # One-shot build + deploy (for CI/CD)
skaffold build                    # Build images only (no deploy)
skaffold deploy                   # Deploy only (using pre-built images)
skaffold delete                   # Remove all deployed resources
skaffold render --output=out.yaml # Render final manifests without deploying (GitOps)
skaffold verify                   # Run integration tests defined in skaffold.yaml
skaffold diagnose                 # Validate skaffold.yaml and print resolved config
skaffold inspect modules          # Show all modules and their dependencies

# ── Environment and profile flags ─────────────────────────────────────────
skaffold run --profile=prod                       # Use a named profile
skaffold run --profile=prod --tag=$(git rev-parse --short HEAD)  # Pin image tag
skaffold dev --kubeconfig=/path/to/kubeconfig     # Use a specific kubeconfig
skaffold run --namespace=staging                  # Deploy to a specific namespace

# ── Image building strategies ─────────────────────────────────────────────
# Defined in skaffold.yaml build section — choose one per environment:
# local: build with local Docker daemon
# googleCloudBuild: build with Cloud Build (no local Docker needed)
# cluster: build inside the Kubernetes cluster (Kaniko)
```

```yaml
# skaffold.yaml with multiple profiles for local/staging/prod
apiVersion: skaffold/v4beta11
kind: Config
metadata:
  name: my-app

build:
  artifacts:
    - image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app
      docker:
        dockerfile: Dockerfile

deploy:
  kubectl:
    manifests: ["k8s/base/**"]

# Profiles override specific sections of the base config
profiles:
  - name: local
    build:
      local:
        push: false              # Load into minikube directly; skip registry push
    deploy:
      kubectl:
        manifests: ["k8s/overlays/local/**"]

  - name: staging
    build:
      googleCloudBuild:          # Build in Cloud Build; no local Docker needed
        projectId: MY_PROJECT
    deploy:
      kubectl:
        manifests: ["k8s/overlays/staging/**"]

  - name: prod
    build:
      googleCloudBuild:
        projectId: MY_PROJECT
        machineType: E2_HIGHCPU_8    # Faster builds for prod releases
    deploy:
      kubectl:
        manifests: ["k8s/overlays/prod/**"]
    test:
      - image: us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app
        structureTests:
          - "./tests/container-structure-test.yaml"
```

---

### 2.10 LIMITS, COMPARISONS & BEST PRACTICES

#### Cloud Code vs Raw CLI

| Task | Raw CLI | Cloud Code | Practical Winner |
|---|---|---|---|
| Deploy to GKE | `kubectl apply -f *.yaml` | Right-click → "Apply to Cluster" | Tie — personal preference |
| Stream pod logs | `kubectl logs -f POD -n NS` | Click pod → "View Logs" | Cloud Code (no pod name lookup) |
| Debug a running container | Manual `delve` / `jdwp` / `debugpy` setup | "Debug on Kubernetes" one-click | **Cloud Code** (significant time saving) |
| Browse Secret Manager | `gcloud secrets versions access latest --secret=X` | Sidebar tree | Cloud Code (discoverability) |
| Generate GCP API code | Read docs → write from scratch | API Explorer → Generate | **Cloud Code** |
| CI/CD pipelines | Shell scripts + gcloud/kubectl | Not applicable (IDE-only) | **Raw CLI** |
| Skaffold dev loop | `skaffold dev` | "Run on Kubernetes" | Tie (same command wrapped) |
| Multi-cluster management | `kubectl config use-context` + scripts | Context selector in sidebar | Tie |

#### Best Practices

- **Commit `skaffold.yaml` to version control** — it is the contract between the Cloud Code dev loop and your CI/CD pipeline; the same file should work in both contexts
- **Use Skaffold profiles** to manage local/staging/prod differences in a single file rather than maintaining separate skaffold files per environment
- **Never store Secret Manager values in `.env` files** committed to Git — use Cloud Code's `${secret:...}` substitution in `launch.json` for local dev
- **Use Cloud Code YAML IntelliSense** for Kubernetes manifests — it catches schema errors before `kubectl apply`, preventing failed deployments
- **Install Cloud Code in Cloud Shell Editor** for a zero-local-setup IDE with full GCP integration — especially useful for teams onboarding to Kubernetes


---

## PART 4 — CROSS-SERVICE PATTERNS & DEVELOPER WORKFLOW GUIDE

The three services in this cheatsheet address different layers of the GCP developer experience: Cloud Shell provides the execution environment, Cloud Code provides the IDE integration and inner development loop, and Deployment Manager provides infrastructure provisioning. They are most powerful when used together as complementary tools rather than alternatives.

---

### 4.1 CHOOSING THE RIGHT TOOL FOR THE JOB

| Task | Cloud Shell | Cloud Code | Deployment Manager |
|---|---|---|---|
| Quick one-off admin command | ✅ **Best** | ❌ Too heavy | ❌ N/A |
| Develop a containerized application | ❌ Limited IDE | ✅ **Best** (Skaffold dev loop) | ❌ N/A |
| Deploy to GKE | ✅ CLI (`kubectl apply`) | ✅ **Best** (visual + debug) | ❌ N/A |
| Deploy to Cloud Run | ✅ CLI (`gcloud run deploy`) | ✅ **Best** (emulator + explorer) | ❌ N/A |
| Provision GCP infrastructure | ✅ (`gcloud` / `terraform`) | ❌ N/A | ✅ **Best** (legacy); prefer Terraform |
| New infra project (greenfield) | ✅ + Terraform | ❌ N/A | ⚠️ Consider Terraform instead |
| Browse and manage secrets | ❌ CLI only | ✅ **Best** (sidebar + injection) | ❌ N/A |
| Debug a running container | ❌ | ✅ **Best** (one-click attach) | ❌ N/A |
| CI/CD pipelines | ✅ **Best** (`gcloud` scripts) | ✅ (`skaffold run` in pipeline) | ✅ (`gcloud dm` in pipeline) |
| Multi-project admin tasks | ✅ **Best** (gcloud configs) | ❌ | ❌ |
| Kubernetes YAML authoring | ❌ (vim only) | ✅ **Best** (IntelliSense + validation) | ❌ N/A |
| Guided onboarding / runbooks | ✅ **Best** (Cloud Shell Tutorials) | ❌ | ❌ |

---

### 4.2 COMBINED DEVELOPER WORKFLOW

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PHASE 1: INFRASTRUCTURE PROVISIONING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tool: DEPLOYMENT MANAGER (or Terraform)
  Tool: CLOUD SHELL (execution environment)

  Author config.yaml + templates/ in local editor or Cloud Shell Editor
  gcloud deployment-manager deployments create my-infra --config=config.yaml --preview
  Review: confirm VPC, Cloud SQL, GCS bucket, IAM bindings are correct
  gcloud deployment-manager deployments update my-infra  → apply
  Capture outputs: DB connection name, network selfLink, etc.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PHASE 2: APPLICATION DEVELOPMENT (Inner Loop)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tool: CLOUD CODE (VS Code or IntelliJ)
  Tool: CLOUD SHELL (ad-hoc diagnostics)

  Cloud Code: "Run on Kubernetes" → skaffold dev starts
  Edit source code → Cloud Code triggers rebuild → redeploys to minikube/GKE
  Secret Manager sidebar: inject DB password into local run config
  Cloud SQL Explorer: run SQL queries against dev database inline
  Kubernetes Explorer: stream pod logs; exec into containers

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PHASE 3: DEBUGGING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tool: CLOUD CODE (primary)
  Tool: CLOUD SHELL (secondary / ad-hoc)

  Cloud Code: "Debug on Kubernetes" → IDE breakpoints in running container
  Cloud Code Log Viewer: stream filtered Cloud Logging in Output panel
  Cloud Shell: ad-hoc kubectl exec, gcloud logging read, gcloud sql connect

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PHASE 4: PROMOTION TO PRODUCTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tool: CLOUD CODE (Skaffold prod profile)
  Tool: CLOUD SHELL (infra updates)
  Tool: DEPLOYMENT MANAGER (config changes)

  Cloud Code / Skaffold: skaffold run --profile=prod → build + push + deploy
  Cloud Shell: gcloud deployment-manager deployments update infra --config=config-v2.yaml
  Cloud Code: Cloud Run Explorer → verify revisions; adjust traffic splits
  Cloud Shell: gcloud logging read to verify production health post-deploy
```

---

### 4.3 IAM ROLES SUMMARY

| Service | IAM Role | Description |
|---|---|---|
| Cloud Shell | `roles/cloudshell.admin` | Administer Cloud Shell for all org users (org-level) |
| Cloud Shell | `roles/cloudshell.user` | Use Cloud Shell; auto-granted to all GCP users |
| Cloud Code | `roles/container.developer` | Deploy to GKE clusters via Cloud Code |
| Cloud Code | `roles/run.developer` | Deploy Cloud Run services via Cloud Code |
| Cloud Code | `roles/secretmanager.secretAccessor` | Read secrets in Secret Manager sidebar |
| Cloud Code | `roles/cloudsql.client` | Connect to Cloud SQL via Auth Proxy in Cloud Code |
| Cloud Code | `roles/logging.viewer` | Stream Cloud Logging in the Log Viewer panel |
| Cloud Code | `roles/monitoring.viewer` | View Cloud Monitoring dashboards |
| Cloud Code | `roles/artifactregistry.writer` | Push container images from Skaffold builds |
| Deployment Manager | `roles/deploymentmanager.editor` | Create, update, and delete deployments |
| Deployment Manager | `roles/deploymentmanager.viewer` | Read-only view of deployments and resources |
| DM Service Account | `roles/editor` | Default; overly broad — **scope down to specific roles** |
| DM Service Account | `roles/compute.admin` | Create/manage GCE resources via DM |
| DM Service Account | `roles/cloudsql.admin` | Create/manage Cloud SQL instances via DM |
| DM Service Account | `roles/storage.admin` | Create/manage GCS buckets via DM |
| DM Service Account | `roles/iam.serviceAccountAdmin` | Create service accounts via DM |
| All services | `roles/iam.serviceAccountUser` | Act as service accounts attached to resources |

> 💡 **TIP:** Deployment Manager's service account (`PROJECT_NUMBER@cloudservices.gserviceaccount.com`) defaults to `roles/editor`. For production, replace this with the specific roles matching the resource types in your deployment. This follows least-privilege and passes security audits.

---

### 4.4 QUICK REFERENCE COMMAND CHEATSHEET

```bash
# ════════════════════════════════════════════════════════════════════════════════
# CLOUD SHELL — UTILITY COMMANDS
# ════════════════════════════════════════════════════════════════════════════════

# Open a file in Cloud Shell Editor
cloudshell edit FILE_NAME

# Open current directory as workspace
cloudshell open-workspace .

# Launch an interactive tutorial
cloudshell launch-tutorial ./tutorial.md

# Get the HTTPS preview URL for a local port
cloudshell get-web-preview-url -p 8080

# ════════════════════════════════════════════════════════════════════════════════
# GCLOUD — CONFIGURATION & AUTH
# ════════════════════════════════════════════════════════════════════════════════

# Show current active configuration
gcloud config list

# Set project / region / zone defaults
gcloud config set project MY_PROJECT_ID
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Named configurations for multi-project workflows
gcloud config configurations create prod-config
gcloud config configurations list
gcloud config configurations activate prod-config
gcloud config configurations activate default

# Run a command against a different project without switching
gcloud compute instances list --project=OTHER_PROJECT

# Impersonate a service account (always unset after)
gcloud config set auth/impersonate_service_account sa@project.iam.gserviceaccount.com
gcloud config unset auth/impersonate_service_account

# Refresh ADC (Application Default Credentials)
gcloud auth application-default login

# ════════════════════════════════════════════════════════════════════════════════
# TMUX — SESSION MANAGEMENT IN CLOUD SHELL
# ════════════════════════════════════════════════════════════════════════════════

# Start named session
tmux new-session -s main

# Start detached session with a command
tmux new-session -d -s bg-job -x 220 -y 50
tmux send-keys -t bg-job 'terraform apply -auto-approve' Enter

# List sessions
tmux list-sessions

# Reattach to session
tmux attach-session -t main

# Kill session
tmux kill-session -t main

# ════════════════════════════════════════════════════════════════════════════════
# CLOUD SHELL — SECURE BASTION OPERATIONS
# ════════════════════════════════════════════════════════════════════════════════

# SSH to GCE VM via IAP (no public IP required)
gcloud compute ssh VM_NAME --zone=ZONE --tunnel-through-iap

# SCP via IAP
gcloud compute scp LOCAL_FILE VM_NAME:~/dest/ --zone=ZONE --tunnel-through-iap

# Fetch GKE cluster credentials
gcloud container clusters get-credentials CLUSTER --zone=ZONE --project=PROJECT

# Start kubectl proxy for Web Preview access
kubectl proxy --port=8080 &

# Connect to Cloud SQL (gcloud auto-starts Auth Proxy)
gcloud sql connect INSTANCE_NAME --user=root --database=mydb

# ════════════════════════════════════════════════════════════════════════════════
# SKAFFOLD — CLOUD CODE'S ENGINE (standalone usage)
# ════════════════════════════════════════════════════════════════════════════════

# Install Skaffold
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold && sudo mv skaffold /usr/local/bin

# Continuous dev loop (watch + rebuild + redeploy)
skaffold dev --port-forward

# One-time build + deploy
skaffold run --port-forward

# Build only (no deploy)
skaffold build

# Build + push to specific registry
skaffold build --default-repo=us-central1-docker.pkg.dev/PROJECT/repo

# Deploy with a named profile
skaffold run --profile=prod

# Render manifests without deploying (GitOps review)
skaffold render --output=rendered.yaml

# Run post-deploy verification
skaffold verify

# Debug mode (inject debug agents; attach debugger)
skaffold debug --port-forward

# Delete all deployed resources
skaffold delete

# ════════════════════════════════════════════════════════════════════════════════
# KUBECTL — VIA CLOUD CODE CONTEXT
# ════════════════════════════════════════════════════════════════════════════════

# List contexts (clusters Cloud Code has configured)
kubectl config get-contexts
kubectl config use-context CONTEXT_NAME

# Common diagnostic commands
kubectl get pods -n NAMESPACE -o wide
kubectl get events -n NAMESPACE --sort-by='.lastTimestamp'
kubectl describe pod POD_NAME -n NAMESPACE
kubectl logs POD_NAME -n NAMESPACE -f --tail=100
kubectl logs POD_NAME -n NAMESPACE -c CONTAINER_NAME -f
kubectl exec -it POD_NAME -n NAMESPACE -- /bin/bash
kubectl top nodes
kubectl top pods -n NAMESPACE

# Port-forward to a service or pod
kubectl port-forward svc/my-service 8080:80 -n NAMESPACE
kubectl port-forward pod/my-pod 8080:8080 -n NAMESPACE

# Apply and delete manifests
kubectl apply -f k8s/
kubectl delete -f k8s/
kubectl apply -k kustomize/overlays/prod/   # Kustomize

# ════════════════════════════════════════════════════════════════════════════════
# DEPLOYMENT MANAGER — FULL LIFECYCLE
# ════════════════════════════════════════════════════════════════════════════════

# ── CREATE ────────────────────────────────────────────────────────────────────
gcloud deployment-manager deployments create DEPLOYMENT \
  --config=config.yaml \
  --preview

gcloud deployment-manager deployments update DEPLOYMENT    # apply preview

gcloud deployment-manager deployments create DEPLOYMENT \
  --config=config.yaml \
  --labels=env=prod,team=platform

# ── READ / INSPECT ────────────────────────────────────────────────────────────
gcloud deployment-manager deployments list
gcloud deployment-manager deployments describe DEPLOYMENT
gcloud deployment-manager resources list --deployment=DEPLOYMENT
gcloud deployment-manager resources describe RESOURCE --deployment=DEPLOYMENT
gcloud deployment-manager manifests list --deployment=DEPLOYMENT
gcloud deployment-manager manifests describe MANIFEST --deployment=DEPLOYMENT \
  --format="value(expandedConfig)"

# ── QUERY OUTPUTS ────────────────────────────────────────────────────────────
gcloud deployment-manager deployments describe DEPLOYMENT \
  --format="table(outputs.name, outputs.finalValue)"

gcloud deployment-manager deployments describe DEPLOYMENT --format=json \
  | jq -r '.outputs[] | select(.name=="FIELD_NAME") | .finalValue'

# ── UPDATE ────────────────────────────────────────────────────────────────────
gcloud deployment-manager deployments update DEPLOYMENT --config=config-v2.yaml --preview
gcloud deployment-manager deployments cancel-preview DEPLOYMENT
gcloud deployment-manager deployments update DEPLOYMENT
gcloud deployment-manager deployments update DEPLOYMENT \
  --update-labels=version=2,last-updated=$(date +%Y%m%d)

# ── DELETE ────────────────────────────────────────────────────────────────────
gcloud deployment-manager deployments delete DEPLOYMENT --quiet
gcloud deployment-manager deployments delete DEPLOYMENT --delete-policy=ABANDON

# ── COMPOSITE TYPES ───────────────────────────────────────────────────────────
gcloud deployment-manager types list
gcloud deployment-manager types create MY_TYPE --template=templates/my.py
gcloud deployment-manager types delete MY_TYPE

# ── OPERATIONS ────────────────────────────────────────────────────────────────
gcloud deployment-manager operations list
gcloud deployment-manager operations describe OPERATION_NAME
gcloud deployment-manager operations wait OPERATION_NAME
```

