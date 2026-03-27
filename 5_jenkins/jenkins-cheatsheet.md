# 🔧 Jenkins Comprehensive Cheatsheet

> A complete reference for Jenkins — covering architecture, pipeline syntax, shared libraries, agents, credentials, JCasC, security, observability, and real-world patterns.

---

## 📚 Table of Contents

1. [Core Concepts & Architecture](#1-core-concepts--architecture)
2. [Installation & Initial Setup](#2-installation--initial-setup)
3. [Jobs & Build Types](#3-jobs--build-types)
4. [Declarative Pipeline — Complete Reference](#4-declarative-pipeline--complete-reference)
5. [Scripted Pipeline](#5-scripted-pipeline)
6. [Jenkinsfile Patterns & Best Practices](#6-jenkinsfile-patterns--best-practices)
7. [Shared Libraries](#7-shared-libraries)
8. [Agents & Distributed Builds](#8-agents--distributed-builds)
9. [Credentials & Secrets Management](#9-credentials--secrets-management)
10. [SCM Integration](#10-scm-integration)
11. [Build Tools Integration](#11-build-tools-integration)
12. [Testing & Quality Gates](#12-testing--quality-gates)
13. [Artifacts & Archiving](#13-artifacts--archiving)
14. [Notifications & Integrations](#14-notifications--integrations)
15. [Jenkins Configuration as Code (JCasC)](#15-jenkins-configuration-as-code-jcasc)
16. [Job DSL Plugin](#16-job-dsl-plugin)
17. [Pipeline Optimisation & Performance](#17-pipeline-optimisation--performance)
18. [Security & Hardening](#18-security--hardening)
19. [Monitoring, Observability & Maintenance](#19-monitoring-observability--maintenance)
20. [Real-World Pipeline Patterns & Recipes](#20-real-world-pipeline-patterns--recipes)

---

## 1. Core Concepts & Architecture

### What is Jenkins?

Jenkins is an open-source automation server used to implement **Continuous Integration (CI)** and **Continuous Delivery/Deployment (CD)**. It sits at the centre of a DevOps toolchain, orchestrating the flow from code commit to production deployment.

```
DevOps Toolchain — where Jenkins fits:

Developer → Git Push → [Jenkins CI] → Test → Build → [Jenkins CD] → Deploy
                            │                               │
                       Lint, Unit Test              Staging → Prod
                       SonarQube, SAST             Helm, Terraform
                       Package, Archive             Notifications
```

**CI/CD Philosophy:**
- **CI** — merge code frequently, run automated tests on every commit, catch bugs early
- **CD (Delivery)** — every passing build is deployable; manual approval gate before prod
- **CD (Deployment)** — every passing build is automatically deployed to production

---

### Jenkins Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    JENKINS CONTROLLER                           │
│                                                                 │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │  Build Queue │  │  Plugin Manager │  │  Credential Store│  │
│  │  (waiting)   │  │  (extensions)   │  │  (secrets)       │  │
│  └──────────────┘  └─────────────────┘  └──────────────────┘  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  $JENKINS_HOME                                          │   │
│  │  jobs/ workspace/ plugins/ config.xml secrets/ logs/   │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────────┘
                        │  SSH / JNLP / WebSocket
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │  Agent 1   │ │  Agent 2   │ │  Agent 3   │
   │ linux-build│ │  windows   │ │  k8s-pod   │
   │ 2 executors│ │ 1 executor │ │ ephemeral  │
   └────────────┘ └────────────┘ └────────────┘
```

---

### Controller vs Agent Responsibilities

| Responsibility | Controller | Agent |
|---|---|---|
| Job scheduling & queue | ✅ | ❌ |
| UI and API | ✅ | ❌ |
| Plugin management | ✅ | ❌ |
| Build execution | ⚠️ (avoid) | ✅ |
| Workspace for builds | ⚠️ (avoid) | ✅ |
| Checkout code | ⚠️ (avoid) | ✅ |

**Communication protocols:**
- **SSH** — controller initiates SSH to agent, runs `remoting.jar`; requires network access from controller
- **JNLP (inbound)** — agent initiates outbound connection; useful when agents are behind NAT/firewall
- **WebSocket** — JNLP over WebSocket; useful when only port 443 is available

> ⚠️ **Warning:** Never run builds on the controller. Set controller executors to 0 in `Manage Jenkins → Nodes → Built-In Node → # of executors`.

---

### $JENKINS_HOME Layout

```bash
$JENKINS_HOME/
├── config.xml              # Main Jenkins system configuration
├── jobs/                   # One subdirectory per job
│   └── my-pipeline/
│       ├── config.xml      # Job configuration
│       └── builds/         # Build history and logs
│           └── 42/
│               ├── log     # Console output
│               └── build.xml
├── workspace/              # Build workspaces (on controller — avoid builds here)
├── plugins/                # Installed plugins (.jpi / .hpi files)
├── secrets/                # Encrypted credential store, master.key
│   ├── master.key          # Encryption key — MUST be backed up
│   └── hudson.util.Secret
├── logs/                   # Jenkins internal logs
├── nodes/                  # Agent configurations
├── users/                  # Local user accounts
└── updates/                # Plugin update cache
```

---

### Build Lifecycle

```
1. TRIGGER         Webhook / cron / manual / upstream
        │
2. QUEUE           Job waits for available executor
        │          (respects quiet period, throttle)
3. ALLOCATE        Jenkins assigns executor on matching agent
        │
4. CHECKOUT        SCM checkout (unless skipDefaultCheckout)
        │
5. BUILD           Stages execute (sh, bat, docker, etc.)
        │
6. POST-ACTIONS    post{} block: archive, notify, cleanup
        │
7. RESULT          SUCCESS / FAILURE / UNSTABLE / ABORTED
```

---

### Key Terminology

| Term | Meaning |
|------|---------|
| **Job / Project** | The configured unit of work (pipeline, freestyle, etc.) |
| **Build** | One execution of a job; identified by build number |
| **Step** | A single action inside a build (`sh`, `echo`, `junit`) |
| **Stage** | A named group of steps; appears on pipeline visualisation |
| **Workspace** | Directory on the agent where build files live |
| **Artifact** | File produced by a build and archived for later use |
| **Fingerprint** | MD5 hash tracking an artifact across jobs |
| **Upstream/Downstream** | Job A triggers job B: A=upstream, B=downstream |
| **Executor** | A single build slot on an agent (one concurrent build) |
| **Executor starvation** | All executors busy; new builds queue and wait |

---

## 2. Installation & Initial Setup

### Docker Quick-Start

```bash
# Pull the Long-Term Support image
docker pull jenkins/jenkins:lts

# Run Jenkins with persistent home directory
docker run -d \
  --name jenkins \
  -p 8080:8080 \          # Jenkins web UI
  -p 50000:50000 \        # JNLP agent communication port
  -v jenkins_home:/var/jenkins_home \   # persist $JENKINS_HOME
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

---

### Docker Compose Setup (Controller + Agent + DinD)

```yaml
# docker-compose.yml — Jenkins controller with Docker-in-Docker agent
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts         # use LTS for stability
    container_name: jenkins-controller
    restart: unless-stopped
    ports:
      - "8080:8080"                    # web UI
      - "50000:50000"                  # JNLP agent port
    volumes:
      - jenkins_home:/var/jenkins_home # persistent config + jobs
      - /var/run/docker.sock:/var/run/docker.sock  # Docker socket (for docker steps)
    environment:
      - JAVA_OPTS=-Xmx2g -Djava.awt.headless=true   # JVM heap + headless
      - JENKINS_OPTS=--prefix=/jenkins               # URL context path
    networks:
      - jenkins-net

  jenkins-agent:
    image: jenkins/inbound-agent:latest  # JNLP inbound agent
    container_name: jenkins-agent
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins:8080  # controller URL (internal DNS)
      - JENKINS_SECRET=${AGENT_SECRET}   # secret from UI: Manage Nodes → agent → secret
      - JENKINS_AGENT_NAME=docker-agent  # must match name configured in UI
      - JENKINS_OPTS=--workDir=/home/jenkins/agent
    volumes:
      - agent_workspace:/home/jenkins/agent
      - /var/run/docker.sock:/var/run/docker.sock  # allow agent to run docker
    networks:
      - jenkins-net
    depends_on:
      - jenkins

  dind:
    image: docker:dind                   # Docker-in-Docker for isolated builds
    container_name: jenkins-dind
    privileged: true                     # required for DinD
    environment:
      - DOCKER_TLS_CERTDIR=/certs        # TLS between DinD and clients
    volumes:
      - dind_certs:/certs/client
    networks:
      - jenkins-net

volumes:
  jenkins_home:
  agent_workspace:
  dind_certs:

networks:
  jenkins-net:
    driver: bridge
```

---

### Kubernetes Deployment (Helm)

```bash
# Add the Jenkins Helm chart repository
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Install Jenkins with custom values
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  --set controller.serviceType=LoadBalancer \
  --set persistence.size=20Gi \
  --values jenkins-values.yaml
```

```yaml
# jenkins-values.yaml — key Helm values
controller:
  numExecutors: 0              # no builds on controller
  adminUser: "admin"
  adminPassword: "${JENKINS_ADMIN_PASSWORD}"  # from secret
  javaOpts: "-Xmx2g -XX:+UseG1GC"
  installPlugins:              # plugins to install at boot
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
  JCasC:                       # inline JCasC config
    configScripts:
      welcome-message: |
        jenkins:
          systemMessage: "Jenkins managed by Helm + JCasC"

persistence:
  enabled: true
  size: 20Gi
  storageClass: "standard"

agent:
  enabled: true                # deploy a default agent pod template
```

---

### Systemd Service

```bash
# /etc/systemd/system/jenkins.service (created by apt/yum install)
# Environment file: /etc/default/jenkins (Debian) or /etc/sysconfig/jenkins (RHEL)
```

```ini
# /etc/default/jenkins — environment for jenkins.service
JENKINS_HOME=/var/lib/jenkins
JENKINS_USER=jenkins
JENKINS_PORT=8080
JENKINS_OPTS="--prefix=/jenkins --httpListenAddress=127.0.0.1"
JAVA_OPTS="-Xmx4g -Xms1g -XX:+UseG1GC -Djava.awt.headless=true -Djenkins.install.runSetupWizard=false"
```

```bash
systemctl start jenkins       # start Jenkins
systemctl stop jenkins        # stop Jenkins
systemctl restart jenkins     # restart (applies JAVA_OPTS changes)
systemctl status jenkins      # check status
journalctl -u jenkins -f      # follow Jenkins logs via journald
```

---

### Nginx Reverse Proxy

```nginx
# /etc/nginx/sites-available/jenkins
upstream jenkins {
    server 127.0.0.1:8080;    # Jenkins listens locally only
}

server {
    listen 443 ssl http2;
    server_name jenkins.example.com;

    ssl_certificate     /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;

    # Required for Jenkins to generate correct redirect URLs
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /jenkins/ {
        proxy_pass http://jenkins/jenkins/;    # note trailing slash

        # WebSocket support — required for Blue Ocean, agent connections
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 90s;                # long builds need this
        proxy_buffering off;                   # stream console output
    }
}

server {
    listen 80;
    server_name jenkins.example.com;
    return 301 https://$host$request_uri;      # redirect HTTP → HTTPS
}
```

> 💡 **Tip:** Set `Jenkins URL` in `Manage Jenkins → System` to `https://jenkins.example.com/jenkins/` — Jenkins uses this to generate absolute links in emails and status checks.

---

## 3. Jobs & Build Types

### Freestyle Project

The simplest job type — configure build steps through the UI. Good for simple scripts and legacy workflows, but lacks version control for the job definition itself.

```bash
# Build step: Execute shell
#!/bin/bash
set -euo pipefail                     # exit on error
echo "Building branch: $GIT_BRANCH"   # $GIT_BRANCH set by Git plugin
mvn clean package -DskipTests=false
```

---

### Multibranch Pipeline

Automatically discovers branches and PRs in a repository. Each branch gets its own pipeline run using that branch's `Jenkinsfile`.

```
Multibranch Pipeline behaviour:
  ├── main          → runs Jenkinsfile from main branch
  ├── feature/auth  → runs Jenkinsfile from feature/auth
  ├── release/1.4   → runs Jenkinsfile from release/1.4
  └── PR-42         → runs Jenkinsfile from PR source branch
```

**Key settings (UI):**
- **Branch Sources** — GitHub/GitLab/Bitbucket/Git
- **Discover branches** — all branches, only PRs, exclude PRs
- **Orphaned Branch Strategy** — delete after N days / keep last N builds
- **Build Strategies** — build on change, named branches, tags

---

### Build Parameters

```groovy
// Declarative — parameters block
pipeline {
    parameters {
        string(name: 'IMAGE_TAG',        // string parameter
               defaultValue: 'latest',
               description: 'Docker image tag to deploy')

        booleanParam(name: 'SKIP_TESTS', // boolean checkbox
                     defaultValue: false,
                     description: 'Skip test stage')

        choice(name: 'ENVIRONMENT',      // dropdown list
               choices: ['dev', 'staging', 'prod'],
               description: 'Target environment')

        password(name: 'API_KEY',        // masked input (stored in build params)
                 defaultValue: '',
                 description: 'API key — use credentials instead for production!')

        text(name: 'CHANGELOG',          // multi-line text
             defaultValue: '',
             description: 'Release notes')
    }
    stages {
        stage('Use params') {
            steps {
                echo "Deploying tag: ${params.IMAGE_TAG}"         // access via params.
                echo "Environment: ${params.ENVIRONMENT}"
                script {
                    if (params.SKIP_TESTS) {                      // boolean param
                        echo "Skipping tests"
                    }
                }
            }
        }
    }
}
```

---

### Build Triggers

```groovy
pipeline {
    triggers {
        // Poll SCM every 5 minutes using H for load distribution
        pollSCM('H/5 * * * *')          // H = hash-based offset, not literal :00

        // Cron — build every night at 2 AM
        cron('H 2 * * *')

        // Trigger when upstream job succeeds
        upstream(upstreamProjects: 'my-library-build',
                 threshold: hudson.model.Result.SUCCESS)
    }
}
```

```bash
# Remote trigger via URL (requires API token)
curl -X POST \
  "https://jenkins.example.com/job/my-pipeline/build?token=BUILD_TOKEN" \
  --user "username:api-token"

# Trigger with parameters
curl -X POST \
  "https://jenkins.example.com/job/my-pipeline/buildWithParameters" \
  --user "username:api-token" \
  --data "IMAGE_TAG=v1.2.3&ENVIRONMENT=staging"
```

> 💡 **Tip:** Always use `H` instead of a literal minute in cron expressions. `H/5 * * * *` spreads builds across the 5-minute window based on job name hash, preventing all jobs from hammering SCM simultaneously at `*/5`.

---

## 4. Declarative Pipeline — Complete Reference

### Overall Structure

```groovy
pipeline {                              // top-level block — required
    agent any                           // where to run — required

    environment {                       // pipeline-wide env vars
        APP_NAME = 'my-service'
        VERSION  = "1.${BUILD_NUMBER}"  // BUILD_NUMBER is built-in
    }

    options {                           // pipeline behaviour options
        timeout(time: 1, unit: 'HOURS')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    parameters {                        // build parameters (see above)
        string(name: 'TAG', defaultValue: 'latest')
    }

    triggers {                          // automated triggers
        pollSCM('H/5 * * * *')
    }

    stages {                            // ordered list of stages
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }

    post {                              // runs after all stages
        always {
            cleanWs()                   // always clean workspace
        }
        failure {
            slackSend color: 'danger', message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

### agent Directive — All Forms

```groovy
// Any available agent
agent any

// No global agent — declare per stage
agent none

// Agent with specific label
agent { label 'linux && docker' }    // label expression

// Docker container as agent
agent {
    docker {
        image 'maven:3.9-eclipse-temurin-17'  // Docker image to run in
        label 'docker-capable'                  // agent that can run Docker
        args '-v $HOME/.m2:/root/.m2'           // pass extra docker run args
        registryUrl 'https://registry.example.com'
        registryCredentialsId 'docker-registry-creds'
        alwaysPull true                          // always pull latest
    }
}

// Build image from Dockerfile in the repo
agent {
    dockerfile {
        filename 'Dockerfile.jenkins'    // default: Dockerfile
        dir 'docker/jenkins'             // directory containing Dockerfile
        additionalBuildArgs '--build-arg JAVA_VERSION=17'
    }
}

// Kubernetes pod as agent
agent {
    kubernetes {
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep", "infinity"]    # keep container alive
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "2"
  - name: docker
    image: docker:24-dind             # Docker-in-Docker sidecar
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
'''
        defaultContainer 'maven'       // default container for sh steps
    }
}
```

---

### options Directive

```groovy
options {
    timeout(time: 2, unit: 'HOURS')            // abort if build takes > 2h
    retry(3)                                    // retry entire pipeline 3x on failure
    timestamps()                               // prefix each log line with timestamp
    skipDefaultCheckout()                      // don't auto-checkout — do it manually
    disableConcurrentBuilds()                  // only one build at a time for this job
    disableConcurrentBuilds(abortPrevious: true) // cancel running build when new one starts
    parallelsAlwaysFailFast()                  // parallel stages: fail immediately on error
    buildDiscarder(logRotator(
        numToKeepStr: '20',                    // keep last 20 builds
        daysToKeepStr: '30',                   // OR builds from last 30 days
        artifactNumToKeepStr: '5',             // but only keep artifacts for last 5
        artifactDaysToKeepStr: '7'
    ))
    quietPeriod(30)                            // wait 30s before starting (collects pushes)
    rateLimitBuilds(throttle: [count: 1,       // max 1 build per hour
                               durationName: 'hour',
                               userBoost: true])
}
```

---

### when Directive — All Conditions

```groovy
stages {
    stage('Deploy to Prod') {
        when {
            branch 'main'                          // only on main branch
        }
        steps { sh 'deploy prod' }
    }

    stage('Deploy to Staging') {
        when {
            branch pattern: 'release/.*',          // regex on branch name
                   comparator: 'REGEXP'
        }
        steps { sh 'deploy staging' }
    }

    stage('Build on tag') {
        when {
            buildingTag()                          // true when building a git tag
        }
        steps { sh 'publish release' }
    }

    stage('Run if env var set') {
        when {
            environment name: 'DEPLOY_ENV',        // env var equals value
                        value: 'production'
        }
        steps { sh './deploy.sh' }
    }

    stage('Run if param is true') {
        when {
            expression { return params.RUN_TESTS } // arbitrary Groovy expression
        }
        steps { sh './test.sh' }
    }

    stage('Changelog contains') {
        when {
            changelog '.*\\[ci deploy\\].*'        // regex on commit messages
        }
        steps { sh 'deploy' }
    }

    stage('Changed files') {
        when {
            changeset 'services/auth/**'           // files matching glob changed
        }
        steps { sh 'build auth-service' }
    }

    stage('Combined conditions') {
        when {
            allOf {                                // ALL must be true (AND)
                branch 'main'
                not { buildingTag() }
                expression { return !params.SKIP_DEPLOY }
            }
        }
        steps { sh 'deploy' }
    }

    stage('Either condition') {
        when {
            anyOf {                                // ANY must be true (OR)
                branch 'main'
                branch 'release/*'
            }
        }
        steps { sh 'deploy' }
    }
}
```

---

### parallel Stages

```groovy
stage('Test') {
    parallel {                                    // all branches run concurrently
        stage('Unit Tests') {
            agent { label 'linux' }
            steps {
                sh 'mvn test -Dtest="Unit*"'
                junit 'target/surefire-reports/*.xml'
            }
        }
        stage('Integration Tests') {
            agent { label 'linux' }
            steps {
                sh 'mvn verify -Dtest="IT*"'
                junit 'target/failsafe-reports/*.xml'
            }
        }
        stage('Security Scan') {
            agent { label 'linux' }
            steps {
                sh 'trivy image myapp:latest'
            }
        }
    }
}
```

---

### matrix — Multi-axis Builds

```groovy
stage('Cross-platform build') {
    matrix {
        axes {
            axis {
                name 'OS'                          // first axis
                values 'linux', 'windows', 'mac'
            }
            axis {
                name 'JAVA_VERSION'                // second axis
                values '11', '17', '21'
            }
        }
        excludes {                                 // exclude invalid combinations
            exclude {
                axis { name 'OS'; values 'mac' }
                axis { name 'JAVA_VERSION'; values '11' }  // no Java 11 on mac
            }
        }
        stages {
            stage('Build') {
                steps {
                    echo "Building on ${OS} with Java ${JAVA_VERSION}"
                    sh './gradlew build'
                }
            }
        }
    }
}
```

---

### Input Step — Human Approval Gate

```groovy
stage('Approve Production Deploy') {
    steps {
        input(
            message: "Deploy ${params.IMAGE_TAG} to PRODUCTION?",
            ok: 'Deploy Now',                     // label for the OK button
            submitter: 'ops-team,alice,bob',       // CSV of users/groups who can approve
            parameters: [
                choice(
                    name: 'REGION',
                    choices: ['us-east-1', 'eu-west-1'],
                    description: 'Deployment region'
                )
            ]
        )
    }
}
```

```groovy
// Input with timeout — auto-abort if no response
stage('Approve') {
    options {
        timeout(time: 24, unit: 'HOURS')   // auto-abort if not approved in 24h
    }
    steps {
        script {
            def approval = input(
                message: 'Proceed with deployment?',
                parameters: [string(name: 'REASON', description: 'Reason for approval')]
            )
            echo "Approved with reason: ${approval}"
        }
    }
}
```

---

### post Block — All Conditions

```groovy
post {
    always {        // always runs — cleanup, notifications
        cleanWs()
    }
    success {       // build succeeded
        archiveArtifacts artifacts: 'dist/**'
    }
    failure {       // build failed
        mail to: 'team@example.com',
             subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
             body: "See: ${env.BUILD_URL}"
    }
    unstable {      // build is UNSTABLE (test failures, quality gate warnings)
        slackSend color: 'warning', message: "UNSTABLE: ${env.JOB_NAME}"
    }
    changed {       // result changed from previous build (failure→success or vice versa)
        echo "Build result changed!"
    }
    fixed {         // was failing, now succeeds
        slackSend color: 'good', message: "FIXED: ${env.JOB_NAME} is back to green"
    }
    regression {    // was succeeding, now fails
        slackSend color: 'danger', message: "REGRESSION: ${env.JOB_NAME} broke"
    }
    aborted {       // build was manually aborted
        echo "Build was aborted"
    }
    cleanup {       // runs LAST, after all other post conditions
        deleteDir()
    }
}
```

---

## 5. Scripted Pipeline

### Groovy Basics for Jenkins

```groovy
// Variables — dynamic typing, def keyword
def name = "Jenkins"
def count = 42
String typed = "explicit type"    // explicit type also works

// String interpolation — GString uses double quotes
def version = "1.2.3"
echo "Version is ${version}"      // "Version is 1.2.3" — interpolated
echo 'Version is ${version}'      // 'Version is ${version}' — literal!

// ⚠️ GString pitfall: credentials in GString are logged!
withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
    // 🔴 Anti-pattern: GString evaluates immediately — API_KEY visible in log!
    sh "curl -H 'Authorization: ${API_KEY}' https://api.example.com"

    // ✅ Single quotes: sh gets literal string, shell does the expansion (masked)
    sh 'curl -H "Authorization: $API_KEY" https://api.example.com'
}

// Collections
def list = ['a', 'b', 'c']
def map = [name: 'alice', age: 30]
echo map.name          // "alice"
echo map['age']        // 30

// Closures (Groovy lambdas)
def greet = { name -> "Hello, ${name}!" }
echo greet("Jenkins")

// List iteration
list.each { item -> echo item }
list.eachWithIndex { item, i -> echo "${i}: ${item}" }
```

---

### node, stage, try/catch

```groovy
// Scripted pipeline structure
node('linux') {                            // allocate executor on 'linux' labeled agent

    stage('Checkout') {
        checkout scm                       // check out source
    }

    stage('Build') {
        try {
            sh 'mvn clean package'
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            error("Build failed: ${e.message}")   // rethrow as pipeline error
        } finally {
            junit 'target/surefire-reports/*.xml' // always publish test results
        }
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {   // conditional stage
            sh './deploy.sh prod'
        } else {
            echo "Skipping deploy for branch ${env.BRANCH_NAME}"
            return                          // skip rest of stage
        }
    }
}
```

---

### Parallel Execution in Scripted Pipeline

```groovy
node {
    stage('Parallel Tests') {
        // parallel() takes a Map of branch-name → closure
        parallel(
            'Unit Tests': {
                node('linux') {
                    sh 'mvn test -Punit'
                }
            },
            'Integration Tests': {
                node('linux') {
                    sh 'mvn verify -Pintegration'
                }
            },
            failFast: true    // abort all branches if any fails
        )
    }
}
```

---

### currentBuild Object

```groovy
// currentBuild — information about the running build
echo currentBuild.result           // SUCCESS / FAILURE / UNSTABLE / null (still running)
echo currentBuild.currentResult    // current result (updates as stages run)
echo currentBuild.displayName      // build display name (default: #42)
echo currentBuild.description      // build description shown in UI

// Set build info programmatically
currentBuild.displayName = "#${BUILD_NUMBER} - ${params.IMAGE_TAG}"
currentBuild.description = "Deployed to ${params.ENVIRONMENT}"

// Duration and timing
echo currentBuild.duration              // milliseconds since build started
echo currentBuild.startTimeInMillis     // epoch millis when build started
echo currentBuild.durationString        // human-readable: "1 min 23 sec"

// Upstream builds
def causes = currentBuild.getBuildCauses()
causes.each { cause ->
    echo "Triggered by: ${cause.shortDescription}"
}
```

---

### env Object

```groovy
// Read built-in environment variables
echo env.BUILD_NUMBER        // "42"
echo env.JOB_NAME            // "my-pipeline"
echo env.JOB_BASE_NAME       // "my-pipeline" (without folder path)
echo env.BUILD_URL           // "https://jenkins.example.com/job/my-pipeline/42/"
echo env.WORKSPACE           // "/home/jenkins/workspace/my-pipeline"
echo env.BRANCH_NAME         // "main" (Multibranch only)
echo env.CHANGE_ID           // PR number (Multibranch PR builds only)
echo env.GIT_COMMIT          // "abc1234..." (set by git checkout)
echo env.GIT_BRANCH          // "origin/main"
echo env.NODE_NAME           // agent name running this stage

// Set environment variable (available to all subsequent steps in same node block)
env.MY_VAR = "hello"
echo env.MY_VAR              // "hello"

// withEnv — scoped env override (preferred over setting env directly)
withEnv(['MAVEN_OPTS=-Xmx512m', 'JAVA_HOME=/opt/java/17']) {
    sh 'mvn build'            // MAVEN_OPTS and JAVA_HOME set only in this block
}
// MAVEN_OPTS no longer set here
```

## 6. Jenkinsfile Patterns & Best Practices

### Safe Shell Commands

```groovy
// sh() with options — don't just use sh 'command'
stages {
    stage('Build') {
        steps {
            // returnStatus: true — get exit code instead of throwing on failure
            script {
                def exitCode = sh(script: 'make test', returnStatus: true)
                if (exitCode != 0) {
                    currentBuild.result = 'UNSTABLE'    // mark unstable, not failed
                    echo "Tests failed with exit code ${exitCode}"
                }
            }

            // returnStdout: true — capture command output as string
            script {
                def version = sh(
                    script: 'git describe --tags --always',
                    returnStdout: true
                ).trim()                               // .trim() removes trailing newline!
                echo "Building version: ${version}"
                env.VERSION = version
            }
        }
    }
}
```

> ⚠️ **Warning:** Always `.trim()` output from `returnStdout: true`. Without it, the string ends with `\n` which causes subtle bugs in comparisons and string interpolation.

---

### Multi-environment Deployment Pattern

```groovy
pipeline {
    agent any
    environment {
        IMAGE = "registry.example.com/myapp:${BUILD_NUMBER}"
    }
    stages {
        stage('Build & Push') {
            steps {
                sh "docker build -t ${IMAGE} ."
                sh "docker push ${IMAGE}"
            }
        }

        stage('Deploy Dev') {
            steps {
                sh "helm upgrade --install myapp ./chart --set image.tag=${BUILD_NUMBER} -n dev"
                sh "kubectl rollout status deployment/myapp -n dev --timeout=120s"
            }
        }

        stage('Integration Tests') {
            steps {
                sh "./run-tests.sh https://dev.example.com"
            }
        }

        stage('Approve Staging') {
            options { timeout(time: 4, unit: 'HOURS') }
            steps {
                input message: "Deploy ${IMAGE} to staging?", ok: 'Deploy'
            }
        }

        stage('Deploy Staging') {
            steps {
                sh "helm upgrade --install myapp ./chart --set image.tag=${BUILD_NUMBER} -n staging"
            }
        }

        stage('Approve Production') {
            when { branch 'main' }        // only allow prod deploy from main
            options { timeout(time: 24, unit: 'HOURS') }
            steps {
                input message: "Deploy to PRODUCTION?",
                      ok: 'Deploy to Prod',
                      submitter: 'ops-lead,cto'    // restrict who can approve
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            steps {
                sh "helm upgrade --install myapp ./chart --set image.tag=${BUILD_NUMBER} -n prod"
                sh "kubectl rollout status deployment/myapp -n prod --timeout=300s"
            }
        }
    }
}
```

---

### Handling Secrets — The Right Way

```groovy
// 🔴 Anti-pattern: secrets in environment variables at pipeline level — visible in logs
environment {
    API_KEY = "super-secret-key"    // hardcoded — terrible!
    DB_PASS = credentials('db-password')  // better, but secret name in config
}

// 🔴 Anti-pattern: echo the secret
withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
    echo "Key is: ${API_KEY}"       // API_KEY appears in logs!
    sh "echo key=${API_KEY}"        // also appears!
}

// ✅ Right way: use single-quoted sh (shell handles the variable, Jenkins masks it)
withCredentials([
    usernamePassword(credentialsId: 'docker-hub',
                     usernameVariable: 'DOCKER_USER',
                     passwordVariable: 'DOCKER_PASS')
]) {
    sh '''
        set +x                          # disable xtrace to prevent leaking
        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
        set -x                          # re-enable xtrace
    '''
}

// ✅ Using credentials() in environment block (auto-masked)
environment {
    DEPLOY_KEY = credentials('prod-deploy-key')    // sets DEPLOY_KEY (SSH key path),
                                                   // DEPLOY_KEY_USR, DEPLOY_KEY_PSW
}
```

---

### Dynamic Stage Generation

```groovy
// Build stages dynamically from a list — useful for microservices
def services = ['auth', 'api', 'frontend', 'worker']

pipeline {
    agent any
    stages {
        stage('Build Services') {
            steps {
                script {
                    // Build parallel map at runtime
                    def buildStages = services.collectEntries { service ->
                        ["Build ${service}": {
                            dir("services/${service}") {     // change to service dir
                                sh "docker build -t ${service}:${BUILD_NUMBER} ."
                            }
                        }]
                    }
                    parallel buildStages                     // run all in parallel
                }
            }
        }
    }
}
```

---

### GitOps-style Pipeline

```groovy
pipeline {
    agent any
    environment {
        APP_REPO    = 'https://github.com/org/myapp.git'
        MANIFESTS_REPO = 'https://github.com/org/k8s-manifests.git'
        IMAGE       = "registry.example.com/myapp:${GIT_COMMIT[0..7]}"
    }
    stages {
        stage('Build & Push Image') {
            steps {
                sh "docker build -t ${IMAGE} ."
                withCredentials([usernamePassword(credentialsId: 'registry-creds',
                                                  usernameVariable: 'REG_USER',
                                                  passwordVariable: 'REG_PASS')]) {
                    sh 'echo "$REG_PASS" | docker login registry.example.com -u "$REG_USER" --password-stdin'
                }
                sh "docker push ${IMAGE}"
            }
        }

        stage('Update Manifests') {
            steps {
                // Check out the GitOps manifests repo
                dir('manifests') {
                    git url: MANIFESTS_REPO, credentialsId: 'github-app'
                    sh """
                        # Update image tag in manifest using sed
                        sed -i 's|image: registry.example.com/myapp:.*|image: ${IMAGE}|g' \
                            apps/myapp/deployment.yaml
                    """
                    sh """
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins CI"
                        git add apps/myapp/deployment.yaml
                        git commit -m "ci: update myapp to ${IMAGE}"
                        git push origin main
                    """
                }
            }
        }
    }
}
```

---

## 7. Shared Libraries

### Library Structure

```
my-jenkins-library/                   # Git repository
├── vars/                             # Global CPS-transformed steps
│   ├── buildDocker.groovy            # def call() → pipeline step: buildDocker(...)
│   ├── deployHelm.groovy
│   └── notify.groovy
├── src/                              # Regular Groovy classes (NOT CPS-safe)
│   └── com/example/
│       ├── GitUtils.groovy           # helper class
│       └── SlackMessage.groovy       # non-CPS utilities
├── resources/                        # Static files
│   ├── config/
│   │   └── maven-settings.xml        # loaded with libraryResource()
│   └── scripts/
│       └── setup-env.sh
└── test/                             # Unit tests (Spock / JenkinsPipelineUnit)
    └── com/example/
        └── GitUtilsSpec.groovy
```

---

### Writing vars/ Steps

```groovy
// vars/buildDocker.groovy — callable as buildDocker(image: 'myapp', tag: '1.0')
def call(Map config = [:]) {
    // Validate required parameters
    if (!config.image) {
        error("buildDocker: 'image' parameter is required")
    }

    // Default values
    def tag   = config.tag ?: env.BUILD_NUMBER         // use BUILD_NUMBER if no tag
    def push  = config.get('push', true)               // default: push to registry
    def args  = config.buildArgs ?: ''

    // Access pipeline steps via implicit 'this' context
    echo "Building Docker image: ${config.image}:${tag}"

    sh "docker build ${args} -t ${config.image}:${tag} ."

    if (push) {
        withCredentials([usernamePassword(
                credentialsId: config.credentialsId ?: 'docker-registry',
                usernameVariable: 'REG_USER',
                passwordVariable: 'REG_PASS')]) {
            sh 'echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin'
            sh "docker push ${config.image}:${tag}"
        }
    }

    return "${config.image}:${tag}"     // return the full image reference
}
```

```groovy
// vars/notify.groovy — multi-channel notification step
def call(Map config = [:]) {
    def status  = config.status ?: currentBuild.currentResult
    def message = config.message ?: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${status}"
    def color   = status == 'SUCCESS' ? 'good' : 'danger'

    if (config.slack != false) {                         // send to Slack by default
        slackSend(
            channel: config.slackChannel ?: '#builds',
            color: color,
            message: "${message} | <${env.BUILD_URL}|View Build>"
        )
    }

    if (config.email) {                                  // email only if specified
        mail(to: config.email,
             subject: message,
             body: "See: ${env.BUILD_URL}")
    }
}
```

```groovy
// Jenkinsfile — using the library
@Library('my-jenkins-library@main') _   // load library, _ imports all vars

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    def image = buildDocker(
                        image: 'registry.example.com/myapp',
                        tag: "${GIT_COMMIT[0..7]}",
                        credentialsId: 'ecr-creds'
                    )
                    env.DOCKER_IMAGE = image
                }
            }
        }
    }
    post {
        always {
            notify(slack: true, slackChannel: '#deployments')
        }
    }
}
```

---

### src/ Classes (Non-CPS)

```groovy
// src/com/example/GitUtils.groovy — NOT CPS-transformed
package com.example

class GitUtils implements Serializable {    // Serializable required for pipeline

    private def steps    // reference to pipeline steps (sh, echo, etc.)

    GitUtils(steps) {    // pass 'this' from vars/ to access steps
        this.steps = steps
    }

    // Get abbreviated git commit hash
    String getShortCommit() {
        return steps.sh(script: 'git rev-parse --short HEAD',
                        returnStdout: true).trim()
    }

    // Parse semver tag
    Map parseSemver(String tag) {
        def matcher = tag =~ /^v?(\d+)\.(\d+)\.(\d+)/
        if (!matcher) return [:]
        return [major: matcher[0][1], minor: matcher[0][2], patch: matcher[0][3]]
    }
}
```

```groovy
// vars/buildAndTag.groovy — using the helper class
import com.example.GitUtils

def call(Map config = [:]) {
    def git = new GitUtils(this)       // pass 'this' for steps access
    def commit = git.getShortCommit()
    echo "Short commit: ${commit}"
}
```

---

### Loading Resources

```groovy
// resources/scripts/setup-db.sh

// vars/setupDatabase.groovy
def call() {
    // Load a resource file from the library
    def script = libraryResource('scripts/setup-db.sh')
    writeFile file: 'setup-db.sh', text: script     // write to workspace
    sh 'bash setup-db.sh'
}

// Load a config template
def call(String env) {
    def template = libraryResource("config/settings-${env}.xml")
    writeFile file: 'settings.xml', text: template
}
```

---

### Testing with JenkinsPipelineUnit

```groovy
// test/NotifySpec.groovy — unit test for notify.groovy
import com.lesfurets.jenkins.unit.*
import org.junit.*
import static com.lesfurets.jenkins.unit.MethodSignature.method

class NotifySpec extends BasePipelineTest {

    @Before
    void setUp() {
        super.setUp()
        // Mock the slackSend step
        helper.registerAllowedMethod('slackSend', [Map])
        helper.registerAllowedMethod('mail', [Map])
    }

    @Test
    void 'should send slack on success'() {
        binding.setVariable('currentBuild',
            [currentResult: 'SUCCESS', result: 'SUCCESS'])
        binding.setVariable('env',
            [JOB_NAME: 'test-job', BUILD_NUMBER: '1', BUILD_URL: 'http://...'])

        def script = loadScript('vars/notify.groovy')
        script.call(slack: true, slackChannel: '#test')

        // Verify slackSend was called with correct color
        def calls = helper.callStack.findAll { it.methodName == 'slackSend' }
        assert calls.size() == 1
        assert calls[0].args[0].color == 'good'
    }
}
```

---

## 8. Agents & Distributed Builds

### Kubernetes Plugin — Pod Templates

```groovy
// Declarative — kubernetes agent
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    pipeline: my-service          # label for identification
spec:
  serviceAccountName: jenkins    # K8s service account with needed RBAC
  containers:
  - name: jnlp                   # JNLP sidecar — required, connects to controller
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"

  - name: maven                  # build container
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep", "infinity"]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx1g"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2        # mount Maven cache for speed

  - name: docker                 # Docker-in-Docker for image builds
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  volumes:
  - name: maven-cache
    hostPath:
      path: /data/maven-cache     # persistent cache on node disk
      type: DirectoryOrCreate
'''
            defaultContainer 'maven'    // sh steps run in this container by default
            retries 2                   // retry pod creation on failure
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'   // runs in 'maven' container
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {    // explicitly target docker container
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

---

### stash and unstash

```groovy
// stash — save files to survive agent boundaries
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps {
                sh 'mvn clean package -DskipTests'
                stash(
                    name: 'build-artifacts',          // name to reference later
                    includes: 'target/*.jar,Dockerfile', // what to stash
                    excludes: 'target/*-sources.jar'  // what to exclude
                )
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'linux' }
                    steps {
                        unstash 'build-artifacts'     // restore stashed files
                        sh 'mvn test'
                    }
                }
                stage('Integration Tests') {
                    agent { label 'linux' }
                    steps {
                        unstash 'build-artifacts'     // each agent gets its own copy
                        sh 'mvn verify -Pintegration'
                    }
                }
            }
        }

        stage('Docker Build') {
            agent { label 'docker' }
            steps {
                unstash 'build-artifacts'             // Dockerfile + JAR on docker agent
                sh 'docker build -t myapp .'
            }
        }
    }
}
```

> ⚠️ **Warning:** Stash is not designed for large files — it uploads through the controller. For large artifacts (> 50MB), use a shared filesystem, S3, or Artifactory instead.

---

### Docker Plugin

```groovy
// docker.build() — build an image
node('docker') {
    def image = docker.build("myapp:${BUILD_NUMBER}", "--build-arg VERSION=1.0 .")

    // docker.image().inside() — run steps inside a container
    docker.image('maven:3.9').inside('-v $HOME/.m2:/root/.m2') {
        sh 'mvn test'    // runs inside maven container
    }

    // docker.withRegistry() — authenticate and push
    docker.withRegistry('https://registry.example.com', 'registry-creds') {
        image.push()                          // push with original tag
        image.push('latest')                  // push with additional tag
    }
}
```

---

### Agent Label Expressions

```groovy
// Single label
agent { label 'linux' }

// AND — both labels must be on same agent
agent { label 'linux && docker' }

// OR — any agent with either label
agent { label 'ubuntu || debian' }

// NOT — exclude agents with label
agent { label 'linux && !arm64' }

// Complex — parentheses for grouping
agent { label '(linux || mac) && docker && !staging' }
```

> 💡 **Tip:** Design a labelling taxonomy upfront:
> - **OS:** `linux`, `windows`, `mac`
> - **Capability:** `docker`, `k8s`, `gpu`
> - **Team:** `team-payments`, `team-auth`
> - **Environment:** `prod-capable`, `staging-only`

---

### Workspace Management

```groovy
// cleanWs() — clean workspace after build
post {
    always {
        cleanWs()                               // delete entire workspace
        cleanWs(patterns: [[pattern: 'target/', type: 'INCLUDE']])  // selective
    }
}

// Custom workspace
agent {
    node {
        label 'linux'
        customWorkspace '/jenkins/workspaces/myapp'  // fixed path (careful with parallel builds!)
    }
}

// deleteDir() — delete current directory
steps {
    dir('temp-download') {
        // work in temp-download/
        sh 'curl -O https://example.com/file.zip'
        sh 'unzip file.zip'
    }
    // back in workspace root
    deleteDir()    // deletes current dir (workspace) — use with caution!
}
```

---

## 9. Credentials & Secrets Management

### Credential Types and withCredentials

```groovy
withCredentials([
    // Username + password (e.g., Docker Hub, database)
    usernamePassword(
        credentialsId: 'docker-hub',
        usernameVariable: 'DOCKER_USER',    // env var name for username
        passwordVariable: 'DOCKER_PASS'     // env var name for password
    ),

    // Secret text (e.g., API token, bearer token)
    string(
        credentialsId: 'sonar-token',
        variable: 'SONAR_TOKEN'
    ),

    // Secret file (e.g., kubeconfig, certificate)
    file(
        credentialsId: 'kubeconfig-prod',
        variable: 'KUBECONFIG'              // path to temp file with secret
    ),

    // SSH private key
    sshUserPrivateKey(
        credentialsId: 'github-ssh-key',
        keyFileVariable: 'SSH_KEY_FILE',    // path to temp private key file
        usernameVariable: 'SSH_USER'        // associated username
    ),

    // Username:password as single string (for HTTP Basic)
    usernameColonPassword(
        credentialsId: 'nexus-creds',
        variable: 'NEXUS_CREDS'             // "username:password" format
    )
]) {
    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
    sh 'curl -u "$NEXUS_CREDS" https://nexus.example.com/repository/maven-releases/'
}
```

---

### credentials() in environment Block

```groovy
environment {
    // Username/password credential — sets THREE variables
    DOCKER = credentials('docker-hub')
    // Sets: DOCKER_USR (username), DOCKER_PSW (password), DOCKER (user:pass)

    // Secret text — sets the variable directly
    SONAR_TOKEN = credentials('sonar-token')
    // Sets: SONAR_TOKEN="actual-secret-value"
}
```

---

### HashiCorp Vault Integration

```groovy
// Install: HashiCorp Vault Plugin
withVault(
    configuration: [
        vaultUrl: 'https://vault.example.com',
        vaultCredentialId: 'vault-approle',    // AppRole credential in Jenkins
        engineVersion: 2
    ],
    vaultSecrets: [
        [
            path: 'secret/data/myapp/prod',    // Vault path
            engineVersion: 2,
            secretValues: [
                [envVar: 'DB_PASSWORD', vaultKey: 'db_password'],  // Vault key → env var
                [envVar: 'API_SECRET',  vaultKey: 'api_secret']
            ]
        ]
    ]
) {
    sh 'echo "DB password length: ${#DB_PASSWORD}"'   // use secret (masked in logs)
}
```

---

### SSH Agent Plugin

```groovy
// Use SSH key for Git operations without exposing the key file
sshagent(['github-deploy-key']) {
    sh 'git clone git@github.com:org/repo.git'
    sh 'git push origin main'

    // SSH agent loads the key into an in-memory agent — no file on disk!
    sh 'ssh -T git@github.com'    // test GitHub SSH access
}
```

---

### Credential Scope Hierarchy

```
┌─────────────────────────────────────────────────┐
│  SYSTEM scope                                   │
│  (Jenkins system use only — not available to    │
│   pipeline jobs)                                │
│  ┌─────────────────────────────────────────┐   │
│  │  GLOBAL scope                           │   │
│  │  (available to ALL jobs everywhere)     │   │
│  │  ┌───────────────────────────────────┐ │   │
│  │  │  FOLDER scope                     │ │   │
│  │  │  (available only within           │ │   │
│  │  │   that folder's jobs)             │ │   │
│  │  └───────────────────────────────────┘ │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

> 💡 **Tip:** Use folder-scoped credentials for team isolation. Team A's production credentials stored in `Team-A/` folder can't be accessed by jobs in `Team-B/`.

---

## 10. SCM Integration

### Git Checkout Options

```groovy
// Shorthand — resolves correctly in Multibranch pipelines
checkout scm

// Full GitSCM configuration
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [
        // Shallow clone — only last N commits (much faster for large repos)
        [$class: 'CloneOption',
         shallow: true,
         depth: 1,                             // only latest commit
         noTags: false,                         // still fetch tags
         timeout: 30],

        // Sparse checkout — only get specific directories
        [$class: 'SparseCheckoutPaths',
         sparseCheckoutPaths: [
             [$class: 'SparseCheckoutPath', path: 'services/auth/'],
             [$class: 'SparseCheckoutPath', path: 'shared/']
         ]],

        // Clean before checkout (like git clean -fdx)
        [$class: 'CleanBeforeCheckout'],

        // Submodule handling
        [$class: 'SubmoduleOption',
         recursiveSubmodules: true,
         trackingSubmodules: false,
         reference: '']
    ],
    userRemoteConfigs: [[
        url: 'git@github.com:org/repo.git',
        credentialsId: 'github-ssh-key'
    ]]
])
```

---

### Git Environment Variables

```groovy
// These are set automatically after checkout scm
echo env.GIT_COMMIT          // full SHA: "abc1234def5678..."
echo env.GIT_BRANCH          // "origin/main" or "origin/feature/auth"
echo env.GIT_URL             // "https://github.com/org/repo.git"
echo env.GIT_LOCAL_BRANCH    // "main" (branch name without remote prefix)
echo env.GIT_AUTHOR_NAME     // author of last commit
echo env.GIT_AUTHOR_EMAIL

// Multibranch-specific variables
echo env.BRANCH_NAME         // "main", "feature/auth", "PR-42"
echo env.CHANGE_ID           // PR number for pull request builds
echo env.CHANGE_URL          // URL to the pull request
echo env.CHANGE_TITLE        // PR title
echo env.CHANGE_AUTHOR

// Useful patterns
def shortCommit = env.GIT_COMMIT[0..7]        // "abc1234"
def isTag = env.TAG_NAME != null              // true if building a tag
def isPR = env.CHANGE_ID != null             // true if building a PR
```

---

### GitHub Integration — Status Checks

```groovy
// Post commit status to GitHub — requires GitHub plugin + credentials
stage('Tests') {
    steps {
        githubNotify(
            context: 'Unit Tests',            // check name in GitHub UI
            description: 'Running unit tests...',
            status: 'PENDING',
            credentialsId: 'github-token'
        )
        sh 'mvn test'
    }
    post {
        success {
            githubNotify context: 'Unit Tests',
                         description: 'All tests passed',
                         status: 'SUCCESS'
        }
        failure {
            githubNotify context: 'Unit Tests',
                         description: 'Tests failed',
                         status: 'FAILURE'
        }
    }
}
```

---

### Webhooks — GitHub Setup

```bash
# GitHub repository → Settings → Webhooks → Add webhook
# Payload URL:   https://jenkins.example.com/github-webhook/
# Content type:  application/json
# Secret:        (generate random secret, store in Jenkins credential)
# Events:        Push events + Pull request events
```

> ⚠️ **Warning:** Jenkins must be accessible from GitHub's IP ranges. Check `https://api.github.com/meta` for current GitHub IP ranges. If Jenkins is behind a firewall, use polling or a webhook relay (smee.io for dev, ngrok for testing).

---

## 11. Build Tools Integration

### Maven

```groovy
pipeline {
    agent any
    tools {
        maven 'Maven-3.9'    // name configured in Global Tool Configuration
        jdk 'OpenJDK-17'
    }
    environment {
        MAVEN_OPTS = '-Xmx1g -XX:+TieredCompilation'
    }
    stages {
        stage('Build') {
            steps {
                // withMaven — captures fingerprints, publishes test results automatically
                withMaven(
                    maven: 'Maven-3.9',
                    mavenSettingsConfig: 'central-mirror-settings',   // settings.xml credential
                    publisherStrategy: 'IMPLICIT'    // auto-publish JUnit results
                ) {
                    sh 'mvn clean verify -B'    // -B: batch mode, no progress output
                }
            }
        }
    }
}
```

---

### Gradle

```groovy
stages {
    stage('Build') {
        steps {
            // Use Gradle wrapper — don't install Gradle on agents
            sh './gradlew clean build --no-daemon'    // --no-daemon: important in CI!
            // Daemon stays alive between runs — causes memory issues on ephemeral agents

            // Publish build scan to Gradle Enterprise
            withGradle {
                sh './gradlew build --scan'
            }
        }
    }
}
```

> ⚠️ **Warning:** Never use `gradle build` — always use `./gradlew`. The wrapper ensures the correct Gradle version for the project and doesn't require Gradle pre-installed on agents.

---

### Node.js

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-20'    // configured in Global Tool Configuration
    }
    stages {
        stage('Install') {
            steps {
                // npm ci — use this in CI, not npm install
                // ci: uses package-lock.json exactly, fails if lock is out of date
                sh 'npm ci --prefer-offline'     // --prefer-offline: use cache when possible
            }
        }
        stage('Test') {
            steps {
                sh 'npm test -- --reporter junit --reporter-option output=test-results.xml'
                junit 'test-results.xml'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }
    }
}
```

---

### Docker Build and Push

```groovy
stages {
    stage('Build Image') {
        steps {
            script {
                def shortCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                def imageTag = "${shortCommit}-${BUILD_NUMBER}"

                // Multi-stage build with cache from registry
                docker.withRegistry('https://registry.example.com', 'registry-creds') {
                    def image = docker.build(
                        "myapp:${imageTag}",
                        "--build-arg BUILD_NUMBER=${BUILD_NUMBER} " +
                        "--cache-from registry.example.com/myapp:cache " +
                        "--target production " +    // multi-stage: build only 'production' target
                        "."
                    )
                    image.push()                    // push with imageTag
                    image.push('latest')            // also push as 'latest'

                    // Update cache image
                    sh "docker tag myapp:${imageTag} registry.example.com/myapp:cache"
                    sh "docker push registry.example.com/myapp:cache"
                }

                env.IMAGE_TAG = imageTag            // save for later stages
            }
        }
    }
}
```

---

### JFrog Artifactory

```groovy
// Publish artifacts to Artifactory
stage('Publish') {
    steps {
        rtMavenRun(                               // Artifactory Maven run
            tool: 'Maven-3.9',
            pom: 'pom.xml',
            goals: 'clean deploy',
            deployerId: 'ARTIFACTORY_DEPLOYER'
        )
        rtPublishBuildInfo(                        // publish build info to Artifactory
            serverId: 'artifactory-prod'
        )
    }
}

// Promote build between repositories
stage('Promote') {
    steps {
        rtPromote(
            serverId: 'artifactory-prod',
            targetRepo: 'libs-release-local',      // promote to release repo
            copy: true,
            status: 'Released'
        )
    }
}
```

## 12. Testing & Quality Gates

### JUnit Plugin

```groovy
stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            // Publish test results — always run even if tests fail
            junit(
                testResults: 'target/surefire-reports/*.xml',   // glob pattern
                allowEmptyResults: false,          // fail if no results found
                skipPublishingChecks: false,       // publish to GitHub Checks API
                keepLongStdio: false               // don't store full stdout (saves disk)
            )
        }
    }
}
```

---

### SonarQube Quality Gate

```groovy
// Requires: SonarQube server configured in Manage Jenkins → System
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube-Production') {  // name configured in Jenkins
            sh '''
                mvn sonar:sonar \
                  -Dsonar.projectKey=my-service \
                  -Dsonar.projectName="My Service" \
                  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
            '''
        }
    }
}

stage('Quality Gate') {
    // Wait for SonarQube webhook — do NOT poll
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true   // fail pipeline if quality gate fails
        }
    }
}
```

> 💡 **Tip:** Configure SonarQube to send a webhook back to `$JENKINS_URL/sonarqube-webhook/` — this lets `waitForQualityGate` receive async notification instead of polling.

---

### Code Coverage — JaCoCo

```groovy
post {
    always {
        jacoco(
            execPattern: 'target/**/*.exec',       // JaCoCo .exec file
            classPattern: 'target/classes',
            sourcePattern: 'src/main/java',
            // Thresholds — pipeline becomes UNSTABLE if not met
            minimumInstructionCoverage: '70',      // 70% instruction coverage
            minimumBranchCoverage: '60',
            minimumLineCoverage: '70',
            // Delta thresholds — fail if coverage DROPS by more than N%
            deltaInstructionCoverage: '5',         // coverage can't drop > 5%
            changeBuildStatus: true
        )
    }
}
```

---

### Warnings Next Generation Plugin

```groovy
post {
    always {
        recordIssues(
            enabledForFailure: true,               // record even if build failed
            aggregatingResults: true,              // merge results from multiple tools
            tools: [
                checkStyle(pattern: 'target/checkstyle-result.xml'),
                spotBugs(pattern: 'target/spotbugsXml.xml', useRankAsPriority: true),
                pmdParser(pattern: 'target/pmd.xml'),
                taskScanner(highTags: 'FIXME', normalTags: 'TODO', lowTags: 'NOTE',
                            includePattern: '**/*.java')
            ],
            qualityGates: [
                // UNSTABLE if > 0 new errors
                [threshold: 1, type: 'NEW_ERROR', unstable: true],
                // FAILURE if total errors > 10
                [threshold: 10, type: 'TOTAL_ERROR', unstable: false]
            ]
        )
    }
}
```

---

### OWASP Dependency Check

```groovy
stage('Security Scan') {
    steps {
        dependencyCheck(
            additionalArguments: '--scan . --format XML --format HTML',
            odcInstallation: 'OWASP-DC'           // tool configured in Global Tools
        )
    }
    post {
        always {
            dependencyCheckPublisher(
                pattern: 'dependency-check-report.xml',
                failedTotalCritical: 0,            // fail on any CRITICAL CVE
                unstableTotalHigh: 5,              // unstable on > 5 HIGH CVEs
                unstableTotalMedium: 20
            )
        }
    }
}
```

---

### Quality Gate Patterns

```groovy
// unstable() — mark build as yellow/unstable rather than red/failed
// Useful for: test failures that don't block deployment, quality warnings
stage('Test') {
    steps {
        script {
            def result = sh(script: 'mvn test', returnStatus: true)
            if (result != 0) {
                unstable("Tests failed — marking build unstable")  // yellow, not red
                // Downstream jobs can check: if (currentBuild.result == 'UNSTABLE')
            }
        }
    }
}

// error() — hard fail the pipeline
stage('Security') {
    steps {
        script {
            def criticalCount = sh(
                script: "grep -c 'CRITICAL' security-report.txt || true",
                returnStdout: true
            ).trim().toInteger()
            if (criticalCount > 0) {
                error("Found ${criticalCount} critical vulnerabilities — aborting!")
            }
        }
    }
}
```

---

## 13. Artifacts & Archiving

### archiveArtifacts

```groovy
post {
    success {
        archiveArtifacts(
            artifacts: 'target/*.jar,target/*.war,dist/**',  // comma-separated globs
            excludes: 'target/*-sources.jar',                // exclude source jars
            fingerprint: true,              // generate MD5 fingerprint for tracking
            allowEmptyArchive: false,       // fail if no artifacts match
            onlyIfSuccessful: true          // only archive on successful builds
        )
    }
}

// Archive specific files with directory structure
archiveArtifacts artifacts: 'reports/**/*.html', fingerprint: true
```

---

### copyArtifacts Plugin

```groovy
// Copy artifacts from another job into this build's workspace
stage('Use Upstream Artifact') {
    steps {
        copyArtifacts(
            projectName: 'my-library-pipeline',       // source job name
            selector: lastSuccessful(),               // which build to copy from
            // Other selectors: specific(buildNumber: '42'), lastCompleted(), upstream()
            filter: 'target/*.jar',                   // which files to copy
            target: 'lib/',                           // destination in workspace
            flatten: false,                           // preserve directory structure
            optional: false                           // fail if source not found
        )
    }
}

// Copy from specific build number
copyArtifacts(
    projectName: 'my-library-pipeline',
    selector: specific(buildNumber: params.LIBRARY_BUILD),
    filter: '*.jar'
)
```

---

### Stash vs Archive Decision Matrix

```
Decision: stash or archive?

stash:                              archive:
✅ Files needed later in same       ✅ Long-term storage (days/weeks)
   pipeline run                     ✅ Accessible from other jobs
✅ Across agent boundaries          ✅ Download from UI
✅ Temporary (auto-deleted)         ✅ Artifact history across builds
✅ Fast (small files)               ✅ Deployment artifacts
❌ Large files (>50MB) — uses       ❌ Temp files not needed after run
   controller as intermediary
❌ Not accessible from other jobs
```

---

### Docker Image Tagging Strategy

```groovy
// Multi-tag strategy — tag once, push with multiple tags
stage('Push Image') {
    steps {
        script {
            def shortCommit = env.GIT_COMMIT[0..7]
            def semver = sh(script: 'git describe --tags --exact-match 2>/dev/null || echo ""',
                          returnStdout: true).trim()

            docker.withRegistry('https://registry.example.com', 'registry-creds') {
                def image = docker.build("myapp:${shortCommit}")

                image.push(shortCommit)          // registry.example.com/myapp:abc1234
                image.push(BUILD_NUMBER)         // registry.example.com/myapp:42
                image.push('latest')             // registry.example.com/myapp:latest

                if (semver) {
                    image.push(semver)           // registry.example.com/myapp:v1.2.3
                    image.push('stable')         // registry.example.com/myapp:stable
                }
            }
        }
    }
}
```

---

## 14. Notifications & Integrations

### Email — Extended Email Plugin

```groovy
post {
    failure {
        emailext(
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            to: '${DEFAULT_RECIPIENTS}',           // uses plugin's default recipients
            replyTo: 'noreply@example.com',
            mimeType: 'text/html',
            body: '''
                <h2>Build Failed</h2>
                <p><b>Job:</b> ${JOB_NAME}<br/>
                <b>Build:</b> ${BUILD_NUMBER}<br/>
                <b>Status:</b> ${BUILD_STATUS}</p>
                <p><a href="${BUILD_URL}">View Build</a></p>
                <h3>Recent Changes</h3>
                ${CHANGES_SINCE_LAST_SUCCESS}
                <h3>Console Output (last 100 lines)</h3>
                <pre>${BUILD_LOG, maxLines=100}</pre>
            '''
        )
    }
    fixed {
        mail(
            to: 'team@example.com',
            subject: "FIXED: ${env.JOB_NAME} is back to green",
            body: "Build ${env.BUILD_NUMBER} succeeded. ${env.BUILD_URL}"
        )
    }
}
```

---

### Slack Notifications

```groovy
// Simple message
slackSend(
    channel: '#deployments',
    color: 'good',                    // 'good'=green, 'warning'=yellow, 'danger'=red
    message: "Deployed ${env.IMAGE_TAG} to prod ✅"
)

// Rich Block Kit message
slackSend(
    channel: '#deployments',
    blocks: """[
        {
            "type": "header",
            "text": {"type": "plain_text", "text": "🚀 Deployment Complete"}
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": "*Job:*\\n${env.JOB_NAME}"},
                {"type": "mrkdwn", "text": "*Status:*\\n${currentBuild.currentResult}"},
                {"type": "mrkdwn", "text": "*Build:*\\n#${env.BUILD_NUMBER}"},
                {"type": "mrkdwn", "text": "*Duration:*\\n${currentBuild.durationString}"}
            ]
        },
        {
            "type": "actions",
            "elements": [{
                "type": "button",
                "text": {"type": "plain_text", "text": "View Build"},
                "url": "${env.BUILD_URL}"
            }]
        }
    ]"""
)
```

---

### Multi-channel Notification Library Function

```groovy
// vars/notify.groovy — reusable notification step
def call(Map config = [:]) {
    def result     = config.result     ?: currentBuild.currentResult
    def jobName    = env.JOB_NAME
    def buildNum   = env.BUILD_NUMBER
    def buildUrl   = env.BUILD_URL
    def duration   = currentBuild.durationString

    def emoji      = result == 'SUCCESS' ? '✅' : (result == 'UNSTABLE' ? '⚠️' : '❌')
    def color      = result == 'SUCCESS' ? 'good' : (result == 'UNSTABLE' ? 'warning' : 'danger')
    def message    = "${emoji} *${jobName}* #${buildNum}: *${result}* (${duration})"

    // Slack
    if (config.slack != false) {
        slackSend(
            channel: config.channel ?: '#ci-cd',
            color: color,
            message: "${message} | <${buildUrl}|View>"
        )
    }

    // Email on failure only
    if (result == 'FAILURE' && config.email) {
        mail(to: config.email, subject: "FAILED: ${jobName} #${buildNum}", body: buildUrl)
    }

    // PagerDuty for production failures
    if (result == 'FAILURE' && config.pagerduty) {
        httpRequest(
            url: 'https://events.pagerduty.com/v2/enqueue',
            httpMode: 'POST',
            contentType: 'APPLICATION_JSON',
            requestBody: """{"routing_key": "${config.pagerduty}",
                            "event_action": "trigger",
                            "payload": {"summary": "${jobName} failed", "severity": "critical"}}"""
        )
    }
}
```

---

### Outbound Webhooks — httpRequest Plugin

```groovy
// Post JSON to an arbitrary endpoint
stage('Notify Deploy API') {
    steps {
        script {
            def payload = """
            {
                "service": "${env.JOB_BASE_NAME}",
                "version": "${env.IMAGE_TAG}",
                "environment": "${params.ENVIRONMENT}",
                "build_url": "${env.BUILD_URL}",
                "commit": "${env.GIT_COMMIT}"
            }
            """.stripIndent()

            httpRequest(
                url: 'https://deploys.example.com/api/events',
                httpMode: 'POST',
                contentType: 'APPLICATION_JSON',
                requestBody: payload,
                customHeaders: [[name: 'Authorization', value: "Bearer ${DEPLOY_API_TOKEN}",
                                 maskValue: true]],   // maskValue: mask in logs!
                validResponseCodes: '200:299'
            )
        }
    }
}
```

---

## 15. Jenkins Configuration as Code (JCasC)

### jenkins.yaml — Full Structure

```yaml
# $JENKINS_HOME/jenkins.yaml (or path from $CASC_JENKINS_CONFIG)
jenkins:
  systemMessage: "Jenkins — managed by JCasC. Do not edit manually."
  numExecutors: 0                    # no builds on controller
  mode: EXCLUSIVE                    # only builds with controller label run here (mode=EXCLUSIVE means nothing, since 0 executors)
  quietPeriod: 5                     # seconds to wait before starting triggered build
  labelString: "controller"

security:
  globalJobDslSecurityConfiguration:
    useScriptSecurity: true          # sandbox Job DSL scripts

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "docker-hub"
              username: "${DOCKER_HUB_USER}"        # resolved from env var
              password: "${DOCKER_HUB_PASS}"
              description: "Docker Hub credentials"
          - string:
              scope: GLOBAL
              id: "sonar-token"
              secret: "${SONAR_TOKEN}"
          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: "github-deploy-key"
              username: "git"
              privateKeySource:
                directEntry:
                  privateKey: "${GITHUB_DEPLOY_KEY}"  # multiline env var

unclassified:
  slackNotifier:
    teamDomain: "myteam"
    tokenCredentialId: "slack-token"
    room: "#ci-cd"

  sonarGlobalConfiguration:
    buildWrapperEnabled: true
    installations:
      - name: "SonarQube-Production"
        serverUrl: "https://sonar.example.com"
        credentialsId: "sonar-token"

tool:
  jdk:
    installations:
      - name: "OpenJDK-17"
        home: ""
        properties:
          - installSource:
              installers:
                - adoptOpenJdkInstaller:
                    id: "jdk-17.0.9+9"              # from AdoptOpenJDK
  maven:
    installations:
      - name: "Maven-3.9"
        home: ""
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.6"                      # version to download
  nodejs:
    installations:
      - name: "NodeJS-20"
        properties:
          - installSource:
              installers:
                - nodeJSInstaller:
                    id: "20.11.0"
                    npmPackagesRefreshHours: 72
```

---

### JCasC — Security and Auth

```yaml
jenkins:
  securityRealm:
    # Option 1: Local database (simple)
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${ADMIN_PASSWORD}"

    # Option 2: LDAP
    ldap:
      configurations:
        - server: "ldap://ldap.example.com:389"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=users"
          userSearch: "uid={0}"
          groupSearchBase: "ou=groups"
          managerDN: "cn=jenkins,ou=service-accounts,dc=example,dc=com"
          managerPasswordSecret: "${LDAP_PASSWORD}"

  authorizationStrategy:
    # Role-based access control (requires Role Strategy plugin)
    roleBased:
      roles:
        global:
          - name: "admin"
            description: "Full administrator access"
            permissions:
              - "Overall/Administer"
            assignments:
              - "admin-user"
              - "ops-team"
          - name: "developer"
            description: "Read access + build trigger"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"
              - "Job/Workspace"
            assignments:
              - "authenticated"    # all authenticated users
```

---

### JCasC — Kubernetes Cloud

```yaml
jenkins:
  clouds:
    - kubernetes:
        name: "kubernetes"
        serverUrl: ""               # empty = use in-cluster config (when Jenkins is in K8s)
        namespace: "jenkins"
        jenkinsUrl: "http://jenkins.jenkins.svc.cluster.local:8080"  # internal K8s DNS
        jenkinsTunnel: "jenkins-agent.jenkins.svc.cluster.local:50000"
        connectTimeout: 30
        readTimeout: 30
        maxRequestsPerHostStr: "64"

        templates:
          - name: "default"
            label: "k8s"
            nodeUsageMode: NORMAL
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:latest"
                alwaysPullImage: false
                resourceRequestMemory: "256Mi"
                resourceRequestCpu: "100m"
                resourceLimitMemory: "512Mi"
                resourceLimitCpu: "500m"
            podRetention: never     # delete pod immediately after build
            idleMinutes: 0
```

---

### JCasC — Global Pipeline Libraries

```yaml
unclassified:
  globalLibraries:
    libraries:
      - name: "company-pipeline-lib"    # name used in @Library('company-pipeline-lib')
        defaultVersion: "main"          # default branch/tag if not specified
        allowVersionOverride: true      # allow @Library('lib@feature-branch')
        includeInChangesets: false      # don't show library changes in job changesets
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/org/jenkins-shared-library.git"
                credentialsId: "github-token"
```

---

### JCasC in Docker

```bash
# Pass JCasC config path via environment variable
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /host/path/jenkins.yaml:/var/jenkins_home/casc_configs/jenkins.yaml \
  -e CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs \
  -e DOCKER_HUB_USER=myuser \             # secrets passed as env vars
  -e DOCKER_HUB_PASS=mypassword \
  jenkins/jenkins:lts
```

```dockerfile
# Dockerfile — bake JCasC into custom Jenkins image
FROM jenkins/jenkins:lts
ENV CASC_JENKINS_CONFIG=/usr/share/jenkins/ref/casc_configs
COPY jenkins.yaml /usr/share/jenkins/ref/casc_configs/jenkins.yaml

# Pre-install plugins for faster startup
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```

---

### Exporting and Applying JCasC

```bash
# Export current Jenkins config to YAML (great starting point!)
curl -u admin:API_TOKEN \
  https://jenkins.example.com/configuration-as-code/export \
  > jenkins-current.yaml

# Validate config without applying
curl -X POST -u admin:API_TOKEN \
  https://jenkins.example.com/configuration-as-code/check \
  --data-binary @jenkins.yaml

# Apply config without restarting (hot reload)
curl -X POST -u admin:API_TOKEN \
  https://jenkins.example.com/configuration-as-code/apply
```

---

## 16. Job DSL Plugin

### Seed Job Pattern

```
Seed Job Pattern:

  Git repo: jobs/
  ├── teams/
  │   ├── payments/jobs.groovy      # defines jobs for payments team
  │   └── auth/jobs.groovy
  └── shared/pipeline-template.groovy

  Seed Job (Pipeline):
    1. Checks out the jobs repo
    2. Runs Job DSL on all *.groovy files
    3. Jenkins creates/updates all defined jobs

  Result: all jobs defined in code, seed job is the only manual job
```

```groovy
// Jenkinsfile for the seed job itself
pipeline {
    agent any
    stages {
        stage('Generate Jobs') {
            steps {
                checkout scm    // check out the job definitions repo
                jobDsl(
                    targets: 'jobs/**/*.groovy',       // process all groovy files
                    removedJobAction: 'DELETE',         // delete jobs no longer in code
                    removedViewAction: 'DELETE',
                    failOnMissingPlugin: true,
                    unstableOnDeprecation: true
                )
            }
        }
    }
}
```

---

### Job DSL — Pipeline Jobs

```groovy
// jobs/teams/payments/jobs.groovy — creates jobs via DSL

// Create a pipeline job
pipelineJob('payments/build-and-test') {
    description('Builds and tests the payments service')

    definition {
        cpsScm {                                    // use Jenkinsfile from SCM
            scm {
                git {
                    remote {
                        url('https://github.com/org/payments.git')
                        credentials('github-token')
                    }
                    branch('*/main')
                }
            }
            scriptPath('Jenkinsfile')               // path within the repo
            lightweight(true)                       // lightweight checkout for Jenkinsfile
        }
    }

    triggers {
        githubPush()                                // trigger on GitHub webhook
    }

    logRotator {
        numToKeep(20)
        daysToKeep(30)
    }
}

// Create multibranch pipeline
multibranchPipelineJob('payments/branches') {
    branchSources {
        github {
            id('payments-github')
            repoOwner('myorg')
            repository('payments')
            credentialsId('github-token')
            traits {
                gitHubBranchDiscovery { strategyId(3) }   // all branches
                gitHubPullRequestDiscovery { strategyId(2) }
            }
        }
    }
    orphanedItemStrategy {
        discardOldItems {
            numToKeep(5)
            daysToKeep(7)
        }
    }
}

// Create folder with team structure
folder('payments') {
    description('Payments team jobs')
    displayName('💳 Payments Team')
    properties {
        folderCredentialsProperty {
            domainCredentials {
                // folder-level credentials visible only to this folder's jobs
            }
        }
    }
}
```

---

### Job DSL — Generating Jobs from a List

```groovy
// Generate a job per microservice from a list
def services = [
    [name: 'auth',     repo: 'org/auth-service',    team: 'platform'],
    [name: 'payments', repo: 'org/payments-service', team: 'payments'],
    [name: 'orders',   repo: 'org/orders-service',   team: 'commerce'],
]

services.each { svc ->
    folder(svc.team)                               // create team folder if not exists

    multibranchPipelineJob("${svc.team}/${svc.name}") {
        displayName(svc.name.capitalize())
        branchSources {
            github {
                id("${svc.name}-source")
                repoOwner('org')
                repository(svc.repo.split('/')[1])
                credentialsId('github-token')
            }
        }
    }
}
```

---

### configure {} Block — Raw XML Manipulation

```groovy
// For Job DSL features not yet supported — use configure {} for raw XML
pipelineJob('my-job') {
    configure { project ->
        // project is the raw XML Node — navigate and modify directly
        project / 'properties' << {
            'com.example.CustomProperty' {           // add unsupported property
                myValue('custom-value')
            }
        }

        // Modify existing element
        (project / 'blockBuildWhenUpstreamBuilding').value = true
    }
}
```

---

## 17. Pipeline Optimisation & Performance

### @NonCPS Annotation — CPS Explained

```
CPS (Continuation Passing Style) Transformation:
Jenkins transforms pipeline Groovy code so it can be:
  - Serialized to disk (pipeline durability — survive restarts)
  - Suspended and resumed at any point
  - Executed across multiple JVM restarts

This transformation has limitations:
  - Some Groovy constructs don't serialize (iterators, closures with state)
  - Complex Groovy code that works outside Jenkins may fail in pipelines
```

```groovy
// @NonCPS — marks a method as NOT CPS-transformed
// Use when: complex Groovy that fails in CPS context, performance-critical utility methods

// 🔴 Anti-pattern: complex Groovy in CPS context (may fail)
def parseConfig(String yaml) {
    def result = [:]
    yaml.readLines().each { line ->          // iterator + closure — may fail in CPS
        def parts = line.split('=')
        result[parts[0]] = parts[1]
    }
    return result
}

// ✅ Right way: mark non-serializable operations @NonCPS
@NonCPS
def parseConfig(String yaml) {
    def result = [:]
    yaml.readLines().each { line ->          // @NonCPS: runs without CPS transformation
        def parts = line.split('=')
        if (parts.size() == 2) result[parts[0].trim()] = parts[1].trim()
    }
    return result                            // ⚠️ return value must be serializable
}
```

> ⚠️ **Warning:** `@NonCPS` methods cannot call CPS-transformed methods (like `sh`, `echo`, `checkout`). Only use `@NonCPS` for pure computation functions that don't invoke pipeline steps.

---

### Parallelisation Strategies

```groovy
// Pattern 1: fan-out (parallel) then fan-in (sequential with results)
stage('Test') {
    parallel {
        stage('Unit') {
            steps {
                sh 'mvn test -Punit'
                stash name: 'unit-results', includes: 'target/surefire-reports/**'
            }
        }
        stage('Integration') {
            steps {
                sh 'mvn test -Pintegration'
                stash name: 'it-results', includes: 'target/failsafe-reports/**'
            }
        }
    }
}
stage('Publish Results') {     // fan-in: collect from all parallel branches
    steps {
        unstash 'unit-results'
        unstash 'it-results'
        junit '**/surefire-reports/*.xml, **/failsafe-reports/*.xml'
    }
}

// Pattern 2: dynamic parallel map from list
stage('Deploy to all regions') {
    steps {
        script {
            def regions = ['us-east-1', 'eu-west-1', 'ap-southeast-1']
            def deployStages = regions.collectEntries { region ->
                ["Deploy ${region}": {
                    sh "helm upgrade --install myapp ./chart --kube-context ${region}"
                    sh "kubectl rollout status deployment/myapp --context ${region}"
                }]
            }
            parallel deployStages    // deploy to all regions simultaneously
        }
    }
}
```

---

### Pipeline Durability Settings

```groovy
// Why this matters: durability = how aggressively Jenkins checkpoints pipeline state to disk
// Higher durability = survives controller restart mid-build; lower durability = faster builds

pipeline {
    options {
        // PERFORMANCE_OPTIMIZED — fastest, minimal disk writes
        // If Jenkins restarts mid-build, the build is lost (must restart from scratch)
        // Best for: short CI builds, ephemeral pipelines
        durabilityHint('PERFORMANCE_OPTIMIZED')

        // MAX_SURVIVABILITY — writes to disk after every step
        // Best for: long-running deployments, IaC pipelines that must not be re-run
        // durabilityHint('MAX_SURVIVABILITY')
    }
}
```

---

### Caching Strategies

```groovy
// Maven cache on persistent agent workspace (static agents)
pipeline {
    agent { label 'linux-persistent' }    // use same agent for cache locality
    options {
        skipDefaultCheckout()             // we'll do it manually after cache setup
    }
    stages {
        stage('Build') {
            steps {
                checkout scm
                // .m2 cache lives in the agent's HOME — persists between builds
                sh 'mvn clean package -Dmaven.repo.local=$HOME/.m2/repository'
            }
        }
    }
}

// Docker layer cache — from registry
stage('Docker Build') {
    steps {
        sh '''
            # Pull cache layer from registry before building
            docker pull registry.example.com/myapp:cache || true

            # Use --cache-from to reuse cached layers
            docker build \
                --cache-from registry.example.com/myapp:cache \
                --tag myapp:${BUILD_NUMBER} .

            # Update cache image
            docker tag myapp:${BUILD_NUMBER} registry.example.com/myapp:cache
            docker push registry.example.com/myapp:cache
        '''
    }
}
```

---

### milestone Step — Out-of-Order Build Prevention

```groovy
// Why this matters: without milestones, older builds can complete AFTER newer ones
// resulting in older code being deployed after newer code

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                milestone(1)                      // build 43 passes milestone 1 first
                sh 'mvn package'                  // if build 42 is still here, it's aborted!
            }
        }
        stage('Deploy') {
            steps {
                milestone(2)                      // same protection for deploy stage
                sh 'helm upgrade --install myapp ./chart'
            }
        }
    }
}
```

---

## 18. Security & Hardening

### Script Security — Sandbox

```
Pipeline Script Security Model:

Sandbox (default for Multibranch/remote Jenkinsfiles):
  - Limited Groovy operations allowed
  - Blocked: file I/O, network, reflection, system calls
  - Allowlisted methods only
  - Admin must approve new method signatures

Approved Scripts (for trusted sources):
  - Full Groovy capabilities
  - Applied to: inline pipeline scripts in UI (trusted)
  - Risk: arbitrary code execution — use sparingly

Recommendation:
  - Keep sandbox enabled for SCM Jenkinsfiles
  - Put complex logic in Shared Libraries (loaded as trusted)
  - Review and minimise script approval signatures
```

---

### CSRF Protection and API Tokens

```bash
# ✅ Always use API tokens, never passwords for CI/CD
# Create token: User → Configure → Add new Token

# Using crumb (CSRF token) for older Jenkins or form submissions
CRUMB=$(curl -s -u "user:token" \
  "https://jenkins.example.com/crumbIssuer/api/json" \
  | jq -r '.crumb')

curl -X POST \
  -H "Jenkins-Crumb: ${CRUMB}" \
  -u "user:token" \
  "https://jenkins.example.com/job/my-pipeline/build"

# Modern API tokens don't need crumbs for REST API calls
curl -X POST \
  -u "${JENKINS_USER_ID}:${JENKINS_API_TOKEN}" \
  "https://jenkins.example.com/job/my-pipeline/build"
```

---

### Role-Based Access Control

```yaml
# JCasC — Role Strategy Plugin
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions: ["Overall/Administer"]
            assignments: ["admin"]

          - name: "developer"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Cancel"
              - "Job/Read"
              - "Job/Workspace"
              - "View/Read"
            assignments: ["authenticated"]

        items:                          # item roles scoped to matching jobs
          - name: "ops-deploy"
            pattern: ".*/prod/.*"       # regex matching job paths
            permissions:
              - "Job/Build"
              - "Job/Configure"
            assignments: ["ops-team"]

        nodes:
          - name: "node-manager"
            permissions: ["Computer/Configure", "Computer/Connect"]
            assignments: ["ops-team"]
```

---

### Supply Chain Security — Plugin Management

```bash
# plugins.txt — pinned plugin versions for reproducible installs
# Format: plugin-id:version

# Workflow and Pipeline
workflow-aggregator:596.v8c21c963d92d
pipeline-stage-view:2.33

# SCM
git:5.2.1
github:1.37.3
github-branch-source:1776.v803a_ccc2cb_e2

# Credentials
credentials:1311.vcf0a_900b_37c2
credentials-binding:677.vdc24b_d328cf9

# Install from file (in Docker image or via jenkins-plugin-cli)
# jenkins-plugin-cli --plugin-file plugins.txt

# Verify checksums — download and verify plugin signatures
jenkins-plugin-cli --plugin-file plugins.txt --verify-checksums
```

---

### Agent-to-Controller Security

```groovy
// ✅ Always specify required label — don't use agent any for sensitive builds
pipeline {
    agent { label 'trusted-agent' }    // only run on known-good agents
}

// ⚠️ Warning: agent any with untrusted workspaces is a security risk
// Compromised agent can exfiltrate credentials, workspace data

// Restrict what agents can do — Manage Jenkins → Security → Agent Protocols
// Enable: Inbound TCP Agent Protocol/4 (JNLP4)
// Disable: Legacy protocols (JNLP, JNLP2, JNLP3) — they lack encryption
```

---

## 19. Monitoring, Observability & Maintenance

### Prometheus Metrics

```yaml
# Prometheus scrape config for Jenkins
scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/prometheus'          # requires Prometheus Metrics plugin
    scheme: 'https'
    static_configs:
      - targets: ['jenkins.example.com']
    basic_auth:
      username: 'prometheus'
      password_file: '/etc/prometheus/jenkins-token'
```

```
Key Jenkins metrics to alert on:
┌─────────────────────────────────────────────────────────────────┐
│ Metric                           │ Alert Threshold              │
├─────────────────────────────────────────────────────────────────┤
│ jenkins_queue_size_value         │ > 10 for > 5 minutes         │
│ jenkins_executor_count_value     │ utilisation > 90%            │
│ jenkins_builds_failed_total      │ spike compared to baseline   │
│ jenkins_builds_duration_seconds  │ > 2x typical p99             │
│ jvm_memory_used_bytes            │ > 85% of heap max            │
│ jvm_gc_pause_seconds             │ > 1s GC pause (old gen leak) │
└─────────────────────────────────────────────────────────────────┘
```

---

### Groovy System Scripts

```groovy
// Run at: Manage Jenkins → Script Console

// List all jobs and their last build status
Jenkins.instance.getAllItems(Job).each { job ->
    def lastBuild = job.getLastBuild()
    def status = lastBuild ? lastBuild.getResult() : 'NEVER_BUILT'
    println "${job.fullName}: ${status}"
}

// Disable all jobs matching a pattern
Jenkins.instance.getAllItems(Job).findAll {
    it.fullName.contains('temp-')         // match jobs with 'temp-' in name
}.each { job ->
    job.disabled = true
    println "Disabled: ${job.fullName}"
}

// Kill all stuck builds (running > 2 hours)
def cutoff = System.currentTimeMillis() - (2 * 60 * 60 * 1000)   // 2 hours ago
Jenkins.instance.getAllItems(Job).each { job ->
    job.builds.findAll { build ->
        build.isBuilding() && build.startTimeInMillis < cutoff
    }.each { build ->
        build.doStop()
        println "Stopped: ${job.fullName} #${build.number}"
    }
}

// Trigger parameterised build programmatically
def job = Jenkins.instance.getItemByFullName('my-pipeline')
def params = [
    new StringParameterValue('ENVIRONMENT', 'staging'),
    new BooleanParameterValue('SKIP_TESTS', false)
]
def future = job.scheduleBuild2(0, new ParametersAction(params))
println "Triggered build: ${future.get().number}"

// Clean up old workspaces (free disk space)
Jenkins.instance.nodes.each { node ->
    node.workspaceList.each { workspace ->
        def path = workspace.getRemote()
        println "Workspace: ${path}"
        // workspace.deleteContents()    // uncomment to actually delete
    }
}
```

---

### Backup Strategy

```bash
# What to back up — $JENKINS_HOME contents:
# ✅ config.xml                  — main config
# ✅ jobs/*/config.xml            — job configs
# ✅ jobs/*/builds/*/build.xml   — build history metadata
# ✅ nodes/*/config.xml          — agent configs
# ✅ users/                       — user accounts
# ✅ secrets/                     — credentials (encrypt at rest!)
# ✅ plugins/*.jpi                — installed plugins
# ✅ *.xml (root level)           — plugin configs
#
# ❌ workspace/                   — can be rebuilt from source
# ❌ cache/                       — download cache, rebuildable
# ❌ jobs/*/builds/*/log          — optional: build logs (large)

# Minimal backup script
rsync -av \
  --exclude='workspace/' \
  --exclude='cache/' \
  --exclude='**/log' \
  /var/lib/jenkins/ \
  s3://my-backup-bucket/jenkins/$(date +%Y-%m-%d)/
```

---

## 20. Real-World Pipeline Patterns & Recipes

### Full CI Pipeline

```groovy
pipeline {
    agent { label 'linux && docker' }
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
        disableConcurrentBuilds()
    }
    environment {
        IMAGE     = "registry.example.com/${env.JOB_BASE_NAME}"
        SHORT_SHA = "${GIT_COMMIT[0..7]}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test -- --reporter junit'
                        junit 'test-results.xml'
                    }
                }
                stage('SAST') {
                    steps { sh 'semgrep --config=auto --json > semgrep.json || true' }
                }
                stage('Dependency Check') {
                    steps { sh 'npm audit --audit-level=high' }
                }
            }
        }
        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE}:${SHORT_SHA} ."
                sh "docker push ${IMAGE}:${SHORT_SHA}"
            }
        }
    }
    post {
        always { cleanWs() }
        success { notify(slack: true) }
        failure { notify(slack: true, pagerduty: env.PAGERDUTY_KEY) }
    }
}
```

---

### Full CD Pipeline

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'IMAGE_TAG', description: 'Image tag from CI pipeline')
        choice(name: 'START_FROM', choices: ['dev', 'staging', 'prod'],
               description: 'Skip earlier environments')
    }
    stages {
        stage('Deploy Dev') {
            when { expression { params.START_FROM == 'dev' } }
            steps {
                sh "helm upgrade --install myapp ./chart -n dev --set image.tag=${params.IMAGE_TAG}"
                sh "kubectl rollout status deployment/myapp -n dev --timeout=120s"
            }
        }
        stage('Integration Tests') {
            when { expression { params.START_FROM == 'dev' } }
            steps { sh './integration-tests.sh https://dev.example.com' }
        }
        stage('Approve → Staging') {
            options { timeout(time: 4, unit: 'HOURS') }
            steps {
                input message: "Deploy ${params.IMAGE_TAG} to Staging?",
                      submitter: 'dev-leads,qa-team'
            }
        }
        stage('Deploy Staging') {
            when { expression { params.START_FROM in ['dev', 'staging'] } }
            steps {
                sh "helm upgrade --install myapp ./chart -n staging --set image.tag=${params.IMAGE_TAG}"
            }
        }
        stage('Smoke Tests') {
            steps { sh './smoke-tests.sh https://staging.example.com' }
        }
        stage('Approve → Production') {
            options { timeout(time: 24, unit: 'HOURS') }
            steps {
                input message: "🚨 Deploy ${params.IMAGE_TAG} to PRODUCTION?",
                      ok: 'Deploy to Production',
                      submitter: 'cto,vp-engineering,ops-lead'
            }
        }
        stage('Deploy Production') {
            steps {
                milestone(10)    // prevent out-of-order deploys
                sh "helm upgrade --install myapp ./chart -n prod --set image.tag=${params.IMAGE_TAG} --atomic --timeout 10m"
                // --atomic: rollback automatically if deploy fails
            }
        }
    }
    post {
        success { notify(message: "✅ Deployed ${params.IMAGE_TAG} to prod", slack: true) }
        failure { notify(message: "❌ Deploy FAILED", slack: true, pagerduty: env.PD_KEY) }
    }
}
```

