# 🏗️ Terraform Comprehensive Cheatsheet

> A complete reference for Terraform — Infrastructure as Code — covering theory, HCL syntax, providers, state, modules, and production best practices.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
   - [What is Terraform?](#what-is-terraform)
   - [Declarative vs Imperative IaC](#declarative-vs-imperative-iac)
   - [The Terraform Workflow](#the-terraform-workflow)
   - [Providers](#providers-overview)
   - [Resources vs Data Sources](#resources-vs-data-sources)
   - [State](#state-overview)
   - [The Dependency Graph (DAG)](#the-dependency-graph-dag)
   - [Idempotency](#idempotency)
   - [Terraform vs Alternatives](#terraform-vs-alternatives)
   - [HCL Syntax Overview](#hcl-syntax-overview)
2. [Installation & Setup](#2-installation--setup)
   - [Installing Terraform](#installing-terraform)
   - [tfenv — Version Management](#tfenv--version-management)
   - [terraform init](#terraform-init)
   - [Environment Variables](#environment-variables)
   - [Cloud Provider Authentication](#cloud-provider-authentication)
3. [HCL Syntax & Language Features](#3-hcl-syntax--language-features)
   - [Blocks, Arguments & Expressions](#blocks-arguments--expressions)
   - [Types](#types)
   - [String Interpolation & Heredoc](#string-interpolation--heredoc)
   - [Operators](#operators)
   - [for Expressions](#for-expressions)
   - [Splat Expressions](#splat-expressions)
   - [Conditional Expressions](#conditional-expressions)
   - [Dynamic Blocks](#dynamic-blocks)
   - [locals](#locals)
   - [Built-in Functions](#built-in-functions)
   - [templatefile() and file()](#templatefile-and-file)
   - [JSON & YAML Functions](#json--yaml-functions)
4. [Providers](#4-providers)
   - [Provider Block Configuration](#provider-block-configuration)
   - [Version Constraints](#version-constraints)
   - [Provider Aliases](#provider-aliases)
   - [required_providers & Lock File](#required_providers--lock-file)
5. [Resources](#5-resources)
   - [Resource Block Anatomy](#resource-block-anatomy)
   - [Meta-Arguments](#meta-arguments)
   - [count vs for_each](#count-vs-for_each)
   - [lifecycle Blocks](#lifecycle-blocks)
   - [Resource Addressing & References](#resource-addressing--references)
   - [Timeouts Block](#timeouts-block)
6. [Data Sources](#6-data-sources)
7. [Variables](#7-variables)
   - [Input Variables](#input-variables)
   - [Variable Files & Precedence](#variable-files--precedence)
   - [Output Values](#output-values)
   - [locals vs Variables](#locals-vs-variables)
   - [Complex Types & Validation](#complex-types--validation)
8. [State Management](#8-state-management)
   - [What State Contains](#what-state-contains)
   - [Remote Backends](#remote-backends)
   - [State Locking](#state-locking)
   - [terraform state Commands](#terraform-state-commands)
   - [terraform import](#terraform-import)
   - [Sensitive Data in State](#sensitive-data-in-state)
9. [Modules](#9-modules)
   - [Module Sources](#module-sources)
   - [Module Block](#module-block)
   - [Root vs Child Modules](#root-vs-child-modules)
   - [Module Structure Conventions](#module-structure-conventions)
10. [Workspaces](#10-workspaces)
11. [CLI Commands](#11-cli-commands)
12. [Expressions & Functions — Deep Dive](#12-expressions--functions--deep-dive)
13. [Backends & Remote State](#13-backends--remote-state)
14. [Provisioners](#14-provisioners)
15. [Testing & Validation](#15-testing--validation)
16. [Best Practices & Patterns](#16-best-practices--patterns)
17. [Quick Reference Tables](#17-quick-reference-tables)

---

## 1. Core Concepts & Theory

### What is Terraform?

Terraform is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp. It lets you define, provision, and manage infrastructure across any cloud or on-premises environment using a high-level declarative configuration language (HCL).

**Why use Terraform?**
- **Version-controlled infrastructure** — treat infra like application code (PRs, reviews, history)
- **Multi-cloud / multi-provider** — AWS, GCP, Azure, Kubernetes, Cloudflare, Datadog, and 3,000+ providers in one tool
- **Automation** — repeatably provision identical environments (dev, staging, prod)
- **Drift detection** — `terraform plan` reveals when real infra differs from declared state
- **Self-documenting** — the code IS the documentation of what exists

---

### Declarative vs Imperative IaC

```
Imperative (Ansible, shell scripts):       Declarative (Terraform):
"Do this, then this, then that"            "Here is the desired end state"

Step 1: create VPC                         resource "aws_vpc" "main" {
Step 2: if VPC exists, skip                  cidr_block = "10.0.0.0/16"
Step 3: create subnet                      }
Step 4: if subnet exists, skip
...                                        Terraform figures out the steps.
```

Terraform compares **desired state** (your HCL) against **current state** (state file) and calculates the minimum set of API calls to converge them. You don't script the *how* — you declare the *what*.

---

### The Terraform Workflow

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  WRITE  │────▶│  INIT   │────▶│  PLAN   │────▶│  APPLY  │
│  .tf    │     │providers│     │dry run  │     │execute  │
│  files  │     │backend  │     │diff     │     │changes  │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
                                                      │
                                                      ▼
                                               ┌─────────┐
                                               │ DESTROY │
                                               │(optional│
                                               │teardown)│
                                               └─────────┘

Write   → Author .tf files describing desired infrastructure
Init    → Download providers, configure backend, install modules
Plan    → Terraform calculates what will change (read-only, safe to run anytime)
Apply   → Execute the plan — provision/modify/delete real resources
Destroy → Tear down all resources managed by this configuration
```

---

### Providers (Overview)

Providers are **plugins** that bridge Terraform to external APIs. Each provider implements resource types and data sources for a specific platform.

```
Terraform Core
     │
     ├── AWS Provider    → calls AWS APIs  (EC2, S3, RDS, IAM...)
     ├── GCP Provider    → calls GCP APIs  (GCE, GCS, GKE...)
     ├── Azure Provider  → calls Azure APIs (VMs, Storage, AKS...)
     ├── K8s Provider    → calls Kubernetes API
     ├── GitHub Provider → calls GitHub API
     └── ... 3,000+ providers on registry.terraform.io
```

Providers are distributed separately from Terraform core and downloaded during `terraform init`.

---

### Resources vs Data Sources

```hcl
# RESOURCE — creates and manages infrastructure (Terraform owns its lifecycle)
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"    # Terraform creates and manages this
}

# DATA SOURCE — reads existing infrastructure (Terraform does NOT manage it)
data "aws_s3_bucket" "existing" {
  bucket = "some-bucket-created-elsewhere"  # Terraform reads this, won't delete it
}
```

| | Resource | Data Source |
|--|---------|------------|
| **Keyword** | `resource` | `data` |
| **Lifecycle** | Managed by Terraform | External — read-only |
| **In state** | Yes | No (refreshed on each plan) |
| **Creates infra** | Yes | No |
| **Reference** | `aws_s3_bucket.my_bucket.id` | `data.aws_s3_bucket.existing.id` |

---

### State (Overview)

Terraform **state** is a JSON file (`terraform.tfstate`) that maps your HCL configuration to the real-world resources it manages.

```
HCL Config          State File              Real Infrastructure
──────────────      ──────────────────      ──────────────────────
resource            {                       AWS EC2 instance
"aws_instance"  ←── "aws_instance.web": ←── i-0abc123def456
"web" { ... }       "id": "i-0abc...",      (actually running)
                    "ami": "ami-...",
                    ...
                    }
```

**Why state is critical:**
- Maps resource names in code → real resource IDs
- Tracks metadata needed for updates (e.g. resource ARNs)
- Enables dependency tracking across your config
- Detects drift between declared config and real world
- Enables `plan` to show a diff without hitting every API

> ⚠️ **Warning:** State can contain sensitive values (passwords, keys). Protect it like a secret. Never commit `terraform.tfstate` to version control.

---

### The Dependency Graph (DAG)

Terraform builds a **Directed Acyclic Graph** of all resources to determine creation order and parallelism.

```
aws_vpc.main
    │
    ├──▶ aws_subnet.public  ──▶ aws_instance.web ──▶ aws_eip.web
    │                                 │
    └──▶ aws_subnet.private           └──▶ (depends on security group)
    │
    └──▶ aws_internet_gateway.igw
    │
    └──▶ aws_security_group.web ──────▶ aws_instance.web
```

- Resources with no dependencies are created in **parallel**
- Resources with `depends_on` or implicit references are serialised
- `terraform graph | dot -Tsvg > graph.svg` renders the full graph

---

### Idempotency

Terraform is idempotent: running `terraform apply` on an already-applied configuration produces **zero changes**.

```bash
# First apply: creates 3 resources
terraform apply    # Plan: 3 to add, 0 to change, 0 to destroy

# Second apply (no changes to .tf files):
terraform apply    # Plan: 0 to add, 0 to change, 0 to destroy ← idempotent

# If someone manually changes infra outside Terraform (drift):
terraform apply    # Plan: 0 to add, 1 to change, 0 to destroy ← detects drift
```

---

### Terraform vs Alternatives

| Tool | Type | Model | Scope | State |
|------|------|-------|-------|-------|
| **Terraform** | IaC | Declarative | Multi-cloud, any API | Yes (tfstate) |
| **Pulumi** | IaC | Declarative (real code: Python/TS/Go) | Multi-cloud | Yes (Pulumi Cloud) |
| **Ansible** | Config Mgmt | Procedural/Imperative | Servers + some cloud | No (agentless) |
| **CloudFormation** | IaC | Declarative (JSON/YAML) | AWS only | Yes (CF stacks) |
| **CDK** | IaC | Declarative (code → CF) | AWS only | Via CloudFormation |
| **Crossplane** | IaC | Declarative (K8s CRDs) | Multi-cloud | Kubernetes etcd |

**Terraform sweet spot:** multi-cloud, API-driven infrastructure provisioning with a large provider ecosystem.

---

### HCL Syntax Overview

```hcl
# HCL — HashiCorp Configuration Language
# Human-readable, JSON-compatible, designed for infrastructure config

# BLOCK — the fundamental unit. Has a type, optional labels, and a body.
block_type "label_1" "label_2" {
  argument_name = expression_value   # argument: key = value pair
}

# Examples of block types:
terraform {}          # terraform settings block (no labels)
provider "aws" {}     # provider block (1 label: provider name)
resource "aws_vpc" "main" {}    # resource block (2 labels: type + local name)
variable "region" {}            # variable block (1 label: variable name)
module "network" {}             # module block (1 label: module name)

# Comments
# Single-line comment
// Also single-line (less common)
/* Multi-line
   comment */
```

---

## 2. Installation & Setup

### Installing Terraform

```bash
# --- macOS (Homebrew — recommended) ---
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform version   # verify

# --- macOS/Linux (manual binary) ---
TERRAFORM_VERSION="1.7.5"
curl -LO "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip"
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform version

# --- Linux (apt — Ubuntu/Debian) ---
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# --- Windows (Chocolatey) ---
choco install terraform

# --- Windows (Scoop) ---
scoop install terraform

# --- Windows (manual) ---
# Download zip from https://developer.hashicorp.com/terraform/downloads
# Extract terraform.exe to a directory in your PATH
```

---

### tfenv — Version Management

`tfenv` manages multiple Terraform versions — essential for working across projects with different version requirements.

```bash
# Install tfenv (macOS)
brew install tfenv

# Install tfenv (Linux/manual)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc

# --- Usage ---
tfenv list-remote               # list all available Terraform versions
tfenv install 1.7.5             # install a specific version
tfenv install latest            # install the latest stable version
tfenv use 1.7.5                 # switch to a specific version globally
tfenv list                      # list locally installed versions

# Pin a version per project (creates .terraform-version file)
echo "1.7.5" > .terraform-version
# tfenv automatically uses this version in the directory
```

---

### terraform init

`terraform init` is always the first command to run in a new or cloned Terraform project.

```bash
# Basic init — downloads providers and sets up backend
terraform init

# Key flags:
terraform init -upgrade              # upgrade providers to latest allowed version
terraform init -reconfigure          # reconfigure backend (re-enter all settings)
terraform init -migrate-state        # migrate state to a new backend configuration
terraform init -backend=false        # skip backend initialisation (only download providers)

# Pass backend config values (for partial config — see Section 13)
terraform init -backend-config="bucket=my-state-bucket"
terraform init -backend-config=backend.hcl   # load from file

# What init does:
# 1. Reads required_providers from .tf files
# 2. Downloads provider plugins into .terraform/providers/
# 3. Creates/updates .terraform.lock.hcl
# 4. Initialises the configured backend
# 5. Downloads any referenced modules into .terraform/modules/
```

> 💡 **Tip:** Always commit `.terraform.lock.hcl` to version control. Never commit the `.terraform/` directory itself (add to `.gitignore`).

---

### Environment Variables

```bash
# --- Input variables via environment ---
# Any TF_VAR_<name> is passed as the value of variable "<name>"
export TF_VAR_region="us-east-1"
export TF_VAR_db_password="supersecret"      # never hardcode secrets in .tf files
export TF_VAR_instance_count="3"

# --- Logging ---
export TF_LOG="DEBUG"    # levels: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG="TRACE"    # most verbose — shows all provider API calls
export TF_LOG_PATH="./terraform.log"         # write logs to a file instead of stderr
unset TF_LOG             # disable logging

# --- Directories ---
export TF_DATA_DIR=".terraform_cache"        # override .terraform/ directory location
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"  # cache provider binaries
                                              # (saves re-downloading across projects)

# --- CLI arguments (applied to all commands) ---
export TF_CLI_ARGS="-no-color"               # disable ANSI colours (useful in CI)
export TF_CLI_ARGS_plan="-compact-warnings"  # flags for specific sub-command
export TF_CLI_ARGS_apply="-auto-approve"     # auto-approve all applies (CI use only)

# --- Workspace ---
export TF_WORKSPACE="staging"                # select workspace without running command

# --- Other ---
export TF_INPUT=0                            # disable interactive prompts (fail if var missing)
export TF_IN_AUTOMATION=1                    # adjusts output for CI environments (less verbose)
```

---

### Cloud Provider Authentication

```bash
# --- AWS ---
# Option 1: Environment variables (recommended for CI)
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"

# Option 2: AWS Profile (recommended for local development)
export AWS_PROFILE="my-profile"              # uses ~/.aws/credentials

# Option 3: IAM Role (EC2 instance / ECS task / Lambda — no keys needed)
# Just run on the resource — SDK picks up role automatically

# Option 4: In provider block (NOT recommended — avoid hardcoding)
provider "aws" {
  region     = "us-east-1"
  # access_key = "..."  # Don't do this
}

# --- GCP ---
# Option 1: Service account key file (CI)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# Option 2: Application Default Credentials (local dev)
gcloud auth application-default login

# Option 3: In provider block
provider "google" {
  project     = "my-project-id"
  region      = "us-central1"
  credentials = file("service-account.json")  # avoid — use env vars instead
}

# --- Azure ---
# Option 1: Azure CLI (local dev)
az login

# Option 2: Service Principal (CI)
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-client-secret"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"

# Option 3: Managed Identity (Azure VMs/pipelines)
export ARM_USE_MSI=true
```

---

## 3. HCL Syntax & Language Features

### Blocks, Arguments & Expressions

```hcl
# Block syntax: type [labels...] { body }
resource "aws_instance" "web" {          # block type + 2 labels
  ami           = "ami-0c55b159cbfafe1f0" # argument (string literal)
  instance_type = var.instance_type       # argument (variable reference)
  count         = 3                       # argument (number literal)
  tags          = local.common_tags       # argument (local reference)

  # Nested block (no = sign, no label)
  network_interface {
    network_interface_id = aws_network_interface.main.id
    device_index         = 0
  }
}
```

---

### Types

```hcl
# PRIMITIVE TYPES
variable "name"    { type = string }   # "hello"
variable "count"   { type = number }   # 42 or 3.14
variable "enabled" { type = bool   }   # true or false

# COLLECTION TYPES
variable "zones" {
  type    = list(string)               # ordered, allows duplicates: ["a", "b", "a"]
  default = ["us-east-1a", "us-east-1b"]
}

variable "tags" {
  type    = map(string)                # key-value pairs, all same type
  default = { env = "prod", team = "platform" }
}

variable "unique_ids" {
  type    = set(string)                # unordered, unique values: {"a", "b"}
  default = ["sg-111", "sg-222"]       # input as list, stored as set
}

# STRUCTURAL TYPES
variable "network_config" {
  type = object({                      # fixed set of named attributes, can be different types
    cidr_block        = string
    enable_dns        = bool
    max_subnets       = number
  })
}

variable "ip_list" {
  type    = tuple([string, number, bool])  # fixed-length, ordered, each element typed
  default = ["10.0.0.1", 8080, true]
}

# any — Terraform infers the type at runtime
variable "flexible" { type = any }

# complex nested type
variable "subnets" {
  type = list(object({
    cidr = string
    az   = string
    public = bool
  }))
}
```

---

### String Interpolation & Heredoc

```hcl
# String interpolation: embed expressions in strings with ${}
locals {
  name       = "myapp"
  env        = "prod"
  full_name  = "${local.name}-${local.env}"       # "myapp-prod"
  bucket_arn = "arn:aws:s3:::${aws_s3_bucket.main.id}"
}

# Heredoc syntax: multi-line strings
resource "aws_instance" "web" {
  user_data = <<-EOT
    #!/bin/bash
    echo "Hello from ${local.env}" > /tmp/hello.txt
    yum update -y
    yum install -y nginx
    systemctl start nginx
  EOT
  # <<-EOT strips leading whitespace (the dash before EOT)
  # <<EOT (without dash) preserves all leading whitespace
}

# Escape a literal ${ with $${ (avoid template interpolation)
locals {
  bash_script = "echo $${HOME}"   # renders as: echo ${HOME}
}

# Template strings with %{ directives
locals {
  items = ["a", "b", "c"]
  joined = "%{ for item in local.items }${item},%{ endfor }"
  # renders as: a,b,c,
}
```

---

### Operators

```hcl
# --- Arithmetic ---
locals {
  sum     = 5 + 3       # 8
  diff    = 10 - 4      # 6
  product = 3 * 4       # 12
  quotient = 10 / 3     # 3.3333...
  modulo  = 10 % 3      # 1
}

# --- Comparison (return bool) ---
locals {
  eq      = "a" == "a"   # true
  neq     = "a" != "b"   # true
  lt      = 3 < 5        # true
  lte     = 3 <= 3       # true
  gt      = 5 > 3        # true
  gte     = 5 >= 5       # true
}

# --- Logical ---
locals {
  and_op  = true && false   # false
  or_op   = true || false   # true
  not_op  = !true           # false
}
```

---

### for Expressions

```hcl
# --- List → List ---
variable "names" {
  default = ["alice", "bob", "carol"]
}

locals {
  upper_names = [for name in var.names : upper(name)]
  # Result: ["ALICE", "BOB", "CAROL"]

  # With index
  indexed = [for i, name in var.names : "${i}: ${name}"]
  # Result: ["0: alice", "1: bob", "2: carol"]

  # With if filter
  long_names = [for name in var.names : name if length(name) > 3]
  # Result: ["alice", "carol"]
}

# --- List → Map ---
locals {
  name_lengths = { for name in var.names : name => length(name) }
  # Result: { alice = 5, bob = 3, carol = 5 }
}

# --- Map → Map ---
variable "instance_types" {
  default = {
    web = "t3.micro"
    api = "t3.small"
    db  = "t3.large"
  }
}

locals {
  # Transform map values
  prefixed = { for key, val in var.instance_types : key => "aws:${val}" }
  # Result: { web = "aws:t3.micro", api = "aws:t3.small", ... }

  # Filter map entries
  small_only = { for k, v in var.instance_types : k => v if v != "t3.large" }
}

# --- Map → List ---
locals {
  instance_list = [for k, v in var.instance_types : "${k}=${v}"]
  # Result: ["web=t3.micro", "api=t3.small", "db=t3.large"]
}
```

---

### Splat Expressions

```hcl
# Splat expressions extract a single attribute from a list of objects
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

output "instance_ids" {
  # [*] splat — equivalent to [for o in aws_instance.server : o.id]
  value = aws_instance.server[*].id
  # Result: ["i-111", "i-222", "i-333"]
}

output "private_ips" {
  value = aws_instance.server[*].private_ip
}

# Splat works on nested attributes too
output "subnet_ids" {
  value = aws_instance.server[*].network_interface[0].subnet_id
}

# Legacy splat (.*) for older-style resources — avoid in new code
# output "old_style" { value = aws_instance.server.*.id }
```

---

### Conditional Expressions

```hcl
# Ternary: condition ? true_value : false_value
locals {
  env            = "prod"
  instance_type  = local.env == "prod" ? "t3.large" : "t3.micro"
  # Result: "t3.large"

  replica_count  = var.high_availability ? 3 : 1

  # Nested ternary (use sparingly — can get unreadable)
  tier = var.env == "prod" ? "premium" : (var.env == "staging" ? "standard" : "free")
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  # Conditional resource creation using count
  count = var.create_instance ? 1 : 0
}

# Conditional output value
output "endpoint" {
  value = var.use_https ? "https://${aws_lb.main.dns_name}" : "http://${aws_lb.main.dns_name}"
}
```

---

### Dynamic Blocks

```hcl
# Dynamic blocks generate repeated nested blocks from a collection
# Useful when the number of nested blocks depends on input variables

variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 22,  protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # Dynamic block — generates one "ingress" block per rule in the list
  dynamic "ingress" {
    for_each = var.ingress_rules          # iterate over the collection
    content {                             # content block defines the nested block body
      from_port   = ingress.value.port    # ingress.value = current element
      to_port     = ingress.value.port    # ingress.key   = index/key
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = "Rule ${ingress.key}"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Dynamic block with conditional (skip if empty)
resource "aws_launch_template" "app" {
  name_prefix = "app-"

  dynamic "tag_specifications" {
    for_each = var.enable_tagging ? ["instance", "volume"] : []
    content {
      resource_type = tag_specifications.value
      tags          = local.common_tags
    }
  }
}
```

---

### locals

```hcl
# locals block — define computed, reusable values within a module
# Not configurable from outside (unlike variables)
# Can reference other locals, variables, resource outputs

locals {
  # Simple derived values
  env    = var.environment                               # rename for brevity
  region = var.aws_region

  # Computed name prefix — used everywhere for consistent naming
  name_prefix = "${var.project}-${var.environment}"

  # Reusable tag map — applied to all resources
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.team
    CostCenter  = var.cost_center
  }

  # Merged tags (base + resource-specific)
  instance_tags = merge(local.common_tags, {
    Role = "webserver"
    Name = "${local.name_prefix}-web"
  })

  # Conditional values
  is_production  = var.environment == "production"
  instance_type  = local.is_production ? "t3.large" : "t3.micro"
  replica_count  = local.is_production ? 3 : 1

  # CIDR calculations
  vpc_cidr       = "10.${var.vpc_number}.0.0/16"
  public_cidrs   = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i)]
  private_cidrs  = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i + 10)]
}
```

> 💡 **Tip:** Centralise all name/tag construction in `locals.tf`. This ensures naming consistency and makes refactoring easy — change the prefix in one place, everything updates.

---

### Built-in Functions

```hcl
# --- STRING FUNCTIONS ---
locals {
  upper     = upper("hello")               # "HELLO"
  lower     = lower("WORLD")               # "world"
  title     = title("hello world")         # "Hello World"
  trimmed   = trimspace("  hello  ")       # "hello"
  replaced  = replace("hello world", "world", "terraform")  # "hello terraform"
  split_val = split(",", "a,b,c")          # ["a", "b", "c"]
  joined    = join("-", ["a", "b", "c"])   # "a-b-c"
  formatted = format("%.2f", 3.14159)      # "3.14"
  formatted2 = formatlist("item %d", [1, 2, 3])  # ["item 1", "item 2", "item 3"]
  substr_val = substr("hello world", 0, 5)  # "hello"
  padded    = format("%-10s|", "hi")       # "hi        |"

  # String checks
  has_prefix = startswith("terraform-s3", "terraform")  # true
  has_suffix = endswith("myfile.tf", ".tf")              # true
  contains_s = strcontains("hello world", "world")       # true
}

# --- NUMERIC FUNCTIONS ---
locals {
  abs_val   = abs(-5)          # 5
  ceil_val  = ceil(4.2)        # 5
  floor_val = floor(4.8)       # 4
  min_val   = min(3, 1, 5, 2)  # 1
  max_val   = max(3, 1, 5, 2)  # 5
  pow_val   = pow(2, 10)       # 1024
  log_val   = log(100, 10)     # 2
  signum    = signum(-5)       # -1
}

# --- COLLECTION FUNCTIONS ---
locals {
  # List operations
  length_l  = length(["a", "b", "c"])          # 3
  element_v = element(["a", "b", "c"], 1)      # "b"
  index_v   = index(["a", "b", "c"], "b")      # 1
  slice_v   = slice(["a","b","c","d"], 1, 3)   # ["b", "c"]
  concat_v  = concat(["a", "b"], ["c", "d"])   # ["a", "b", "c", "d"]
  flatten_v = flatten([[1, 2], [3, [4, 5]]])   # [1, 2, 3, 4, 5]
  distinct_v = distinct(["a", "b", "a", "c"]) # ["a", "b", "c"]
  reverse_v = reverse([1, 2, 3])              # [3, 2, 1]
  sort_v    = sort(["c", "a", "b"])           # ["a", "b", "c"]
  contains_v = contains(["a", "b"], "a")      # true

  # Map operations
  keys_v    = keys({ a = 1, b = 2 })           # ["a", "b"]
  values_v  = values({ a = 1, b = 2 })         # [1, 2]
  lookup_v  = lookup({ a = 1, b = 2 }, "a", 0) # 1 (with default 0)
  merge_v   = merge({ a = 1 }, { b = 2 })      # { a = 1, b = 2 }
  has_key   = contains(keys({ a = 1 }), "a")   # true

  # Set operations
  setintersection_v = setintersection(["a","b"], ["b","c"])  # toset(["b"])
  setunion_v        = setunion(["a","b"], ["b","c"])         # toset(["a","b","c"])
  setsubtract_v     = setsubtract(["a","b","c"], ["b"])      # toset(["a","c"])
}

# --- ENCODING FUNCTIONS ---
locals {
  b64enc = base64encode("hello world")   # "aGVsbG8gd29ybGQ="
  b64dec = base64decode("aGVsbG8gd29ybGQ=")  # "hello world"
  urlenc = urlencode("hello world")      # "hello+world"
  sha256 = sha256("mydata")             # hex SHA-256 hash
  md5    = md5("mydata")                # hex MD5 hash (NOT for security)
  uuid   = uuid()                       # generates a new random UUID each run
}

# --- FILESYSTEM FUNCTIONS ---
locals {
  content    = file("${path.module}/scripts/init.sh")  # read file as string
  b64content = filebase64("${path.module}/certs/ca.pem")  # read file as base64
  md5file    = filemd5("${path.module}/config.json")   # MD5 of file contents
  # path.module  = directory of current module
  # path.root    = directory of root module
  # path.cwd     = current working directory
}

# --- DATE/TIME FUNCTIONS ---
locals {
  timestamp_now = timestamp()        # current UTC time: "2024-03-15T10:30:00Z"
  formatted_ts  = formatdate("YYYY-MM-DD", timestamp())  # "2024-03-15"
  # Note: timestamp() changes every plan run — use carefully
}

# --- TYPE CONVERSION FUNCTIONS ---
locals {
  to_string = tostring(42)               # "42"
  to_number = tonumber("42")             # 42
  to_bool   = tobool("true")             # true
  to_list   = tolist(toset(["a","b"]))   # ["a", "b"]
  to_set    = toset(["a", "b", "a"])     # toset(["a", "b"])
  to_map    = tomap({ a = "1", b = "2" })
}

# --- IP NETWORKING FUNCTIONS ---
locals {
  vpc_cidr    = "10.0.0.0/16"
  # Calculate subnets — cidrsubnet(prefix, newbits, netnum)
  subnet_0    = cidrsubnet("10.0.0.0/16", 8, 0)   # "10.0.0.0/24"
  subnet_1    = cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
  subnet_10   = cidrsubnet("10.0.0.0/16", 8, 10)  # "10.0.10.0/24"

  # Get specific host address in a CIDR range
  first_host  = cidrhost("10.0.0.0/24", 1)   # "10.0.0.1"
  last_host   = cidrhost("10.0.0.0/24", 254) # "10.0.0.254"

  # Get subnet mask
  netmask     = cidrnetmask("10.0.0.0/24")   # "255.255.255.0"
}
```

---

### templatefile() and file()

```hcl
# file() — read a file as a plain string
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  key_name      = aws_key_pair.main.key_name

  # Inline script from file
  user_data = file("${path.module}/scripts/bootstrap.sh")
}

# templatefile() — render a file as a template with variable substitution
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Pass a map of variables to the template
  user_data = templatefile("${path.module}/templates/init.sh.tpl", {
    db_host    = aws_db_instance.main.address
    db_name    = var.db_name
    app_port   = var.app_port
    region     = var.aws_region
    log_level  = var.environment == "prod" ? "warn" : "debug"
  })
}
```

```bash
# templates/init.sh.tpl — referenced by templatefile() above
#!/bin/bash
# Variables are substituted from the map passed to templatefile()
DB_HOST="${db_host}"
DB_NAME="${db_name}"
APP_PORT="${app_port}"
REGION="${region}"
LOG_LEVEL="${log_level}"

echo "Configuring app in $REGION"
echo "DB_HOST=$DB_HOST" >> /etc/app.env
echo "DB_NAME=$DB_NAME" >> /etc/app.env
echo "APP_PORT=$APP_PORT" >> /etc/app.env

# Template directives
%{ for host in extra_hosts ~}
echo "${host}" >> /etc/hosts
%{ endfor ~}
```

---

### JSON & YAML Functions

```hcl
# jsonencode() — encode a Terraform value as JSON string
resource "aws_iam_policy" "s3_access" {
  name   = "s3-access"
  # Encode a native HCL map as JSON (avoids messy heredoc JSON)
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.main.arn}/*"
      }
    ]
  })
}

# jsondecode() — parse a JSON string into a Terraform value
locals {
  config_json = file("${path.module}/config.json")
  config      = jsondecode(local.config_json)
  # Access fields: local.config.database.host
}

# yamlencode() — encode a value as YAML string
resource "kubernetes_config_map" "app" {
  metadata { name = "app-config" }
  data = {
    "config.yaml" = yamlencode({
      port     = var.app_port
      debug    = var.enable_debug
      database = {
        host = var.db_host
        port = 5432
      }
    })
  }
}

# yamldecode() — parse YAML string to Terraform value
locals {
  values_yaml = file("${path.module}/values.yaml")
  helm_values = yamldecode(local.values_yaml)
}
```

---

## 4. Providers

### Provider Block Configuration

```hcl
# versions.tf (or providers.tf) — declare provider requirements
terraform {
  required_version = ">= 1.5.0"         # minimum Terraform version

  required_providers {
    aws = {
      source  = "hashicorp/aws"          # registry.terraform.io/hashicorp/aws
      version = "~> 5.0"                 # allow 5.x but not 6.x
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

# Provider configuration block — configure the provider itself
provider "aws" {
  region = var.aws_region                # required: which region to target

  # Optional: assume an IAM role (for cross-account access)
  assume_role {
    role_arn = "arn:aws:iam::123456789:role/TerraformRole"
  }

  # Optional: default tags applied to ALL resources created by this provider
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
    }
  }
}
```

---

### Version Constraints

```hcl
# Version constraint operators:
# = 1.2.3     — exact version only
# != 1.2.3    — any version except this one
# > 1.2.3     — greater than
# >= 1.2.3    — greater than or equal
# < 1.2.3     — less than
# <= 1.2.3    — less than or equal
# ~> 1.2.3    — allows rightmost: >=1.2.3, <1.3.0 (patch-level updates only)
# ~> 1.2      — allows rightmost: >=1.2.0, <2.0.0 (minor-level updates)

required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.0"        # allows 5.x.x updates, blocks 6.0.0
  }
  helm = {
    source  = "hashicorp/helm"
    version = ">= 2.10, < 3.0" # multiple constraints (AND)
  }
  random = {
    source  = "hashicorp/random"
    version = "= 3.5.1"       # pin to exact version (most reproducible)
  }
}
```

> 💡 **Tip:** Use `~> X.Y` (minor-pinned) for most providers — it allows patch updates (bug fixes) while protecting against breaking changes in major versions. Pin exact versions (`= X.Y.Z`) in production for maximum reproducibility.

---

### Provider Aliases

```hcl
# Use provider aliases to configure the same provider multiple times
# (e.g. multi-region, multi-account, or multi-environment)

# Default provider (no alias)
provider "aws" {
  region = "us-east-1"
}

# Additional provider configurations with aliases
provider "aws" {
  alias  = "us_west"           # alias identifier
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu_central"
  region = "eu-central-1"
}

# Use a specific provider alias in a resource
resource "aws_instance" "web_east" {
  # Uses default provider (us-east-1)
  ami           = "ami-east-123"
  instance_type = "t3.micro"
}

resource "aws_instance" "web_west" {
  provider      = aws.us_west    # use the aliased provider
  ami           = "ami-west-456"
  instance_type = "t3.micro"
}

resource "aws_s3_bucket" "backup" {
  provider = aws.eu_central
  bucket   = "my-eu-backup-bucket"
}

# Pass aliased providers to modules
module "replication" {
  source = "./modules/replication"

  providers = {
    aws.primary   = aws          # map module's "aws.primary" to default aws
    aws.secondary = aws.us_west  # map module's "aws.secondary" to aliased aws.us_west
  }
}
```

---

### required_providers & Lock File

```hcl
# The .terraform.lock.hcl file is automatically created/updated by terraform init
# It records exact provider versions and hashes for reproducibility

# .terraform.lock.hcl (auto-generated — do NOT edit manually)
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"                    # exact version locked
  constraints = "~> 5.0"                    # the constraint from required_providers
  hashes = [
    "h1:abc123...",                          # hash of the provider binary
    "zh:def456...",                          # hash for verification
  ]
}
```

```bash
# Always commit .terraform.lock.hcl to version control
git add .terraform.lock.hcl

# Update lock file after changing version constraints
terraform init -upgrade          # upgrades to latest allowed version + updates lock

# Add hashes for additional platforms (e.g. for CI running on Linux when you develop on Mac)
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64
```

---

## 5. Resources

### Resource Block Anatomy

```hcl
# resource "<PROVIDER_TYPE>" "<LOCAL_NAME>" { ... }
# PROVIDER_TYPE: identifies the provider and resource type (e.g. aws_instance)
# LOCAL_NAME:    unique name within the module (used for references, not in cloud)

resource "aws_instance" "web_server" {
  # --- Required arguments (provider-specific) ---
  ami           = "ami-0c55b159cbfafe1f0"   # Amazon Machine Image ID
  instance_type = "t3.micro"                # EC2 instance size

  # --- Optional arguments ---
  key_name               = aws_key_pair.deployer.key_name
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id

  # --- Nested blocks (provider-specific) ---
  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  # --- Tags ---
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
    Role = "webserver"
  })

  # --- Meta-arguments (available on ALL resources) ---
  depends_on = [aws_internet_gateway.main]   # explicit dependency
  provider   = aws.us_west                   # use non-default provider alias
  count      = 2                             # create 2 copies

  lifecycle {
    prevent_destroy = true                   # refuse to destroy this resource
  }
}
```

---

### Meta-Arguments

```hcl
# Meta-arguments are available on ALL resource types

# 1. depends_on — explicit dependency (use when implicit deps aren't enough)
resource "aws_instance" "app" {
  ami           = "ami-123"
  instance_type = "t3.micro"

  # Explicit dependency — Terraform doesn't know this instance needs the policy
  # because there's no attribute reference between them
  depends_on = [
    aws_iam_role_policy_attachment.app_policy,
    aws_s3_bucket.assets,
  ]
}

# 2. provider — select a non-default provider alias
resource "aws_s3_bucket" "eu_backup" {
  provider = aws.eu_central    # use provider "aws" with alias "eu_central"
  bucket   = "eu-backup-data"
}

# 3. lifecycle — control resource lifecycle behaviour
resource "aws_db_instance" "main" {
  lifecycle {
    create_before_destroy = true   # create new before destroying old (zero-downtime)
    prevent_destroy       = true   # error if plan includes destroying this resource
    ignore_changes        = [      # don't update these attributes even if config changes
      password,                    # DB password managed outside Terraform
      tags["LastModified"],        # auto-updated tag, ignore drift
    ]
    replace_triggered_by = [       # force replacement if these values change
      aws_launch_template.app.latest_version,
    ]
  }
}
```

---

### count vs for_each

```hcl
# --- count: create N identical (or nearly identical) resources ---
resource "aws_instance" "server" {
  count = 3                              # creates server[0], server[1], server[2]

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Use count.index (0, 1, 2) to differentiate
  tags = {
    Name = "server-${count.index}"      # server-0, server-1, server-2
  }
}

# Reference by index
output "first_server_ip" {
  value = aws_instance.server[0].private_ip
}
output "all_server_ips" {
  value = aws_instance.server[*].private_ip  # splat
}

# --- for_each: create resources keyed by a map or set ---
variable "servers" {
  default = {
    web  = { type = "t3.small",  az = "us-east-1a" }
    api  = { type = "t3.medium", az = "us-east-1b" }
    jobs = { type = "t3.large",  az = "us-east-1c" }
  }
}

resource "aws_instance" "server" {
  for_each = var.servers             # creates server["web"], server["api"], server["jobs"]

  ami               = "ami-0c55b159cbfafe1f0"
  instance_type     = each.value.type    # each.value = current map value
  availability_zone = each.value.az

  tags = {
    Name = "server-${each.key}"          # each.key = "web", "api", "jobs"
  }
}

# Reference by key
output "web_server_ip" {
  value = aws_instance.server["web"].private_ip
}

# for_each with a set (keys and values are the same)
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key    # "alice", "bob", "carol"
}
```

**count vs for_each — when to use each:**

| | `count` | `for_each` |
|--|---------|-----------|
| **Use when** | Creating N identical resources | Resources need different configs or names |
| **Index type** | Integer (0, 1, 2...) | String key (stable) |
| **Removal** | Removes and recreates all higher-indexed items | Only removes the specific item |
| **Referencing** | `resource[0]`, `resource[*]` | `resource["key"]` |
| **Risk** | Reordering list = plan shows many replacements | Keyed — safe to reorder |

> ⚠️ **Warning:** Avoid using `count` with a list of strings as indices. If you remove an item from the middle of the list, Terraform will want to recreate all resources after the removed index. Use `for_each` with `toset()` instead.

---

### lifecycle Blocks

```hcl
resource "aws_db_instance" "primary" {
  identifier     = "prod-db"
  engine         = "postgres"
  instance_class = "db.t3.medium"
  password       = var.db_password

  lifecycle {
    # --- create_before_destroy ---
    # Default: destroy old → create new (causes downtime)
    # With this: create new → update references → destroy old
    create_before_destroy = true

    # --- prevent_destroy ---
    # Causes terraform plan to FAIL if this resource would be destroyed
    # Use on databases, stateful resources you never want accidentally deleted
    prevent_destroy = true

    # --- ignore_changes ---
    # Terraform won't update these attributes even if config differs from state
    # Useful for: externally managed values, auto-generated fields
    ignore_changes = [
      password,               # managed by secrets rotation, not Terraform
      tags["LastModified"],   # auto-updated by AWS
      ami,                    # allow ASG to update AMI without Terraform
    ]
    # Ignore ALL changes (use with extreme caution — disables drift detection)
    # ignore_changes = all

    # --- replace_triggered_by ---
    # Force replacement of this resource when a referenced resource/attribute changes
    # Useful for rolling updates of instances when launch template version changes
    replace_triggered_by = [
      aws_launch_template.app.latest_version,
    ]
  }
}

# Precondition and postcondition (Terraform 1.2+)
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      # Validated BEFORE the resource is created/updated
      condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
      error_message = "instance_type must be a t3 type. Got: ${var.instance_type}"
    }

    postcondition {
      # Validated AFTER the resource is created/updated
      condition     = self.public_ip != null
      error_message = "Expected a public IP to be assigned."
    }
  }
}
```

---

### Resource Addressing & References

```hcl
# Resource address format:
# <resource_type>.<local_name>                        (in root module)
# module.<module_name>.<resource_type>.<local_name>   (in module)
# module.<a>.module.<b>.<resource_type>.<local_name>  (nested modules)

# Reference an attribute of a resource
resource "aws_subnet" "web" {
  vpc_id            = aws_vpc.main.id           # reference vpc's id attribute
  cidr_block        = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]
}

# Reference from count-created resources
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Reference from for_each-created resources
resource "aws_route53_record" "app" {
  for_each = aws_instance.server    # iterate over all instances created by for_each

  zone_id = aws_route53_zone.main.zone_id
  name    = "${each.key}.${var.domain}"
  type    = "A"
  ttl     = 300
  records = [each.value.public_ip]  # each.value = the instance object
}

# Reference a module output
resource "aws_instance" "app" {
  subnet_id = module.networking.public_subnet_ids[0]  # from module output
  vpc_security_group_ids = [module.security.web_sg_id]
}
```

---

### Timeouts Block

```hcl
resource "aws_db_instance" "main" {
  identifier     = "prod-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  # Timeouts — how long to wait for operations before failing
  # Available operations depend on the resource type
  timeouts {
    create = "60m"   # wait up to 60 min for create (default: varies)
    update = "80m"   # wait up to 80 min for updates (e.g. restoring from snapshot)
    delete = "40m"   # wait up to 40 min for deletion
  }
}
```

---

## 6. Data Sources

```hcl
# Data source: query existing infrastructure — Terraform doesn't manage it

# --- Fetch the latest Amazon Linux 2 AMI ---
data "aws_ami" "amazon_linux" {
  most_recent = true                  # return the newest matching AMI
  owners      = ["amazon"]            # filter by owner (amazon, self, account ID)

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]  # glob pattern
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use in a resource
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id   # reference data source output
  instance_type = "t3.micro"
}

# --- Fetch an existing VPC ---
data "aws_vpc" "default" {
  default = true    # get the default VPC
}

data "aws_vpc" "prod" {
  id = var.vpc_id   # by specific ID
}

data "aws_vpc" "by_tag" {
  tags = {
    Environment = "production"
    Name        = "prod-vpc"
  }
}

# --- Fetch existing subnets ---
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.prod.id]
  }
  tags = {
    Tier = "private"
  }
}

# --- Fetch a secret from AWS Secrets Manager ---
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db-password"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}

# --- Fetch available AZs ---
data "aws_availability_zones" "available" {
  state = "available"   # only AZs that are operational
}

locals {
  azs = data.aws_availability_zones.available.names   # ["us-east-1a", "us-east-1b", ...]
}

# --- Fetch IAM policy document (generates JSON policy) ---
data "aws_iam_policy_document" "s3_access" {
  statement {
    effect  = "Allow"
    actions = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = ["${aws_s3_bucket.main.arn}/*"]
  }

  statement {
    effect  = "Allow"
    actions = ["s3:ListBucket"]
    resources = [aws_s3_bucket.main.arn]
  }
}

resource "aws_iam_policy" "s3" {
  policy = data.aws_iam_policy_document.s3_access.json   # pre-rendered JSON
}

# --- Fetch DNS zone ---
data "aws_route53_zone" "main" {
  name         = "example.com."   # trailing dot is required
  private_zone = false
}
```

> 💡 **Tip:** Always prefer `data "aws_iam_policy_document"` over `jsonencode()` or heredoc JSON for IAM policies. It has HCL syntax, supports conditions, and is easier to review in PRs.

---

## 7. Variables

### Input Variables

```hcl
# variables.tf — define all input variables for the module/root config

# Simple string variable
variable "aws_region" {
  description = "AWS region to deploy resources in"   # always include description
  type        = string
  default     = "us-east-1"                           # optional default value
}

# Number variable
variable "instance_count" {
  description = "Number of EC2 instances to create"
  type        = number
  default     = 2
}

# Boolean variable
variable "enable_monitoring" {
  description = "Enable detailed monitoring on EC2 instances"
  type        = bool
  default     = false
}

# Required variable (no default — user MUST provide a value)
variable "db_password" {
  description = "Password for the database. Store in .tfvars or pass via TF_VAR_db_password"
  type        = string
  sensitive   = true   # value is redacted in plan/apply output and logs
}

# Variable with validation
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod. Got: '${var.environment}'"
  }
}

# Variable with multiple validations
variable "instance_type" {
  type    = string
  default = "t3.micro"

  validation {
    condition     = startswith(var.instance_type, "t3.") || startswith(var.instance_type, "t4g.")
    error_message = "Must be a t3 or t4g family instance type."
  }

  validation {
    condition     = !contains(["t3.nano"], var.instance_type)  # exclude tiny instances
    error_message = "t3.nano is too small for this workload."
  }
}
```

---

### Variable Files & Precedence

```bash
# Variable definition files:
# terraform.tfvars       — automatically loaded
# *.auto.tfvars          — automatically loaded (alphabetical order)
# custom.tfvars          — must be specified with -var-file flag
```

```hcl
# terraform.tfvars (or *.auto.tfvars) — automatically loaded
aws_region      = "us-east-1"
environment     = "staging"
instance_count  = 3

# Complex types in .tfvars
tags = {
  Team    = "platform"
  Project = "myapp"
}

instance_configs = [
  { type = "t3.small",  az = "us-east-1a" },
  { type = "t3.medium", az = "us-east-1b" },
]
```

**Variable precedence (lowest to highest):**

```
1. Default values in variable {} blocks                 (lowest priority)
2. terraform.tfvars (auto-loaded)
3. terraform.tfvars.json (auto-loaded)
4. *.auto.tfvars files (alphabetical order)
5. -var-file flag                     (right-most -var-file wins)
6. -var flag                          (right-most -var wins)
7. TF_VAR_<name> environment variables (highest priority)
```

```bash
# -var-file: load a specific vars file
terraform plan -var-file="env/prod.tfvars"

# Multiple -var-file flags (later files win on conflicts)
terraform plan \
  -var-file="base.tfvars" \
  -var-file="prod.tfvars" \
  -var-file="secrets.tfvars"

# -var: inline override (wins over all files)
terraform apply -var="instance_count=5" -var="environment=prod"

# TF_VAR_* environment variables
export TF_VAR_db_password="my-secret-password"
export TF_VAR_instance_count=10
```

---

### Output Values

```hcl
# outputs.tf — expose values from a module or root config

# Basic output
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

# Sensitive output (redacted in CLI output, but still stored in state)
output "db_connection_string" {
  description = "Database connection string — treat as secret"
  value       = "postgresql://${var.db_user}:${aws_db_instance.main.password}@${aws_db_instance.main.address}:5432/${var.db_name}"
  sensitive   = true    # redacted in terminal output
}

# Output a complex value
output "public_subnet_ids" {
  description = "IDs of all public subnets"
  value       = [for s in aws_subnet.public : s.id]
}

# Output a map
output "instance_ips" {
  description = "Map of instance name to private IP"
  value       = { for k, v in aws_instance.server : k => v.private_ip }
}

# Output with depends_on (rarely needed — for when output depends on a non-attribute side effect)
output "app_url" {
  value      = "https://${aws_lb.main.dns_name}"
  depends_on = [aws_route53_record.app]   # ensure DNS is ready before outputting
}
```

```bash
# Read outputs from CLI
terraform output                   # show all outputs
terraform output vpc_id            # show specific output value
terraform output -raw vpc_id       # raw value (no quotes — useful in scripts)
terraform output -json             # all outputs as JSON (for CI/CD parsing)
terraform output -json | jq '.vpc_id.value'  # pipe to jq for scripting
```

---

### locals vs Variables

```hcl
# Use VARIABLES for:
# - Values that change between environments or deployments
# - Values the caller (user, CI, parent module) needs to configure
# - Values that should be documented as module inputs

variable "environment" { type = string }
variable "instance_type" { type = string }

# Use LOCALS for:
# - Derived/computed values (not directly configurable)
# - Avoiding repetition within a module
# - Encapsulating complex expressions
# - Building tag maps, name prefixes, CIDR ranges

locals {
  # Derived from variables — not directly configurable
  is_prod     = var.environment == "production"
  name_prefix = "myapp-${var.environment}"

  # Complex computation (cleaner as a local)
  azs         = slice(data.aws_availability_zones.available.names, 0, 3)

  # Reusable expression
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

---

### Complex Types & Validation

```hcl
# Complex type variables with full annotation

variable "subnets" {
  description = "List of subnet configurations"
  type = list(object({
    name       = string
    cidr_block = string
    az         = string
    public     = bool
  }))
  default = [
    { name = "web-a", cidr_block = "10.0.1.0/24", az = "us-east-1a", public = true  },
    { name = "web-b", cidr_block = "10.0.2.0/24", az = "us-east-1b", public = true  },
    { name = "app-a", cidr_block = "10.0.11.0/24", az = "us-east-1a", public = false },
  ]
}

variable "alarm_thresholds" {
  description = "Map of metric name to alarm threshold"
  type        = map(number)
  default     = {
    cpu_percent    = 80
    memory_percent = 90
    disk_percent   = 85
  }
}

# Nested object with optional fields (Terraform 1.3+)
variable "database_config" {
  type = object({
    engine          = string
    engine_version  = string
    instance_class  = string
    storage_gb      = number
    multi_az        = optional(bool, false)         # optional with default
    backup_days     = optional(number, 7)
    deletion_protection = optional(bool, true)
  })
}

# Using complex variable in resources
resource "aws_subnet" "all" {
  for_each = { for s in var.subnets : s.name => s }  # convert list to map by name

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az
  map_public_ip_on_launch = each.value.public

  tags = { Name = each.key }
}
```

---

## 8. State Management

### What State Contains

```json
// terraform.tfstate — JSON file (simplified example)
{
  "version": 4,
  "terraform_version": "1.7.5",
  "serial": 42,              // increments on each apply
  "lineage": "abc123...",    // unique ID for this state file
  "outputs": {
    "vpc_id": { "value": "vpc-0abc123", "type": "string" }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [{
        "schema_version": 1,
        "attributes": {
          "id":         "vpc-0abc123def456",
          "cidr_block": "10.0.0.0/16",
          "arn":        "arn:aws:ec2:us-east-1:123456789:vpc/vpc-0abc123"
          // ... all resource attributes
        }
      }]
    }
  ]
}
```

---

### Remote Backends

```hcl
# Always use a remote backend in team/production environments
# Remote backends provide: shared state, locking, versioning

# --- AWS S3 + DynamoDB (most common) ---
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"    # S3 bucket (must exist first)
    key            = "prod/network/terraform.tfstate"  # path within bucket
    region         = "us-east-1"

    encrypt        = true                           # encrypt state at rest
    dynamodb_table = "terraform-state-lock"         # DynamoDB table for locking
    # Table must have a String hash key named "LockID"

    # Optional: assume role for cross-account state
    role_arn       = "arn:aws:iam::123456789:role/TerraformStateRole"
  }
}

# --- GCP GCS ---
terraform {
  backend "gcs" {
    bucket      = "mycompany-terraform-state"
    prefix      = "prod/network"       # equivalent to S3 key prefix
    # GCS handles locking automatically via object versioning
  }
}

# --- Azure Blob Storage ---
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "myterraformstate"
    container_name       = "tfstate"
    key                  = "prod.network.tfstate"
  }
}

# --- Terraform Cloud / HCP Terraform ---
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-network"    # specific workspace
      # OR: use tags to match multiple workspaces
      # tags = ["prod", "network"]
    }
  }
}
```

```bash
# Create the S3 bucket and DynamoDB table for state (one-time setup)
# Often done with a separate "bootstrap" Terraform config or CLI commands

aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

---

### State Locking

State locking prevents concurrent operations that could corrupt state.

```
Developer A runs: terraform apply    →  Acquires lock on DynamoDB/GCS
Developer B runs: terraform apply    →  Error: state is locked (A is running)
                                         LockID: abc123, owner: alice@example.com
Developer A finishes               →  Releases lock
Developer B runs again             →  Acquires lock, proceeds
```

```bash
# If Terraform crashes mid-apply, the lock may remain
# Force-unlock (use with caution — ensure no apply is actually running)
terraform force-unlock <LOCK_ID>

# Example:
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        abc-123-def-456
#   Path:      mycompany-terraform-state/prod/network/terraform.tfstate
#   Operation: OperationTypeApply
#   Who:       alice@example.com
#   Created:   2024-03-15T10:30:00Z

terraform force-unlock abc-123-def-456
```

---

### terraform state Commands

```bash
# List all resources in state
terraform state list
# aws_instance.web
# aws_vpc.main
# aws_subnet.public[0]
# module.network.aws_subnet.private["us-east-1a"]

# Show detailed state of a specific resource
terraform state show aws_instance.web
terraform state show 'aws_subnet.public[0]'        # quote brackets
terraform state show 'module.network.aws_vpc.main'  # in a module

# Move a resource in state (rename without destroying/recreating)
# Use when: renaming a resource, moving it into/out of a module
terraform state mv aws_instance.old_name aws_instance.new_name
terraform state mv 'aws_instance.web' 'module.compute.aws_instance.web'

# Remove a resource from state (WITHOUT destroying it)
# Use when: you want Terraform to forget about a resource (stop managing it)
terraform state rm aws_instance.web
terraform state rm 'module.old_module.aws_instance.web'

# Pull current state from remote backend to stdout (as JSON)
terraform state pull > backup.tfstate

# Push a local state file to remote backend (DANGEROUS — overwrites remote!)
terraform state push backup.tfstate
```

> ⚠️ **Warning:** `terraform state push` overwrites the remote state. Only use this for disaster recovery after taking a backup. Incorrect state manipulation can cause Terraform to delete real infrastructure.

---

### terraform import

Bring existing infrastructure under Terraform management without recreating it.

```bash
# Import syntax: terraform import <resource_address> <real_resource_id>
terraform import aws_instance.web i-0abc123def456789
terraform import aws_vpc.main vpc-0123456789abcdef0
terraform import 'aws_subnet.public[0]' subnet-abc123      # with index
terraform import 'aws_instance.server["web"]' i-abc123    # with for_each key

# Module imports
terraform import 'module.network.aws_vpc.main' vpc-0123456789

# Import block (Terraform 1.5+ — declarative import in .tf files)
import {
  id = "i-0abc123def456789"                    # real cloud resource ID
  to = aws_instance.web                         # Terraform resource address
}

# After import, Terraform generates the config (1.6+ with -generate-config-out)
terraform plan -generate-config-out=generated.tf

# Workflow:
# 1. Write resource block with required attributes
# 2. Run terraform import
# 3. Run terraform plan — check for any drift (should show 0 changes ideally)
# 4. Update config to match any attributes shown as changing
```

> 💡 **Tip:** Terraform 1.5+ `import` blocks are the preferred approach — they're reviewable in PRs and can be checked in CI before applying. The `terraform import` CLI command is still useful for quick one-off imports.

---

### Sensitive Data in State

```bash
# State stores ALL resource attributes — including passwords, keys, connection strings
# Protect state with:

# 1. Enable encryption at rest for your backend
#    S3: SSE-S3 or SSE-KMS encryption
#    GCS: Default Google-managed encryption (or CMEK)
#    Azure: Storage service encryption

# 2. Restrict access to the state bucket/container
#    Only Terraform CI/CD role + admins should have read access

# 3. Enable versioning on the state bucket
#    Allows rollback if state is corrupted

# 4. Never commit terraform.tfstate to git
echo "*.tfstate" >> .gitignore
echo "*.tfstate.backup" >> .gitignore
echo ".terraform/" >> .gitignore

# 5. Use sensitive = true on outputs and variables containing secrets
#    (redacts value in CLI output, but still stored in state)

# 6. Consider using Vault or Secrets Manager for secrets,
#    fetched at runtime via data sources (not stored as Terraform state)
```

---

## 9. Modules

### Module Sources

```hcl
# Local path (relative)
module "networking" {
  source = "./modules/networking"
}

# Local path (sibling directory)
module "shared" {
  source = "../shared-modules/logging"
}

# Terraform Registry (public)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"           # always pin version for Registry modules
}

# Git (HTTPS)
module "internal" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/networking"
  # Double slash (//) separates repo URL from subdirectory path
}

# Git with specific ref (tag, branch, or commit)
module "internal_pinned" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/networking?ref=v1.2.0"
}

# Git (SSH)
module "private" {
  source = "git::git@github.com:myorg/terraform-modules.git//modules/networking?ref=main"
}

# S3 bucket
module "s3_module" {
  source = "s3::https://s3.amazonaws.com/my-tf-modules/networking.zip"
}

# GCS bucket
module "gcs_module" {
  source = "gcs::https://www.googleapis.com/storage/v1/my-tf-modules/networking.zip"
}
```

---

### Module Block

```hcl
# Calling (instantiating) a module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # required
  version = "~> 5.0"                          # required for registry modules

  # Pass input variables to the module
  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"

  azs             = local.azs
  private_subnets = local.private_cidrs
  public_subnets  = local.public_cidrs

  enable_nat_gateway = true
  single_nat_gateway = !local.is_production   # single NAT for non-prod (cheaper)
  one_nat_gateway_per_az = local.is_production # HA NAT for prod

  tags = local.common_tags
}

# Consuming module outputs
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.small"
  subnet_id     = module.vpc.private_subnets[0]   # module output
  vpc_security_group_ids = [module.security_groups.app_sg_id]
}

output "vpc_id" {
  value = module.vpc.vpc_id     # pass module output to root output
}

# Module with explicit provider passing
module "eu_resources" {
  source = "./modules/regional"

  providers = {
    aws = aws.eu_central    # pass aliased provider to module
  }
}
```

---

### Root vs Child Modules

```
Root Module (the directory you run terraform commands in)
│
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
│
├── module "vpc"  ───────────▶  ./modules/networking/   (child module)
│                                 ├── main.tf
│                                 ├── variables.tf
│                                 └── outputs.tf
│
├── module "compute" ────────▶  ./modules/compute/       (child module)
│                                 └── module "asg" ────▶  (nested child)
│
└── module "database" ───────▶  terraform-aws-modules/rds/aws  (registry)
```

---

### Module Structure Conventions

```
modules/networking/
├── main.tf          # primary resources
├── variables.tf     # all input variables with descriptions
├── outputs.tf       # all outputs with descriptions
├── versions.tf      # required_providers and terraform version constraint
├── locals.tf        # local value computations
├── data.tf          # data sources
└── README.md        # module documentation (generate with terraform-docs)
```

```hcl
# modules/networking/outputs.tf — expose module outputs
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "List of IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "List of IDs of public subnets"
  value       = aws_subnet.public[*].id
}
```

> 💡 **Tip:** Every module should have a `versions.tf` with `required_providers` and a minimum `required_version` for Terraform. This ensures the module fails fast with a clear error if run on an incompatible version, rather than with a cryptic provider error.

---

## 10. Workspaces

```hcl
# Workspaces allow multiple state files from the same configuration
# Each workspace has its own terraform.tfstate

# Use workspace name in config to differentiate resources
locals {
  env           = terraform.workspace        # "default", "staging", "prod"
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  replica_count = terraform.workspace == "prod" ? 3 : 1
}

resource "aws_instance" "app" {
  instance_type = local.instance_type
  tags = {
    Environment = terraform.workspace
  }
}
```

```bash
# Workspace commands
terraform workspace new staging      # create and switch to new workspace
terraform workspace new prod
terraform workspace list             # list all workspaces (* = current)
# * default
#   staging
#   prod
terraform workspace select staging   # switch to existing workspace
terraform workspace show             # show current workspace name
terraform workspace delete staging   # delete workspace (must have no state or use -force)

# State is stored per workspace:
# Local:  terraform.tfstate.d/<workspace>/terraform.tfstate
# S3:     <bucket>/<key_prefix>/<workspace>/terraform.tfstate
```

**Workspaces vs separate directories:**

| | Workspaces | Separate Directories |
|--|-----------|---------------------|
| **Same config** | ✅ Reuses same .tf files | ❌ Must copy or use modules |
| **Different configs** | ❌ Same code path | ✅ Completely independent |
| **Isolation** | Partial (same backend, different state) | Full (separate backend) |
| **Best for** | Same infra, different scale | Environments with different architecture |
| **Risk** | Easy to apply to wrong workspace | Low — entirely separate |

> ⚠️ **Warning:** Workspaces are NOT a good fit for environment isolation (dev/staging/prod) when environments have meaningfully different configurations, different IAM permissions, or different AWS accounts. Use separate directories + backends for true environment isolation.

---

## 11. CLI Commands

```bash
# --- terraform init ---
terraform init                          # initialise working directory
terraform init -upgrade                 # upgrade providers to latest allowed version
terraform init -reconfigure             # reconfigure backend (ignore existing state)
terraform init -migrate-state           # move state to newly configured backend
terraform init -backend=false           # skip backend init (only install providers)
terraform init -backend-config="key=value"  # pass backend config (partial config pattern)

# --- terraform validate ---
terraform validate                      # check HCL syntax and internal consistency
# Does NOT check against real APIs — fast, offline check

# --- terraform fmt ---
terraform fmt                           # format all .tf files in current directory
terraform fmt -recursive                # format recursively (all subdirectories)
terraform fmt -check                    # exit non-zero if any files need formatting (CI use)
terraform fmt -diff                     # show diff of formatting changes
terraform fmt -write=false              # print formatted output, don't write files

# --- terraform plan ---
terraform plan                          # show execution plan
terraform plan -out=tfplan              # save plan to file (apply exactly this later)
terraform plan -var="env=prod"          # inline variable override
terraform plan -var-file="prod.tfvars" # load variables from file
terraform plan -target=aws_instance.web # plan only this resource (and its deps)
terraform plan -refresh-only            # only check for drift (don't plan changes)
terraform plan -destroy                 # show destruction plan

# --- terraform apply ---
terraform apply                         # apply changes (prompts for confirmation)
terraform apply -auto-approve           # skip confirmation (CI/CD use only)
terraform apply tfplan                  # apply a saved plan file (no prompt)
terraform apply -target=aws_instance.web  # apply only specific resource
terraform apply -var="env=prod"         # with variable override
terraform apply -replace=aws_instance.web  # force replacement of specific resource
terraform apply -refresh=false          # skip refreshing state (faster, less safe)
terraform apply -parallelism=20         # number of concurrent operations (default: 10)

# --- terraform destroy ---
terraform destroy                       # destroy all managed resources (prompts)
terraform destroy -auto-approve         # skip confirmation ⚠️
terraform destroy -target=aws_instance.web  # destroy only specific resource

# --- terraform output ---
terraform output                        # show all root module outputs
terraform output vpc_id                 # show specific output
terraform output -raw vpc_id            # output raw value (no quotes — for scripting)
terraform output -json                  # all outputs as JSON
terraform output -json | jq -r '.vpc_id.value'   # pipe to jq

# --- terraform console ---
# Interactive REPL for evaluating Terraform expressions
# Has access to current state and config
terraform console
# > cidrsubnet("10.0.0.0/16", 8, 1)
# "10.0.1.0/24"
# > length(var.subnets)
# 3
# > aws_vpc.main.id
# "vpc-0abc123"

# --- terraform graph ---
terraform graph                         # DOT-format dependency graph to stdout
terraform graph | dot -Tsvg > graph.svg # render as SVG (requires graphviz)
terraform graph -type=plan              # graph for planned changes only

# --- terraform taint / untaint (DEPRECATED — use -replace) ---
# terraform taint aws_instance.web      # marks for replacement on next apply
# terraform untaint aws_instance.web    # remove taint mark
# Use instead:
terraform apply -replace=aws_instance.web   # force replacement without tainting

# --- terraform providers ---
terraform providers                     # show provider requirements and source addresses
terraform providers lock                # update lock file for current providers
terraform providers lock -platform=linux_amd64  # add platform-specific hashes
terraform providers mirror ./mirror/    # download providers to local directory

# --- terraform version ---
terraform version                       # show Terraform + provider versions
```

---

## 12. Expressions & Functions — Deep Dive

```hcl
# --- for expressions with if filtering ---
locals {
  all_instances = {
    web  = { type = "t3.small",  env = "prod" }
    api  = { type = "t3.medium", env = "prod" }
    dev  = { type = "t3.micro",  env = "dev"  }
  }

  # Filter: keep only prod instances
  prod_instances = {
    for k, v in local.all_instances : k => v if v.env == "prod"
  }
  # Result: { web = {...}, api = {...} }

  # Transform + filter in one expression
  prod_types = [
    for k, v in local.all_instances : v.type if v.env == "prod"
  ]
  # Result: ["t3.small", "t3.medium"]
}

# --- Nested for expressions ---
variable "regions" {
  default = ["us-east-1", "us-west-2"]
}
variable "envs" {
  default = ["dev", "prod"]
}

locals {
  # Cartesian product using setproduct
  region_env_combos = [
    for pair in setproduct(var.regions, var.envs) : {
      region = pair[0]
      env    = pair[1]
      name   = "${pair[1]}-${pair[0]}"
    }
  ]
  # Result: [{region="us-east-1", env="dev", name="dev-us-east-1"}, ...]
}

# --- flatten() for nested lists ---
variable "team_users" {
  default = {
    platform = ["alice", "bob"]
    backend  = ["carol", "dave", "eve"]
  }
}

locals {
  # Flatten nested list of lists into single list
  all_users = flatten([for team, users in var.team_users : users])
  # Result: ["alice", "bob", "carol", "dave", "eve"]

  # Flatten with transformation
  user_team_pairs = flatten([
    for team, users in var.team_users : [
      for user in users : { user = user, team = team }
    ]
  ])
  # Result: [{user="alice",team="platform"}, {user="bob",team="platform"}, ...]
}

# Use flatten to create for_each resources from nested structure
resource "aws_iam_user_group_membership" "membership" {
  for_each = {
    for pair in local.user_team_pairs : "${pair.user}-${pair.team}" => pair
  }
  user   = each.value.user
  groups = [each.value.team]
}

# --- merge() for combining maps ---
locals {
  base_tags = { ManagedBy = "terraform", Project = "myapp" }
  env_tags  = { Environment = var.environment }
  svc_tags  = { Service = "api", Version = var.app_version }

  # Merge multiple maps — rightmost key wins on conflict
  all_tags  = merge(local.base_tags, local.env_tags, local.svc_tags)
}

# --- lookup() — safe map access with default ---
locals {
  instance_types = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
  # lookup(map, key, default) — returns default if key not found
  instance = lookup(local.instance_types, var.environment, "t3.micro")
}

# --- try() — suppress errors from failed expressions ---
locals {
  # try() evaluates expressions left-to-right, returning the first non-error
  # Useful for optional/nullable attributes
  db_port = try(var.database_config.port, 5432)         # use port if set, else 5432
  tag_val = try(aws_instance.web.tags["Name"], "unknown") # handle missing tag

  # can() — returns true if expression would succeed without error
  has_port  = can(var.database_config.port)
}

# --- coalesce() and coalescelist() ---
locals {
  # coalesce() returns first non-null, non-empty argument
  region   = coalesce(var.override_region, var.aws_region, "us-east-1")

  # coalescelist() returns first non-empty list
  subnets  = coalescelist(var.explicit_subnets, data.aws_subnets.default.ids)
}

# --- one() — extract single item from single-element collection ---
locals {
  # Returns null if empty, errors if more than 1 element
  the_vpc_id = one(data.aws_vpcs.selected.ids)  # ensures exactly 0 or 1 VPCs matched
}

# --- zipmap() — build a map from two lists ---
locals {
  names  = ["web", "api", "db"]
  ids    = ["i-111", "i-222", "i-333"]
  name_to_id = zipmap(local.names, local.ids)
  # Result: { web = "i-111", api = "i-222", db = "i-333" }
}

# --- setproduct() — cartesian product ---
locals {
  envs     = ["dev", "staging", "prod"]
  services = ["api", "worker", "cron"]

  all_combos = setproduct(local.envs, local.services)
  # [["dev","api"], ["dev","worker"], ..., ["prod","cron"]]
}

# --- range() --- generate number sequences
locals {
  nums_0_to_4  = range(5)           # [0, 1, 2, 3, 4]
  nums_1_to_5  = range(1, 6)        # [1, 2, 3, 4, 5]
  even_0_to_8  = range(0, 10, 2)    # [0, 2, 4, 6, 8]  (step=2)
}

# --- toset(), tolist(), tomap() ---
locals {
  with_dupes = ["a", "b", "a", "c"]
  unique_set = toset(local.with_dupes)   # {"a", "b", "c"} — deduplicates
  back_list  = tolist(local.unique_set)  # ["a", "b", "c"] — ordered list from set
}

# --- Regular expressions ---
locals {
  version     = "v1.23.456"
  # regex() returns first match (errors if no match)
  semver      = regex("v(\\d+)\\.(\\d+)\\.(\\d+)", local.version)
  major       = local.semver[0]     # "1"
  minor       = local.semver[1]     # "23"

  # regexall() returns all matches as a list
  all_ips     = regexall("\\d+\\.\\d+\\.\\d+\\.\\d+", "IPs: 10.0.0.1 and 192.168.1.1")
  # ["10.0.0.1", "192.168.1.1"]

  # replace() with regex
  sanitized   = replace("my app name!", "/[^a-z0-9-]/", "-")  # "my-app-name-"
}

# --- CIDR functions ---
locals {
  vpc_cidr    = "10.0.0.0/16"

  # cidrsubnet(prefix, newbits, netnum) — carve out subnets
  # newbits: how many bits to add to the prefix
  # netnum:  subnet number
  public_a  = cidrsubnet(local.vpc_cidr, 8, 1)    # 10.0.1.0/24
  public_b  = cidrsubnet(local.vpc_cidr, 8, 2)    # 10.0.2.0/24
  private_a = cidrsubnet(local.vpc_cidr, 8, 11)   # 10.0.11.0/24
  private_b = cidrsubnet(local.vpc_cidr, 8, 12)   # 10.0.12.0/24

  # Generate N subnets programmatically
  public_subnets  = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i + 1)]
  private_subnets = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i + 11)]

  # cidrhost() — get specific host in a CIDR
  gateway_ip = cidrhost(local.public_a, 1)    # "10.0.1.1"
  broadcast  = cidrhost(local.public_a, -1)   # "10.0.1.255"
}
```

---

## 13. Backends & Remote State

```hcl
# --- Full S3 + DynamoDB backend ---
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "environments/prod/network/terraform.tfstate"
    region         = "us-east-1"

    encrypt        = true                            # AES-256 server-side encryption
    dynamodb_table = "terraform-state-lock"          # for state locking
    kms_key_id     = "arn:aws:kms:us-east-1:..."    # optional KMS key

    # IAM role to assume for state operations
    role_arn       = "arn:aws:iam::123456789:role/TerraformStateRole"
    session_name   = "TerraformStateProd"
  }
}

# --- Full GCS backend ---
terraform {
  backend "gcs" {
    bucket      = "mycompany-terraform-state"
    prefix      = "environments/prod/network"
    credentials = "service-account.json"    # or use GOOGLE_APPLICATION_CREDENTIALS env
  }
}

# --- Full Azure backend ---
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "myterraformstate12345"   # globally unique
    container_name       = "tfstate"
    key                  = "prod.network.tfstate"
    use_azuread_auth     = true                      # use Azure AD auth (recommended)
  }
}

# --- HTTP backend (generic) ---
terraform {
  backend "http" {
    address        = "https://my-state-server.example.com/state/myapp"
    lock_address   = "https://my-state-server.example.com/lock/myapp"
    unlock_address = "https://my-state-server.example.com/lock/myapp"
    username       = "terraform"
    password       = var.state_server_password   # ⚠️ vars not allowed in backend config
    # Use environment variables instead:
    # TF_HTTP_USERNAME and TF_HTTP_PASSWORD
  }
}

# --- Partial backend configuration ---
# Leave sensitive/environment-specific values out of the backend block
# and pass them at init time (common in CI/CD)
terraform {
  backend "s3" {
    # Only static, non-sensitive values here
    region = "us-east-1"
    encrypt = true
    dynamodb_table = "terraform-state-lock"
    # bucket and key are passed via -backend-config flag or file
  }
}
```

```bash
# Pass remaining backend config at init time
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="key=prod/network/terraform.tfstate"

# Or from a file:
terraform init -backend-config=backend-prod.hcl
```

```hcl
# backend-prod.hcl (NOT a .tf file — passed to -backend-config)
bucket = "mycompany-terraform-state"
key    = "environments/prod/network/terraform.tfstate"
```

```hcl
# --- terraform_remote_state: read outputs from another state file ---
# Cross-stack references pattern: read outputs from a "foundation" stack

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "environments/prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use outputs from the remote state
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.app_sg_id]
}

# --- State migration between backends ---
# 1. Update the backend block in your config
# 2. Run: terraform init -migrate-state
#    Terraform will copy state from old backend to new backend
# 3. Verify: terraform state list (should show same resources)
# 4. Remove old state from previous backend manually (after confirming)
```

---

## 14. Provisioners

> ⚠️ **Warning:** Provisioners are a **last resort**. They add significant complexity, reduce idempotency, and can fail in ways Terraform can't recover from automatically. Always prefer cloud-native alternatives (user_data, cloud-init, Packer images, SSM Run Command).

```hcl
# --- local-exec: run commands on the Terraform executor machine ---
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  provisioner "local-exec" {
    # Run on the machine running terraform (not the remote instance)
    command = "echo ${self.public_ip} >> known_hosts.txt"

    # Optional settings
    working_dir = "/tmp"
    interpreter = ["bash", "-c"]        # default on Unix
    environment = {
      INSTANCE_IP = self.public_ip
      ENV         = var.environment
    }
  }

  # local-exec on destroy
  provisioner "local-exec" {
    when    = destroy                    # only run on resource destruction
    command = "echo 'Removing ${self.id} from inventory'"
    on_failure = continue               # continue even if command fails
    # on_failure = fail (default)       # fail the plan if command fails
  }
}

# --- connection block: configure SSH/WinRM for remote provisioners ---
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deploy.key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/terraform-key.pem")
    host        = self.public_ip
    timeout     = "5m"                  # wait up to 5 min for SSH to be ready
  }

  # --- remote-exec: run commands on the remote resource ---
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
      "sudo systemctl start nginx",
      "sudo systemctl enable nginx",
    ]
  }

  # --- file provisioner: copy a file or directory to the remote ---
  provisioner "file" {
    source      = "config/nginx.conf"   # local path
    destination = "/tmp/nginx.conf"     # remote path
  }

  provisioner "remote-exec" {
    inline = ["sudo cp /tmp/nginx.conf /etc/nginx/nginx.conf"]
  }
}

# --- WinRM connection (Windows instances) ---
resource "aws_instance" "windows" {
  ami           = "ami-windows-2022"
  instance_type = "t3.medium"

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = rsadecrypt(self.password_data, file("~/.ssh/key.pem"))
    host     = self.public_ip
    https    = true
    insecure = true
    timeout  = "10m"
  }

  provisioner "remote-exec" {
    inline = ["powershell -Command Set-ExecutionPolicy RemoteSigned -Force"]
  }
}

# --- Better alternatives to provisioners ---
resource "aws_instance" "better_way" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # user_data: runs at first boot, managed by the OS, no SSH needed
  user_data = templatefile("${path.module}/templates/bootstrap.sh.tpl", {
    db_host   = aws_db_instance.main.address
    log_level = var.log_level
  })

  # Even better: bake software into custom AMI with Packer
  # Then just reference the pre-baked AMI — no provisioners at all
}
```

---

## 15. Testing & Validation

### terraform validate & plan

```bash
# Syntax and configuration validation (fast, offline)
terraform validate
# Success: The configuration is valid.
# Error: An argument named "instanec_type" is not expected here.

# Format check in CI
terraform fmt -check -recursive

# Dry run — plan shows what WOULD happen
terraform plan -out=tfplan
# Plan: 3 to add, 1 to change, 0 to destroy.
```

---

### Validation, Preconditions & Postconditions

```hcl
# Variable validation (Terraform 0.13+)
variable "cidr_block" {
  type = string

  validation {
    # condition must evaluate to true; false triggers error_message
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "cidr_block must be a valid IPv4 CIDR notation (e.g. '10.0.0.0/16')."
  }

  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) <= 24
    error_message = "CIDR block must be /24 or larger (prefix length <= 24)."
  }
}

# Precondition — evaluated before resource create/update
# Postcondition — evaluated after resource create/update
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "The AMI ${var.ami_id} must be x86_64 architecture."
    }

    postcondition {
      condition     = self.state == "running"
      error_message = "Instance did not reach running state. Got: ${self.state}"
    }
  }
}

# check block (Terraform 1.5+) — continuous assertions, non-blocking
# Unlike preconditions, a failed check produces a WARNING not an error
check "health_check" {
  data "http" "app_health" {
    url = "https://${aws_lb.main.dns_name}/health"
  }

  assert {
    condition     = data.http.app_health.status_code == 200
    error_message = "Application health check failed. Status: ${data.http.app_health.status_code}"
  }
}

check "certificate_expiry" {
  data "tls_certificate" "app" {
    url = "https://${aws_lb.main.dns_name}"
  }

  assert {
    condition     = timecmp(data.tls_certificate.app.certificates[0].not_after, timeadd(timestamp(), "720h")) > 0
    error_message = "TLS certificate expires within 30 days — renew immediately!"
  }
}
```

---

### Testing Ecosystem

```bash
# --- Terratest (Go integration tests) ---
# Install Go, then:
go mod init test
go get github.com/gruntwork-io/terratest/modules/terraform

# test/vpc_test.go (example Terratest)
# func TestVpc(t *testing.T) {
#   opts := &terraform.Options{TerraformDir: "../"}
#   defer terraform.Destroy(t, opts)
#   terraform.InitAndApply(t, opts)
#   vpcId := terraform.Output(t, opts, "vpc_id")
#   assert.NotEmpty(t, vpcId)
# }
go test -v -timeout 30m ./test/...

# --- tflint (linting) ---
brew install tflint
tflint --init                     # download rule plugins
tflint                            # lint current directory
tflint --recursive                # lint all subdirectories

# .tflint.hcl config:
# plugin "aws" {
#   enabled = true
#   version = "0.27.0"
#   source  = "github.com/terraform-linters/tflint-ruleset-aws"
# }

# --- checkov (security scanning) ---
pip install checkov
checkov -d .                      # scan current directory
checkov -f main.tf                # scan specific file
checkov --framework terraform     # specify framework
checkov -d . --compact            # compact output

# --- tfsec (security scanning) ---
brew install tfsec
tfsec .                          # scan for security issues
tfsec . --minimum-severity HIGH  # only show high+ severity

# --- terraform-docs (documentation generation) ---
brew install terraform-docs
terraform-docs markdown . > README.md     # generate README from module
terraform-docs markdown table . --output-file README.md  # table format

# --- Infracost (cost estimation) ---
brew install infracost
infracost auth login
infracost breakdown --path .      # estimate monthly cost of current config
infracost diff --path . --compare-to baseline.json  # diff cost changes
```

---

## 16. Best Practices & Patterns

### Repository Structure

```
# Option 1: Environment directories (recommended for most teams)
infrastructure/
├── modules/                    # reusable modules
│   ├── networking/
│   ├── compute/
│   └── database/
├── environments/
│   ├── dev/
│   │   ├── main.tf             # calls modules, dev-specific config
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── terraform.tfvars    # dev-specific values
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars    # prod-specific values (no secrets)
└── scripts/                    # helper scripts, bootstrap, CI tools

# Option 2: Workspace-based (simpler, same config, different state)
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── locals.tf
├── terraform.tfvars            # shared defaults
├── dev.tfvars                  # -var-file=dev.tfvars
└── prod.tfvars                 # -var-file=prod.tfvars
```

---

### File Organisation Conventions

```
# Standard module/root file layout:
main.tf         # primary resource definitions
variables.tf    # all input variable declarations
outputs.tf      # all output value declarations
versions.tf     # terraform{} block: required_version + required_providers
locals.tf       # locals {} blocks (computed/derived values)
data.tf         # data source declarations
providers.tf    # provider {} configuration blocks (for multi-provider configs)

# Optional:
iam.tf          # IAM resources (policies, roles, attachments)
security.tf     # security groups, NACLs, KMS keys
dns.tf          # Route53 / DNS records
monitoring.tf   # CloudWatch, alerts, dashboards
```

---

### Naming Conventions

```hcl
# Resource local names: lowercase, underscores (snake_case)
resource "aws_vpc" "main" {}              # ✅
resource "aws_vpc" "MainVPC" {}           # ❌ avoid CamelCase
resource "aws_vpc" "main-vpc" {}          # ❌ avoid hyphens (use underscores)

# Variable names: lowercase, underscores
variable "instance_type" {}               # ✅
variable "InstanceType" {}                # ❌

# Output names: lowercase, underscores, descriptive
output "vpc_id" {}                        # ✅
output "public_subnet_ids" {}             # ✅
output "db_connection_string" {}          # ✅

# Module names: lowercase, underscores or hyphens
module "networking" {}                    # ✅
module "vpc_networking" {}               # ✅

# Cloud resource names (what shows up in AWS console):
# Use: ${var.project}-${var.environment}-${purpose}
# e.g: myapp-prod-api, myapp-staging-db, myapp-dev-vpc

locals {
  name_prefix = "${var.project}-${var.environment}"
  vpc_name    = "${local.name_prefix}-vpc"
  db_name     = "${local.name_prefix}-${var.db_identifier}"
}
```

---

### Tagging Strategy

```hcl
# locals.tf — centralised tag management
locals {
  # Base tags applied to everything
  base_tags = {
    ManagedBy   = "terraform"
    Repository  = "github.com/myorg/infrastructure"
    Project     = var.project
    Environment = var.environment
    CostCenter  = var.cost_center
    Owner       = var.team
  }

  # Function-specific tag additions
  network_tags = merge(local.base_tags, {
    Layer = "network"
  })

  compute_tags = merge(local.base_tags, {
    Layer = "compute"
    PatchGroup = "standard"
  })
}

# Apply at provider level (AWS default_tags)
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = local.base_tags   # applied to ALL resources automatically
  }
}

# Resource-specific tags (merged with default_tags)
resource "aws_instance" "web" {
  tags = {
    Name = "${local.name_prefix}-web"    # instance-specific tags
    Role = "webserver"
  }
}
```

---

### CI/CD Integration

```yaml
# GitHub Actions workflow for Terraform
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: environments/prod

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5       # pin exact version in CI

      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        if: github.event_name == 'pull_request'

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

```bash
# Atlantis (GitOps for Terraform — runs plan/apply from PRs)
# atlantis.yaml in repo root:
# version: 3
# projects:
#   - name: prod-network
#     dir: environments/prod/network
#     workspace: default
#     autoplan:
#       when_modified: ["*.tf", "*.tfvars", "../../../modules/**/*.tf"]
#       enabled: true

# GitLab CI (.gitlab-ci.yml)
# Plan on MR, Apply on merge to main — similar pattern to GitHub Actions above
```

---

### Key Best Practices

```hcl
# 1. Pin versions — always
terraform {
  required_version = ">= 1.5.0, < 2.0.0"   # never use no constraint
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"   # minor-pinned
    }
  }
}

# 2. Use sensitive = true for secrets
variable "db_password" {
  sensitive = true   # redacted in output
}

output "db_connection" {
  sensitive = true   # redacted in output
  value     = "...${var.db_password}..."
}

# 3. Don't use -target regularly
# -target is for emergencies only — it can cause state drift where
# some resources are managed separately from the rest.
# Always apply the full configuration.

# 4. Use prevent_destroy on critical resources
resource "aws_s3_bucket" "data_lake" {
  lifecycle {
    prevent_destroy = true   # requires explicit removal of this block to delete
  }
}

# 5. Separate state per environment — never share state across envs
# environments/dev/  → dev state bucket / prefix
# environments/prod/ → prod state bucket / prefix

# 6. Never run terraform in CI without a plan review step on PRs
# Plan on PR → human reviews → merge → apply

# 7. Use data sources instead of hardcoded IDs
data "aws_ami" "amazon_linux" {         # ✅ dynamic lookup
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name"; values = ["amzn2-ami-hvm-*"] }
}
# ami = "ami-0c55b159cbfafe1f0"         # ❌ hardcoded — becomes stale
```

---

## 17. Quick Reference Tables

### All CLI Commands

| Command | Description | Key Flags |
|---------|-------------|-----------|
| `terraform init` | Initialise working directory | `-upgrade`, `-reconfigure`, `-migrate-state`, `-backend-config` |
| `terraform validate` | Validate HCL syntax | _(none commonly used)_ |
| `terraform fmt` | Format .tf files | `-check`, `-recursive`, `-diff` |
| `terraform plan` | Preview changes | `-out`, `-var`, `-var-file`, `-target`, `-refresh-only`, `-destroy` |
| `terraform apply` | Apply changes | `-auto-approve`, `-target`, `-replace`, `-parallelism` |
| `terraform destroy` | Destroy all resources | `-auto-approve`, `-target` |
| `terraform output` | Show outputs | `-json`, `-raw` |
| `terraform console` | Interactive REPL | _(none)_ |
| `terraform graph` | Show dependency graph | `-type=plan` |
| `terraform show` | Show state or plan file | `-json` |
| `terraform state list` | List resources in state | _(none)_ |
| `terraform state show` | Show resource in state | _(none)_ |
| `terraform state mv` | Move resource in state | _(none)_ |
| `terraform state rm` | Remove resource from state | _(none)_ |
| `terraform state pull` | Download state from backend | _(none)_ |
| `terraform state push` | Upload state to backend | _(use with extreme caution)_ |
| `terraform import` | Import existing resource | _(none)_ |
| `terraform workspace new` | Create workspace | _(none)_ |
| `terraform workspace select` | Switch workspace | _(none)_ |
| `terraform workspace list` | List workspaces | _(none)_ |
| `terraform workspace delete` | Delete workspace | `-force` |
| `terraform providers` | Show provider info | _(none)_ |
| `terraform providers lock` | Update lock file | `-platform` |
| `terraform providers mirror` | Download providers locally | _(path)_ |
| `terraform version` | Show Terraform version | _(none)_ |
| `terraform force-unlock` | Release stuck state lock | `<lock_id>` |
| `terraform get` | Download/update modules | `-update` |
| `terraform refresh` | Sync state with real world | _(deprecated — use plan -refresh-only)_ |
| `terraform taint` | Mark for replacement | _(deprecated — use apply -replace)_ |

---

### HCL Type System

| Type | Category | Example | Notes |
|------|----------|---------|-------|
| `string` | Primitive | `"hello"` | UTF-8 text |
| `number` | Primitive | `42` or `3.14` | Integer or float |
| `bool` | Primitive | `true` / `false` | Boolean |
| `list(type)` | Collection | `["a","b","c"]` | Ordered, allows duplicates |
| `set(type)` | Collection | `toset(["a","b"])` | Unordered, unique values |
| `map(type)` | Collection | `{a = 1, b = 2}` | Key-value, all values same type |
| `object({...})` | Structural | `{name="x", port=80}` | Named attrs, can be different types |
| `tuple([...])` | Structural | `["a", 1, true]` | Fixed length, positional types |
| `any` | Special | _(anything)_ | Type inferred at runtime |

---

### Built-in Functions by Category

| Category | Functions |
|----------|-----------|
| **String** | `format`, `formatlist`, `upper`, `lower`, `title`, `trim`, `trimspace`, `trimprefix`, `trimsuffix`, `replace`, `split`, `join`, `substr`, `indent`, `chomp`, `startswith`, `endswith`, `strcontains` |
| **Numeric** | `abs`, `ceil`, `floor`, `max`, `min`, `pow`, `log`, `signum`, `parseint` |
| **Collection** | `length`, `element`, `index`, `slice`, `concat`, `flatten`, `distinct`, `reverse`, `sort`, `contains`, `keys`, `values`, `lookup`, `merge`, `zipmap`, `setproduct`, `setintersection`, `setunion`, `setsubtract`, `toset`, `tolist`, `tomap`, `range`, `one`, `coalesce`, `coalescelist` |
| **Encoding** | `base64encode`, `base64decode`, `urlencode`, `jsonencode`, `jsondecode`, `yamlencode`, `yamldecode`, `csvdecode` |
| **Filesystem** | `file`, `filebase64`, `filemd5`, `filesha256`, `filesha512`, `templatefile`, `abspath`, `basename`, `dirname`, `pathexpand` |
| **Date/Time** | `timestamp`, `formatdate`, `timeadd`, `timecmp` |
| **Hash/Crypto** | `md5`, `sha1`, `sha256`, `sha512`, `bcrypt`, `uuid`, `uuidv5` |
| **IP Network** | `cidrhost`, `cidrnetmask`, `cidrsubnet`, `cidrsubnets`, `cidrcontains` |
| **Type Convert** | `tostring`, `tonumber`, `tobool`, `try`, `can` |
| **Regex** | `regex`, `regexall`, `replace` |

---

### lifecycle Meta-Arguments

| Argument | Type | Effect |
|----------|------|--------|
| `create_before_destroy` | `bool` | Create replacement before destroying original (zero-downtime updates) |
| `prevent_destroy` | `bool` | Error if plan would destroy this resource — protects critical resources |
| `ignore_changes` | `list` or `all` | Ignore specified attributes during plan/apply (or all changes) |
| `replace_triggered_by` | `list` | Force replacement if referenced attribute changes |
| `precondition` | block | Assert condition before create/update — fails plan if false |
| `postcondition` | block | Assert condition after create/update — fails apply if false |

---

### Variable Precedence (Lowest → Highest)

| Priority | Source | Notes |
|----------|--------|-------|
| 1 (lowest) | `default` in `variable {}` block | Falls back to this if nothing else provides value |
| 2 | `terraform.tfvars` | Auto-loaded from working directory |
| 3 | `terraform.tfvars.json` | Auto-loaded JSON variant |
| 4 | `*.auto.tfvars` | Auto-loaded, alphabetical order |
| 5 | `*.auto.tfvars.json` | Auto-loaded JSON variant |
| 6 | `-var-file` flag | Right-most `-var-file` wins |
| 7 | `-var` flag | Right-most `-var` wins |
| 8 (highest) | `TF_VAR_<name>` | Environment variable — overrides everything |

---

### Resource Meta-Arguments Summary

| Meta-Argument | Purpose | Example |
|---------------|---------|---------|
| `count` | Create N copies, integer indexed | `count = 3` |
| `for_each` | Create copies keyed by map/set | `for_each = var.servers` |
| `depends_on` | Explicit dependency declaration | `depends_on = [aws_iam_role_policy.main]` |
| `provider` | Use non-default provider | `provider = aws.eu_central` |
| `lifecycle` | Control resource lifecycle | `lifecycle { prevent_destroy = true }` |

---

### Common for Expression Patterns

| Pattern | Expression | Result |
|---------|-----------|--------|
| List → UPPER list | `[for s in var.names : upper(s)]` | `["ALICE", "BOB"]` |
| List → filtered list | `[for s in var.names : s if length(s) > 3]` | Only long names |
| List → map | `{for s in var.names : s => length(s)}` | `{alice=5, bob=3}` |
| Map → list of values | `[for k, v in var.map : v]` | All values as list |
| Map → filtered map | `{for k, v in var.map : k => v if v != "skip"}` | Filtered map |
| Map → transformed map | `{for k, v in var.map : k => upper(v)}` | Values uppercased |
| Nested list flatten | `flatten([for item in var.nested : item.list])` | Single flat list |
| Index in list loop | `[for i, v in var.list : "${i}:${v}"]` | `["0:a", "1:b"]` |
| Cartesian product | `setproduct(var.list_a, var.list_b)` | All combinations |
| Build map from lists | `zipmap(var.keys, var.values)` | `{k1=v1, k2=v2}` |
| List deduplication | `toset(var.list_with_dupes)` | Unique values |

---

*Generated with ❤️ — For Terraform 1.5+*
*Official docs: [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform) | Registry: [registry.terraform.io](https://registry.terraform.io)*
