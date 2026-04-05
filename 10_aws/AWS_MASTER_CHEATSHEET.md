# AWS Master Reference Cheatsheet
> Senior Solutions Architect Edition — All Services, CLI, Limits, Patterns

---

# Table of Contents
1. [Compute](#1-compute)
2. [Storage](#2-storage)
3. [Databases](#3-databases)
4. [Networking & Content Delivery](#4-networking--content-delivery)
5. [Security, Identity & Compliance](#5-security-identity--compliance)
6. [Management & Governance](#6-management--governance)
7. [Developer Tools](#7-developer-tools)
8. [Analytics](#8-analytics)
9. [AI & Machine Learning](#9-ai--machine-learning)
10. [Application Integration](#10-application-integration)
11. [Business Applications](#11-business-applications)
12. [Front-End Web & Mobile](#12-front-end-web--mobile)
13. [IoT](#13-iot)
14. [Containers](#14-containers)
15. [Migration & Transfer](#15-migration--transfer)
16. [End-User Computing](#16-end-user-computing)
17. [Media Services](#17-media-services)
18. [Cost Management](#18-cost-management)
19. [Blockchain & Quantum](#19-blockchain--quantum)
20. [Satellite & Robotics](#20-satellite--robotics)
21. [Global Infrastructure Overview](#21-global-infrastructure-overview)
22. [Shared Responsibility Model](#22-aws-shared-responsibility-model)
23. [Well-Architected Framework](#23-aws-well-architected-framework)
24. [IAM Policy Anatomy](#24-iam-policy-anatomy)
25. [Networking Deep Dive](#25-aws-networking-deep-dive)
26. [Service Limits — Top 20 Quotas](#26-aws-service-limits--top-20-most-hit-quotas)
27. [CLI Quick Reference](#27-cli-quick-reference)
28. [Cost Optimization Cheat Cards](#28-cost-optimization-cheat-cards)
29. [Disaster Recovery Patterns](#29-disaster-recovery-patterns)

---

# 1. Compute

> AWS Compute covers a broad spectrum from long-running virtual machines to ephemeral serverless functions and purpose-built HPC clusters. Choosing the right compute primitive determines cost, performance, and operational burden.

---

## 1.1 EC2 (Elastic Compute Cloud)

**What it is:** Virtual servers in the cloud. You control the OS, networking, storage, and lifecycle. The foundational IaaS building block of AWS.

**Instance Families:**

| Family | Optimized For | Example Types |
|--------|--------------|---------------|
| M (General) | Balanced CPU/RAM | m7i, m7g, m6i, m5 |
| C (Compute) | High CPU | c7i, c7g, c6i, c5n |
| R (Memory) | High RAM | r7i, r7g, r6i, r5 |
| X (Memory) | Extreme RAM (up to 24TB) | x2idn, x1e |
| I (Storage) | NVMe local SSD | i4i, i3en |
| D (Dense) | HDD dense storage | d3en |
| H (HDD) | Throughput HDD | h1 |
| P/G/Inf (Accel) | GPU/ML | p4d, g5, inf2, trn1 |
| F (FPGA) | FPGAs | f1 |
| T (Burstable) | Burst CPU | t4g, t3a, t3 |
| Mac | macOS workloads | mac1, mac2 |
| Hpc | HPC/MPI | hpc6a, hpc7g |

**Purchase Models:**

| Model | Discount | Use Case | Commitment |
|-------|---------|---------|------------|
| On-Demand | 0% | Dev/test, unpredictable | None |
| Reserved (1yr) | ~40% | Steady-state workloads | 1 year |
| Reserved (3yr) | ~60% | Long-running baseline | 3 years |
| Savings Plans (Compute) | ~66% | Flexible family/region | 1–3 yr |
| Spot | ~90% | Fault-tolerant, batch | Interruptible |
| Dedicated Host | varies | License compliance, tenancy | On-demand/Reserved |
| Dedicated Instance | varies | Dedicated hardware | No host affinity |
| Capacity Reservations | On-Demand price | Reserve capacity in AZ | None |

**Placement Groups:**

| Type | Description | Use Case |
|------|-------------|---------|
| Cluster | Same rack, low latency | HPC, tightly coupled |
| Spread | Different racks/AZs | HA, max isolation |
| Partition | Groups of racks | Hadoop, Kafka, Cassandra |

**Key Features:**
- AMIs (Amazon Machine Images) — snapshot-based OS images, shareable across accounts/regions
- User Data — bootstrap scripts run at launch
- Instance Metadata Service (IMDSv2) — token-based metadata retrieval
- Enhanced Networking (ENA, SR-IOV) — up to 100 Gbps
- Elastic Fabric Adapter (EFA) — RDMA-capable, HPC networking
- Nitro Hypervisor — bare-metal performance, dedicated hardware security
- EC2 Hibernate — saves RAM to EBS root volume, fast resume
- Elastic Network Interface (ENI) — multiple NICs per instance
- EC2 Auto Scaling — scale in/out based on demand

**Common Use Cases:**
- Web/app servers requiring OS-level control
- Lift-and-shift migrations
- HPC workloads (EFA + placement groups)
- Windows workloads with BYOL licensing
- GPU-accelerated ML training

**Key CLI Commands:**
```bash
# Launch instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --subnet-id subnet-12345678 \
  --security-group-ids sg-12345678 \
  --iam-instance-profile Name=MyRole \
  --user-data file://bootstrap.sh \
  --count 1

# Describe instances with filter
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PrivateIpAddress]' \
  --output table

# Stop / Start / Terminate
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Create AMI
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "MyApp-v1.0" \
  --no-reboot

# Spot request
aws ec2 request-spot-instances \
  --spot-price "0.05" \
  --instance-count 2 \
  --type "one-time" \
  --launch-specification file://spec.json

# Describe Spot price history
aws ec2 describe-spot-price-history \
  --instance-types m5.large \
  --product-descriptions "Linux/UNIX" \
  --start-time 2024-01-01T00:00:00Z

# Modify instance type (must be stopped)
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --instance-type "{\"Value\": \"m5.xlarge\"}"

# Create Auto Scaling group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateId=lt-12345,Version='$Latest' \
  --min-size 2 --max-size 10 --desired-capacity 4 \
  --vpc-zone-identifier "subnet-11111,subnet-22222"
```

**Pricing Model:**
- On-Demand: per-second billing (minimum 60 seconds) for Linux; per-hour for Windows
- Reserved: partial or full upfront; significant discounts
- Spot: market price, can be reclaimed with 2-min warning
- Data transfer: $0.09/GB out to internet (first 10TB/month)

**Integration Points:** EBS, EFS, ENI, Elastic IP, Auto Scaling, ALB/NLB, CloudWatch, SSM, IAM instance profiles, VPC, Route 53, CloudFormation

**Gotchas / Limits:**
- Default limit: 32 vCPUs On-Demand per region (request increase)
- Spot interruption notice is 2 minutes — use Spot interruption handler in ASG
- IMDSv1 is insecure; enforce IMDSv2 via policy/launch template
- Cluster placement groups limited to single AZ
- EBS root volume NOT deleted by default if you change `DeleteOnTermination`
- Hibernate requires encrypted root EBS volume
- T-series credits exhaust at 100% CPU; unlimited mode incurs extra cost

---

## 1.2 Lambda

**What it is:** Serverless, event-driven compute. You upload code; AWS runs it in response to triggers. No servers to manage, auto-scales to zero.

**Key Features:**
- Runtimes: Node.js, Python, Java, .NET, Go, Ruby, custom (container image up to 10 GB)
- Memory: 128 MB – 10,240 MB (CPU scales proportionally)
- Timeout: max 15 minutes
- Ephemeral `/tmp` storage: 512 MB – 10,240 MB
- Lambda Layers — share common code/libraries across functions
- Lambda Extensions — monitoring, security sidecars
- Provisioned Concurrency — pre-warm instances to eliminate cold starts
- Reserved Concurrency — cap max instances for a function
- Function URLs — built-in HTTPS endpoint without API Gateway
- SnapStart (Java) — snapshot initialized execution environment for fast startup
- Container image support — package with Docker (up to 10 GB)
- Lambda@Edge — runs at CloudFront edge locations (max 5 s timeout on viewer events)
- Event Source Mappings — SQS, Kinesis, DynamoDB Streams polling

**Common Use Cases:**
- API backends (behind API Gateway or ALB)
- Event-driven processing (S3 uploads, DynamoDB Streams)
- Scheduled tasks (EventBridge cron)
- Real-time data transformation in Kinesis/Firehose
- Image/video processing
- Chatbots, webhooks

**Key CLI Commands:**
```bash
# Create function
aws lambda create-function \
  --function-name my-function \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler app.handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 512

# Invoke synchronously
aws lambda invoke \
  --function-name my-function \
  --payload '{"key":"value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json

# Invoke asynchronously
aws lambda invoke \
  --function-name my-function \
  --invocation-type Event \
  --payload '{"key":"value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json

# Update function code
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Publish version
aws lambda publish-version --function-name my-function

# Create alias
aws lambda create-alias \
  --function-name my-function \
  --name prod \
  --function-version 5

# Add event source mapping (SQS)
aws lambda create-event-source-mapping \
  --function-name my-function \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
  --batch-size 10

# Put provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 50

# List functions
aws lambda list-functions --query 'Functions[*].[FunctionName,Runtime,Timeout]' --output table
```

**Pricing Model:**
- Requests: $0.20 per 1M requests
- Duration: $0.0000166667 per GB-second
- Free tier: 1M requests + 400,000 GB-seconds/month forever
- Provisioned Concurrency: separate per-second charge

**Integration Points:** API Gateway, ALB, S3, DynamoDB, SQS, SNS, EventBridge, Kinesis, Step Functions, CloudWatch, X-Ray, Secrets Manager, VPC

**Gotchas / Limits:**
- Default concurrent executions: 1,000 per region (soft limit)
- 250 MB deployment package (unzipped), 50 MB zipped (S3 bypasses this)
- Cold starts most severe in Java and .NET — use SnapStart or Provisioned Concurrency
- VPC Lambda adds ENI attachment latency (mitigated by Hyperplane ENI)
- `/tmp` is shared across warm invocations — do NOT store secrets there
- Lambda@Edge: 128 MB max memory, US-East-1 deployment only

---

## 1.3 ECS (Elastic Container Service)

**What it is:** AWS-native container orchestration service. Runs Docker containers on either EC2 (you manage instances) or Fargate (serverless). No Kubernetes control plane to manage.

**Key Features:**
- Task Definitions — JSON blueprint specifying containers, CPU, memory, networking
- Services — maintain desired task count, integrate with ALB/NLB
- Clusters — logical grouping of EC2 or Fargate capacity
- Capacity Providers — link ASGs or Fargate to clusters
- Service Auto Scaling — scale tasks based on CloudWatch metrics
- ECS Anywhere — run ECS tasks on on-premises/other cloud servers
- Exec into container: `ecs execute-command`
- Service Connect — built-in service mesh via Cloud Map
- Task-level IAM roles (via Task Role)

**Common Use Cases:**
- Microservices on containers without Kubernetes complexity
- Batch jobs with containerized workloads
- CI/CD pipelines deploying containerized apps
- Web APIs with blue/green deployments via CodeDeploy

**Key CLI Commands:**
```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create cluster
aws ecs create-cluster --cluster-name production

# Create service
aws ecs create-service \
  --cluster production \
  --service-name web-api \
  --task-definition web-api:5 \
  --desired-count 4 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-111,subnet-222],securityGroups=[sg-abc],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=web,containerPort=8080"

# Update service (rolling deploy)
aws ecs update-service \
  --cluster production \
  --service web-api \
  --task-definition web-api:6 \
  --force-new-deployment

# List tasks
aws ecs list-tasks --cluster production --service-name web-api

# Exec into container
aws ecs execute-command \
  --cluster production \
  --task arn:aws:ecs:... \
  --container web \
  --interactive \
  --command "/bin/sh"

# Describe services
aws ecs describe-services \
  --cluster production \
  --services web-api \
  --query 'services[0].deployments'
```

**Pricing Model:**
- EC2 launch type: pay for EC2 instances only
- Fargate: per vCPU/second and per GB-memory/second
- ECS itself is free; you pay for underlying compute

**Integration Points:** ECR, ALB/NLB, CloudWatch, IAM, Secrets Manager, EFS, FSx, Service Connect, Cloud Map, CodeDeploy, CloudFormation

**Gotchas / Limits:**
- Task definition revisions are immutable — you always create new ones
- `awsvpc` networking mode is required for Fargate
- Container CPU units are out of 1024 (1024 = 1 vCPU)
- Service quotas: 2,000 tasks per service, 500 services per cluster
- ECS Exec requires SSM agent in container + IAM permissions

---

## 1.4 EKS (Elastic Kubernetes Service)

**What it is:** Managed Kubernetes control plane. AWS manages etcd, API server, and control plane availability. You manage worker nodes (EC2) or use Fargate for serverless pods.

**Key Features:**
- Managed node groups — EC2 nodes with automated patching and scaling
- Self-managed nodes — full control over node lifecycle
- EKS Fargate profiles — run pods serverless (no node management)
- EKS Anywhere — run EKS on-prem or other clouds
- EKS Add-ons — managed VPC CNI, CoreDNS, kube-proxy, EBS CSI driver
- AWS Load Balancer Controller — provisions ALB/NLB from Ingress/Service
- IRSA (IAM Roles for Service Accounts) — pod-level IAM via OIDC
- EKS Blueprints / CDK patterns for day-1 cluster config
- Karpenter — open-source node autoprovisioner (faster than Cluster Autoscaler)
- GuardDuty EKS Runtime Monitoring

**Common Use Cases:**
- Kubernetes-native microservices
- ML workloads with GPU nodes (Karpenter + GPU node class)
- Multi-tenant platforms
- GitOps deployments (Flux, ArgoCD)

**Key CLI Commands:**
```bash
# Create cluster (eksctl)
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# Update kubeconfig
aws eks update-kubeconfig --name prod-cluster --region us-east-1

# Create managed node group
aws eks create-nodegroup \
  --cluster-name prod-cluster \
  --nodegroup-name gpu-workers \
  --node-role arn:aws:iam::123456789012:role/NodeRole \
  --instance-types g5.xlarge \
  --scaling-config minSize=0,maxSize=10,desiredSize=2 \
  --subnets subnet-111 subnet-222

# List add-ons
aws eks list-addons --cluster-name prod-cluster

# Create add-on
aws eks create-addon \
  --cluster-name prod-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/ebs-csi-role

# Describe cluster
aws eks describe-cluster --name prod-cluster \
  --query 'cluster.[name,version,status,endpoint]'

# Associate OIDC provider (for IRSA)
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster --approve
```

**Pricing Model:**
- Control plane: $0.10/hour per cluster (~$72/month)
- Worker nodes: EC2 on-demand/reserved/spot pricing
- Fargate pods: per vCPU/second + per GB-memory/second
- EKS Anywhere: $0.90/cluster/hour for connected clusters

**Integration Points:** EC2 Auto Scaling, ALB/NLB, ECR, IAM, EBS/EFS/FSx CSI drivers, CloudWatch Container Insights, GuardDuty, VPC, Route 53, Secrets Manager, AppMesh

**Gotchas / Limits:**
- EKS control plane upgrade must go one minor version at a time
- Fargate doesn't support DaemonSets or privileged containers
- Default: 110 pods per node (ENI-based limit — use prefix delegation to increase)
- Managed node group AMI must match cluster Kubernetes version
- EKS EOL follows upstream Kubernetes: ~14 months support per version

---

## 1.5 AWS Fargate

**What it is:** Serverless compute engine for containers. Removes the need to manage EC2 instances — you define CPU/memory at the task/pod level, and AWS provisions the underlying infrastructure.

**Key Features:**
- Works with both ECS and EKS
- Per-task isolation — each task gets its own micro-VM (Firecracker)
- No shared kernel between tasks
- Fargate Spot — up to 70% discount, interruptible
- EFS integration for persistent storage
- VPC networking with ENI per task

**Pricing Model:**
- ECS Fargate: vCPU/second ($0.04048/vCPU-hr) + GB-memory/second ($0.004445/GB-hr)
- EKS Fargate: slightly different rates
- Fargate Spot: ~70% discount

**Gotchas / Limits:**
- No GPU support on Fargate
- No privileged containers
- No host networking
- 20 GB ephemeral storage (expandable to 200 GB)
- Fargate Spot not available for EKS

---

## 1.6 AWS Batch

**What it is:** Fully managed batch computing. Dynamically provisions optimal compute (EC2, Spot, Fargate) to run hundreds of thousands of containerized batch jobs.

**Key Features:**
- Job Queues — prioritized queues linked to Compute Environments
- Compute Environments — managed or unmanaged EC2/Fargate/Spot fleets
- Job Definitions — Docker container spec for each job type
- Array Jobs — run many identical jobs with a single submission
- Multi-node Parallel Jobs — distributed computing with EFA
- Fair-share scheduling — equitable compute distribution across teams

**Common Use Cases:**
- Genomics, financial risk analysis, rendering, ML training
- ETL pipelines, media transcoding

```bash
# Submit job
aws batch submit-job \
  --job-name my-batch-job \
  --job-queue high-priority \
  --job-definition my-job-def:3 \
  --parameters '{"inputBucket":"my-bucket","key":"data.csv"}'

# Describe jobs
aws batch describe-jobs --jobs job-id-1 job-id-2

# List jobs in queue
aws batch list-jobs --job-queue high-priority --job-status RUNNING
```

**Pricing Model:** Pay only for EC2/Fargate used by jobs. Batch orchestration is free.

**Gotchas / Limits:**
- Job duration max: 14 days
- Spot interruptions in Batch jobs: jobs are automatically retried up to `attempts` count
- Compute environment min vCPUs: set to 0 to scale to zero

---

## 1.7 Elastic Beanstalk

**What it is:** PaaS — upload your application code and Beanstalk handles provisioning, load balancing, auto-scaling, and monitoring. Supports multiple platforms.

**Key Features:**
- Supported platforms: Node.js, Java, .NET, PHP, Ruby, Python, Go, Docker
- Deployment policies: All at once, Rolling, Rolling with extra batch, Immutable, Blue/Green
- Managed platform updates
- Environment tiers: Web Server and Worker
- `.ebextensions` — customize EC2 config via YAML/JSON
- Saved configurations and environment cloning

**Pricing Model:** Free for Beanstalk itself; pay for underlying EC2, ELB, RDS, etc.

**Gotchas / Limits:**
- Immutable deployments briefly double instance count
- `.ebextensions` execute before app deployment
- For Docker, supports single container and multi-container (ECS-based)

---

## 1.8 AWS Lightsail

**What it is:** Simplified VPS service — predictable monthly pricing, pre-configured stacks (WordPress, LAMP, Node.js, etc.). Designed for developers who want simplicity.

**Pricing Model:** Flat monthly fee ($3.50–$160+/month) including compute, storage, and data transfer.

**Gotchas:** Limited to Lightsail-native resources; VPC peering to AWS VPC costs extra.

---

## 1.9 AWS Outposts

**What it is:** Bring AWS hardware to your data center. Fully managed rack(s) running AWS infrastructure on-premises with native AWS APIs.

**Variants:**
- Outposts Rack — full 42U rack (from 1 to many)
- Outposts Servers — 1U/2U servers for smaller deployments

**Key Features:**
- Runs EC2, ECS, EKS, RDS, EMR, ElastiCache, ALB locally
- Local gateway (LGW) for on-prem network routing
- S3 on Outposts for local object storage
- Consistent API with AWS cloud

**Pricing:** Hardware + infrastructure support subscription (3-year term, ~$200K–$500K+/yr depending on config)

---

## 1.10 App Runner

**What it is:** Fully managed service to deploy containerized web applications and APIs. Just provide a container image or source code — no infrastructure management.

**Key Features:**
- Auto-scaling from 0 to N instances
- Built-in load balancing and TLS
- Supports containers from ECR or source from CodeCommit/GitHub
- VPC connector for private resources

**Pricing:** Per vCPU-second and per GB-memory-second of provisioned capacity + per request.

---

## 1.11 AWS Wavelength

**What it is:** Deploy AWS compute and storage at the edge of 5G networks (inside telecom providers). Ultra-low latency (<10 ms) to mobile devices.

**Use Cases:** AR/VR, live video, autonomous vehicles, real-time gaming.

---

## 1.12 Nitro Enclaves

**What it is:** Isolated compute environments inside EC2 instances for processing highly sensitive data. No persistent storage, no interactive access, no external networking.

**Use Cases:** Private key operations, credential processing, genomics PII processing.

---

## 1.13 AWS ParallelCluster

**What it is:** Open-source tool to deploy and manage HPC clusters on AWS. Automates provisioning of head node, compute nodes, shared storage, and job schedulers.

**Key Features:**
- Supports Slurm, AWS Batch as schedulers
- EFA networking for MPI workloads
- FSx for Lustre integration for shared scratch storage

---

## 1.14 Serverless Application Repository (SAR)

**What it is:** Managed repository of serverless applications. Publish and deploy pre-built Lambda-based apps (your own or community-shared).

---

---

# 2. Storage

> AWS Storage services span object, block, file, and archival tiers. Selecting the right type depends on access patterns, latency requirements, cost sensitivity, and protocol needs.

---

## 2.1 S3 (Simple Storage Service)

**What it is:** Infinitely scalable object storage. Store and retrieve any amount of data from anywhere. Industry-standard for data lakes, backups, static hosting, and application data.

**Storage Classes:**

| Class | Use Case | Durability | Retrieval | Min Duration |
|-------|----------|-----------|-----------|-------------|
| S3 Standard | Frequent access | 11 9s | ms | None |
| S3 Intelligent-Tiering | Unknown access | 11 9s | ms | 30 days (monitoring fee) |
| S3 Standard-IA | Infrequent, rapid | 11 9s | ms | 30 days |
| S3 One Zone-IA | Infrequent, single AZ | 11 9s | ms | 30 days |
| S3 Glacier Instant | Archive, ms retrieval | 11 9s | ms | 90 days |
| S3 Glacier Flexible | Archive, minutes-hours | 11 9s | min–hr | 90 days |
| S3 Glacier Deep Archive | Long-term, 12hr retrieval | 11 9s | 12–48hr | 180 days |
| S3 Express One Zone | Ultra-low latency | 11 9s | single-digit ms | 1 hour |

**Key Features:**
- Versioning — protect against accidental deletes and overwrites
- MFA Delete — require MFA to delete versioned objects
- Replication (CRR/SRR) — cross-region or same-region replication
- S3 Lifecycle Policies — automate transitions and expirations
- Object Lock (WORM) — Compliance or Governance mode, Retention Period + Legal Hold
- Multipart Upload — required for objects >5 GB, optional >100 MB
- Transfer Acceleration — uses CloudFront edge locations
- S3 Access Points — named network endpoints with distinct policies per application
- S3 Object Lambda — transform data on retrieval
- Event Notifications — to SQS, SNS, Lambda, EventBridge
- Requester Pays — requester bears data transfer and retrieval costs
- Presigned URLs — time-limited access without credentials
- S3 Batch Operations — invoke Lambda or standard ops on billions of objects
- Server-Side Encryption: SSE-S3 (AES-256), SSE-KMS, SSE-C, DSSE-KMS
- Bucket Policy + ACL + Access Analyzer
- S3 Inventory — CSV/ORC/Parquet listing of objects and metadata
- Storage Lens — organization-wide analytics and insights

**Common Use Cases:**
- Data lake storage (raw/processed data, queried by Athena)
- Application asset storage (images, videos, user uploads)
- Static website hosting
- Log archival and compliance
- Backup target for EC2, RDS, on-prem

**Key CLI Commands:**
```bash
# Create bucket
aws s3 mb s3://my-bucket --region us-east-1

# Copy local to S3
aws s3 cp ./data.csv s3://my-bucket/data/

# Sync directory
aws s3 sync ./local-dir s3://my-bucket/prefix/ --delete

# List objects
aws s3 ls s3://my-bucket/prefix/ --recursive --human-readable

# Presigned URL (valid 1 hour)
aws s3 presign s3://my-bucket/file.pdf --expires-in 3600

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Put bucket policy
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://policy.json

# Configure lifecycle rule
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# Enable server-side encryption (SSE-S3 default)
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Enable replication
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json

# S3 Batch Operations — create job
aws s3control create-job \
  --account-id 123456789012 \
  --operation '{"LambdaInvoke":{"FunctionArn":"arn:aws:lambda:..."}}' \
  --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},"Location":{"ObjectArn":"arn:aws:s3:::manifest-bucket/manifest.csv","ETag":"etag"}}' \
  --report '{"Bucket":"arn:aws:s3:::report-bucket","Prefix":"reports/","Format":"Report_CSV_20180820","Enabled":true,"ReportScope":"AllTasks"}' \
  --priority 10 \
  --role-arn arn:aws:iam::123456789012:role/batch-ops-role

# Block all public access
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**Pricing Model:**
- Storage: $0.023/GB-month (Standard)
- Requests: $0.0004/1000 PUT; $0.0004/10000 GET (Standard)
- Data transfer out: $0.09/GB (first 10TB)
- Free inbound transfer, free S3-to-CloudFront transfer
- Lifecycle transitions incur per-request fees

**Integration Points:** CloudFront (OAC), Lambda (event triggers), Athena (queries), Glue (Catalog), Redshift Spectrum, SageMaker, Macie, CloudTrail data events, KMS, IAM, EventBridge, DataSync, Snow Family

**Gotchas / Limits:**
- Max object size: 5 TB (5 GB per PUT; use multipart for larger)
- Bucket names must be globally unique
- Eventual consistency for overwrite PUTs is now strong consistency (since Dec 2020)
- S3 is regional — replicate explicitly for cross-region DR
- Default: 100 buckets per account (soft limit to 1000)
- Requester pays cannot be used with anonymous access
- SSE-KMS counts against KMS API quotas
- Object Lock requires versioning to be enabled
- Glacier minimum storage durations incur charges even if deleted early

---

## 2.2 EBS (Elastic Block Store)

**What it is:** Persistent block storage volumes for EC2. Think of it as an external hard drive attached to a single EC2 instance (except io2 Block Express multi-attach).

**Volume Types:**

| Type | Description | Max IOPS | Max Throughput | Use Case |
|------|-------------|----------|--------------|---------|
| gp3 | General Purpose SSD | 16,000 | 1,000 MB/s | Default OS, DBs, dev |
| gp2 | General Purpose SSD (legacy) | 16,000 | 250 MB/s | Legacy workloads |
| io2 Block Express | Provisioned IOPS SSD | 256,000 | 4,000 MB/s | Critical DBs, SAP |
| io1 | Provisioned IOPS SSD (legacy) | 64,000 | 1,000 MB/s | I/O intensive |
| st1 | Throughput Optimized HDD | 500 | 500 MB/s | Big data, logs |
| sc1 | Cold HDD | 250 | 250 MB/s | Archival, infrequent |

**Key Features:**
- Snapshots — incremental backups stored in S3 (you don't see the bucket)
- Fast Snapshot Restore (FSR) — eliminate snap restore latency
- Snapshot Lifecycle Manager (DLM) — automate snapshot schedules
- EBS Multi-Attach — io1/io2 can attach to up to 16 Nitro EC2 instances simultaneously
- Encryption — AES-256 via KMS; snapshot of encrypted volume is encrypted
- Elastic Volumes — modify type, IOPS, throughput, size without detaching
- Recycle Bin — protect snapshots from accidental deletion

**Common Use Cases:**
- OS boot volumes
- Database data volumes (MySQL, Postgres, Oracle)
- High-performance transactional workloads

```bash
# Create volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 500 \
  --iops 10000 \
  --throughput 500 \
  --availability-zone us-east-1a \
  --encrypted

# Attach volume
aws ec2 attach-volume \
  --volume-id vol-12345 \
  --instance-id i-12345 \
  --device /dev/xvdf

# Create snapshot
aws ec2 create-snapshot \
  --volume-id vol-12345 \
  --description "Weekly backup"

# Modify volume (Elastic Volumes)
aws ec2 modify-volume \
  --volume-id vol-12345 \
  --volume-type io2 \
  --iops 20000

# Enable DLM lifecycle policy
aws dlm create-lifecycle-policy \
  --description "Daily snapshots" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/dlm-role \
  --policy-details file://dlm-policy.json
```

**Pricing Model:**
- gp3: $0.08/GB-month; IOPS above 3000: $0.005/IOPS-month; Throughput above 125 MB/s: $0.04/MB/s-month
- io2: $0.125/GB-month + $0.065/provisioned IOPS (up to 32K)
- Snapshots: $0.05/GB-month (incremental)

**Integration Points:** EC2, Lambda (not directly), ECS, EKS (EBS CSI driver), DLM, KMS, CloudWatch, AWS Backup

**Gotchas / Limits:**
- EBS volumes are AZ-specific — cannot move across AZs directly (snapshot → restore)
- gp2 IOPS scales with size (3 IOPS/GB max 16K) — gp3 decouples IOPS from size
- Multi-Attach requires cluster-aware file system (GFS2, OCFS2)
- Snapshots lock the volume briefly during first snapshot creation
- Max volume size: 64 TB (io2 Block Express), 16 TB (gp3)

---

## 2.3 EFS (Elastic File System)

**What it is:** Fully managed NFS (v4.1/v4.0) file system for Linux. Scales automatically from kilobytes to petabytes. Can be mounted on thousands of EC2 instances and Lambda concurrently.

**Storage Classes:**
- EFS Standard — frequently accessed, multi-AZ
- EFS Standard-IA — infrequently accessed, multi-AZ (lower cost)
- EFS One Zone — single AZ, Standard or IA tiers
- Lifecycle Management — auto-move files to IA after 7/14/30/60/90 days

**Performance Modes:**
- General Purpose (default) — latency-sensitive apps
- Max I/O — highly parallel, slightly higher latency (legacy; use elastic throughput)

**Throughput Modes:**
- Elastic (recommended) — bursts up to 3 GB/s read / 1 GB/s write, pay per use
- Provisioned — set throughput regardless of storage size
- Bursting — throughput scales with storage size

**Key Features:**
- Multi-AZ (Standard) or single-AZ (One Zone)
- Mount targets — ENI per AZ for VPC access
- EFS Access Points — application-specific entry point with user/group enforcement
- Encryption at rest (KMS) and in transit (TLS)

```bash
# Create EFS
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=my-efs

# Create mount target
aws efs create-mount-target \
  --file-system-id fs-12345678 \
  --subnet-id subnet-12345678 \
  --security-groups sg-12345678

# Mount on EC2
sudo mount -t efs -o tls fs-12345678:/ /mnt/efs

# Install amazon-efs-utils first
sudo yum install amazon-efs-utils
```

**Pricing Model:**
- Standard: $0.30/GB-month; Standard-IA: $0.025/GB-month + $0.01/GB retrieval
- Elastic throughput: $0.03/GB read, $0.06/GB write
- One Zone: 47% cheaper than multi-AZ

**Integration Points:** EC2, ECS (via volume mounts), EKS (EFS CSI driver), Lambda (mounted via EFS Access Points), DataSync, AWS Backup

**Gotchas / Limits:**
- Linux only (NFS); not supported on Windows without NFS client
- EFS is regional — all AZs in a region share one file system
- Max file size: 52 TB
- Throughput limit per file system depends on mode
- Lambda EFS mount adds latency; keep functions in same AZ as mount target

---

## 2.4 FSx Family

### FSx for Lustre
High-performance parallel file system for HPC and ML workloads.

**Key Features:**
- Scratch (temporary) or Persistent deployment types
- S3 integration — lazy load from S3, write back to S3
- Sub-millisecond latency, up to 1 TB/s aggregate throughput
- POSIX-compliant
- SSD or HDD storage

**Use Cases:** ML training (SageMaker), HPC, media rendering, genomics

```bash
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --subnet-ids subnet-12345 \
  --lustre-configuration \
    DeploymentType=PERSISTENT_2,PerUnitStorageThroughput=250,DataCompressionType=LZ4
```

### FSx for Windows File Server
Fully managed Windows native file system with SMB, DFS, AD integration.

**Key Features:**
- SMB protocol (v2.0–v3.1.1)
- Active Directory integration (self-managed or AWS Managed AD)
- Shadow Copies (VSS), NTFS, DFS Namespaces
- Multi-AZ or Single-AZ deployment
- SSD or HDD storage

**Use Cases:** Windows workloads migrating from on-prem, .NET app file shares, home directories

### FSx for NetApp ONTAP
Managed NetApp ONTAP file system — supports NFS, SMB, iSCSI.

**Key Features:**
- Multi-protocol access (NFS, CIFS/SMB, iSCSI)
- NetApp SnapMirror replication
- FlexCache (caching volumes), FlexClone (instant clones)
- Storage efficiency (dedup, compression, thin provisioning)

**Use Cases:** Enterprise workloads needing ONTAP features, lift-and-shift from on-prem NetApp

### FSx for OpenZFS
Managed OpenZFS file system. High-performance NFS with ZFS features.

**Key Features:**
- NFS v3 and v4.x
- ZFS snapshots (up to 1 million), clones, compression
- Up to 10 GB/s throughput, sub-0.5 ms latency

**Use Cases:** Linux workloads, dev/test cloning, media production

---

## 2.5 S3 Glacier (Standalone)

**What it is:** The original Glacier service (now largely superseded by S3 Glacier storage classes). Separate vault-based API for archives.

**Retrieval Options:** Expedited (1–5 min), Standard (3–5 hr), Bulk (5–12 hr)
**Pricing:** Storage from $0.004/GB-month; retrieval fees per GB + per request

---

## 2.6 Storage Gateway

Connects on-premises environments to AWS cloud storage via standard protocols.

| Type | Protocol | Backed By | Use Case |
|------|----------|-----------|---------|
| S3 File Gateway | NFS, SMB | S3 | File share → S3 |
| FSx File Gateway | SMB | FSx for Windows | On-prem Windows cache |
| Volume Gateway (Stored) | iSCSI | S3 + EBS Snapshots | Local + cloud backup |
| Volume Gateway (Cached) | iSCSI | S3 (primary) | Low-latency cloud volumes |
| Tape Gateway | VTL (iSCSI) | S3 Glacier | Replace physical tapes |

**Pricing:** Per GB of data written/read + underlying storage (S3/EBS)

**Gotchas:**
- Requires VM (VMware, Hyper-V, KVM) or hardware appliance on-prem
- Local cache disk size determines performance
- Tape Gateway emulates physical tape libraries for backup software compatibility

---

## 2.7 AWS Backup

**What it is:** Centralized backup management across AWS services. Define backup plans, policies, and vaults in one place.

**Supported Resources:** EC2, EBS, RDS, Aurora, DynamoDB, EFS, FSx, S3, Storage Gateway, DocumentDB, Neptune, VMware on-prem

**Key Features:**
- Backup Plans — schedule + lifecycle rules
- Backup Vaults — encrypted storage containers
- AWS Backup Vault Lock — WORM protection (compliance)
- Cross-account and cross-region backup copy
- Organization-wide backup policies via AWS Organizations

```bash
# Create backup plan
aws backup create-backup-plan --backup-plan file://plan.json

# Create backup selection (resources to protect)
aws backup create-backup-selection \
  --backup-plan-id plan-id \
  --backup-selection file://selection.json

# Start on-demand backup
aws backup start-backup-job \
  --backup-vault-name Default \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/MyTable \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupRole
```

**Pricing:** Per GB of backup storage + per GB restored

---

## 2.8 Snow Family

| Service | Capacity | Use Case |
|---------|---------|---------|
| Snowcone | 8 TB HDD / 14 TB SSD | Edge computing, small transfers |
| Snowball Edge Storage Optimized | 80 TB | Large migrations, edge |
| Snowball Edge Compute Optimized | 28 TB NVMe + EC2 | GPU/compute at edge |
| Snowmobile | 100 PB (truck) | Exabyte-scale migration |

**Key Features:**
- DataSync agent pre-installed on Snowcone
- Snowball Edge runs EC2 instances and Lambda locally
- AWS OpsHub GUI for local management
- End-to-end encryption (AES-256)

**Pricing:** Per-device rental fee + per-day usage + data transfer (inbound to S3 free)

**Gotchas:** Device ordering takes 1–2 weeks; shipping time adds to migration timeline

---

## 2.9 Transfer Family

**What it is:** Managed SFTP, FTPS, FTP, and AS2 server backed by S3 or EFS. No server infrastructure to manage.

**Protocols:** SFTP, FTPS, FTP (within VPC only), AS2 (EDI)

**Key Features:**
- Custom identity providers (Cognito, Lambda, Active Directory, LDAP)
- Endpoint types: public, VPC, VPC_ENDPOINT
- Logical directories — map S3 paths to virtual directories
- Transfer Family Workflows — post-upload processing via Lambda

**Pricing:** Per protocol endpoint-hour + per GB of data uploaded/downloaded

---

---

# 3. Databases

> AWS offers purpose-built databases for relational, key-value, document, in-memory, graph, time-series, ledger, and wide-column workloads. Choose the engine that matches your data model, consistency requirements, and access patterns.

---

## 3.1 RDS (Relational Database Service)

**What it is:** Managed relational database service. AWS handles provisioning, patching, backups, Multi-AZ failover, and read replicas.

**Supported Engines:**

| Engine | Versions | Notable Feature |
|--------|---------|----------------|
| MySQL | 5.7, 8.0 | Widest community compatibility |
| PostgreSQL | 12–16 | Extensions, JSONB, PostGIS |
| MariaDB | 10.x | MySQL-compatible, open source |
| Oracle | EE, SE2 | BYOL or License Included |
| SQL Server | Express/Web/Standard/Enterprise | Windows auth, SSRS |
| Db2 | 11.5 | IBM enterprise workloads |

**Key Features:**
- Multi-AZ deployment — synchronous standby in another AZ (auto-failover ~1–2 min)
- Read Replicas — async replication; up to 15 replicas per source
- Automated backups — point-in-time restore (up to 35 days)
- Manual snapshots — retained indefinitely
- Storage Auto Scaling — automatically increase storage when 10% free
- Enhanced Monitoring — OS metrics at 1–60 second granularity
- Performance Insights — database load visualization (14 days free, 2 yr paid)
- RDS Proxy — serverless connection pooler, reduces connection overhead
- IAM database authentication (MySQL, PostgreSQL)
- Encryption at rest (KMS) and in transit (SSL/TLS)
- Parameter Groups and Option Groups for engine configuration
- Maintenance windows for patching

**Common Use Cases:**
- OLTP workloads: e-commerce, SaaS applications
- WordPress, Drupal, and other CMS platforms
- ERP and CRM systems

```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-mysql \
  --db-instance-class db.m7g.large \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password 'MySecurePass123!' \
  --allocated-storage 200 \
  --storage-type gp3 \
  --iops 3000 \
  --multi-az \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name my-subnet-group \
  --backup-retention-period 14 \
  --deletion-protection \
  --storage-encrypted

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-replica \
  --source-db-instance-identifier prod-mysql \
  --db-instance-class db.m7g.large

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-mysql \
  --db-snapshot-identifier prod-mysql-snap-20240115

# Restore from point-in-time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-mysql \
  --target-db-instance-identifier prod-mysql-restored \
  --restore-time 2024-01-15T10:00:00Z

# Modify instance (with apply-immediately or next maintenance window)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql \
  --db-instance-class db.m7g.2xlarge \
  --apply-immediately

# Describe events
aws rds describe-events \
  --source-identifier prod-mysql \
  --source-type db-instance \
  --duration 1440
```

**Pricing Model:**
- Instance: per hour (varies by class and engine)
- Storage: $0.115/GB-month (gp2), $0.10/GB-month (gp3) + IOPS above baseline
- Multi-AZ: ~2x instance cost
- Backup: free up to DB storage size; excess at $0.095/GB-month
- Data transfer: standard rates

**Integration Points:** EC2, Lambda, ECS, EKS (via RDS Proxy), Secrets Manager, IAM, CloudWatch, SNS (events), DMS, Database Migration, AWS Backup

**Gotchas / Limits:**
- Multi-AZ standby cannot be used for reads — that's what read replicas are for
- Failover promotes standby and renames DNS — clients must use DNS endpoint (not IP)
- Storage can only increase, NOT decrease on running instance
- Read replicas can be promoted to standalone (breaks replication)
- Oracle/SQL Server Multi-AZ use different mechanism (Mirroring/Always On) — check engine limits
- Default max connections depends on instance size (RAM-based formula)

---

## 3.2 Amazon Aurora

**What it is:** AWS-built, cloud-optimized relational database. MySQL and PostgreSQL compatible with 5x MySQL performance and 3x PostgreSQL performance claims. Decoupled compute and storage.

**Key Features:**
- Distributed storage — 6 copies across 3 AZs, 4/6 for writes, 3/6 for reads
- Storage grows in 10 GB increments automatically
- Aurora Replicas — up to 15 read replicas with sub-10ms replica lag
- Aurora Serverless v2 — scales compute in fine-grained increments (0.5–128 ACUs)
- Aurora Global Database — primary region + up to 5 secondary read regions; RPO <1s, RTO <1 min
- Aurora Multi-Master — multiple write nodes (MySQL only, consider Serverless v2 instead)
- Backtrack — rewind DB without restore (MySQL only, up to 72 hours)
- Fast Clone — copy DB in seconds (copy-on-write)
- Aurora ML — call SageMaker/Comprehend from SQL
- Zero-ETL integration with Redshift

**Aurora Endpoints:**
- Cluster endpoint — always points to primary writer
- Reader endpoint — load balances across all replicas
- Custom endpoint — target specific replica subset
- Instance endpoint — specific instance

**Aurora Serverless v2:**
- Minimum capacity: 0.5 ACUs (~0.9 GB RAM)
- Scales instantly, unlike v1 which pauses
- Supports Multi-AZ, Global Database, read replicas

```bash
# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username admin \
  --master-user-password 'MySecurePass!' \
  --vpc-security-group-ids sg-12345 \
  --db-subnet-group-name aurora-subnet-group \
  --storage-encrypted \
  --enable-cloudwatch-logs-exports '["postgresql"]'

# Add writer instance
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-instance-1 \
  --db-cluster-identifier prod-aurora \
  --db-instance-class db.r7g.large \
  --engine aurora-postgresql

# Add read replica
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-reader-1 \
  --db-cluster-identifier prod-aurora \
  --db-instance-class db.r7g.large \
  --engine aurora-postgresql

# Failover (promote replica to primary)
aws rds failover-db-cluster \
  --db-cluster-identifier prod-aurora \
  --target-db-instance-identifier prod-aurora-reader-1

# Create Aurora Serverless v2 cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-serverless \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=64 \
  --master-username admin \
  --master-user-password 'MyPass!' \
  --db-subnet-group-name aurora-subnet-group
```

**Pricing Model:**
- ACUs (Aurora Capacity Units): $0.12/ACU-hour (Serverless v2)
- Instance pricing: similar to RDS but Aurora-specific classes
- Storage: $0.10/GB-month + $0.20/1M I/O requests
- Backtrack: $0.012/million change records
- Global Database: standard + data transfer between regions

**Gotchas / Limits:**
- Aurora storage minimum is 10 GB, automatically grows
- Can't reduce allocated storage
- Aurora Serverless v2 minimum 0.5 ACU (never scales to 0 like v1)
- Backtrack not available on PostgreSQL-compatible Aurora
- Global Database secondary regions are read-only
- Multi-Master Aurora (MySQL) not recommended for new workloads — use Serverless v2

---

## 3.3 DynamoDB

**What it is:** Serverless, fully managed NoSQL key-value and document database. Single-digit millisecond performance at any scale. No servers, no connections, no schema.

**Key Concepts:**
- Table — collection of items
- Item — row (max 400 KB)
- Attribute — column
- Primary Key: Partition Key only (hash key) OR Partition Key + Sort Key (composite)
- LSI (Local Secondary Index) — alternate sort key, same partition key, created at table creation
- GSI (Global Secondary Index) — alternate partition+sort key, created anytime, eventually consistent

**Capacity Modes:**

| Mode | Description | Use Case |
|------|-------------|---------|
| Provisioned | Set RCU/WCU; can use Auto Scaling | Predictable traffic |
| On-Demand | Pay per request | Unknown/spiky traffic |

- 1 RCU = 1 strongly consistent read/s (4 KB) = 2 eventually consistent reads/s
- 1 WCU = 1 write/s (1 KB)

**Key Features:**
- DynamoDB Streams — ordered log of item changes; triggers Lambda
- DynamoDB Accelerator (DAX) — in-memory cache, microsecond reads, API-compatible
- Global Tables — multi-region active-active replication
- Point-in-Time Recovery (PITR) — restore to any second in last 35 days
- TTL (Time to Live) — auto-expire items
- Transactions — ACID across multiple items/tables (TransactGetItems/TransactWriteItems)
- Conditional writes — optimistic locking
- PartiQL — SQL-compatible query language
- Export to S3 — no RCU consumption, Parquet/DynamoDB JSON
- Import from S3 — bulk load from CSV/Parquet/DynamoDB JSON
- Kinesis Data Streams for DynamoDB — stream to Kinesis

**Common Use Cases:**
- Session stores, shopping carts (key-value)
- Gaming leaderboards (high write throughput)
- IoT telemetry
- Serverless app backends (with Lambda)
- Ad tech (sub-ms lookups)

```bash
# Create table
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=CustomerId,AttributeType=S \
    AttributeName=OrderId,AttributeType=S \
  --key-schema \
    AttributeName=CustomerId,KeyType=HASH \
    AttributeName=OrderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --table-class STANDARD

# Put item
aws dynamodb put-item \
  --table-name Orders \
  --item '{"CustomerId":{"S":"cust-123"},"OrderId":{"S":"ord-456"},"Amount":{"N":"99.99"}}'

# Get item
aws dynamodb get-item \
  --table-name Orders \
  --key '{"CustomerId":{"S":"cust-123"},"OrderId":{"S":"ord-456"}}' \
  --consistent-read

# Query (same partition key)
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "CustomerId = :cid AND begins_with(OrderId, :prefix)" \
  --expression-attribute-values '{":cid":{"S":"cust-123"},":prefix":{"S":"ord-2024"}}'

# Scan (use sparingly)
aws dynamodb scan \
  --table-name Orders \
  --filter-expression "Amount > :amt" \
  --expression-attribute-values '{":amt":{"N":"50"}}'

# Update item
aws dynamodb update-item \
  --table-name Orders \
  --key '{"CustomerId":{"S":"cust-123"},"OrderId":{"S":"ord-456"}}' \
  --update-expression "SET #s = :status" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":status":{"S":"SHIPPED"}}'

# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name Orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Enable streams
aws dynamodb update-table \
  --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Create GSI
aws dynamodb update-table \
  --table-name Orders \
  --attribute-definitions AttributeName=Status,AttributeType=S \
  --global-secondary-index-updates \
    '[{"Create":{"IndexName":"StatusIndex","KeySchema":[{"AttributeName":"Status","KeyType":"HASH"}],"Projection":{"ProjectionType":"ALL"}}}]'
```

**Pricing Model (On-Demand):**
- Read: $0.25/million RRU (Request Units)
- Write: $1.25/million WRU
- Storage: $0.25/GB-month (Standard), $0.10/GB-month (Standard-IA)
- DAX: per node-hour
- Global Tables: +$1.875/million WRU per replicated write

**Integration Points:** Lambda (Streams), API Gateway, AppSync, Kinesis, S3 (Export/Import), DAX, CloudWatch, IAM, Glue (ETL), EMR

**Gotchas / Limits:**
- Item max size: 400 KB
- Partition key must be well-distributed — hot partitions cause throttling
- LSI cannot be added after table creation
- On-Demand has 2x throughput ceiling compared to provisioned
- Strongly consistent reads cannot be used with GSIs
- DynamoDB is not suitable for complex JOINs or ad-hoc queries
- Default limit: 256 tables per region

---

## 3.4 ElastiCache for Redis

**What it is:** Managed Redis in-memory data store. Sub-millisecond latency for caching, session management, pub/sub, leaderboards, and more.

**Deployment Modes:**
- Cluster Mode Disabled — single shard, up to 5 replicas
- Cluster Mode Enabled — up to 500 shards, horizontal scaling

**Key Features:**
- Redis 6.x, 7.x support
- Multi-AZ with auto-failover
- Read replicas per shard
- Persistence (RDB snapshots, AOF)
- Encryption in-transit (TLS) and at-rest (KMS)
- AUTH token + RBAC (Redis 6+)
- Serverless ElastiCache — auto-scales, pay per data stored and ECPUs
- ElastiCache Global Datastore — cross-region replication

```bash
# Create Redis cluster
aws elasticache create-replication-group \
  --replication-group-id prod-redis \
  --description "Production Redis" \
  --num-cache-clusters 3 \
  --cache-node-type cache.r7g.large \
  --engine redis \
  --engine-version 7.0 \
  --multi-az-enabled \
  --automatic-failover-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "MySecureToken123!" \
  --cache-subnet-group-name redis-subnet-group

# Describe replication groups
aws elasticache describe-replication-groups \
  --replication-group-id prod-redis

# Failover
aws elasticache test-failover \
  --replication-group-id prod-redis \
  --node-group-id 0001
```

**Pricing Model:** Per node-hour based on node type; data transfer; backup storage.

**Gotchas:**
- ElastiCache is NOT accessible from the internet (VPC only)
- Cluster Mode Enabled requires Redis client with cluster support
- AOF persistence increases cost and recovery time
- Serverless ElastiCache: cold start can add latency after extended idle

---

## 3.5 ElastiCache for Memcached

**What it is:** Managed Memcached — pure caching, multithreaded, no persistence, no replication.

**Key Features:**
- Auto Discovery — clients find nodes automatically
- Up to 20 nodes per cluster
- Best for simple horizontal caching

**When to use Memcached vs Redis:**

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Persistence | Yes | No |
| Replication | Yes | No |
| Data types | Rich (lists, sets, sorted sets) | Simple strings |
| Pub/Sub | Yes | No |
| Clustering | Yes | Horizontal sharding |
| Multi-threading | No (single) | Yes |

---

## 3.6 Amazon Neptune

**What it is:** Managed graph database. Supports Property Graph (Gremlin, openCypher) and RDF/SPARQL.

**Key Features:**
- Multi-AZ, up to 15 read replicas
- Neptune Analytics — graph analytics engine (in-memory)
- Neptune Streams — change log for graph changes
- Bulk load from S3 (CSV format)
- Neptune ML — GNN-based ML on graph data via SageMaker

**Use Cases:** Knowledge graphs, fraud detection, recommendation engines, social networks, identity graphs

**Pricing:** Instance hours + storage ($0.10/GB-month) + I/O + backup

---

## 3.7 Amazon DocumentDB

**What it is:** MongoDB-compatible managed document database. NOT running MongoDB under the hood — custom implementation of MongoDB wire protocol.

**Key Features:**
- Compatible with MongoDB 3.6, 4.0, 5.0 drivers and tools
- Storage grows in 10 GB increments automatically
- Up to 15 read replicas
- PITR, automated backups
- Global Clusters (cross-region)
- Elastic Clusters — horizontally scalable (millions of reads/writes per second)

**Use Cases:** Content management, catalogs, user profiles, JSON document storage

**Gotchas:** Not 100% MongoDB API compatible — test your workload; some aggregation operators missing

---

## 3.8 Amazon Keyspaces (for Apache Cassandra)

**What it is:** Serverless, Cassandra-compatible managed wide-column database.

**Key Features:**
- Compatible with CQL (Cassandra Query Language) 3.x
- Serverless — scales automatically, pay per request
- PITR (35 days)
- Encryption at rest and in transit
- Multi-Region replication (via CQL)

**Use Cases:** Time-series IoT data, high-velocity writes, distributed workloads ported from Cassandra

---

## 3.9 Amazon Timestream

**What it is:** Serverless time-series database. Automatically scales ingestion, storage, and querying of time-series data.

**Key Features:**
- SQL-like query language with time-series functions (interpolation, smoothing, approximation)
- Memory + magnetic store tiers
- Automated data lifecycle (move from memory to magnetic tier)
- Timestream Live Analytics (streaming data via Kinesis)
- Grafana, Tableau, QuickSight integrations

**Use Cases:** DevOps metrics, IoT sensor data, application monitoring, financial tick data

---

## 3.10 Amazon QLDB (Quantum Ledger Database)

**What it is:** Immutable, cryptographically verifiable ledger database. Complete history of all changes, cannot be altered.

**Key Features:**
- PartiQL query language
- Journal — append-only, immutable transaction log
- Cryptographic verification via SHA-256 hash digest
- Serverless

**Use Cases:** Financial transactions audit trail, supply chain provenance, healthcare records

**Note:** For decentralized ledger needs, see Managed Blockchain. QLDB is centralized and AWS-managed.

---

## 3.11 MemoryDB for Redis

**What it is:** Redis-compatible in-memory database with durability. Unlike ElastiCache, MemoryDB is a primary database (not just a cache) — data persists via Multi-AZ transaction log.

**Key Features:**
- Redis API compatible (all data types, commands)
- Microsecond reads, single-digit ms writes
- Multi-AZ, durable (transactional log across AZs)
- Up to 500 shards

**When to use MemoryDB vs ElastiCache Redis:**
- MemoryDB: primary database requiring Redis API + durability
- ElastiCache Redis: caching layer in front of another database

---

## 3.12 RDS Proxy

**What it is:** Fully managed database proxy for RDS and Aurora. Pools and shares connections to improve application scalability and resilience.

**Key Features:**
- Supports MySQL, PostgreSQL, SQL Server, MariaDB, Aurora
- Reduces "too many connections" errors from Lambda/ECS at scale
- Automatic failover to standby in <30 seconds (faster than native)
- IAM authentication and Secrets Manager integration
- Pinning — maintain connection to specific DB instance for session state

**Pricing:** $0.015 per vCPU-hour of the database instance being proxied

**Gotchas:**
- RDS Proxy only accessible from VPC (no public endpoint)
- Multiplexing not available for all workloads (pinning may reduce benefit)


---

# 4. Networking & Content Delivery

> AWS networking services provide the foundation for connectivity, security, traffic management, and content delivery at global scale.

---

## 4.1 VPC (Virtual Private Cloud)

**What it is:** Logically isolated network in AWS. You control IP ranges, subnets, route tables, gateways, and security policies.

**Core Components:**

| Component | Description |
|-----------|-------------|
| CIDR Block | IP range for the VPC (e.g., 10.0.0.0/16) |
| Subnet | Sub-range of VPC CIDR, tied to one AZ |
| Route Table | Rules for traffic routing; 1 per subnet |
| Internet Gateway (IGW) | Enables public internet access for subnet |
| NAT Gateway | Outbound-only internet for private subnets |
| VPC Endpoint | Private connectivity to AWS services |
| Security Group | Stateful firewall at ENI level |
| NACL | Stateless firewall at subnet level |
| VPC Flow Logs | Capture IP traffic metadata |
| DHCP Options Set | DNS/NTP settings for VPC |

**Security Group vs NACL:**

| Feature | Security Group | NACL |
|---------|---------------|------|
| Stateful | Yes | No |
| Level | Instance (ENI) | Subnet |
| Allow/Deny | Allow only | Allow and Deny |
| Default | Deny all in, allow all out | Allow all |
| Rule evaluation | All rules evaluated | Rules in number order |

**VPC Endpoints:**
- Gateway Endpoint — S3 and DynamoDB; routes via route table, free
- Interface Endpoint (PrivateLink) — other services; ENI with private IP, hourly + data charge
- Gateway Load Balancer Endpoint — route traffic through virtual appliances

**Key CLI Commands:**
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'

# Create subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'

# Create Internet Gateway and attach
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345 \
  --vpc-id vpc-12345678

# Create NAT Gateway
aws ec2 allocate-address --domain vpc  # Get EIP
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id eipalloc-12345

# Create route table
aws ec2 create-route-table --vpc-id vpc-12345678

# Add route to NAT GW
aws ec2 create-route \
  --route-table-id rtb-12345 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-12345

# Create security group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web tier" \
  --vpc-id vpc-12345678

# Add inbound rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogs-role

# Create Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-111 subnet-222 \
  --security-group-ids sg-12345 \
  --private-dns-enabled

# VPC Peering
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaa \
  --peer-vpc-id vpc-bbb \
  --peer-region us-west-2
```

**Pricing Model:**
- VPC, subnets, route tables, IGW: free
- NAT Gateway: $0.045/hr + $0.045/GB processed
- Interface Endpoints (PrivateLink): $0.01/AZ/hr + $0.01/GB
- VPC Flow Logs: CloudWatch Logs or S3 storage costs

**Gotchas / Limits:**
- Default VPC: 5 per region (soft limit)
- Subnets: 200 per VPC; VPCs: 5 per region (default)
- CIDR cannot overlap for VPC peering
- Peering is non-transitive — use Transit Gateway for hub-and-spoke
- Security groups are regional; reference by ID across VPCs with peering
- NAT Gateway in each AZ for HA (not cross-AZ by default)
- NACLs are stateless — must allow response traffic explicitly

---

## 4.2 Route 53

**What it is:** Scalable DNS web service and domain registrar. Supports public/private hosted zones, health checks, and traffic routing policies.

**Routing Policies:**

| Policy | Description | Use Case |
|--------|-------------|---------|
| Simple | Single resource | Basic DNS |
| Weighted | Route % of traffic to each endpoint | A/B testing, gradual migration |
| Latency | Route to lowest latency region | Multi-region apps |
| Failover | Primary/secondary based on health check | Active-passive DR |
| Geolocation | Route based on user's geographic location | Compliance, localization |
| Geoproximity | Route based on location + bias | Traffic shifting |
| Multivalue Answer | Return multiple IPs (basic load balancing) | Simple HA without ELB |
| IP-Based | Route based on client IP CIDR | ISP-based routing |

**Key Features:**
- Health Checks — monitor endpoint availability; trigger DNS failover
- Private Hosted Zones — resolve within VPC(s) only
- Alias records — AWS-specific; no TTL charge; CNAME at apex domain
- DNSSEC signing
- Route 53 Resolver (formerly DNS+) — hybrid DNS resolution
- Resolver Endpoints: Inbound (on-prem → AWS) and Outbound (AWS → on-prem)
- Route 53 Resolver DNS Firewall — block DNS queries to malicious domains
- Traffic Flow — visual routing policy builder

```bash
# Create hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference unique-string-1

# Create DNS record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-1234.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Create health check
aws route53 create-health-check \
  --caller-reference hc-1 \
  --health-check-config \
    "Type=HTTPS,FullyQualifiedDomainName=api.example.com,Port=443,RequestInterval=30,FailureThreshold=3"

# List hosted zones
aws route53 list-hosted-zones

# List records
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890
```

**Pricing Model:**
- Hosted zone: $0.50/month (first 25 zones), $0.10/month after
- DNS queries: $0.40/million (first 1B), $0.20/million after
- Health checks: $0.50/month standard; $1.00/month with string matching
- Traffic Flow: $50/month per policy record

**Gotchas / Limits:**
- Alias records are free for queries to AWS resources
- TTL minimum is 0 (instant failover), but DNS propagation delay still exists
- Health checks for private resources need CloudWatch alarm-based health checks
- Default: 500 hosted zones per account; 10,000 records per zone

---

## 4.3 CloudFront

**What it is:** Global CDN with 450+ edge locations. Caches content at the edge for low-latency delivery. Also provides security (WAF, Shield) and serverless edge compute.

**Key Features:**
- Origins: S3, ALB, EC2, API Gateway, MediaPackage, custom HTTP
- Origin Access Control (OAC) — secures S3 origins (replaces OAI)
- Cache Behaviors — route patterns to different origins with separate cache/TTL settings
- Cache Policies and Origin Request Policies — fine-grained cache key control
- Signed URLs and Signed Cookies — restrict access to paid/authenticated content
- Field-Level Encryption — encrypt specific form fields at edge
- CloudFront Functions — lightweight JS at edge (viewer request/response), sub-ms, 1M free/month
- Lambda@Edge — Node.js/Python at regional edge (all 4 event types), heavier compute
- Real-Time Logs — stream access logs to Kinesis
- CloudFront Access Logs — standard logs to S3
- Geo Restriction — allowlist or blocklist countries
- WebSocket support
- gRPC support
- HTTP/2 and HTTP/3 (QUIC)
- Continuous Deployment — safe deploy with staging distribution

**Lambda@Edge vs CloudFront Functions:**

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| Events | Viewer Req/Resp | All 4 (incl. Origin) |
| Runtime | JS only | Node.js, Python |
| Timeout | 1ms | 5s (viewer) / 30s (origin) |
| Memory | 2 MB | 128MB–10GB |
| Deploy region | All edge PoPs | us-east-1 (replicated) |
| Price | Very cheap | Lambda + data transfer |

```bash
# Create distribution
aws cloudfront create-distribution --distribution-config file://dist-config.json

# Create invalidation
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*" "/images/*" "/index.html"

# Get distribution
aws cloudfront get-distribution --id EDFDVBD6EXAMPLE

# List distributions
aws cloudfront list-distributions \
  --query 'DistributionList.Items[*].[Id,DomainName,Origins.Items[0].DomainName]' \
  --output table

# Update origin (requires ETag from get-distribution)
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --distribution-config file://updated-config.json \
  --if-match E2QWRUHEXAMPLE
```

**Pricing Model:**
- Data transfer out: $0.0085–$0.25/GB (varies by region)
- HTTP requests: $0.0075–$0.016/10K requests
- Lambda@Edge: standard Lambda pricing × 3x for edge
- CloudFront Functions: $0.10/million invocations
- Invalidations: first 1,000 paths/month free; $0.005/path after

**Gotchas / Limits:**
- Default TTL: 24 hours; use Cache-Control/Expires headers to override
- CloudFront propagation after change: 5–15 minutes
- Invalidations not instant — use versioned file names instead for production
- Lambda@Edge functions must be deployed in us-east-1
- Max response size for Lambda@Edge: 1.3 MB (viewer), 40 MB (origin)

---

## 4.4 API Gateway

**What it is:** Fully managed API service. Create, publish, and manage REST, HTTP, and WebSocket APIs at any scale.

**API Types:**

| Type | Protocol | Use Case | Latency |
|------|----------|---------|---------|
| REST API | HTTP/1.1 | Full-featured, edge-optimized | Medium |
| HTTP API | HTTP/1.1 | Low-latency, cheaper (~70% cheaper) | Low |
| WebSocket API | WS | Real-time, bidirectional | Low |

**Key Features (REST API):**
- Stages — deployment environments (dev/staging/prod)
- Stage Variables — environment-specific config (like Lambda alias)
- Usage Plans + API Keys — rate limiting and quota management
- Request/Response Transformations (Mapping Templates)
- Caching — cache responses at stage level (TTL 300s default)
- Custom domain names + ACM certificates
- Resource policies — IP allow/deny
- Canary deployments — % traffic to new stage
- Edge-optimized (via CloudFront) vs Regional vs Private (VPC only)
- Integration types: Lambda, HTTP, AWS service, Mock, VPC Link

**HTTP API:**
- Lower latency and cost
- JWT authorizers, Lambda authorizers, IAM
- Automatic deployments
- No caching, no request/response transforms

**WebSocket API:**
- Connection ID per connected client
- Routes: `$connect`, `$disconnect`, `$default`, custom
- `@connections` API to push to specific clients

```bash
# Create HTTP API
aws apigatewayv2 create-api \
  --name my-http-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:my-function

# Create REST API
aws apigateway create-rest-api --name my-rest-api

# Create deployment
aws apigateway create-deployment \
  --rest-api-id abc123 \
  --stage-name prod

# Create usage plan
aws apigateway create-usage-plan \
  --name standard \
  --throttle burstLimit=200,rateLimit=100 \
  --quota limit=10000,period=MONTH

# Create API key
aws apigateway create-api-key \
  --name my-api-key \
  --enabled

# Get integrations (HTTP API)
aws apigatewayv2 get-integrations --api-id abc123
```

**Pricing Model:**
- REST API: $3.50/million API calls; cache $0.02–$3.84/hr depending on size
- HTTP API: $1.00/million (first 300M); $0.90/million after
- WebSocket: $1.00/million messages + $0.25/million connection-minutes

**Gotchas / Limits:**
- Default throttle: 10,000 RPS per region, 5,000 burst
- HTTP API does NOT support API key-based auth natively — use Lambda authorizer
- REST API caching not available in HTTP API
- 29-second integration timeout (Lambda, HTTP) — design for it
- Private APIs require VPC Endpoint for access

---

## 4.5 Load Balancers (ALB / NLB / CLB / GWLB)

### Application Load Balancer (ALB)
Layer 7, HTTP/HTTPS/gRPC. Routes based on path, host, headers, query strings, source IP.

**Key Features:**
- Target types: EC2, IP, Lambda, ECS
- Path-based, host-based, header-based, query-string routing
- Sticky sessions (AWSALB cookie)
- WebSocket support
- Redirect rules (HTTP → HTTPS, URL redirects)
- Fixed response (return custom HTTP code/body)
- WAF integration
- Access Logs to S3
- gRPC traffic routing

### Network Load Balancer (NLB)
Layer 4, TCP/UDP/TLS. Ultra-low latency, preserves source IP, handles millions of RPS.

**Key Features:**
- Static IP per AZ (Elastic IP assignable)
- TLS termination
- Target types: EC2, IP, ALB
- Health checks: TCP, HTTP, HTTPS
- Cross-zone load balancing (disabled by default; costs data transfer)
- PrivateLink integration (expose service to other VPCs)
- UDP support

### Classic Load Balancer (CLB)
Legacy, Layer 4+7. Use ALB or NLB for all new workloads.

### Gateway Load Balancer (GWLB)
Layer 3. Deploy, scale, and manage third-party network virtual appliances (firewalls, IDS/IPS, deep packet inspection).

**Key Features:**
- GENEVE protocol on port 6081
- Works with AWS Marketplace appliances (Palo Alto, Fortinet, etc.)
- Transparent to the network

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name prod-alb \
  --type application \
  --subnets subnet-111 subnet-222 \
  --security-groups sg-12345

# Create target group
aws elbv2 create-target-group \
  --name web-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345 \
  --health-check-path /health \
  --target-type instance

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/... \
  --targets Id=i-1234 Id=i-5678

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/...

# Create listener rule (path-based)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --priority 10 \
  --conditions '[{"Field":"path-pattern","Values":["/api/*"]}]' \
  --actions '[{"Type":"forward","TargetGroupArn":"arn:aws:..."}]'
```

**Pricing Model:**
- ALB: $0.008/hr + $0.008/LCU-hour (LCU = connections + bandwidth + rule evaluations)
- NLB: $0.008/hr + $0.006/NLCU-hour
- GWLB: $0.004/hr + $0.0035/GB

---

## 4.6 AWS Direct Connect

**What it is:** Dedicated private network connection from on-premises to AWS. Consistent throughput, reduced latency, not over public internet.

**Connection Types:**
- Dedicated: 1 Gbps, 10 Gbps, 100 Gbps (physical cross-connect at DX location)
- Hosted: Subrate speeds (50 Mbps – 10 Gbps) via DX Partner

**Key Features:**
- Virtual Interfaces (VIFs): Private VIF (VPC), Public VIF (AWS public services), Transit VIF (TGW)
- Direct Connect Gateway — connect to multiple VPCs/regions from single DX
- LAG (Link Aggregation Group) — bundle multiple connections
- BGP routing, supports bidirectional forwarding detection (BFD)
- SiteLink — private WAN between on-prem sites via DX backbone

**Pricing:** Port-hour + data transfer out (lower than internet rates)

**Gotchas:**
- DX setup time: 30–90 days for dedicated connections
- DX is NOT redundant by default — add second connection + VPN backup
- SiteLink data transfer billed separately

---

## 4.7 AWS Transit Gateway (TGW)

**What it is:** Hub-and-spoke network transit service. Connect thousands of VPCs, VPNs, and Direct Connect across accounts and regions through a central gateway.

**Key Features:**
- Up to 5,000 VPC attachments per TGW
- Multicast support
- Route tables per TGW — segment network traffic
- Inter-region peering
- TGW Network Manager — global network monitoring
- TGW Connect — SD-WAN integration via GRE

```bash
# Create TGW
aws ec2 create-transit-gateway \
  --description "Central TGW" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable

# Attach VPC
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-12345 \
  --vpc-id vpc-12345 \
  --subnet-ids subnet-111 subnet-222

# Attach VPN
aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-12345 \
  --type ipsec.1 \
  --transit-gateway-id tgw-12345
```

**Pricing:** $0.05/attachment/hr + $0.02/GB processed

---

## 4.8 AWS Global Accelerator

**What it is:** Network service that routes traffic to optimal regional endpoints using the AWS global backbone. Static anycast IP addresses for your applications.

**Key Features:**
- 2 static Anycast IPs (global entry points)
- Routes to ALB, NLB, EC2, or Elastic IPs
- Traffic Dials — control % traffic per endpoint group
- Health checks with automatic failover
- ~60% improvement in network performance vs public internet

**vs CloudFront:** Global Accelerator is for non-HTTP protocols or where you need TCP-level acceleration + static IPs. CloudFront is HTTP/S CDN with caching.

**Pricing:** $0.025/hr per accelerator + $0.015/GB data transfer (premium over standard)

---

## 4.9 AWS PrivateLink

**What it is:** Expose services to other VPCs and AWS accounts via private IPs without VPC peering or internet routing. Built on NLB.

**Key Features:**
- Service Provider creates NLB-backed VPC Endpoint Service
- Consumers create Interface VPC Endpoint → gets private IP in their VPC
- Cross-account, cross-region supported
- AWS services (SSM, Secrets Manager, ECR, etc.) all expose PrivateLink endpoints

---

## 4.10 AWS Network Firewall

**What it is:** Managed stateful firewall for VPCs. Supports Suricata rules, domain filtering, and TLS inspection.

**Key Features:**
- Stateful and stateless rule groups
- Suricata-compatible IPS rules
- Domain-based filtering (SNI/DNS)
- TLS inspection (decryption for inspection)
- Centralized or distributed deployment models

**Pricing:** $0.395/hr per firewall endpoint + $0.065/GB processed

---

## 4.11 AWS Shield

| Tier | Description | Cost |
|------|-------------|------|
| Standard | Always-on basic DDoS protection | Free |
| Advanced | 24/7 DDoS Response Team (DRT), cost protection, Advanced detection | $3,000/month |

**Shield Advanced:**
- Protects EC2, ELB, CloudFront, Global Accelerator, Route 53
- Proactive engagement from DRT during attacks
- DDoS cost credits (usage spike reimbursement)
- WAF integration

---

## 4.12 AWS WAF (Web Application Firewall)

**What it is:** Application-layer firewall. Protect against OWASP Top 10, bots, scrapers, and custom threats.

**Key Features:**
- Web ACLs — collection of rules applied to CloudFront, ALB, API Gateway, AppSync, Cognito
- Rule types: AWS Managed Rules, IP sets, Regex patterns, Rate-based rules, Geo-match
- Bot Control — managed rule group for bot mitigation
- Fraud Control — account takeover/creation protection
- CAPTCHA and Challenge actions
- Logging to S3, CloudWatch, Kinesis

**Pricing:** $5/Web ACL/month + $1/rule/month + $0.60/million requests

---

## 4.13 AWS App Mesh

**What it is:** Service mesh using Envoy proxy. Provides traffic management, observability, and security for microservices.

**Key Features:**
- Virtual services, virtual nodes, virtual routers, virtual gateways
- Traffic shaping (weighted routing, retry policies, circuit breakers)
- mTLS for inter-service encryption
- Works with ECS, EKS, EC2

---

## 4.14 AWS Cloud Map

**What it is:** Service discovery for cloud resources. Register and discover services by name using DNS or API queries.

**Key Features:**
- Health checks (custom or Route 53)
- Supports EC2, ECS, EKS, Lambda, IP, DNS namespaces


---

# 5. Security, Identity & Compliance

> AWS security services form a layered defense from identity management through threat detection, data protection, and compliance automation.

---

## 5.1 IAM (Identity and Access Management)

**What it is:** Manages authentication and authorization for all AWS services. Every API call is authorized through IAM.

**Core Constructs:**
- **Users** — individual people or applications (legacy; prefer IAM roles)
- **Groups** — collection of users sharing policies
- **Roles** — temporary credentials; assumed by AWS services, users, or external identities
- **Policies** — JSON documents defining permissions

**Policy Types:**

| Type | Attached To | Notes |
|------|------------|-------|
| Identity-based | User, Group, Role | Defines what the identity can do |
| Resource-based | Resource (S3, KMS, etc.) | Defines who can access the resource |
| SCP (Service Control Policy) | AWS Org OU or Account | Maximum permission boundary; doesn't grant |
| Permission Boundary | IAM User or Role | Limits max permissions an identity can have |
| Session Policy | AssumeRole call | Further restricts temporary credentials |
| ACL | S3, WAF (legacy) | Cross-account access without IAM |

**Policy Evaluation Logic (in order):**
1. Explicit Deny (in any policy) → **DENY** (always wins)
2. SCP allows? → If no SCP allows → DENY
3. Resource-based policy allows? → **ALLOW** (for cross-account)
4. Identity-based policy allows? → ALLOW
5. Permission Boundary allows? → If no → DENY
6. Session policy allows? → If no → DENY

**Key Features:**
- MFA enforcement
- Password policies
- Access Analyzer — find cross-account and public access
- IAM Credential Report — all users and credential status
- Last-accessed information — identify unused permissions
- IAM Roles Anywhere — use X.509 certs for non-AWS workloads
- Service-linked roles — pre-defined roles for AWS services

```bash
# Create IAM user
aws iam create-user --user-name developer-jane

# Attach policy to user
aws iam attach-user-policy \
  --user-name developer-jane \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Create role with trust policy
aws iam create-role \
  --role-name LambdaExecRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name LambdaExecRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create inline policy
aws iam put-role-policy \
  --role-name LambdaExecRole \
  --policy-name DynamoDBAccess \
  --policy-document file://dynamo-policy.json

# List attached policies
aws iam list-attached-role-policies --role-name LambdaExecRole

# Create IAM policy
aws iam create-policy \
  --policy-name S3ReadOnly \
  --policy-document file://s3-read-policy.json

# Simulate policy (dry run permissions check)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
  --action-names s3:GetObject s3:PutObject

# Generate credential report
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d

# Create access key
aws iam create-access-key --user-name developer-jane

# Enable MFA
aws iam enable-mfa-device \
  --user-name developer-jane \
  --serial-number arn:aws:iam::123456789012:mfa/developer-jane \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

**Gotchas / Limits:**
- Default: 5,000 IAM users per account; 1,000 roles; 10 managed policies per user/role
- SCPs don't grant permissions; they only restrict what can be granted
- Root account should never be used for day-to-day operations; protect with hardware MFA
- IAM is global (not regional)
- Roles have no long-term credentials — temporary STS credentials only
- Permission Boundary must be set on both the role AND the policy for full boundary effect

---

## 5.2 AWS STS (Security Token Service)

**What it is:** Issues temporary security credentials (Access Key ID, Secret Key, Session Token) for IAM roles or federated users.

**Key APIs:**
- `AssumeRole` — assume a role (cross-account or same account)
- `AssumeRoleWithSAML` — federated users from enterprise IdP
- `AssumeRoleWithWebIdentity` — federated from Cognito, OIDC (IRSA for EKS)
- `GetSessionToken` — temp credentials for IAM user (MFA enforcement)
- `GetFederationToken` — legacy federated user credentials

```bash
# Assume role
aws sts assume-role \
  --role-arn arn:aws:iam::111111111111:role/CrossAccountRole \
  --role-session-name "deployment-session" \
  --duration-seconds 3600

# Get caller identity
aws sts get-caller-identity
```

**Gotchas:** Default session duration 1 hour (max 12 hr for roles); max 1 hr for AssumeRoleWithWebIdentity unless role allows longer.

---

## 5.3 Amazon Cognito

**What it is:** Identity management for web and mobile apps. Handles user registration, authentication, and authorization.

**Components:**

| Component | Purpose |
|-----------|---------|
| User Pools | User directory; sign-up/sign-in, JWT tokens |
| Identity Pools (Federated) | Exchange tokens for AWS credentials (STS) |

**User Pools Key Features:**
- Built-in UI, customizable
- MFA (TOTP, SMS)
- Social login (Google, Facebook, Apple, Amazon)
- Enterprise federation (SAML 2.0, OIDC)
- Lambda triggers (pre/post sign-up, pre-token generation, custom auth)
- Advanced Security (adaptive authentication, breach detection)
- Groups and roles

**Identity Pools:**
- Map authenticated/unauthenticated users to IAM roles
- Supports Cognito User Pools + external IdPs + anonymous access

```bash
# Create user pool
aws cognito-idp create-user-pool \
  --pool-name MyAppUsers \
  --policies PasswordPolicy='{MinimumLength=12,RequireUppercase=true,RequireLowercase=true,RequireNumbers=true}'

# Create app client
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_XXXXX \
  --client-name my-app \
  --generate-secret false \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH

# Admin create user
aws cognito-idp admin-create-user \
  --user-pool-id us-east-1_XXXXX \
  --username user@example.com \
  --temporary-password 'TempPass123!'
```

**Pricing:** MAU-based pricing (Monthly Active Users). First 50K MAU free.

---

## 5.4 AWS Secrets Manager

**What it is:** Manage, retrieve, and rotate secrets (DB passwords, API keys, credentials) securely. Built-in rotation via Lambda.

**Key Features:**
- Automatic rotation — native support for RDS, Aurora, Redshift, DocumentDB; custom via Lambda
- Cross-account access via resource policies
- Versioning and staging labels (AWSCURRENT, AWSPENDING, AWSPREVIOUS)
- Encryption via KMS
- Integration with RDS Proxy, ECS, Lambda, EKS

```bash
# Create secret
aws secretsmanager create-secret \
  --name /prod/db/password \
  --secret-string '{"username":"admin","password":"MyPass123"}' \
  --kms-key-id arn:aws:kms:...

# Get secret value
aws secretsmanager get-secret-value \
  --secret-id /prod/db/password \
  --version-stage AWSCURRENT

# Rotate secret immediately
aws secretsmanager rotate-secret \
  --secret-id /prod/db/password \
  --rotation-lambda-arn arn:aws:lambda:... \
  --rotation-rules AutomaticallyAfterDays=30

# List secrets
aws secretsmanager list-secrets --filter Key=name,Values=/prod/
```

**Pricing:** $0.40/secret/month + $0.05/10K API calls

**vs SSM Parameter Store:**
- Secrets Manager: auto-rotation, purpose-built for secrets, higher cost
- SSM Parameter Store: cheaper/free (Standard tier), no native rotation, but good for config

---

## 5.5 AWS KMS (Key Management Service)

**What it is:** Managed service for creating and controlling encryption keys. Integrated with most AWS services for envelope encryption.

**Key Types:**

| Type | Control | Use Case |
|------|---------|---------|
| AWS Managed Key | AWS managed | Default for service encryption |
| Customer Managed Key (CMK) | You manage rotation, policy | Compliance requirements |
| AWS Owned Key | AWS internal | Not directly visible |

**Key Features:**
- Symmetric (AES-256-GCM) and Asymmetric (RSA, ECC) keys
- Key rotation — annual (AWS Managed: automatic; CMK: opt-in or on-demand)
- Key policies — resource-based policy on each key
- Grants — delegate key use without updating key policy
- Multi-Region Keys — replicate across regions (same key ID)
- CloudHSM-backed keys (custom key store)
- Envelope encryption — data key encrypted by CMK; data encrypted by data key

```bash
# Create CMK
aws kms create-key \
  --description "Prod DB encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --policy file://key-policy.json

# Create alias
aws kms create-alias \
  --alias-name alias/prod-db-key \
  --target-key-id key-id

# Encrypt data
aws kms encrypt \
  --key-id alias/prod-db-key \
  --plaintext fileb://secret.txt \
  --output text --query CiphertextBlob | base64 --decode > encrypted.bin

# Decrypt
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text --query Plaintext | base64 --decode

# Enable key rotation
aws kms enable-key-rotation --key-id key-id

# Describe key
aws kms describe-key --key-id alias/prod-db-key
```

**Pricing:** $1/CMK/month + $0.03/10K API calls

**Gotchas:**
- KMS is regional — Multi-Region Keys must be explicitly enabled
- Key deletion has mandatory 7–30 day waiting period (cannot be immediate)
- KMS API has quotas — high-frequency encryption should use Data Key Caching

---

## 5.6 AWS CloudHSM

**What it is:** Hardware Security Module (HSM) in the cloud. FIPS 140-2 Level 3 validated. You manage the HSM, keys, and users — AWS cannot access your keys.

**Key Features:**
- Dedicated HSM hardware (not shared)
- Customer-managed: you are sole key custodian
- Supports PKCS#11, JCE, CNG/KSP, OpenSSL PKCS#11 library
- Cluster mode — 2+ HSMs for HA
- Can be used as custom key store for KMS

**Use Cases:** Regulatory requirements needing FIPS 140-2 L3, Oracle TDE, SSL/TLS offload, document signing

**Pricing:** ~$1.60/HSM/hour per instance

---

## 5.7 AWS ACM (Certificate Manager)

**What it is:** Provision, manage, and deploy SSL/TLS certificates for AWS services.

**Key Features:**
- Free public certificates (DV certificates for ALB, CloudFront, API Gateway)
- Auto-renewal
- DNS validation (CNAME record) or email validation
- Private CA (ACM PCA) — manage private certificate authority
- Import external certificates

**Gotchas:**
- ACM certificates cannot be downloaded — only usable on integrated AWS services
- ACM is regional — CloudFront certificates must be in us-east-1
- Private CA costs $400/month per CA + $0.75/certificate

---

## 5.8 Amazon GuardDuty

**What it is:** Intelligent threat detection using ML and threat intelligence. Monitors CloudTrail, VPC Flow Logs, DNS logs, S3 data events, EKS audit logs, Lambda network activity.

**Key Features:**
- No agent required — analyzes AWS data sources
- 300+ threat findings organized by severity
- Multi-account via Organizations
- GuardDuty Malware Protection — scan EBS volumes and S3 objects
- GuardDuty RDS Protection — detect anomalous login attempts
- Automated response via EventBridge → Lambda

**Pricing:** Based on volume of analyzed events (CloudTrail events, DNS queries, VPC Flow Logs). Varies by data volume.

---

## 5.9 Amazon Inspector v2

**What it is:** Automated vulnerability management for EC2, Lambda, and container images in ECR. Continuously scans for CVEs and network exposure.

**Key Features:**
- Agentless for EC2 (uses SSM) or agent-based
- Inspector Score — prioritized by exploitability
- SBOM (Software Bill of Materials) export
- CIS benchmarks for EC2

**Pricing:** Per EC2 instance-month + per container image scan + per Lambda function scan.

---

## 5.10 Amazon Macie

**What it is:** ML-powered sensitive data discovery for S3. Automatically finds PII, credentials, financial data.

**Key Features:**
- Sensitive data jobs — scan S3 buckets on schedule or on-demand
- Findings sent to EventBridge, Security Hub
- Custom data identifiers — regex + keyword patterns

---

## 5.11 AWS Security Hub

**What it is:** Central security findings aggregator and compliance checker. Consolidates findings from GuardDuty, Inspector, Macie, Firewall Manager, IAM Access Analyzer, and partner tools.

**Key Features:**
- Security standards: CIS AWS Foundations, AWS Foundational, PCI DSS, NIST 800-53
- Automated responses via EventBridge/Step Functions/Lambda
- Cross-account aggregation via Organizations
- Custom insights and dashboards

---

## 5.12 Amazon Detective

**What it is:** Security investigation service. Automatically collects and organizes CloudTrail, VPC Flow Logs, GuardDuty findings into a behavior graph for forensic investigation.

**Key Features:**
- Visual graphs of activity over time
- Deep dives into EC2 instances, IAM roles, IP addresses
- Integration with GuardDuty and Security Hub

---

## 5.13 IAM Identity Center (SSO)

**What it is:** Centralized workforce identity management. Single sign-on to AWS accounts and business applications from one place.

**Key Features:**
- Integrates with external IdPs (Okta, Azure AD, ADFS) via SAML 2.0 / SCIM
- Permission Sets — define what users can do in each account
- Multi-account access across AWS Organizations
- Attribute-Based Access Control (ABAC) with user attributes

---

## 5.14 AWS Directory Service

| Type | Description | Use Case |
|------|-------------|---------|
| Managed Microsoft AD | Fully managed Active Directory (Windows Server 2019) | Enterprise AD, trusts |
| AD Connector | Proxy to existing on-prem AD | Reuse existing AD |
| Simple AD | Samba-based lightweight LDAP | Dev/test, small orgs |

---

## 5.15 AWS Resource Access Manager (RAM)

**What it is:** Share AWS resources across accounts without VPC peering. Share subnets, Transit Gateways, Route 53 Resolver rules, License Manager configs, and more.

---

## 5.16 AWS Firewall Manager

**What it is:** Centrally manage WAF rules, Shield Advanced, Security Groups, Network Firewall, and Route 53 DNS Firewall across all accounts in an AWS Organization.

**Key Features:**
- Security policies — enforce WAF web ACLs on all ALBs/CloudFront
- Automatic remediation for non-compliant resources
- Requires AWS Organizations and designated administrator account


---

# 6. Management & Governance

> AWS management tools provide visibility, control, automation, and governance across your AWS environment.

---

## 6.1 CloudWatch

**What it is:** Monitoring and observability platform. Collects metrics, logs, events, and traces from AWS and custom sources.

**Key Components:**

### CloudWatch Metrics
- Namespace → Metric → Dimensions
- Default resolution: 1 minute; High resolution: 1 second (custom)
- Retention: 3 hr (1s), 15 days (1min), 63 days (5min), 15 months (1hr)
- Math expressions across metrics
- Anomaly Detection — ML-based dynamic thresholds

### CloudWatch Alarms
- States: OK, ALARM, INSUFFICIENT_DATA
- Actions: SNS, Auto Scaling, EC2 actions, Systems Manager OpsCenter
- Composite Alarms — combine multiple alarms with AND/OR
- Evaluate over periods (M out of N)

### CloudWatch Logs
- Log Groups → Log Streams
- Log Insights — interactive query language (stats, filter, sort, parse)
- Subscriptions — stream to Lambda, Kinesis, Firehose
- Metric Filters — extract metrics from log patterns
- Log anomaly detection
- Live Tail — real-time log streaming in console

### CloudWatch Synthetics
- Canary scripts — monitor endpoints and APIs via Puppeteer/Selenium
- Blueprints for heartbeat, API canary, UI canary

### CloudWatch RUM (Real User Monitoring)
- Collect client-side performance data from web applications

### CloudWatch Evidently
- Feature flags and A/B testing

### Application Signals
- APM (Application Performance Monitoring) using OpenTelemetry

```bash
# Put custom metric
aws cloudwatch put-metric-data \
  --namespace MyApp/Performance \
  --metric-data MetricName=ResponseTime,Value=250,Unit=Milliseconds \
  --dimensions Name=Environment,Value=prod

# Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPU \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345678 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789012:ops-alerts

# Get metric stats
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345 \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-16T00:00:00Z \
  --period 3600 \
  --statistics Average Maximum

# Log Insights query
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50'

# Create log metric filter
aws logs put-metric-filter \
  --log-group-name /aws/lambda/my-function \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp,metricValue=1,unit=Count

# Get log groups
aws logs describe-log-groups --prefix /aws/lambda

# Put subscription filter (to Kinesis)
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/my-function \
  --filter-name ship-to-kinesis \
  --filter-pattern "" \
  --destination-arn arn:aws:kinesis:us-east-1:123456789012:stream/log-stream
```

**Pricing:**
- Custom metrics: $0.30/metric/month
- Dashboards: $3/dashboard/month (free tier: 3 dashboards, 50 metrics)
- Alarms: $0.10/alarm/month standard; $0.30/high-resolution
- Logs ingestion: $0.50/GB; storage: $0.03/GB-month
- Synthetics: $0.0012/canary run

---

## 6.2 AWS CloudTrail

**What it is:** Records all API calls made in your AWS account. Audit log of "who did what, when, from where."

**Key Features:**
- Management Events — control plane operations (default, free tier 90 days)
- Data Events — S3 object operations, Lambda invocations, DynamoDB item operations (extra cost)
- Insights Events — detect unusual API activity
- Trail — configure log delivery to S3 + CloudWatch Logs
- Multi-region and Organization trails
- Log file integrity validation (SHA-256 hash chain)
- CloudTrail Lake — SQL-based querying of events (up to 7-year retention)

```bash
# Create trail
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name my-cloudtrail-bucket \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --cloud-watch-logs-log-group-arn arn:aws:logs:...:log-group:CloudTrail \
  --cloud-watch-logs-role-arn arn:aws:iam::...:role/CloudTrailRole

# Start logging
aws cloudtrail start-logging --name org-trail

# Look up events (last 90 days)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket \
  --start-time 2024-01-01T00:00:00Z
```

**Pricing:** Free (90-day management event history). Trail delivery: first copy of management events free per region; additional trails $2/100K events. Data events: $0.10/100K.

---

## 6.3 AWS Config

**What it is:** Continuous compliance monitoring. Records resource configurations and changes; evaluates against compliance rules.

**Key Features:**
- Configuration Recorder — records all resource configs and changes
- Config Rules — AWS Managed (~200+) or custom Lambda-based
- Conformance Packs — bundled rules for CIS, NIST, PCI
- Auto-Remediation — link rules to SSM Automation documents
- Configuration History and Snapshots
- Advanced Queries — SQL-like queries across resource inventory
- Aggregators — multi-account, multi-region view

```bash
# Get resource compliance
aws configservice get-compliance-details-by-resource \
  --resource-type AWS::S3::Bucket \
  --resource-id my-bucket

# List non-compliant resources
aws configservice get-compliance-summary-by-resource-type \
  --resource-types AWS::EC2::SecurityGroup

# Describe config rules
aws configservice describe-config-rules

# Start remediation
aws configservice start-remediation-execution \
  --config-rule-name restricted-ssh \
  --resource-keys '[{"resourceType":"AWS::EC2::SecurityGroup","resourceId":"sg-12345"}]'
```

**Pricing:** $0.003/configuration item recorded; $0.001/rule evaluation/month

---

## 6.4 AWS Systems Manager (SSM)

**What it is:** Operational hub for managing EC2 instances and on-prem servers at scale. No SSH/RDP required.

**Key Capabilities:**

| Capability | Description |
|-----------|-------------|
| Session Manager | Browser-based shell/RDP; no open ports, no SSH keys |
| Run Command | Execute scripts/commands across instances |
| Patch Manager | Automated OS patching with compliance reporting |
| State Manager | Ensure instances maintain desired state |
| Inventory | Collect software/config inventory from instances |
| Parameter Store | Hierarchical config/secret storage |
| Automation | Runbooks for complex operations (start/stop, AMI creation) |
| OpsCenter | Aggregate operational issues (OpsItems) |
| Incident Manager | Incident response with runbooks and escalation |
| Fleet Manager | View and manage EC2 fleet in console |
| Change Manager | Change request workflows |

**SSM Parameter Store:**
- Standard Tier: free, 10,000 parameters, 4 KB max
- Advanced Tier: 100,000 parameters, 8 KB max, parameter policies, $0.05/parameter/month
- SecureString: encrypted with KMS
- Hierarchy: `/prod/app/db/password`

```bash
# Start Session Manager session
aws ssm start-session --target i-1234567890abcdef0

# Run command across multiple instances
aws ssm send-command \
  --targets '[{"Key":"tag:Environment","Values":["prod"]}]' \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["sudo systemctl status nginx"]' \
  --output-s3-bucket-name my-ssm-output-bucket

# Get command invocation results
aws ssm get-command-invocation \
  --command-id cmd-id \
  --instance-id i-12345678

# Put SSM Parameter
aws ssm put-parameter \
  --name "/prod/db/password" \
  --value "MySecretPass123!" \
  --type SecureString \
  --key-id alias/prod-db-key \
  --overwrite

# Get SSM Parameter
aws ssm get-parameter \
  --name "/prod/db/password" \
  --with-decryption

# Get parameters by path
aws ssm get-parameters-by-path \
  --path "/prod/db/" \
  --with-decryption \
  --recursive

# Create Automation execution
aws ssm start-automation-execution \
  --document-name "AWS-CreateManagedLinuxInstance" \
  --parameters '{"AmiId":["ami-12345"],"InstanceType":["t3.medium"]}'

# List patch compliance
aws ssm describe-instance-patch-states \
  --instance-ids i-12345 i-67890
```

---

## 6.5 AWS Organizations

**What it is:** Central account governance. Manage multiple AWS accounts with hierarchical OUs, SCPs, and consolidated billing.

**Key Features:**
- Root → OUs (Organizational Units) → Accounts
- SCPs — maximum permission boundary for all principals in scope
- Delegated administrator — designate member account to manage specific services
- Tag policies — enforce tagging standards
- Backup policies — centrally enforce AWS Backup plans
- AI opt-out policies — opt out of AI services data usage
- CloudTrail and Config Organization integration
- Consolidated billing — all accounts in one bill, volume discounts shared

**SCP Evaluation:** SCPs define the maximum allowed actions. They do NOT grant permissions.

```bash
# List accounts
aws organizations list-accounts

# Create OU
aws organizations create-organizational-unit \
  --parent-id r-root-id \
  --name Production

# Attach SCP
aws organizations attach-policy \
  --policy-id p-12345678 \
  --target-id ou-12345678

# Create SCP
aws organizations create-policy \
  --name DenyRootAccountActions \
  --type SERVICE_CONTROL_POLICY \
  --description "Deny root account API calls" \
  --content file://scp.json
```

---

## 6.6 AWS Control Tower

**What it is:** Landing Zone automation. Set up and govern a secure, compliant multi-account AWS environment with best practices out of the box.

**Key Features:**
- Account Factory — self-service account provisioning via Service Catalog
- Guardrails — preventive (SCPs) and detective (Config rules) controls
- ~300 built-in guardrails (AWS Foundational, CIS, NIST, PCI)
- Control Tower Dashboard — compliance status across all accounts
- Account Factory for Terraform (AFT) — IaC-based account vending

---

## 6.7 AWS CloudFormation

**What it is:** Infrastructure as Code (IaC) — define AWS resources in JSON/YAML templates; CloudFormation provisions and manages them.

**Key Features:**
- Stacks — deployed instance of a template
- Change Sets — preview changes before applying
- Stack Sets — deploy across multiple accounts/regions
- Drift Detection — detect manual changes to stack resources
- Nested Stacks — modular templates
- Custom Resources — Lambda-backed resources for non-native services
- CloudFormation Registry — extend with private resource types
- Rollback — automatic on failure; retain resources on policy

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name prod-vpc \
  --template-body file://vpc-template.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Key=Project,Value=MyApp

# Create change set
aws cloudformation create-change-set \
  --stack-name prod-vpc \
  --change-set-name add-subnet \
  --template-body file://vpc-v2.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod

# Execute change set
aws cloudformation execute-change-set \
  --change-set-name add-subnet \
  --stack-name prod-vpc

# Detect drift
aws cloudformation detect-stack-drift --stack-name prod-vpc

# Deploy StackSet to org
aws cloudformation create-stack-set \
  --stack-set-name baseline-config \
  --template-body file://baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

# Describe stack events
aws cloudformation describe-stack-events \
  --stack-name prod-vpc \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

**Pricing:** Free (pay only for provisioned resources)

**Gotchas:**
- Stack update can fail and auto-rollback — ensure update policies
- Circular dependencies cause deploy failures — use DependsOn or !Ref carefully
- Deletion policies (Retain, Snapshot, Delete) on resources
- CloudFormation macros via Lambda for template transforms

---

## 6.8 AWS CDK (Cloud Development Kit)

**What it is:** Define AWS infrastructure using familiar programming languages (TypeScript, Python, Java, Go, C#). Synthesizes to CloudFormation.

**Key Concepts:**
- App → Stack → Construct (L1/L2/L3)
- L1 — CFn resources directly
- L2 — higher-level opinionated constructs
- L3 — patterns/solutions (e.g., `QueueProcessingFargateService`)

---

## 6.9 AWS SAM (Serverless Application Model)

**What it is:** CloudFormation extension for serverless apps. Simplified syntax for Lambda, API Gateway, DynamoDB, SQS, SNS.

**Key Features:**
- `sam local invoke` — test Lambda locally with Docker
- `sam deploy` — package and deploy serverless apps
- `sam pipeline` — set up CI/CD pipelines

---

## 6.10 AWS Trusted Advisor

**What it is:** Best-practice recommendations across 5 categories: Cost Optimization, Performance, Security, Fault Tolerance, Service Limits.

**Tiers:**
- All accounts: 7 core checks (security/limits)
- Business/Enterprise Support: 500+ checks, programmatic access, CloudWatch integration

---

## 6.11 AWS Compute Optimizer

**What it is:** ML-based recommendations for right-sizing EC2, Lambda, ECS, EBS, Auto Scaling Groups.

**Key Features:**
- 14 days of utilization data analyzed
- Shows current vs recommended specs with projected savings/performance
- Enhanced Infrastructure Metrics (paid): 3 months of data

---

## 6.12 AWS Well-Architected Tool

**What it is:** Conduct Well-Architected Reviews against the 6 pillars. Generate improvement plan with milestone tracking. Custom Lenses for specific workload types.


---

# 7. Developer Tools

> AWS Developer Tools streamline the software development lifecycle from code to deployment with native CI/CD, testing, and debugging services.

---

## 7.1 AWS CodeCommit
**What it is:** Managed Git repositories. Hosted in AWS with IAM integration.
**Key Features:** Pull requests, triggers (Lambda/SNS), branch protection, SSH/HTTPS access.
**Pricing:** $1/user/month (5 active users free). Effectively superseded by GitHub/GitLab integrations.
**⚠️ Note:** AWS announced end of new customer onboarding for CodeCommit in 2024.

---

## 7.2 AWS CodeBuild
**What it is:** Fully managed CI build service. Compile, test, and produce artifacts without managing build servers.
**Key Features:**
- `buildspec.yml` defines build phases (install, pre_build, build, post_build)
- Docker support — build Docker images, push to ECR
- VPC support for accessing private resources
- Caching (S3 or local) to speed up builds
- Test reports (JUnit, NUnit, TestNG, Cucumber, VisualStudio)
- Environment variables + Secrets Manager integration

```bash
# Start build
aws codebuild start-build \
  --project-name my-app-build \
  --source-version main

# List builds
aws codebuild list-builds-for-project --project-name my-app-build

# Batch get builds
aws codebuild batch-get-builds --ids build-id-1 build-id-2
```
**Pricing:** Per build minute ($0.005/min for general1.small)

---

## 7.3 AWS CodeDeploy
**What it is:** Automate code deployments to EC2, Lambda, ECS, and on-premises servers.
**Deployment Types:**

| Type | Description |
|------|-------------|
| In-place | Stop → deploy → restart (EC2/on-prem only) |
| Blue/Green (EC2) | Launch new ASG, cut over ELB |
| Blue/Green (ECS) | Shift traffic between task sets via ALB |
| Linear/Canary (Lambda) | Gradually shift traffic to new version |

**`appspec.yml`:** Defines deployment steps and lifecycle hooks.

**Pricing:** Free for EC2/on-prem. Lambda/ECS: $0.02/1000 deployments.

---

## 7.4 AWS CodePipeline
**What it is:** Continuous delivery pipeline service. Orchestrate source → build → test → deploy stages.
**Key Features:**
- Triggers: CodeCommit, GitHub, S3, ECR image push
- Actions: CodeBuild, CodeDeploy, Lambda, ECS, CloudFormation, Elastic Beanstalk, manual approval
- Parallel and sequential stages
- Variable passing between stages

```bash
# Start pipeline execution
aws codepipeline start-pipeline-execution --name my-pipeline

# Get pipeline state
aws codepipeline get-pipeline-state --name my-pipeline

# Approve manual approval action
aws codepipeline put-approval-result \
  --pipeline-name my-pipeline \
  --stage-name Approval \
  --action-name ManualApproval \
  --result summary="Looks good",status=Approved \
  --token token-from-get-approval-token
```
**Pricing:** $1/pipeline/month (active pipelines only)

---

## 7.5 AWS CodeArtifact
**What it is:** Managed software artifact repository. Hosts npm, PyPI, Maven, NuGet, Gradle packages with upstream proxy caching.
**Key Features:**
- Upstream repositories — proxy public repos (npmjs, PyPI, Maven Central)
- Domain — groups multiple repositories
- Cross-account access
- VPC endpoint support

```bash
# Login for npm
aws codeartifact login --tool npm --repository my-repo --domain my-domain
```
**Pricing:** $0.05/GB stored + $0.09/GB requested

---

## 7.6 AWS CodeGuru
**What it is:** ML-powered code review (Reviewer) and application performance profiling (Profiler).
- **Reviewer:** Detect security vulnerabilities, coding best practices in Java/Python PRs
- **Profiler:** CPU/memory profiling in production; Lambda support

---

## 7.7 AWS X-Ray
**What it is:** Distributed tracing for applications. Visualize request flow across microservices, identify bottlenecks and errors.
**Key Features:**
- Traces, segments, subsegments
- Service map visualization
- Sampling rules — reduce trace volume
- X-Ray Insights — anomaly detection
- Works with Lambda, EC2, ECS, EKS, API Gateway, Elastic Beanstalk

```bash
# Get service map
aws xray get-service-graph \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```
**Pricing:** $5/million traces recorded; $0.50/million scanned

---

## 7.8 AWS Fault Injection Simulator (FIS)
**What it is:** Chaos engineering service. Run controlled fault injection experiments on AWS resources.
**Actions:** Stop EC2, terminate ECS tasks, inject CPU/memory stress, simulate AZ outages, inject Lambda errors, throttle RDS connections.
**Pricing:** Per action-minute executed.

---

## 7.9 Amazon DevOps Guru
**What it is:** ML-powered operational insights. Detects anomalous operational behaviors and identifies root causes automatically.
**Key Features:**
- Integrates with CloudFormation stacks, tags, or all account resources
- Reactive insights (from anomalies) and Proactive insights (upcoming issues)
- Recommendations link to specific AWS service documentation

---

# 8. Analytics

> AWS Analytics provides a complete, integrated toolchain for data ingestion, storage, processing, and visualization at any scale.

---

## 8.1 Amazon Athena
**What it is:** Serverless, interactive SQL query service for data in S3. Pay per query. Built on Presto/Trino.
**Key Features:**
- Query CSV, JSON, Parquet, ORC, Avro, TSV in S3
- Federated Queries — query RDS, DynamoDB, on-prem via Lambda connectors
- Athena for Apache Spark — run Spark workloads serverlessly
- Integration with AWS Glue Data Catalog
- Workgroups — separate query limits and cost control
- Result caching and saved queries
- CTAS (Create Table As Select) for ETL patterns

```bash
# Start query
aws athena start-query-execution \
  --query-string "SELECT * FROM my_db.orders WHERE year='2024' LIMIT 100" \
  --query-execution-context Database=my_db \
  --result-configuration OutputLocation=s3://my-bucket/athena-results/ \
  --work-group primary

# Get results
aws athena get-query-results --query-execution-id query-id

# Get query status
aws athena get-query-execution --query-execution-id query-id
```
**Pricing:** $5/TB scanned (use Parquet/ORC + partitions to reduce)
**Optimization:** Columnar format (Parquet/ORC) reduces scan by 90%+; partition on date/region

---

## 8.2 Amazon EMR
**What it is:** Managed big data platform. Run Apache Spark, Hive, Presto, HBase, Flink on scalable clusters.
**Deployment Options:**
- EMR on EC2 — traditional cluster (master + core + task nodes)
- EMR Serverless — no cluster management, auto-scales
- EMR on EKS — run Spark as pods on EKS

**Key Features:**
- Auto Scaling for task nodes
- Spot instances for task nodes (cost savings)
- Instance Fleets — diverse instance types for Spot
- EMR Studio — Jupyter-based IDE for data science
- EMR Steps — submit Spark/Hive jobs to cluster
- EMRFS — use S3 as HDFS replacement

```bash
# Create EMR cluster
aws emr create-cluster \
  --name "Prod Spark" \
  --release-label emr-7.0.0 \
  --applications Name=Spark Name=Hadoop \
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m5.xlarge \
    InstanceGroupType=CORE,InstanceCount=4,InstanceType=m5.2xlarge \
    InstanceGroupType=TASK,InstanceCount=8,InstanceType=m5.2xlarge,BidPrice=OnDemandPrice \
  --use-default-roles \
  --ec2-attributes KeyName=my-keypair,SubnetId=subnet-12345 \
  --log-uri s3://my-bucket/emr-logs/ \
  --auto-terminate

# Add step (Spark job)
aws emr add-steps \
  --cluster-id j-12345 \
  --steps '[{"Name":"SparkJob","ActionOnFailure":"CONTINUE","HadoopJarStep":{"Jar":"command-runner.jar","Args":["spark-submit","--class","com.example.Main","s3://my-bucket/app.jar"]}}]'
```
**Pricing:** EC2 costs + EMR premium ($0.012–$0.27/EMR vCore-hour or vCore-hour depending on size)

---

## 8.3 Amazon Kinesis Data Streams
**What it is:** Real-time streaming data service. Ingest GB/s of data and process it with multiple consumers simultaneously.
**Key Concepts:**
- Shard — unit of capacity (1 MB/s in, 2 MB/s out, 1000 writes/s)
- Partition Key — determines shard routing
- Sequence Number — unique ID per record
- Retention: 24 hrs default, up to 365 days (extended)
- On-demand mode — auto-scales shards

```bash
# Create stream
aws kinesis create-stream --stream-name my-stream --shard-count 4

# Put record
aws kinesis put-record \
  --stream-name my-stream \
  --data "$(echo '{"event":"click","userId":"u123"}' | base64)" \
  --partition-key "u123"

# Get shard iterator
aws kinesis get-shard-iterator \
  --stream-name my-stream \
  --shard-id shardId-000000000000 \
  --shard-iterator-type LATEST

# Get records
aws kinesis get-records --shard-iterator AAAA....

# Describe stream
aws kinesis describe-stream-summary --stream-name my-stream
```
**Pricing:** $0.015/shard-hour + $0.014/million records; Extended retention extra.

---

## 8.4 Amazon Kinesis Data Firehose
**What it is:** Fully managed delivery stream. Load streaming data to S3, Redshift, OpenSearch, Splunk, HTTP endpoints. No coding required.
**Key Features:**
- Buffer by size (1–128 MB) or time (60–900 s)
- Transform with Lambda inline
- Dynamic partitioning — route records to different S3 prefixes based on content
- Data format conversion (JSON → Parquet/ORC)
- Direct PUT or Kinesis Data Streams as source

```bash
# Create delivery stream
aws firehose create-delivery-stream \
  --delivery-stream-name my-firehose \
  --s3-destination-configuration \
    RoleARN=arn:aws:iam::...:role/firehose-role,\
    BucketARN=arn:aws:s3:::my-bucket,\
    Prefix=data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/,\
    BufferingHints='{SizeInMBs:128,IntervalInSeconds=300}'
```
**Pricing:** $0.029/GB ingested (first 500 TB/month)

---

## 8.5 AWS Glue
**What it is:** Serverless ETL and data catalog service. Discover, prepare, and combine data for analytics.
**Key Components:**
- **Glue Data Catalog** — Apache Hive Metastore compatible; central metadata store for Athena, EMR, Redshift Spectrum
- **Glue Crawlers** — discover schema from S3, RDS, DynamoDB, JDBC
- **Glue ETL Jobs** — Apache Spark-based ETL (Python/Scala); GlueContext API
- **Glue DataBrew** — visual data preparation (no code)
- **Glue Data Quality** — rule-based data quality checks (DQ Ruleset)
- **Glue Studio** — visual ETL job builder
- **Glue Streaming** — Kafka/Kinesis streaming ETL

```bash
# Start crawler
aws glue start-crawler --name my-s3-crawler

# Start ETL job
aws glue start-job-run \
  --job-name my-etl-job \
  --arguments '--input_path=s3://raw/,--output_path=s3://processed/'

# Get job run status
aws glue get-job-run --job-name my-etl-job --run-id jr-12345

# List tables in catalog
aws glue get-tables --database-name my_database
```
**Pricing:** $0.44/DPU-hour (ETL); Catalog: first million objects/month free

---

## 8.6 AWS Lake Formation
**What it is:** Build, secure, and manage data lakes on S3. Fine-grained column-level access control on top of Glue Catalog.
**Key Features:**
- LF-TBAC — tag-based access control
- Column and row-level security in Athena and Redshift Spectrum
- Data Filters for column masking
- Governed Tables — ACID transactions on S3 data
- Cross-account data sharing

---

## 8.7 Amazon Redshift
**What it is:** Cloud data warehouse. Petabyte-scale OLAP with columnar storage, MPP architecture.
**Key Features:**
- RA3 nodes — compute/storage separation (offload to S3)
- Redshift Serverless — auto-scale, pay per RPU-second
- Redshift Spectrum — query S3 data without loading
- Zero-ETL integrations — Aurora → Redshift, RDS → Redshift, DynamoDB → Redshift automatically
- Materialized Views with auto-refresh
- AQUA (Advanced Query Accelerator) — hardware-accelerated cache
- Data Sharing — share live data across clusters without copying
- Redshift ML — CREATE MODEL SQL → SageMaker Autopilot

```bash
# Create serverless namespace
aws redshift-serverless create-namespace \
  --namespace-name prod-namespace \
  --admin-username admin \
  --admin-user-password 'MyPass123!'

# Create workgroup
aws redshift-serverless create-workgroup \
  --workgroup-name prod-workgroup \
  --namespace-name prod-namespace \
  --base-capacity 8

# Execute query (via Data API)
aws redshift-data execute-statement \
  --cluster-identifier my-cluster \
  --database analytics \
  --db-user admin \
  --sql "SELECT COUNT(*) FROM fact_orders WHERE order_date >= '2024-01-01'"
```
**Pricing:** RA3 nodes from $0.25/node-hour; Serverless: $0.36/RPU-hour; Storage: $0.024/GB-month

---

## 8.8 Amazon OpenSearch Service
**What it is:** Managed OpenSearch (fork of Elasticsearch 7.x) and Kibana/OpenSearch Dashboards. Full-text search, log analytics, observability.
**Key Features:**
- Managed scaling, snapshots, security (fine-grained access control)
- OpenSearch Serverless — auto-scale, pay per OCU-hour
- Cold storage — move indexes to S3-backed storage
- UltraWarm — warm storage tier
- SQL plugin, Observability plugin, Anomaly Detection
- Multiple AZ deployment

---

## 8.9 Amazon MSK (Managed Streaming for Apache Kafka)
**What it is:** Fully managed Apache Kafka. Run Kafka without managing brokers, ZooKeeper, or storage.
**Key Features:**
- MSK Serverless — auto-scale capacity
- MSK Connect — managed Kafka Connect connectors
- Kafka versions 2.x, 3.x
- TLS + SASL authentication
- mTLS between brokers
- IAM authentication (MSK-specific)
- CloudWatch metrics integration

**Pricing:** Per broker-hour + storage per GB-month

---

## 8.10 Amazon QuickSight
**What it is:** Serverless BI and dashboarding. ML-powered insights, embedded analytics.
**Key Features:**
- SPICE — in-memory engine for fast queries
- Reader, Author, Admin pricing tiers
- Embedded dashboards (3rd party embedding)
- ML Insights — forecasting, anomaly detection, Q (NLP queries)
- Row-level security, column-level security

---

# 9. AI & Machine Learning

> AWS AI/ML spans managed ML platforms, foundation model access, and purpose-built AI services for vision, language, and specialized domains.

---

## 9.1 Amazon SageMaker
**What it is:** End-to-end ML platform. Build, train, deploy, and monitor ML models at scale.

**Key Capabilities:**

| Component | Description |
|-----------|-------------|
| SageMaker Studio | Unified IDE for ML (JupyterLab-based) |
| SageMaker Autopilot | AutoML — automatically trains and tunes models |
| Feature Store | Central repository for ML features (online + offline) |
| Pipelines | CI/CD for ML workflows (MLOps) |
| Model Monitor | Detect data/model drift in production |
| JumpStart | Pre-trained models (Hugging Face, Llama, Stability AI) |
| Canvas | No-code ML model building |
| Serverless Inference | Scale to zero between requests |
| Real-time Inference | Persistent endpoints with auto-scaling |
| Batch Transform | Large-scale offline inference |
| Clarify | Bias detection, explainability |
| Experiments | Track training runs, parameters, metrics |
| Ground Truth | Data labeling (human + automated) |
| HyperParameter Tuning (HPO) | Bayesian/random/grid search |
| SageMaker Training Compiler | GPU optimization, faster training |

```bash
# Create training job
aws sagemaker create-training-job \
  --training-job-name my-model-v1 \
  --algorithm-specification \
    TrainingImage=382416733822.dkr.ecr.us-east-1.amazonaws.com/xgboost:1.7-1,\
    TrainingInputMode=File \
  --role-arn arn:aws:iam::123456789012:role/SageMakerRole \
  --input-data-config '[{"ChannelName":"train","DataSource":{"S3DataSource":{"S3Uri":"s3://my-bucket/train/","S3DataType":"S3Prefix"}}}]' \
  --output-data-config S3OutputPath=s3://my-bucket/model-output/ \
  --resource-config InstanceType=ml.m5.xlarge,InstanceCount=1,VolumeSizeInGB=50

# Deploy model
aws sagemaker create-endpoint-config \
  --endpoint-config-name my-model-config \
  --production-variants \
    VariantName=primary,ModelName=my-model,InstanceType=ml.m5.large,InitialInstanceCount=1,InitialVariantWeight=1

aws sagemaker create-endpoint \
  --endpoint-name my-endpoint \
  --endpoint-config-name my-model-config

# Invoke endpoint
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name my-endpoint \
  --content-type text/csv \
  --body "1.5,2.3,0.8" \
  output.json
```
**Pricing:** Per instance-hour for training and inference; per GB for data storage; serverless inference per invocation.

---

## 9.2 Amazon Bedrock
**What it is:** Fully managed service for accessing and customizing foundation models (FMs) from AWS and leading AI companies.

**Available Models (examples):**
- Amazon Titan (text, embeddings, image)
- Anthropic Claude 3 (Haiku, Sonnet, Opus)
- Meta Llama 3
- Mistral
- Stability AI (Stable Diffusion)
- Cohere Command, Embed
- AI21 Jurassic

**Key Capabilities:**
- **Knowledge Bases** — RAG (Retrieval Augmented Generation) with S3 + vector store (OpenSearch Serverless, Pinecone, Aurora, Redis)
- **Agents** — orchestrate multi-step tasks with function calling (Action Groups via Lambda)
- **Guardrails** — content filtering, PII redaction, grounding checks, topic denial
- **Fine-tuning** — customize models with your data (continued pre-training + instruction fine-tuning)
- **Model Evaluation** — automatic and human evaluation of model outputs
- **Provisioned Throughput** — committed throughput for consistent latency

```bash
# Invoke model (Bedrock)
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0 \
  --content-type application/json \
  --accept application/json \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":1024,"messages":[{"role":"user","content":"Explain VPC peering"}]}' \
  output.json

# List available models
aws bedrock list-foundation-models \
  --query 'modelSummaries[*].[modelId,modelName]' --output table

# Start knowledge base ingestion
aws bedrock-agent start-ingestion-job \
  --knowledge-base-id kb-12345 \
  --data-source-id ds-12345
```
**Pricing:** Per 1K input/output tokens (varies by model); Knowledge Bases: per chunk stored + per query.

---

## 9.3 Amazon Rekognition
**What it is:** Image and video analysis. Detect objects, scenes, text, faces, celebrities, content moderation.
**Key Features:**
- Face detection, comparison, search (collection-based)
- Object and scene detection
- Text detection (OCR)
- Content moderation labels
- Video: stored (S3) and streaming (Kinesis Video Streams)
- Custom Labels — train custom object detection models

```bash
# Detect labels
aws rekognition detect-labels \
  --image S3Object='{Bucket=my-bucket,Name=photo.jpg}' \
  --min-confidence 70

# Detect faces
aws rekognition detect-faces \
  --image S3Object='{Bucket=my-bucket,Name=photo.jpg}' \
  --attributes ALL

# Compare faces
aws rekognition compare-faces \
  --source-image S3Object='{Bucket=my-bucket,Name=source.jpg}' \
  --target-image S3Object='{Bucket=my-bucket,Name=target.jpg}'
```
**Pricing:** Per image/minute of video processed.

---

## 9.4 Amazon Textract
**What it is:** Extract text, tables, forms, and signatures from documents. Beyond OCR — understands document structure.
**Key Features:**
- Detect Document Text (raw OCR)
- Analyze Document (forms with key-value pairs, tables)
- Analyze ID (passports, licenses)
- Analyze Expense (receipts, invoices)
- Async API for multi-page PDFs

---

## 9.5 Amazon Comprehend
**What it is:** NLP service — extract insights from text: entities, key phrases, sentiment, language, PII, topics.
**Comprehend Medical:** Extracts medical entities (ICD-10, RxNorm, SNOMED CT) from clinical text.

---

## 9.6 Amazon Transcribe
**What it is:** Speech-to-text. Supports 100+ languages, custom vocabulary, speaker diarization, PII redaction, custom language models.
**Transcribe Medical:** HIPAA-eligible, medical vocabulary.

---

## 9.7 Amazon Polly
**What it is:** Text-to-speech. 60+ voices, 29 languages. Neural TTS voices for natural sound. SSML support.

---

## 9.8 Amazon Lex v2
**What it is:** Build conversational chatbots (voice + text). Uses same tech as Alexa. Supports multi-turn dialogs, slot filling, fulfillment via Lambda.

---

## 9.9 Amazon Kendra
**What it is:** ML-powered enterprise search. Natural language queries across unstructured data (S3, SharePoint, Confluence, RDS, Salesforce).
**Key Features:** Semantic search, incremental learning, relevance tuning, filtering.

---

## 9.10 Amazon Personalize
**What it is:** Real-time personalization and recommendation. Upload user-item interaction data → trained recommendation model → real-time API.

---

## 9.11 Amazon Fraud Detector
**What it is:** ML-powered fraud detection. Train on your historical fraud data to detect account fraud, payment fraud, and abuse.

---

## 9.12 Amazon Q Developer (formerly CodeWhisperer)
**What it is:** AI coding assistant. Real-time code suggestions, security scanning, CLI assistance, code transformation (Java 8/11 → 17).
**Pricing:** Free tier for individuals; Pro tier $19/user/month.

---

# 10. Application Integration

> AWS integration services connect distributed systems and enable event-driven, loosely coupled architectures.

---

## 10.1 Amazon SQS
**What it is:** Fully managed message queue. Decouple producers and consumers. At-least-once delivery.

**Queue Types:**

| Feature | Standard | FIFO |
|---------|---------|------|
| Throughput | Nearly unlimited | 300 TPS (3000 with batching) |
| Ordering | Best-effort | Strict FIFO per MessageGroupId |
| Delivery | At-least-once | Exactly-once |
| Deduplication | No | Yes (5-min window) |

**Key Features:**
- Visibility Timeout — hide message during processing (default 30s, max 12h)
- Dead Letter Queue (DLQ) — after N failed receives
- Long Polling — reduce empty receives, cost (up to 20s wait)
- Message size — up to 256 KB (S3 pointer pattern for larger)
- SSE encryption with KMS
- VPC endpoint support

```bash
# Create queue
aws sqs create-queue \
  --queue-name my-standard-queue \
  --attributes \
    VisibilityTimeout=60,\
    MessageRetentionPeriod=86400,\
    ReceiveMessageWaitTimeSeconds=20

# Create FIFO queue
aws sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

# Send message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --message-body '{"event":"order_placed","orderId":"123"}'

# Receive messages
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --max-number-of-messages 10 \
  --wait-time-seconds 20

# Delete message
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --receipt-handle AQEB...

# Set DLQ
aws sqs set-queue-attributes \
  --queue-url https://... \
  --attributes RedrivePolicy='{"deadLetterTargetArn":"arn:aws:sqs:...:my-dlq","maxReceiveCount":"5"}'
```
**Pricing:** $0.40/million requests (first 1M free/month)

---

## 10.2 Amazon SNS
**What it is:** Fully managed pub/sub messaging and mobile notification. Fan-out messages to multiple endpoints.
**Subscription Protocols:** SQS, Lambda, HTTP/HTTPS, Email, SMS, Mobile Push (APNs, FCM), Kinesis Firehose
**Key Features:**
- Topic-based (Standard and FIFO)
- Message filtering — subscription filter policies
- Message archival to SQS DLQ
- SNS FIFO — exactly-once, ordered (with SQS FIFO subscribers only)
- Large message support via S3 (SNS Extended Client Library)
- Server-side encryption

```bash
# Create topic
aws sns create-topic --name my-topic

# Subscribe SQS
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:my-queue

# Publish message
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --message '{"event":"user_signup","userId":"u123"}' \
  --subject "New User Signup" \
  --message-attributes '{"env":{"DataType":"String","StringValue":"prod"}}'

# Add subscription filter
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:...:sub/12345 \
  --attribute-name FilterPolicy \
  --attribute-value '{"env":["prod"],"eventType":["ORDER","PAYMENT"]}'
```
**Pricing:** $0.50/million publish API requests; $0.40/million SQS deliveries

---

## 10.3 Amazon EventBridge
**What it is:** Serverless event bus. Route events from AWS services, SaaS apps, and custom apps to targets.
**Components:**
- **Default Event Bus** — AWS service events
- **Custom Event Bus** — your application events
- **Partner Event Bus** — SaaS events (Salesforce, Zendesk, Datadog)
- **EventBridge Pipes** — point-to-point event streaming with filtering + enrichment
- **EventBridge Scheduler** — cron/rate-based scheduling (replaces CloudWatch Events cron)

```bash
# Create custom event bus
aws events create-event-bus --name my-app-events

# Put event
aws events put-events \
  --entries '[{
    "Source":"myapp.orders",
    "DetailType":"OrderPlaced",
    "Detail":"{\"orderId\":\"123\",\"amount\":99.99}",
    "EventBusName":"my-app-events"
  }]'

# Create rule
aws events put-rule \
  --name ProcessOrders \
  --event-bus-name my-app-events \
  --event-pattern '{"source":["myapp.orders"],"detail-type":["OrderPlaced"]}' \
  --state ENABLED

# Add Lambda target
aws events put-targets \
  --rule ProcessOrders \
  --event-bus-name my-app-events \
  --targets '[{"Id":"process-lambda","Arn":"arn:aws:lambda:...:function:ProcessOrder"}]'

# Create scheduled rule (every 5 minutes)
aws events put-rule \
  --name every-5-min \
  --schedule-expression "rate(5 minutes)" \
  --state ENABLED
```
**Pricing:** $1/million custom events; $1/million cross-account events; Scheduler: $1/million invocations (first 14M free)

---

## 10.4 AWS Step Functions
**What it is:** Serverless workflow orchestration. Define state machines as JSON/YAML (Amazon States Language). Coordinate distributed components.
**Workflow Types:**

| Type | Max Duration | Pricing | Use Case |
|------|-------------|---------|---------|
| Standard | 1 year | $0.025/1K transitions | Long-running, auditable |
| Express | 5 minutes | $1/M executions + duration | High-volume, event-driven |

**State Types:** Task, Choice, Wait, Parallel, Map, Pass, Succeed, Fail

**Key Features:**
- SDK Integrations — directly call 200+ AWS services (no Lambda needed)
- Activity tasks — external workers pulling work items
- Error handling — Retry and Catch per state
- Step Functions Workflows (Workflow Studio — drag-and-drop)
- Express Workflows emit to CloudWatch Logs/X-Ray

```bash
# Create state machine
aws stepfunctions create-state-machine \
  --name OrderProcessing \
  --definition file://state-machine.json \
  --role-arn arn:aws:iam::123456789012:role/StepFunctionsRole \
  --type STANDARD

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessing \
  --name exec-1234 \
  --input '{"orderId":"123","customerId":"cust-456"}'

# Describe execution
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:...:execution:OrderProcessing:exec-1234
```

---

## 10.5 Amazon MQ
**What it is:** Managed message broker for ActiveMQ and RabbitMQ. For migrating existing JMS/AMQP/STOMP apps without rewriting.
**vs SQS/SNS:** Use MQ when you need standard protocol support (JMS, AMQP, STOMP, MQTT) for lift-and-shift migrations.

---

## 10.6 Amazon AppFlow
**What it is:** No-code integration service to transfer data between SaaS apps (Salesforce, Slack, ServiceNow, Marketo) and AWS (S3, Redshift, Snowflake).

---

## 10.7 AWS AppSync
**What it is:** Managed GraphQL and Pub/Sub API service. Real-time data sync and offline sync for web/mobile apps.
**Key Features:**
- Data sources: DynamoDB, Lambda, RDS, OpenSearch, HTTP
- Real-time subscriptions via WebSocket
- Offline data sync (Amplify DataStore)
- JavaScript and VTL resolvers

---

## 10.8 Amazon MWAA (Managed Workflows for Apache Airflow)
**What it is:** Fully managed Apache Airflow. Deploy DAGs from S3, scale workers automatically.
**Pricing:** Per environment class per hour + per worker-hour.


---

# 11. Business Applications

---

## 11.1 Amazon SES (Simple Email Service)
**What it is:** Cloud email sending service — transactional emails, marketing emails, bulk email.
**Key Features:** DKIM/SPF/DMARC support, Dedicated IPs, reputation dashboard, event publishing (bounces, complaints), Virtual Deliverability Manager
**Pricing:** $0.10/1000 emails sent (within AWS); $0.40/1000 over internet

---

## 11.2 Amazon Connect
**What it is:** Cloud contact center. Omnichannel: voice (PSTN), chat, tasks. Pay-per-use, no licenses.
**Key Features:** Natural language IVR (Lex integration), real-time + historical analytics, agent desktop, outbound campaigns, Contact Lens (ML-powered analytics, sentiment, summarization)
**Pricing:** $0.018/minute inbound voice; per agent/month for add-on features

---

## 11.3 Amazon Pinpoint
**What it is:** Customer engagement and marketing communications. Email, SMS, push notifications, voice, in-app messaging.
**Key Features:** User segments, journeys (automation), A/B testing, event tracking

---

## 11.4 Amazon Chime
**What it is:** Online meetings, video conferencing, chat, phone calls.
**Amazon Chime SDK:** Embed audio/video/messaging into custom apps.

---

## 11.5 AWS Supply Chain
**What it is:** ML-powered supply chain visibility and risk management. Connects to ERP, WMS, TMS systems.

---

# 12. Front-End Web & Mobile

---

## 12.1 AWS Amplify
**What it is:** Full-stack platform for frontend web and mobile developers. Hosting, CI/CD, auth, API, data, storage — all from CLI or console.
**Key Features:**
- Amplify Hosting — static site + SSR hosting (Next.js, Nuxt, Gatsby)
- Amplify Libraries — client-side SDKs for Auth, API, Storage
- Amplify Studio — visual builder for UI components + data models
- CI/CD with Git branch deployments
- Custom domains, SSL, WAF

---

## 12.2 AWS Device Farm
**What it is:** Test mobile and web apps on real physical devices in the cloud (iOS, Android, browsers).
**Test frameworks:** Appium, Calabash, Espresso, XCTest, Selenium

---

## 12.3 Amazon Location Service
**What it is:** Add location functionality to applications: maps, routing, geocoding, geofencing, asset tracking.
**Data providers:** Esri, HERE, Grab
**Pricing:** Per request (maps tiles, geocodes, routes)

---

# 13. IoT

> AWS IoT services manage device connectivity, messaging, processing, and management at scale from edge to cloud.

---

## 13.1 AWS IoT Core
**What it is:** Managed MQTT/HTTP/WebSocket broker for IoT devices. Connect billions of devices securely.
**Key Features:**
- X.509 certificate-based authentication
- IoT Policies — control device permissions
- Rules Engine — process and route device data to 20+ AWS services
- Device Shadow — virtual representation of device state (desired vs reported)
- Fleet Indexing — query all device states with SQL
- MQTT over WebSocket

```bash
# Create IoT thing
aws iot create-thing --thing-name my-sensor-001

# Create policy
aws iot create-policy \
  --policy-name SensorPolicy \
  --policy-document file://iot-policy.json

# Create certificate
aws iot create-keys-and-certificate --set-as-active \
  --certificate-pem-outfile cert.pem \
  --public-key-outfile public.key \
  --private-key-outfile private.key

# Get IoT endpoint
aws iot describe-endpoint --endpoint-type iot:Data-ATS
```
**Pricing:** $0.08–$1.00/million messages depending on message size

---

## 13.2 AWS IoT Greengrass
**What it is:** Extend AWS to edge devices. Run Lambda, Docker containers, and ML inference locally on IoT devices without cloud connectivity.
**Key Features:**
- Greengrass Core v2 (open source)
- Component-based architecture
- Pre-built components (ML inference, Kinesis video stream, Modbus, OPC-UA)
- Local MQTT broker and shadow sync

---

## 13.3 AWS IoT SiteWise
**What it is:** Collect, organize, and analyze industrial equipment data at scale.
**Key Features:**
- Modbus TCP, OPC-UA protocol support (via Greengrass)
- Asset models and hierarchies
- Real-time and historical metrics
- SiteWise Monitor — dashboards for operators

---

## 13.4 AWS IoT Device Defender
**What it is:** Security service for IoT devices. Audit configurations, detect anomalies in device behavior.
**Capabilities:** Audit (device certificates, policies), Detect (statistical anomaly detection), Mitigation Actions

---

## 13.5 AWS IoT Analytics
**What it is:** Managed analytics for IoT data. Cleanse, process, enrich, store, and analyze device data.

---

## 13.6 AWS IoT TwinMaker
**What it is:** Build digital twins of physical environments. Connect data from IoT sensors, video, and business systems into a unified 3D knowledge graph.

---

## 13.7 AWS IoT Events
**What it is:** Detect and respond to events from IoT sensors and applications. State machine-based event detection.

---

## 13.8 AWS FreeRTOS
**What it is:** Open-source RTOS for microcontrollers with AWS IoT connectivity libraries.

---

# 14. Containers

> AWS container services span the full lifecycle from image storage to orchestration and service mesh.

---

## 14.1 Amazon ECR (Elastic Container Registry)
**What it is:** Managed Docker and OCI container image registry.
**Key Features:**
- Private repositories per account/region
- ECR Public Gallery — public image hosting
- Image Vulnerability Scanning (Enhanced via Inspector v2)
- Image immutability — prevent tag overwrites
- Lifecycle policies — auto-delete untagged or old images
- Cross-account access via repository policy
- Replication — cross-region and cross-account
- Pull-through cache — cache images from Docker Hub, Quay, GitHub

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# Build and push
docker build -t my-app .
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3

# Create lifecycle policy
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://lifecycle-policy.json
```
**Pricing:** $0.10/GB-month storage; data transfer standard rates

---

*(ECS, EKS, Fargate, App Mesh, Cloud Map covered in sections 1.3, 1.4, 1.5, 4.13, 4.14)*

## 14.2 AWS Copilot CLI
**What it is:** CLI tool to build, release, and operate containerized applications on ECS and AppRunner.
**Creates:** VPC, ECS Cluster, ALB, ECR repo, CodePipeline from simple `copilot init`.

## 14.3 EKS Anywhere / ECS Anywhere
- **EKS Anywhere:** Run EKS on-prem using VMware vSphere, bare metal, Nutanix, or Snow devices.
- **ECS Anywhere:** Register any server (on-prem, other cloud) as an ECS external instance. $0.01025/vCPU-hour.

---

# 15. Migration & Transfer

---

## 15.1 AWS Migration Hub
**What it is:** Central hub to track migrations across AWS and partner tools. Aggregates discovery and migration status.

## 15.2 AWS Application Discovery Service
**What it is:** Discover on-prem servers, processes, and network connections. Two modes:
- Agentless (via VMware vCenter connector)
- Agent-based (AWS Discovery Agent on each server)

## 15.3 AWS Application Migration Service (MGN)
**What it is:** Rehost (lift-and-shift) any application to AWS. Continuously replicates source servers to AWS using block-level replication. Replaces CloudEndure.
**Key Steps:** Install agent → continuous replication → launch test instance → cutover → terminate source

## 15.4 AWS Database Migration Service (DMS)
**What it is:** Migrate databases to AWS with minimal downtime. Supports homogeneous (Oracle→Oracle) and heterogeneous (Oracle→Aurora).
**Key Features:**
- Replication instance (t3.medium to r5.24xlarge)
- Endpoints (source + target) with full and CDC (change data capture)
- Schema Conversion Tool (SCT) for heterogeneous migrations
- Serverless DMS (auto-scales replication capacity)
- Data validation option

```bash
# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.r5.large \
  --allocated-storage 100 \
  --multi-az

# Create endpoints
aws dms create-endpoint \
  --endpoint-identifier source-postgres \
  --endpoint-type source \
  --engine-name postgres \
  --server-name my-pg-server.example.com \
  --port 5432 \
  --database-name mydb \
  --username admin \
  --password 'MyPass'

# Create migration task (full load + CDC)
aws dms create-replication-task \
  --replication-task-identifier full-load-cdc \
  --source-endpoint-arn arn:aws:dms:...:endpoint:source \
  --target-endpoint-arn arn:aws:dms:...:endpoint:target \
  --replication-instance-arn arn:aws:dms:...:rep:instance \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json
```

## 15.5 AWS DataSync
**What it is:** Automate data transfers between on-prem and AWS storage, or between AWS storage services.
**Supports:** NFS, SMB, HDFS, object storage (S3-compatible) on-prem; S3, EFS, FSx on AWS.
**Features:** Scheduling, bandwidth throttling, data validation, encryption in-transit, 10 Gbps transfers.

## 15.6 VM Import/Export
**What it is:** Import VM images from VMware, Hyper-V, or Azure as EC2 AMIs. Export EC2 instances back to VMs.

## 15.7 AWS Mainframe Modernization
**What it is:** Managed runtime to modernize COBOL/PL/I mainframe applications to AWS via refactoring or replatforming.

---

# 16. End-User Computing

---

## 16.1 Amazon WorkSpaces
**What it is:** Managed virtual desktops (VDI). Windows or Linux desktops delivered as a service.
**Pricing:** Monthly ($35–$80/user) or hourly bundles
**WorkSpaces Personal vs WorkSpaces Pools:**
- Personal: persistent desktop per user
- Pools: non-persistent, session-based (for task workers)

## 16.2 Amazon AppStream 2.0
**What it is:** Stream desktop applications to browsers. No installation needed on user's device.
**Use Cases:** Distribute CAD, GIS, data science tools; remote labs.

## 16.3 Amazon WorkSpaces Web
**What it is:** Low-cost, fully managed web browser to access internal web applications securely. No data downloaded to endpoints.

## 16.4 NICE DCV
**What it is:** High-performance remote display protocol. Stream 3D, GPU-accelerated applications (CAD, simulation) from EC2 to thin clients.

---

# 17. Media Services

---

## 17.1 AWS Elemental MediaConvert
**What it is:** File-based video transcoding. Convert video files from one format to another at scale.
**Key Features:** 400+ input formats, H.264/H.265/AV1 output, DRM (Widevine, PlayReady, FairPlay), captions, ABR packaging (HLS, DASH, CMAF).

## 17.2 AWS Elemental MediaLive
**What it is:** Live video encoding. Broadcast-grade live stream encoding to channels (HLS, RTMP, MPTS).
**Key Features:** Redundant encoding pipelines, input failover, real-time alerting.

## 17.3 AWS Elemental MediaPackage
**What it is:** Video packaging and origination. Package live and VOD content for multiple devices.
**Key Features:** HLS, DASH, CMAF, MSS output; DVR functionality; CDN integration.

## 17.4 Amazon Interactive Video Service (IVS)
**What it is:** Managed live streaming solution for interactive apps (Twitch-like). Ultra-low latency (<5s).
**Key Features:** RTMPS ingest, HLS playback, timed metadata, chat, real-time streams (WebRTC).

## 17.5 AWS Elemental MediaTailor
**What it is:** Video personalization and monetization. Server-side ad insertion (SSAI) for VOD and live.

## 17.6 Nimble Studio
**What it is:** Managed content production studio in the cloud. Virtual workstations for artists, render farms, shared storage.

---

# 18. Cost Management

> AWS cost tools help control spending through visibility, alerting, automation, and purchasing strategies.

---

## 18.1 AWS Cost Explorer
**What it is:** Visualize and analyze AWS costs and usage over time. Built-in forecasting and rightsizing recommendations.
**Key Features:**
- 13 months of historical data
- Hourly granularity (last 14 days)
- RI/Savings Plans coverage and utilization reports
- API access for programmatic analysis

```bash
# Get cost and usage
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" "UsageQuantity" \
  --group-by Type=DIMENSION,Key=SERVICE
```

## 18.2 AWS Budgets
**What it is:** Set custom cost and usage budgets with alerts.
**Budget Types:** Cost, Usage, Reserved Instance coverage, Savings Plans coverage
**Actions:** Alert via SNS/email; auto-apply SCP or IAM policy when threshold hit.

```bash
# Create cost budget
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

## 18.3 Cost and Usage Report (CUR)
**What it is:** Most detailed cost data. CSV/Parquet to S3; queryable with Athena.
**Data:** Line-item costs, resource IDs, tags, usage type, blended/unblended/amortized costs.

## 18.4 Savings Plans vs Reserved Instances

| Feature | Savings Plans | Reserved Instances |
|---------|-------------|-------------------|
| Flexibility | EC2 (instance family/region) or Compute (any family/region/OS) | Specific instance type + region |
| Commitment | $/hour spend commitment | Specific capacity reservation |
| Discount | Up to 66% | Up to 72% |
| Marketable | No | Yes (RI Marketplace) |

## 18.5 EC2 Spot Instances
- Up to 90% discount for fault-tolerant workloads
- 2-minute interruption notice
- Spot Fleet — mix of Spot + On-Demand across instance types
- Spot Instance Advisor — estimate interruption frequency

## 18.6 AWS Pricing Calculator
**What it is:** Estimate costs for new architectures before building. Group services into estimates.
URL: calculator.aws

## 18.7 AWS Billing Conductor
**What it is:** Customize billing view for internal chargebacks or reselling (MSPs). Create custom pricing rules, pro-forma billing groups.

---

# 19. Blockchain & Quantum

---

## 19.1 Amazon Managed Blockchain
**What it is:** Create and manage blockchain networks using Hyperledger Fabric or Ethereum.
**Key Features:**
- Managed Hyperledger Fabric — private permissioned blockchain for enterprise consortiums
- Ethereum access node — connect to public Ethereum mainnet/testnets without running own node

## 19.2 Amazon QLDB
*(Covered in Section 3.10)*

## 19.3 Amazon Braket
**What it is:** Managed quantum computing service. Experiment with quantum hardware from D-Wave, IonQ, Rigetti, OQC; and quantum circuit simulators.
**Key Features:**
- Braket SDK (Python) — define quantum circuits
- On-demand simulators (SV1, DM1, TN1) — no quantum hardware needed for development
- Hybrid Jobs — combine quantum + classical with SageMaker
**Pricing:** Per task (quantum hardware) or per simulation hour

---

# 20. Satellite & Robotics

---

## 20.1 AWS Ground Station
**What it is:** Fully managed ground station as a service. Communicate with satellites, downlink data to S3/EC2 without building your own antenna network.
**Key Features:** 31+ ground stations globally, contact scheduling, wideband digital output to EC2 (DataDefender EC2 instance type)
**Pricing:** Per contact minute

## 20.2 AWS RoboMaker
**What it is:** Managed simulation and deployment for robotics applications. Built on ROS (Robot Operating System).
**Key Features:** Robot application development environment (Cloud9), Fleet Management, Simulation WorldForge for procedural world generation.


---

# 21. Global Infrastructure Overview

---

## Regions
- Isolated geographic areas, each with 2+ AZs
- ~34 regions globally (2024); more announced regularly
- Resources are region-scoped unless otherwise noted (IAM, Route 53, CloudFront are global)
- Choose regions based on: latency to users, compliance/data residency, service availability, pricing

## Availability Zones (AZs)
- One or more discrete data centers with redundant power, networking, cooling
- ~3 AZs per region (range: 2–6)
- Connected by high-bandwidth, low-latency fiber
- AZ ID (e.g., `use1-az1`) — consistent cross-account identifier (your `us-east-1a` ≠ another account's `us-east-1a`)
- ~108 AZs globally

## Local Zones
- Extension of AWS region placed closer to population centers
- Run latency-sensitive apps (<10ms) closer to end users
- Subset of services available (EC2, EBS, ECS, RDS, ELB, etc.)
- ~30+ Local Zones in cities like Los Angeles, Boston, Dallas, Mumbai

## Wavelength Zones
- AWS infrastructure embedded within 5G telco networks
- Ultra-low latency (<10ms) to mobile devices
- Supported by Verizon, KDDI, Vodafone, SK Telecom

## Edge Locations (CloudFront / Route 53 / Global Accelerator)
- ~450+ Points of Presence (PoPs) globally
- Used by CloudFront CDN, Route 53 DNS, Global Accelerator, Lambda@Edge
- Not full AZs — no EC2, RDS, etc.
- Regional Edge Caches — ~13 larger PoPs between origin and edge locations

## AWS Backbone
- AWS maintains a private global fiber network connecting all regions and edge locations
- Transit Gateway SiteLink and Global Accelerator use this backbone for optimized routing

---

# 22. AWS Shared Responsibility Model

---

## Core Principle
> AWS is responsible for security **OF** the cloud. Customers are responsible for security **IN** the cloud.

---

## Breakdown by Compute Type

| Responsibility | EC2 | ECS/EKS (EC2) | Fargate | Lambda |
|---------------|-----|--------------|---------|--------|
| Physical infrastructure | AWS | AWS | AWS | AWS |
| Hypervisor | AWS | AWS | AWS | AWS |
| Container runtime | Customer | Customer | AWS | AWS |
| OS patching | **Customer** | **Customer** | AWS | AWS |
| Network configuration | Customer | Customer | Customer | Customer |
| Application code | Customer | Customer | Customer | Customer |
| Data encryption | Customer | Customer | Customer | Customer |
| Identity (IAM) | Customer | Customer | Customer | Customer |
| Security groups | Customer | Customer | Customer | Customer |
| Guest OS (EC2) | **Customer** | **Customer** | N/A | N/A |

## AWS-Managed Responsibilities (Always AWS):
- Physical security of data centers
- Hardware maintenance and replacement
- Global network infrastructure
- Hypervisor/virtualization layer
- Managed service software (RDS engine patches, DynamoDB infrastructure)
- Availability of the service (SLA)

## Customer Responsibilities (Always Customer):
- Data classification and protection
- IAM users, roles, policies
- Application-level security
- Network ACLs and Security Groups
- OS and application patching (for unmanaged services)
- Encryption of data at rest and in transit (customer enables; AWS provides tools)
- MFA and credential management

## Managed Services (more AWS responsibility):
For **RDS**: AWS manages OS, database patching, replication, backups.
Customer manages: schema, queries, connection pooling, parameter groups, who can access.

For **S3**: AWS manages infrastructure, redundancy, durability.
Customer manages: bucket policies, access control, encryption, versioning, lifecycle.

---

# 23. AWS Well-Architected Framework

---

## The 6 Pillars

### 1. Operational Excellence
**Principles:**
- Perform operations as code (IaC, runbooks as Lambda)
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure (game days, chaos engineering)
- Learn from all operational events and failures

**Key Services:** CloudFormation, CDK, Systems Manager, CloudWatch, X-Ray, FIS, CodePipeline, Config

---

### 2. Security
**Principles:**
- Implement a strong identity foundation (least privilege, no long-lived credentials)
- Enable traceability
- Apply security at all layers
- Automate security best practices
- Protect data in transit and at rest
- Keep people away from data
- Prepare for security events

**Key Services:** IAM, Organizations, SCP, KMS, CloudTrail, GuardDuty, Security Hub, WAF, Shield, Macie, VPC, Inspector

---

### 3. Reliability
**Principles:**
- Automatically recover from failure
- Test recovery procedures
- Scale horizontally to increase aggregate workload availability
- Stop guessing capacity (use Auto Scaling)
- Manage change in automation

**Key Services:** Route 53, ELB, Auto Scaling, Multi-AZ RDS, Aurora, S3, CloudWatch Alarms, Backup, Resilience Hub

---

### 4. Performance Efficiency
**Principles:**
- Democratize advanced technologies (use managed services)
- Go global in minutes
- Use serverless architectures
- Experiment more often
- Consider mechanical sympathy (match resource type to workload)

**Key Services:** CloudFront, Lambda, Fargate, Aurora Serverless, DynamoDB, ElastiCache, Kinesis, SageMaker, Graviton EC2, Compute Optimizer

---

### 5. Cost Optimization
**Principles:**
- Implement cloud financial management
- Adopt a consumption model (pay for what you use)
- Measure overall efficiency
- Stop spending money on undifferentiated heavy lifting
- Analyze and attribute expenditure

**Key Services:** Savings Plans, Spot Instances, Reserved Instances, Auto Scaling, S3 Intelligent-Tiering, Compute Optimizer, Cost Explorer, Budgets, Lambda (pay per use)

---

### 6. Sustainability
**Principles:**
- Understand your impact
- Establish sustainability goals
- Maximize utilization
- Anticipate and adopt more efficient hardware and software
- Use managed services (AWS optimizes shared infrastructure)
- Reduce downstream impact (minimize data transmitted to devices)

**Key Services:** Graviton (ARM) instances, Spot Instances, Auto Scaling, serverless, S3 Intelligent-Tiering, managed services

---

# 24. IAM Policy Anatomy

---

## Full JSON Policy Structure

```json
{
  "Version": "2012-10-17",           // Policy language version — always use this
  "Id": "MyPolicyId",                // Optional identifier
  "Statement": [
    {
      "Sid": "AllowS3ReadProd",      // Statement ID — optional, must be unique in policy
      "Effect": "Allow",             // Allow | Deny
      "Principal": {                 // WHO — only in resource-based policies
        "AWS": "arn:aws:iam::123456789012:role/MyRole",
        "Service": "lambda.amazonaws.com"
      },
      "Action": [                    // WHAT — list of API actions
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [                  // WHERE — ARNs of resources
        "arn:aws:s3:::prod-bucket",
        "arn:aws:s3:::prod-bucket/*"
      ],
      "Condition": {                 // WHEN — additional conditions
        "StringEquals": {
          "s3:prefix": ["data/"],
          "aws:RequestedRegion": "us-east-1"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "IpAddress": {
          "aws:SourceIp": ["10.0.0.0/8"]
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        }
      }
    }
  ]
}
```

## Policy Types Comparison

| Policy Type | Attached To | Grants Access | Cross-Account | Notes |
|-------------|------------|---------------|---------------|-------|
| Identity-based | User/Group/Role | Yes | Only with resource-based | Most common |
| Resource-based | Resource (S3, KMS, SQS…) | Yes | Yes (if principal from other account) | Only select services support it |
| SCP | Org OU/Account | No (restricts only) | N/A | Can't grant, only limit |
| Permission Boundary | User or Role | No (restricts max) | N/A | Effective = identity ∩ boundary |
| Session Policy | AssumeRole call | No (restricts session) | N/A | Can't exceed role's policies |
| ACL | S3 buckets/objects | Yes | Yes | Legacy; prefer bucket policies |

## Policy Evaluation Logic (Detailed)

```
Request comes in
│
├─ Explicit DENY in ANY policy? → DENY (hard stop)
│
├─ SCP present?
│   └─ SCP allows action? → continue
│   └─ No SCP or SCP denies → DENY
│
├─ Resource-based policy? 
│   └─ Same account: allows? → continue (resource-based alone can allow)
│   └─ Cross-account: needs BOTH resource-based AND identity-based allows
│
├─ Identity-based policy?
│   └─ Allows? → continue
│   └─ No allow → DENY (implicit)
│
├─ Permission Boundary present?
│   └─ Allows? → continue
│   └─ Denies → DENY
│
└─ Session Policy present?
    └─ Allows? → ALLOW
    └─ Denies → DENY
```

## Common Condition Keys

| Key | Description | Example |
|-----|-------------|---------|
| `aws:RequestedRegion` | Restrict to specific regions | Deny non-us-east-1 |
| `aws:SourceIp` | Source IP CIDR | Corporate IP only |
| `aws:SourceVpc` | Source VPC | VPC endpoint only |
| `aws:SourceVpce` | Specific VPC endpoint | Specific endpoint |
| `aws:MultiFactorAuthPresent` | MFA required | Sensitive actions |
| `aws:PrincipalArn` | Specific principal | Cross-service |
| `aws:CalledVia` | Service calling on behalf | e.g., CloudFormation |
| `s3:prefix` | S3 key prefix | Folder-level access |
| `ec2:Region` | EC2 region | Region lock |

---

# 25. AWS Networking Deep Dive

---

## VPC CIDR Planning and Subnet Design

### RFC 1918 Private Ranges
| Block | Range | Common VPC Use |
|-------|-------|----------------|
| 10.0.0.0/8 | 10.0.0.0 – 10.255.255.255 | Large enterprise (use /16 or /8 per region) |
| 172.16.0.0/12 | 172.16.0.0 – 172.31.255.255 | Medium environments |
| 192.168.0.0/16 | 192.168.0.0 – 192.168.255.255 | Small VPCs, lab |

### Recommended Subnet Architecture

```
VPC: 10.0.0.0/16 (65,534 IPs)
│
├── Public Subnet AZ-a:  10.0.0.0/24   (251 usable — 5 reserved by AWS)
├── Public Subnet AZ-b:  10.0.1.0/24
├── Public Subnet AZ-c:  10.0.2.0/24
│
├── Private Subnet AZ-a: 10.0.10.0/23  (509 usable)
├── Private Subnet AZ-b: 10.0.12.0/23
├── Private Subnet AZ-c: 10.0.14.0/23
│
└── Isolated Subnet AZ-a: 10.0.20.0/24  (DBs, no internet route)
    Isolated Subnet AZ-b: 10.0.21.0/24
    Isolated Subnet AZ-c: 10.0.22.0/24
```

**AWS Reserved IPs in Every Subnet (first 4 + last 1):**
- .0 — network address
- .1 — VPC router
- .2 — DNS resolver
- .3 — future use
- .255 — broadcast (not used, but reserved)

### Subnet Tier Definitions

| Tier | Route to IGW | Route to NAT | Use For |
|------|-------------|-------------|---------|
| Public | Yes | No | ALBs, NAT Gateways, Bastion hosts |
| Private | No | Yes | App servers, ECS/EKS tasks |
| Isolated/Data | No | No | RDS, ElastiCache (egress via VPC endpoints only) |

---

## NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|------------|-------------|
| Managed by | AWS | You |
| Availability | Highly available in AZ | Single instance; SPOF |
| Bandwidth | Up to 100 Gbps auto-scaling | Limited by EC2 type |
| Cost | $0.045/hr + $0.045/GB | EC2 + data transfer |
| SG | Cannot attach | Can attach |
| Port forwarding | No | Yes (iptables) |
| Bastion host | No | Yes |
| Recommendation | **Always use** | Legacy only |

---

## VPC Peering vs Transit Gateway vs PrivateLink

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|------------|----------------|-------------|
| Topology | 1:1 (mesh) | Hub-and-spoke | Service consumer-provider |
| Transitive routing | No | Yes | N/A |
| Max connections | Limited (full mesh scales n²) | 5,000 VPCs | Per endpoint |
| Cross-region | Yes | Yes (inter-region) | Yes (some services) |
| On-prem connectivity | No | Yes (VPN/DX) | No |
| Cost | Free (data transfer) | $0.05/hr/attach + $0.02/GB | $0.01/hr/AZ + $0.01/GB |
| Use case | Simple peering between 2 VPCs | Network hub, multiple VPCs | Expose service privately |

---

# 26. AWS Service Limits — Top 20 Most Hit Quotas

| # | Service | Limit | Default | Notes |
|---|---------|-------|---------|-------|
| 1 | EC2 vCPUs (On-Demand) | Running vCPUs | 32–1152 (by family) | Per account per region |
| 2 | EC2 Elastic IPs | 5 | 5 per region | Easy to increase |
| 3 | VPCs per region | 5 | 5 | Soft limit |
| 4 | Subnets per VPC | 200 | 200 | |
| 5 | Security groups per VPC | 2,500 | 2,500 | |
| 6 | Rules per security group | 60 inbound + 60 outbound | 60 | |
| 7 | Lambda concurrent executions | 1,000 | 1,000 per region | Shared across all functions |
| 8 | Lambda function deployment size | 250 MB (unzipped) | 250 MB | 50 MB zipped direct upload |
| 9 | S3 buckets per account | 100 | 100 | Increase to 1,000 |
| 10 | S3 PUT rate per prefix | 3,500 req/s | 3,500 | 5,500 GET per prefix |
| 11 | IAM roles per account | 1,000 | 1,000 | |
| 12 | IAM policies per user/role | 10 managed | 10 | |
| 13 | CloudFormation stacks | 2,000 | 2,000 per region | |
| 14 | RDS instances per region | 40 | 40 | |
| 15 | DynamoDB tables per region | 2,500 | 2,500 | |
| 16 | API Gateway APIs per region | 600 (REST) | 600 | |
| 17 | ECS tasks per service | 2,000 | 2,000 | |
| 18 | SNS topics per account | 100,000 | 100,000 | |
| 19 | Kinesis shards per stream | 500 | 500 per region | |
| 20 | SQS messages in-flight | 120,000 (standard) | 120,000 | 20,000 FIFO |

**How to increase limits:**
```bash
# View quota
aws service-quotas get-service-quota \
  --service-code ec2 \
  --quota-code L-1216C47A

# Request increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 100

# List available quotas for a service
aws service-quotas list-service-quotas --service-code lambda
```

---

# 27. CLI Quick Reference

---

## EC2
```bash
# Find latest Amazon Linux 2023 AMI
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
  "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId'

# List all running instances with name
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,IP:PrivateIpAddress,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table

# Get instance profile
aws ec2 describe-iam-instance-profile-associations

# Describe security groups
aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=vpc-12345 \
  --query 'SecurityGroups[*].{ID:GroupId,Name:GroupName}'
```

## S3
```bash
# Total size of bucket
aws s3 ls s3://my-bucket --recursive --human-readable --summarize | tail -2

# Copy between buckets (cross-region)
aws s3 cp s3://source-bucket/file.csv s3://dest-bucket/file.csv \
  --source-region us-east-1 --region eu-west-1

# Enable S3 Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket my-bucket \
  --accelerate-configuration Status=Enabled

# List bucket policy
aws s3api get-bucket-policy --bucket my-bucket --output text | python3 -m json.tool

# Get bucket location
aws s3api get-bucket-location --bucket my-bucket
```

## IAM
```bash
# Get current identity
aws sts get-caller-identity

# List all IAM users and last activity
aws iam generate-credential-report && sleep 5
aws iam get-credential-report --query Content --output text | base64 -d | column -t -s','

# Find roles with specific policy
aws iam list-entities-for-policy \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Get effective permissions (policy simulation)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
  --action-names s3:DeleteObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# List all inline and managed policies for a role
aws iam list-role-policies --role-name MyRole
aws iam list-attached-role-policies --role-name MyRole
```

## Lambda
```bash
# Get function config
aws lambda get-function-configuration --function-name my-func

# Update env vars
aws lambda update-function-configuration \
  --function-name my-func \
  --environment Variables='{DB_HOST=rds.example.com,ENV=prod}'

# Add Lambda permission (allow API Gateway to invoke)
aws lambda add-permission \
  --function-name my-func \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:api-id/*/POST/users"

# View recent CloudWatch logs
aws logs tail /aws/lambda/my-func --follow --format short
```

## ECS
```bash
# Force redeploy (pull latest image)
aws ecs update-service \
  --cluster prod \
  --service my-service \
  --force-new-deployment

# List running tasks with container IPs
aws ecs list-tasks --cluster prod --service-name my-service \
  | jq -r '.taskArns[]' \
  | xargs -I{} aws ecs describe-tasks --cluster prod --tasks {} \
  --query 'tasks[*].{Task:taskArn,IP:containers[0].networkInterfaces[0].privateIpv4Address}'

# Get service events
aws ecs describe-services --cluster prod --services my-service \
  --query 'services[0].events[:10]' --output table
```

## RDS
```bash
# Get all RDS instances across regions
for region in $(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text); do
  echo "=== $region ==="
  aws rds describe-db-instances \
    --region $region \
    --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,Engine,DBInstanceStatus]' \
    --output table 2>/dev/null
done

# Check parameter group settings
aws rds describe-db-parameters \
  --db-parameter-group-name default.mysql8.0 \
  --query 'Parameters[?ParameterName==`max_connections`]'

# Start RDS instance
aws rds start-db-instance --db-instance-identifier my-dev-db

# Stop RDS instance  
aws rds stop-db-instance --db-instance-identifier my-dev-db
```

## CloudFormation
```bash
# List all stacks and status
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[*].[StackName,StackStatus,LastUpdatedTime]' \
  --output table

# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].Outputs'

# Export values from stacks
aws cloudformation list-exports \
  --query 'Exports[*].[Name,Value]' \
  --output table

# Delete stack (with wait)
aws cloudformation delete-stack --stack-name my-stack
aws cloudformation wait stack-delete-complete --stack-name my-stack
```

## General Productivity Tips
```bash
# Set default region
export AWS_DEFAULT_REGION=us-east-1

# Use profiles
export AWS_PROFILE=prod-account

# Assume role and set env vars
eval $(aws sts assume-role \
  --role-arn arn:aws:iam::111111111111:role/DeployRole \
  --role-session-name deploy \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text | awk '{print "export AWS_ACCESS_KEY_ID="$1"\nexport AWS_SECRET_ACCESS_KEY="$2"\nexport AWS_SESSION_TOKEN="$3}')

# Query with JMESPath and format as table
aws ec2 describe-vpcs \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table

# Get resource costs by tag (Cost Explorer)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=Project
```

---

# 28. Cost Optimization Cheat Cards

---

## Top 10 Cost Optimization Patterns

### 1. Right-Size Compute
**Pattern:** Use AWS Compute Optimizer to identify over-provisioned EC2, Lambda, ECS, and EBS.
**Services:** Compute Optimizer, CloudWatch metrics, Trusted Advisor
**Savings potential:** 20–40%

### 2. Spot Instances for Fault-Tolerant Workloads
**Pattern:** Use Spot for stateless web tiers, batch, CI/CD, ML training, dev/test.
**Services:** EC2 Spot Fleet, ASG with mixed instances + Spot, EMR task nodes, ECS Spot capacity providers, Karpenter Spot node pools
**Savings potential:** Up to 90% vs On-Demand

### 3. Savings Plans and Reserved Instances for Steady-State
**Pattern:** Commit to minimum baseline compute (Compute Savings Plans for flexibility). Use Reserved Instances for specific instance types.
**Services:** Savings Plans (Compute, EC2, SageMaker), Reserved Instances, Reserved DB Instances
**Savings potential:** 40–66%

### 4. S3 Lifecycle and Intelligent-Tiering
**Pattern:** Auto-transition objects to cheaper storage tiers. Enable Intelligent-Tiering for unknown access patterns.
**Services:** S3 Lifecycle Policies, S3 Intelligent-Tiering, S3 Glacier Deep Archive
**Savings potential:** 60–95% on cold data

### 5. Serverless for Spiky/Bursty Workloads
**Pattern:** Replace always-on servers with Lambda + API Gateway. Aurora Serverless v2 for variable DB workloads.
**Services:** Lambda, API Gateway (HTTP API), ECS Fargate Spot, DynamoDB On-Demand, Aurora Serverless v2
**Savings potential:** Pay only for actual usage (scale to zero)

### 6. Auto Scaling and Scheduled Scaling
**Pattern:** Scale down non-production environments nights/weekends. Scale production based on demand.
**Services:** EC2 Auto Scaling, Application Auto Scaling (ECS, DynamoDB, Aurora, Lambda provisioned), RDS stop/start scheduling
**Savings potential:** 30–70% on dev/test

### 7. NAT Gateway Cost Reduction
**Pattern:** NAT Gateway charges $0.045/GB — a hidden major cost. Minimize cross-AZ data through NAT. Use VPC endpoints for S3 and DynamoDB.
**Optimizations:**
- Gateway endpoints for S3/DynamoDB (free)
- Ensure NAT GW is in same AZ as resources
- Consider NAT instance for low-traffic scenarios
- Use PrivateLink for other services instead of routing through NAT

### 8. Data Transfer Optimization
**Pattern:** Keep data transfer within AWS, within regions, within AZs.
**Rules:**
- S3 → CloudFront = free
- AZ → AZ = $0.01/GB
- Region → Region = $0.02/GB
- To internet = $0.09/GB
- Use S3 Select / Athena query pruning to reduce data scanned

### 9. Reserved Capacity for Databases
**Pattern:** RDS and ElastiCache Reserved Instances (1yr/3yr) provide 40–60% savings for production workloads.
**Services:** RDS Reserved Instances, ElastiCache Reserved Nodes, Redshift Reserved Nodes, DynamoDB Reserved Capacity

### 10. Eliminate Waste
**Pattern:** Continuously audit for idle or orphaned resources.
**Checklist:**
- Unattached EBS volumes (snapshot + delete)
- Unused Elastic IPs ($0.005/hr)
- Idle RDS instances (>50% CPU < 1%, < 1 DB connection)
- Old Lambda versions and layers
- Unused Secrets, Parameters
- Forgotten S3 buckets (set lifecycle policies)
- Unused CloudWatch Dashboards ($3/mo each)
- Over-retained CloudWatch Logs
- Test environments running 24/7

**Services:** Trusted Advisor, Cost Explorer, AWS Cost Anomaly Detection, CloudWatch Metrics

---

# 29. Disaster Recovery Patterns

---

## Definitions
- **RPO (Recovery Point Objective):** Maximum acceptable data loss — how much data can you afford to lose?
- **RTO (Recovery Time Objective):** Maximum acceptable downtime — how quickly must you be operational?

Lower RPO/RTO = higher cost. Choose the pattern that satisfies requirements at minimum cost.

---

## DR Pattern Comparison

| Pattern | Description | RPO | RTO | Relative Cost | Key Services |
|---------|-------------|-----|-----|--------------|-------------|
| **Backup & Restore** | Backups stored in S3/Glacier; restore manually during disaster | Hours | Hours–Days | $ (Lowest) | S3, Glacier, AWS Backup, AMI, RDS Snapshots |
| **Pilot Light** | Minimal live infrastructure in DR region; core services running but scaled down | Minutes | 30min–Hours | $$ | EC2 (stopped), RDS read replica, Route 53 failover, AMIs |
| **Warm Standby** | Scaled-down but functional copy of production in DR region; scaled up on event | Seconds–Minutes | Minutes | $$$ | ASG (min capacity), Aurora Global DB, Route 53, CloudFormation |
| **Multi-Site Active-Active** | Full production in 2+ regions, traffic load-balanced simultaneously | Near-zero | Near-zero | $$$$ (Highest) | Global Accelerator/Route 53, DynamoDB Global Tables, Aurora Global DB, CloudFront |

---

## Per-Pattern Architecture Details

### Backup & Restore
```
Primary Region                    DR Region
─────────────────                 ──────────────────
EC2 + RDS → AMIs/Snapshots  ──→  S3/Glacier (cross-region replication)
                                  ↓
                                  Restore EC2 from AMI
                                  Restore RDS from snapshot
                                  Update DNS to new endpoints
```
- **Key services:** AWS Backup (cross-region copy), S3 CRR, DLM for AMIs
- **Automation:** CloudFormation template + Lambda scheduled restore test
- **Test regularly:** Annual or quarterly restore drill

### Pilot Light
```
Primary Region (full scale)       DR Region (minimal)
─────────────────────────         ──────────────────────
Web → App → RDS Primary  ───→    RDS Read Replica (always syncing)
                                  EC2 AMIs (not running)
                                  ASG (desired=0)
                                  During DR: scale up, promote replica, update Route 53
```
- **Key services:** RDS cross-region read replicas, Aurora Global DB, Route 53 failover routing

### Warm Standby
```
Primary Region (full scale)       DR Region (min scale)
─────────────────────────         ──────────────────────
Web (10 instances)  ───────→     Web (2 instances, can scale to 10)
App (20 instances)  ───────→     App (4 instances)
Aurora Primary      ───────→     Aurora Global DB Secondary (writable after promote)
Route 53: 100% → Primary         Route 53: 0% → DR (ready to flip)
```
- Scale up ASGs + promote Aurora Global DB secondary
- Update Route 53 weights to route traffic

### Multi-Site Active-Active
```
Region A (us-east-1)              Region B (eu-west-1)
────────────────────              ─────────────────────
ALB → ECS/EKS Tasks               ALB → ECS/EKS Tasks
DynamoDB Global Table ←──────────→ DynamoDB Global Table
Aurora Global DB Primary           Aurora Global DB Secondary
           ↑                                 ↑
           └──────────────┬──────────────────┘
                  Global Accelerator / CloudFront
                  (Route 53 latency routing)
```
- **Conflict resolution:** DynamoDB last-writer-wins; Aurora: single writer at a time
- **DR scenario:** Remove failed region from Route 53/GA health check → instant failover

---

## DR Services by Category

| Category | Backup & Restore | Pilot Light | Warm Standby | Active-Active |
|----------|-----------------|------------|--------------|--------------|
| Database | RDS Snapshots, Backup | Cross-region Read Replica | Aurora Global DB (secondary) | DynamoDB Global Tables, Aurora Global |
| Compute | AMI copy to DR | AMIs in DR region | ASG min=2 in DR | Full ASG in DR |
| DNS/Routing | Manual DNS update | Route 53 Failover | Route 53 Failover (quick flip) | Global Accelerator + health checks |
| Storage | S3 CRR | S3 CRR | S3 CRR | S3 CRR + multi-region access points |
| Automation | CloudFormation stack | CF Stack pre-deployed | CF Stack deployed + scaled down | Full stack deployed |

---

## RTO/RPO Reference for AWS Services

| Service | RPO | RTO | Notes |
|---------|-----|-----|-------|
| DynamoDB Global Tables | <1 second | <1 minute | Active-active, eventually consistent |
| Aurora Global Database | <1 second | <1 minute | Promote secondary to writer |
| RDS Multi-AZ | Near zero | 1–2 minutes | Automatic failover same region |
| RDS Cross-region replica | Minutes | 30–60 minutes | Manual promote required |
| S3 Cross-Region Replication | Minutes | Immediate (data already there) | After replication completes |
| EBS Snapshots | Daily (or per schedule) | 30–60 minutes (restore) | Faster with Fast Snapshot Restore |
| Backup & Restore (manual) | Hours | Hours | Depends on size + automation |

---

*AWS Master Reference Cheatsheet — Complete*
*Generated for: Senior Solutions Architects | Covers All 20 Service Categories + Architecture Patterns*