---

### Terraform IaC Pipeline

```groovy
pipeline {
    agent { label 'linux' }
    environment {
        TF_WORKSPACE = params.ENVIRONMENT
        TF_VAR_region = params.AWS_REGION
    }
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'])
        choice(name: 'AWS_REGION',  choices: ['us-east-1', 'eu-west-1'])
        choice(name: 'ACTION',      choices: ['plan', 'apply', 'destroy'])
    }
    stages {
        stage('Terraform Init') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: "aws-${params.ENVIRONMENT}"]]) {
                    sh '''
                        terraform init \
                          -backend-config="bucket=tf-state-${TF_WORKSPACE}" \
                          -backend-config="key=myapp/terraform.tfstate" \
                          -reconfigure
                    '''
                }
            }
        }
        stage('Terraform Plan') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: "aws-${params.ENVIRONMENT}"]]) {
                    sh 'terraform plan -out=tfplan -detailed-exitcode'
                    sh 'terraform show -no-color tfplan > plan.txt'
                    archiveArtifacts artifacts: 'plan.txt'
                }
            }
        }
        stage('Approve') {
            when { expression { params.ACTION == 'apply' } }
            options { timeout(time: 4, unit: 'HOURS') }
            steps {
                script {
                    def planText = readFile('plan.txt')
                    input(message: "Review the plan and approve:",
                          parameters: [text(name: 'PLAN', defaultValue: planText,
                                           description: 'Terraform plan output')])
                }
            }
        }
        stage('Terraform Apply') {
            when { expression { params.ACTION == 'apply' } }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: "aws-${params.ENVIRONMENT}"]]) {
                    sh 'terraform apply tfplan'
                }
            }
        }
    }
}
```

---

### Monorepo Pipeline — Change Detection

```groovy
// Detect which services changed and build only those
pipeline {
    agent any
    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // Get list of changed files since last successful build
                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD",
                        returnStdout: true
                    ).trim().split('\n')

                    // Map service directories to changed flag
                    def services = ['auth', 'payments', 'orders', 'frontend']
                    env.CHANGED_SERVICES = services.findAll { svc ->
                        changedFiles.any { file -> file.startsWith("services/${svc}/") }
                    }.join(',')

                    echo "Changed services: ${env.CHANGED_SERVICES}"
                    if (!env.CHANGED_SERVICES) {
                        echo "No service changes detected — skipping builds"
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Build Changed Services') {
            steps {
                script {
                    def changed = env.CHANGED_SERVICES.split(',')
                    def buildStages = changed.collectEntries { svc ->
                        ["Build ${svc}": {
                            dir("services/${svc}") {
                                sh "docker build -t ${svc}:${BUILD_NUMBER} ."
                            }
                        }]
                    }
                    parallel buildStages
                }
            }
        }
    }
}
```

---

### Emergency Hotfix Pipeline

```groovy
// Expedited path: faster feedback, fewer gates, extra notifications
pipeline {
    agent any
    parameters {
        string(name: 'HOTFIX_BRANCH', description: 'Branch containing the hotfix')
        string(name: 'INCIDENT_ID',   description: 'Incident/ticket ID for audit trail')
    }
    options {
        timeout(time: 30, unit: 'MINUTES')    // strict timeout on hotfix
        timestamps()
    }
    stages {
        stage('Hotfix Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: params.HOTFIX_BRANCH]],
                    userRemoteConfigs: [[url: scm.userRemoteConfigs[0].url,
                                        credentialsId: scm.userRemoteConfigs[0].credentialsId]]
                ])
            }
        }
        stage('Critical Tests Only') {
            parallel {
                stage('Smoke Tests') {
                    steps { sh 'mvn test -Psmoke-only' }
                }
                stage('Security Scan') {
                    steps { sh 'trivy image myapp:hotfix --exit-code 1 --severity CRITICAL' }
                }
            }
        }
        stage('Hotfix Approval') {
            options { timeout(time: 2, unit: 'HOURS') }    // short approval window
            steps {
                input(
                    message: "⚡ HOTFIX: Deploy ${params.HOTFIX_BRANCH} to prod? Incident: ${params.INCIDENT_ID}",
                    submitter: 'cto,vp-engineering',
                    ok: 'Deploy Hotfix'
                )
            }
        }
        stage('Deploy Hotfix') {
            steps {
                sh "helm upgrade --install myapp ./chart --set image.tag=hotfix --atomic"
            }
        }
    }
    post {
        always {
            // Extra audit notifications for hotfix deployments
            notify(
                message: "🚨 HOTFIX ${currentBuild.result}: ${params.HOTFIX_BRANCH} - Incident: ${params.INCIDENT_ID}",
                slack: true,
                slackChannel: '#incidents',
                email: 'leadership@example.com'
            )
        }
    }
}
```

---

## Appendix: Quick Reference

### Built-in Environment Variables

| Variable | Value |
|----------|-------|
| `BUILD_NUMBER` | `"42"` |
| `BUILD_URL` | `"https://jenkins.example.com/job/my-pipeline/42/"` |
| `JOB_NAME` | `"team/my-pipeline"` (with folder path) |
| `JOB_BASE_NAME` | `"my-pipeline"` (without folder path) |
| `WORKSPACE` | `"/home/jenkins/workspace/my-pipeline"` |
| `NODE_NAME` | `"agent-1"` or `"built-in"` |
| `BRANCH_NAME` | `"main"` (Multibranch only) |
| `CHANGE_ID` | PR number (Multibranch PR builds) |
| `GIT_COMMIT` | Full SHA after checkout |
| `GIT_BRANCH` | `"origin/main"` |

### Declarative Pipeline Skeleton

```groovy
pipeline {
    agent { label 'linux' }
    options { timestamps(); timeout(time: 1, unit: 'HOURS') }
    environment { APP = 'myapp' }
    parameters { string(name: 'TAG', defaultValue: 'latest') }
    triggers { pollSCM('H/5 * * * *') }
    stages {
        stage('Build') {
            when { branch 'main' }
            steps { sh 'make build' }
        }
    }
    post {
        always  { cleanWs() }
        success { echo 'Success' }
        failure { echo 'Failed' }
    }
}
```

### Plugin Essentials

| Plugin | Purpose |
|--------|---------|
| `workflow-aggregator` | Pipeline support (meta-plugin) |
| `git` | Git SCM integration |
| `github-branch-source` | Multibranch for GitHub |
| `credentials-binding` | `withCredentials` step |
| `kubernetes` | K8s pod agents |
| `configuration-as-code` | JCasC support |
| `job-dsl` | Programmatic job creation |
| `slack` | Slack notifications |
| `junit` | Test result publishing |
| `sonar` | SonarQube integration |
| `blueocean` | Modern pipeline visualisation |

---

*Jenkins version: 2.x LTS recommended. Always test plugin updates in a staging Jenkins instance before applying to production.*

*Reference: [Jenkins Documentation](https://www.jenkins.io/doc/) | [Pipeline Steps Reference](https://www.jenkins.io/doc/pipeline/steps/) | [JCasC Schema](https://www.jenkins.io/projects/jcasc/)*
