# DevOps Phase 7 — Cloud Platforms

> **Learning Goal:** Understand cloud computing from the ground up — master the core services of AWS, Azure, and GCP that every DevOps engineer uses daily: compute, storage, networking, identity, and load balancing.

---

## Table of Contents

1. [What is Cloud Computing?](#1-what-is-cloud-computing)
2. [Cloud Service Models — IaaS, PaaS, SaaS](#2-cloud-service-models--iaas-paas-saas)
3. [AWS vs Azure vs GCP — Comparison](#3-aws-vs-azure-vs-gcp--comparison)
4. [IAM — Identity & Access Management](#4-iam--identity--access-management)
5. [Compute — EC2, VMs, and Instances](#5-compute--ec2-vms-and-instances)
6. [Storage — S3, Blob Storage, GCS](#6-storage--s3-blob-storage-gcs)
7. [VPC — Virtual Private Cloud (Networking)](#7-vpc--virtual-private-cloud-networking)
8. [Load Balancers](#8-load-balancers)
9. [Managed Kubernetes — EKS, AKS, GKE](#9-managed-kubernetes--eks-aks-gke)
10. [Serverless & Managed Services](#10-serverless--managed-services)
11. [Cloud Cost Optimization](#11-cloud-cost-optimization)
12. [Multi-Cloud & Hybrid Cloud](#12-multi-cloud--hybrid-cloud)
13. [Interview Mastery](#13-interview-mastery)

---

## 1. What is Cloud Computing?

### Beginner Explanation

Before cloud computing, if you wanted to run a website, you had to:
1. Buy physical servers ($5,000–$50,000 each)
2. Rent data center space ($1,000–$10,000/month)
3. Hire people to rack, cable, and maintain servers
4. Plan capacity months in advance (guess how much you'd need)
5. If traffic spiked beyond your capacity — your site crashed

**Cloud computing means renting someone else's computers** (Amazon's, Microsoft's, Google's) over the internet, paying only for what you use, and scaling up or down in seconds.

### Technical Definition

Cloud computing is the **on-demand delivery of compute, storage, networking, and other IT resources** via the internet, with pay-as-you-go pricing, managed by a cloud service provider.

### Five Essential Characteristics (NIST Definition)

| Characteristic | Meaning |
|---------------|---------|
| **On-demand self-service** | Provision resources yourself (no phone calls, no tickets) |
| **Broad network access** | Accessible over the internet from any device |
| **Resource pooling** | Provider serves multiple customers from shared infrastructure |
| **Rapid elasticity** | Scale up/down in seconds, appear unlimited |
| **Measured service** | Pay only for what you use (metered like electricity) |

### The Pre-Cloud vs Cloud World

```
PRE-CLOUD (On-Premises):
┌─────────────────────────────────────────────────────────────┐
│  YOUR DATA CENTER                                           │
│                                                             │
│  You own:     Servers, switches, routers, storage           │
│  You manage:  OS, patching, networking, cooling, power      │
│  You pay:     Upfront capital (CapEx) + ongoing operations  │
│  Scale time:  Weeks to months (order → ship → rack → config)│
│  Risk:        Overprovision (waste $) or underprovision     │
│               (site crashes)                                │
└─────────────────────────────────────────────────────────────┘

CLOUD:
┌─────────────────────────────────────────────────────────────┐
│  CLOUD PROVIDER'S DATA CENTER                               │
│                                                             │
│  They own:    Everything physical                           │
│  You manage:  Your apps, data, config (varies by model)     │
│  You pay:     Per hour/second of usage (OpEx)               │
│  Scale time:  Seconds (API call → resource ready)           │
│  Risk:        Only pay for what you use. Scale on demand.   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Cloud Service Models — IaaS, PaaS, SaaS

### The Pizza Analogy

```
WHAT DO YOU MANAGE vs WHAT DOES THE PROVIDER MANAGE?

On-Premises    IaaS          PaaS         SaaS
(Make at home) (Take & bake) (Delivery)   (Dine out)
─────────────  ─────────────  ───────────  ──────────
Applications   Applications   Applications │ Provider
Data           Data           Data         │ manages
Runtime        Runtime        ─────────────│ everything
Middleware     Middleware     │ Provider   │
OS             OS             │ manages    │
Virtualization ─────────────  │ these      │
Servers        │ Provider    │            │
Storage        │ manages     │            │
Networking     │ these       │            │
─────────────  ─────────────  ───────────  ──────────

YOU manage      YOU manage     YOU manage    YOU manage
everything      apps + data    apps + data   nothing
                               (maybe config)
```

### Detailed Comparison

| Model | You Manage | Provider Manages | Examples |
|-------|-----------|-----------------|----------|
| **IaaS** | OS, runtime, apps, data | Hardware, networking, virtualization | EC2, Azure VMs, GCE |
| **PaaS** | Application code, data | Everything below (OS, runtime, scaling) | Heroku, App Engine, Elastic Beanstalk, Azure App Service |
| **SaaS** | Configuration only | Everything | Gmail, Salesforce, Slack, GitHub |
| **FaaS** (serverless) | Individual functions | Everything including scaling | Lambda, Cloud Functions, Azure Functions |
| **CaaS** (containers) | Container images | Orchestration, networking, scaling | EKS, GKE, AKS, ECS Fargate |

### Choosing the Right Model

```
DECISION FLOW:

"I need full control over the OS and networking"
  → IaaS (EC2, VMs)

"I just want to deploy my code, don't care about servers"
  → PaaS (App Engine, Elastic Beanstalk)

"I want to run containers without managing servers"
  → CaaS/Serverless containers (Fargate, Cloud Run)

"I have event-driven, short-lived functions"
  → FaaS (Lambda, Cloud Functions)

"I need a database but don't want to manage it"
  → Managed service (RDS, Cloud SQL, DynamoDB)
```

---

## 3. AWS vs Azure vs GCP — Comparison

### Market Position (2024)

| Provider | Market Share | Strength | Best For |
|----------|-------------|----------|----------|
| **AWS** | ~32% | Broadest service catalog, most mature | Startups to enterprise, largest ecosystem |
| **Azure** | ~23% | Microsoft integration, enterprise | Windows workloads, Office 365 shops, hybrid |
| **GCP** | ~11% | Data/ML/Kubernetes, developer experience | Data analytics, ML, Kubernetes-native |

### Service Name Mapping

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **Compute (VMs)** | EC2 | Virtual Machines | Compute Engine |
| **Serverless compute** | Lambda | Azure Functions | Cloud Functions |
| **Container orchestration** | EKS | AKS | GKE |
| **Serverless containers** | Fargate | Container Apps | Cloud Run |
| **Object storage** | S3 | Blob Storage | Cloud Storage |
| **Block storage** | EBS | Managed Disks | Persistent Disk |
| **File storage** | EFS | Azure Files | Filestore |
| **Relational DB** | RDS / Aurora | Azure SQL | Cloud SQL / AlloyDB |
| **NoSQL DB** | DynamoDB | Cosmos DB | Firestore / Bigtable |
| **Cache** | ElastiCache | Azure Cache for Redis | Memorystore |
| **Message queue** | SQS | Service Bus | Cloud Pub/Sub |
| **VPC/Networking** | VPC | VNet | VPC |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |
| **CDN** | CloudFront | Azure CDN / Front Door | Cloud CDN |
| **Load balancer** | ALB / NLB / CLB | Azure Load Balancer / App Gateway | Cloud Load Balancing |
| **IAM** | IAM | Entra ID (Azure AD) + RBAC | Cloud IAM |
| **Secrets** | Secrets Manager | Key Vault | Secret Manager |
| **Monitoring** | CloudWatch | Azure Monitor | Cloud Monitoring |
| **Logging** | CloudWatch Logs | Log Analytics | Cloud Logging |
| **CI/CD** | CodePipeline / CodeBuild | Azure DevOps | Cloud Build |
| **IaC** | CloudFormation | ARM / Bicep | Deployment Manager |
| **Container Registry** | ECR | ACR | Artifact Registry |

### Regions and Availability Zones

```
REGION: A geographic location (e.g., us-east-1 = Northern Virginia)
  │
  ├── Availability Zone A (us-east-1a): Independent data center(s)
  ├── Availability Zone B (us-east-1b): Independent data center(s)
  └── Availability Zone C (us-east-1c): Independent data center(s)

Each AZ:
  - Separate power, cooling, networking
  - Connected via low-latency private fiber
  - Independent failure domain (fire/flood in one doesn't affect others)

HIGH AVAILABILITY PATTERN:
  Deploy your application in MULTIPLE AZs within the same region.
  If one AZ goes down, traffic fails over to the other AZ(s).

  ┌─────────────────────────────────────────────────┐
  │  Region: us-east-1                              │
  │                                                 │
  │  AZ-a              AZ-b              AZ-c       │
  │  ┌──────┐          ┌──────┐          ┌──────┐  │
  │  │App x2│          │App x2│          │App x2│  │
  │  │DB pri│          │DB rep│          │DB rep│  │
  │  └──────┘          └──────┘          └──────┘  │
  │                                                 │
  │  If AZ-a fails → traffic goes to AZ-b and AZ-c │
  └─────────────────────────────────────────────────┘
```

---

## 4. IAM — Identity & Access Management

### Why IAM is Critical

IAM answers three questions:
1. **Authentication:** WHO are you? (prove your identity)
2. **Authorization:** WHAT can you do? (permissions)
3. **Auditing:** WHAT did you do? (logging)

IAM is the #1 most important service in any cloud — misconfigure it and attackers get access to everything.

### AWS IAM — Deep Dive

```
AWS IAM HIERARCHY:

AWS Account (root)
    │
    ├── IAM Users (humans with long-term credentials)
    │     └── Access Keys (programmatic) or Console Password
    │
    ├── IAM Groups (collection of users)
    │     └── Developers, Admins, ReadOnly
    │
    ├── IAM Roles (temporary credentials, assumed by services/users)
    │     └── EC2 instance role, Lambda execution role, Cross-account role
    │
    └── IAM Policies (JSON documents defining permissions)
          └── Attached to Users, Groups, or Roles
```

### IAM Policy Structure

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadOnly",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "10.0.0.0/8"
                }
            }
        },
        {
            "Sid": "DenyDeleteAnything",
            "Effect": "Deny",
            "Action": "s3:DeleteObject",
            "Resource": "*"
        }
    ]
}
```

**Policy evaluation logic:**
```
1. By default, everything is DENIED (implicit deny)
2. Check all applicable policies
3. If ANY policy has an explicit Deny → DENIED (deny always wins)
4. If ANY policy has an Allow → ALLOWED
5. If no policy mentions the action → DENIED (implicit deny)

Order of precedence: Explicit Deny > Explicit Allow > Implicit Deny
```

### IAM Roles — The Most Important Concept

```
WHY ROLES, NOT USERS FOR SERVICES:

❌ WRONG: Create IAM user for your EC2 instance, put access keys in env vars
   - Long-lived credentials that can be stolen
   - Must rotate manually
   - If leaked → attacker has permanent access

✅ CORRECT: Attach IAM Role to EC2 instance
   - Temporary credentials (rotated automatically every hour)
   - No secrets to manage or leak
   - Credentials provided via instance metadata
   - Principle of least privilege easy to enforce

HOW IT WORKS:
  EC2 Instance → assumes Role → gets temporary credentials
  Lambda Function → executes with Role → has permissions defined in Role
  EKS Pod → IRSA (IAM Roles for Service Accounts) → pod-level IAM
  CI/CD (GitHub Actions) → OIDC → assumes Role (no stored keys!)
```

### AWS IAM Commands

```bash
# Create a user
aws iam create-user --user-name deploy-bot

# Create a role (for EC2)
aws iam create-role \
    --role-name EC2AppRole \
    --assume-role-policy-document file://trust-policy.json

# trust-policy.json:
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}

# Attach policy to role
aws iam attach-role-policy \
    --role-name EC2AppRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile (to attach role to EC2)
aws iam create-instance-profile --instance-profile-name EC2AppProfile
aws iam add-role-to-instance-profile \
    --instance-profile-name EC2AppProfile \
    --role-name EC2AppRole
```

### IAM Best Practices

```
1. NEVER use the root account for daily work
   → Create IAM users/roles, lock root in a safe with MFA

2. Principle of least privilege
   → Grant only permissions needed, nothing more
   → Start with zero permissions, add as needed

3. Use roles, not long-lived access keys
   → For EC2: instance roles
   → For Lambda: execution roles
   → For CI/CD: OIDC federation (no stored keys)
   → For cross-account: assume role

4. Enable MFA everywhere
   → Especially for console access and privilege escalation

5. Use IAM policies with conditions
   → Restrict by IP, time, MFA status, region

6. Audit with CloudTrail
   → Log every API call (who did what, when, from where)

7. Use Service Control Policies (SCPs) in AWS Organizations
   → Guardrails across all accounts (prevent certain actions globally)
```

### Azure & GCP IAM Comparison

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Identity provider | IAM | Entra ID (Azure AD) | Cloud Identity |
| Service identity | IAM Role | Managed Identity | Service Account |
| Permission grouping | Policy | Role Definition | Role |
| Binding | Policy attachment | Role Assignment | IAM Binding |
| Temporary creds | STS AssumeRole | Managed Identity token | Workload Identity |
| Org-level guardrails | SCPs | Azure Policy | Organization Policies |

---

## 5. Compute — EC2, VMs, and Instances

### What is EC2?

EC2 (Elastic Compute Cloud) is AWS's **virtual machine service**. You get a virtual server in the cloud within seconds, choose its size (CPU, memory, storage), and pay per second of usage.

### Instance Types (AWS)

```
NAMING CONVENTION:  m5.xlarge
                    │││  │
                    ││└──└── Size (nano → metal)
                    │└────── Generation (higher = newer)
                    └─────── Family (purpose)

FAMILIES:
  t3/t4g   — Burstable (cheap, for low/variable workloads)
  m5/m6i   — General purpose (balanced CPU + memory)
  c5/c6i   — Compute optimized (CPU-heavy: compilation, encoding)
  r5/r6i   — Memory optimized (RAM-heavy: caches, databases)
  i3/i4i   — Storage optimized (high IOPS: databases, data warehouses)
  p4/p5    — GPU instances (ML training, graphics rendering)
  g5       — Graphics/inference (ML inference, gaming)

SIZES:
  .nano      (2 vCPU, 0.5 GB)   → $0.005/hr
  .micro     (2 vCPU, 1 GB)
  .small     (2 vCPU, 2 GB)
  .medium    (2 vCPU, 4 GB)
  .large     (2 vCPU, 8 GB)     → ~$0.10/hr
  .xlarge    (4 vCPU, 16 GB)
  .2xlarge   (8 vCPU, 32 GB)
  .4xlarge   (16 vCPU, 64 GB)
  ...
  .metal     (96+ vCPU, 384+ GB) → ~$4.50/hr
```

### Launching an EC2 Instance

```bash
# Launch an EC2 instance via CLI
aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1f0 \
    --instance-type t3.medium \
    --key-name my-keypair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --iam-instance-profile Name=EC2AppProfile \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-app-server}]' \
    --user-data file://startup-script.sh \
    --count 1

# startup-script.sh (runs on first boot):
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
docker pull registry.example.com/myapp:latest
docker run -d -p 80:8080 registry.example.com/myapp:latest
```

### EC2 Pricing Models

| Model | Discount | Commitment | Best For |
|-------|---------|-----------|----------|
| **On-Demand** | 0% (full price) | None | Unpredictable workloads, development |
| **Reserved Instances** | Up to 72% | 1 or 3 year | Steady-state production workloads |
| **Savings Plans** | Up to 72% | 1 or 3 year (flexible) | Committed spend, flexible instance types |
| **Spot Instances** | Up to 90% | None (can be interrupted) | Fault-tolerant batch, CI/CD, ML training |

```
SPOT INSTANCES — 90% cheaper but AWS can reclaim with 2-min warning

GOOD USE CASES:
  ✅ CI/CD runners (if interrupted, retry the build)
  ✅ Batch processing (checkpointed, can resume)
  ✅ ML training (checkpointed every N minutes)
  ✅ Stateless workers (behind a queue — just process less)

BAD USE CASES:
  ❌ Production web servers (users see errors)
  ❌ Databases (data corruption risk)
  ❌ Single-instance anything critical
```

### Auto Scaling Groups (ASG)

```
AUTO SCALING GROUP:

┌─────────────────────────────────────────────────────────────────┐
│  ASG: my-app-asg                                                │
│  Min: 2 | Desired: 4 | Max: 20                                  │
│                                                                 │
│  Launch Template: t3.medium, ami-abc123, user-data script       │
│                                                                 │
│  Scaling Policy: Add 2 instances when CPU > 70% for 5 min      │
│                  Remove 1 instance when CPU < 30% for 10 min    │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │Instance 1│ │Instance 2│ │Instance 3│ │Instance 4│           │
│  │  AZ-a    │ │  AZ-b    │ │  AZ-a    │ │  AZ-b    │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                                                                 │
│  Health Check: ELB health check (unhealthy → replace)           │
│  Spread across AZs automatically                                │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create a launch template
aws ec2 create-launch-template \
    --launch-template-name my-app-lt \
    --launch-template-data '{
        "ImageId": "ami-0c55b159cbfafe1f0",
        "InstanceType": "t3.medium",
        "SecurityGroupIds": ["sg-xxx"],
        "IamInstanceProfile": {"Name": "EC2AppProfile"},
        "UserData": "base64-encoded-startup-script"
    }'

# Create ASG
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name my-app-asg \
    --launch-template LaunchTemplateName=my-app-lt,Version='$Latest' \
    --min-size 2 \
    --max-size 20 \
    --desired-capacity 4 \
    --vpc-zone-identifier "subnet-aaa,subnet-bbb" \
    --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/my-app-tg/...
    --health-check-type ELB \
    --health-check-grace-period 300
```

---

## 6. Storage — S3, Blob Storage, GCS

### What is Object Storage?

Object storage (S3) stores data as **objects** in flat "buckets" — unlike file systems (hierarchical folders) or block storage (raw disk). Each object has:
- A key (the "path": `images/photo.jpg`)
- The data (the file content)
- Metadata (content-type, custom headers)

### S3 — Deep Dive

```
S3 FUNDAMENTALS:

Bucket: my-company-assets (globally unique name)
  ├── images/
  │     ├── logo.png         (object)
  │     └── banner.jpg       (object)
  ├── backups/
  │     └── db-2024-01-15.sql.gz
  └── configs/
        └── app-config.json

Key features:
  - 99.999999999% durability (11 nines — designed to never lose data)
  - 99.99% availability
  - Unlimited storage
  - Single object up to 5 TB
  - Versioning (keep all versions of every object)
  - Encryption (at rest + in transit)
  - Lifecycle policies (auto-transition to cheaper storage)
  - Event notifications (trigger Lambda on upload)
```

### S3 Storage Classes

| Class | Use Case | Cost | Retrieval |
|-------|----------|------|-----------|
| **S3 Standard** | Frequently accessed data | $$$ | Instant |
| **S3 Standard-IA** | Infrequent access (>30 days) | $$ | Instant (retrieval fee) |
| **S3 One Zone-IA** | Non-critical infrequent data | $ | Instant (single AZ) |
| **S3 Glacier Instant** | Archive, instant access | $ | Instant (retrieval fee) |
| **S3 Glacier Flexible** | Archive, hours access OK | ¢ | 1–12 hours |
| **S3 Glacier Deep Archive** | Long-term archive | ¢¢ | 12–48 hours |
| **S3 Intelligent-Tiering** | Unknown access pattern | Auto-optimized | Instant |

### S3 Commands & Operations

```bash
# Create bucket
aws s3 mb s3://my-company-assets-prod

# Upload file
aws s3 cp ./backup.sql.gz s3://my-bucket/backups/backup.sql.gz

# Upload directory (recursive)
aws s3 sync ./dist/ s3://my-bucket/website/ --delete

# Download
aws s3 cp s3://my-bucket/config.json ./config.json

# List objects
aws s3 ls s3://my-bucket/images/

# Pre-signed URL (temporary access without credentials)
aws s3 presign s3://my-bucket/private/report.pdf --expires-in 3600
# Returns URL that anyone can use for 1 hour

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket my-bucket \
    --versioning-configuration Status=Enabled
```

### S3 Bucket Policy (Resource-Based Access)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-website-bucket/*"
        },
        {
            "Sid": "DenyUnencryptedUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-bucket/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        }
    ]
}
```

### S3 Lifecycle Policy

```json
{
    "Rules": [
        {
            "ID": "ArchiveOldBackups",
            "Status": "Enabled",
            "Filter": {"Prefix": "backups/"},
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                }
            ],
            "Expiration": {
                "Days": 365
            }
        }
    ]
}
```

### Common S3 Use Cases in DevOps

| Use Case | Pattern |
|----------|---------|
| Static website hosting | S3 + CloudFront CDN |
| CI/CD artifact storage | Build output uploaded after CI |
| Terraform state backend | Remote state locking with DynamoDB |
| Log archival | CloudWatch → S3 via subscription |
| Database backups | Automated nightly dumps to S3 |
| Docker registry storage | ECR uses S3 internally |
| Data lake | Raw data landing zone for analytics |

---

## 7. VPC — Virtual Private Cloud (Networking)

### What is a VPC?

A VPC is your **private, isolated network** in the cloud. It's like having your own data center's network, but virtual. You control:
- IP address ranges
- Subnets (public and private)
- Route tables
- Internet access (who can reach the internet)
- Security (firewalls at instance and subnet level)

### VPC Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16 (65,536 IP addresses)                                    │
│  Region: us-east-1                                                          │
│                                                                             │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────┐ │
│  │  AVAILABILITY ZONE A             │  │  AVAILABILITY ZONE B             │ │
│  │                                  │  │                                  │ │
│  │  ┌────────────────────────────┐  │  │  ┌────────────────────────────┐  │ │
│  │  │ PUBLIC SUBNET              │  │  │  │ PUBLIC SUBNET              │  │ │
│  │  │ 10.0.1.0/24               │  │  │  │ 10.0.2.0/24               │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  │  [ALB]  [NAT Gateway]     │  │  │  │  [ALB]  [Bastion Host]    │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  └──────────────┬─────────────┘  │  │  └──────────────┬─────────────┘  │ │
│  │                 │                 │  │                 │                 │ │
│  │  ┌──────────────┴─────────────┐  │  │  ┌──────────────┴─────────────┐  │ │
│  │  │ PRIVATE SUBNET (App)       │  │  │  │ PRIVATE SUBNET (App)       │  │ │
│  │  │ 10.0.3.0/24               │  │  │  │ 10.0.4.0/24               │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  │  [EC2: App Server]         │  │  │  │  [EC2: App Server]         │  │ │
│  │  │  [EKS Worker Nodes]        │  │  │  │  [EKS Worker Nodes]        │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  └──────────────┬─────────────┘  │  │  └──────────────┬─────────────┘  │ │
│  │                 │                 │  │                 │                 │ │
│  │  ┌──────────────┴─────────────┐  │  │  ┌──────────────┴─────────────┐  │ │
│  │  │ PRIVATE SUBNET (Data)      │  │  │  │ PRIVATE SUBNET (Data)      │  │ │
│  │  │ 10.0.5.0/24               │  │  │  │ 10.0.6.0/24               │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  │  [RDS Primary]             │  │  │  │  [RDS Replica]             │  │ │
│  │  │  [ElastiCache]             │  │  │  │  [ElastiCache Replica]     │  │ │
│  │  │                            │  │  │  │                            │  │ │
│  │  └────────────────────────────┘  │  │  └────────────────────────────┘  │ │
│  └──────────────────────────────────┘  └──────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────┐                                                      │
│  │ Internet Gateway   │  ← Connects VPC to the public internet              │
│  └───────────────────┘                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key VPC Components

| Component | Purpose |
|-----------|---------|
| **VPC** | The overall network container (CIDR block like 10.0.0.0/16) |
| **Subnet** | A sub-range of the VPC (e.g., 10.0.1.0/24 = 256 IPs). Resides in ONE AZ. |
| **Internet Gateway (IGW)** | Allows resources in public subnets to reach the internet |
| **NAT Gateway** | Allows private subnet resources to reach the internet (outbound only — can't be reached from outside) |
| **Route Table** | Rules that determine where network traffic goes |
| **Security Group** | Instance-level firewall (stateful — return traffic auto-allowed) |
| **Network ACL (NACL)** | Subnet-level firewall (stateless — must allow return traffic explicitly) |
| **VPC Peering** | Connect two VPCs (traffic stays on AWS backbone, not internet) |
| **Transit Gateway** | Hub for connecting multiple VPCs and on-prem networks |
| **VPC Endpoints** | Access AWS services (S3, DynamoDB) without going through internet |

### Public vs Private Subnets

```
PUBLIC SUBNET:
  - Has a route to the Internet Gateway (0.0.0.0/0 → igw-xxx)
  - Resources get public IP addresses
  - Directly reachable from the internet
  - Use for: Load balancers, bastion hosts, NAT gateways

PRIVATE SUBNET:
  - NO route to Internet Gateway
  - Resources only have private IPs
  - NOT reachable from the internet
  - Use for: Application servers, databases, internal services
  - Outbound internet via NAT Gateway (for package updates, API calls)

RULE: Put everything in private subnets unless it MUST be public.
Only load balancers and NAT gateways should be in public subnets.
```

### Route Tables

```bash
# Public subnet route table:
Destination        Target
10.0.0.0/16        local          # Traffic within VPC stays local
0.0.0.0/0          igw-abc123     # Everything else → Internet Gateway

# Private subnet route table:
Destination        Target
10.0.0.0/16        local          # Traffic within VPC stays local
0.0.0.0/0          nat-xyz789     # Everything else → NAT Gateway (outbound only)
```

### Security Groups (Instance Firewall)

```bash
# Security Group: web-servers
aws ec2 create-security-group \
    --group-name web-servers \
    --description "Allow HTTP/HTTPS from ALB" \
    --vpc-id vpc-123

# Inbound rules (what can reach the instance)
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxx \
    --protocol tcp \
    --port 8080 \
    --source-group sg-alb    # Only from ALB security group

# Outbound rules (what the instance can reach)
# Default: all outbound allowed (restrict in high-security environments)
```

```
SECURITY GROUP BEST PRACTICES:

✅ Allow traffic from OTHER SECURITY GROUPS, not IP ranges
   - ALB SG allows 80/443 from 0.0.0.0/0 (internet)
   - App SG allows 8080 from ALB SG (not from 0.0.0.0/0)
   - DB SG allows 5432 from App SG only

❌ Don't open 0.0.0.0/0 on non-web ports
   - SSH (22) should be restricted to your IP or bastion SG
   - DB ports should NEVER be open to the internet
```

### Creating a VPC (AWS CLI)

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=production}]'

# Create subnets
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-a}]'
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.3.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-a}]'

# Create Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxx --vpc-id vpc-xxx

# Create NAT Gateway (in public subnet)
aws ec2 allocate-address --domain vpc   # Get Elastic IP first
aws ec2 create-nat-gateway --subnet-id subnet-public-a --allocation-id eipalloc-xxx

# Create route tables and associate
aws ec2 create-route-table --vpc-id vpc-xxx
aws ec2 create-route --route-table-id rtb-public --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
aws ec2 create-route --route-table-id rtb-private --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
```

---

## 8. Load Balancers

### What is a Load Balancer?

A load balancer distributes incoming traffic across multiple servers (targets) to:
1. Prevent any single server from being overwhelmed
2. Provide high availability (if one server dies, traffic goes to others)
3. Enable zero-downtime deployments (drain old, add new)

### Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| Round Robin | Cycles through servers 1→2→3→1→2→3 | Equal-capacity servers |
| Least Connections | Sends to server with fewest active connections | Variable request duration |
| Weighted Round Robin | Higher-weight servers get more traffic | Mixed-capacity servers |
| IP Hash | Same client IP always hits same server | Session affinity (sticky) |
| Random | Random selection | Simple, surprisingly effective |

### AWS Load Balancer Types

```
THREE TYPES:

1. ALB (Application Load Balancer) — Layer 7 (HTTP/HTTPS)
   ─────────────────────────────────────────────────────
   - Routes based on URL path, hostname, headers, query strings
   - WebSocket support
   - HTTP/2 support
   - Can route to different target groups based on rules
   - Best for: Web applications, APIs, microservices
   
   Example:
     /api/*     → API target group
     /images/*  → Static content target group
     /admin/*   → Admin target group

2. NLB (Network Load Balancer) — Layer 4 (TCP/UDP)
   ─────────────────────────────────────────────────────
   - Routes based on IP + port (no HTTP inspection)
   - Ultra-low latency (millions of requests/sec)
   - Static IP addresses
   - Preserves source IP
   - Best for: High-performance TCP apps, gaming, IoT, gRPC

3. CLB (Classic Load Balancer) — Legacy
   ─────────────────────────────────────────────────────
   - Old generation, don't use for new projects
   - Both Layer 4 and Layer 7 (limited)
```

### ALB Configuration

```
ALB ARCHITECTURE:

                    ┌──────────────────────────────┐
Internet ──────────►│   ALB (public subnets)       │
                    │   Listener: 443 (HTTPS)       │
                    │   SSL Certificate: *.example  │
                    └──────────────┬────────────────┘
                                   │
                    ┌──────────────┴────────────────┐
                    │         RULES                  │
                    │                                │
                    │  IF path = /api/*              │
                    │    → Forward to: api-tg        │
                    │                                │
                    │  IF host = admin.example.com   │
                    │    → Forward to: admin-tg      │
                    │                                │
                    │  DEFAULT                       │
                    │    → Forward to: frontend-tg   │
                    └───────┬────────┬───────────────┘
                            │        │
                ┌───────────┘        └───────────────┐
                ▼                                    ▼
    ┌─────────────────────┐              ┌─────────────────────┐
    │  Target Group: api  │              │  Target Group: web   │
    │  Health: /health    │              │  Health: /           │
    │                     │              │                      │
    │  [EC2-1] [EC2-2]   │              │  [EC2-3] [EC2-4]    │
    └─────────────────────┘              └─────────────────────┘
```

### ALB with Terraform (Infrastructure as Code)

```hcl
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "my-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]

  enable_deletion_protection = true
}

# HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# HTTP → HTTPS redirect
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Target Group
resource "aws_lb_target_group" "app" {
  name        = "my-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"

  health_check {
    path                = "/health"
    port                = "8080"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
  }

  deregistration_delay = 30  # Wait 30s for in-flight requests to finish
}
```

### Health Checks

```
LOAD BALANCER HEALTH CHECK:

ALB sends HTTP GET /health to each target every 30 seconds.
  - 200 OK → healthy (receives traffic)
  - 5 consecutive failures → unhealthy (removed from rotation)
  - 2 consecutive successes → healthy again (re-added)

Timeline:
  Target healthy → receiving traffic
  Target fails 1x → still receiving traffic (could be a blip)
  Target fails 3x → still receiving traffic (still below threshold)
  Target fails 5x → UNHEALTHY → removed from rotation
  Target recovers 2x → healthy → added back

THIS IS WHY HEALTH ENDPOINTS MATTER IN YOUR APP:
  GET /health → 200 if ready to serve, 503 if not
```

---

## 9. Managed Kubernetes — EKS, AKS, GKE

### Why Managed Kubernetes?

Self-managed Kubernetes means YOU manage the control plane (API server, etcd, scheduler). That's complex, risky, and time-consuming.

Managed Kubernetes: cloud provider manages the control plane. You manage worker nodes and workloads.

| Feature | Self-Managed | Managed (EKS/AKS/GKE) |
|---------|-------------|----------------------|
| Control plane management | You (patching, HA, backups) | Provider |
| Control plane cost | Your servers | ~$75/month (EKS) or free (GKE Autopilot) |
| Upgrade complexity | Manual, risky | Semi-automated |
| etcd backups | Your responsibility | Provider handles |
| Cloud integration (IAM, LB, storage) | Manual setup | Native integration |
| Support | Community | Provider support + SLA |

### EKS (AWS)

```bash
# Create EKS cluster using eksctl (recommended)
eksctl create cluster \
    --name production \
    --version 1.29 \
    --region us-east-1 \
    --nodegroup-name workers \
    --node-type m5.large \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 10 \
    --managed \
    --with-oidc    # For IRSA (IAM Roles for Service Accounts)

# Update kubeconfig
aws eks update-kubeconfig --name production --region us-east-1

# Now use kubectl as normal
kubectl get nodes
kubectl get pods -A
```

### GKE (Google Cloud)

```bash
# Create GKE Autopilot cluster (Google manages everything including nodes)
gcloud container clusters create-auto production \
    --region us-central1 \
    --release-channel regular

# Standard GKE (you manage node pools)
gcloud container clusters create production \
    --zone us-central1-a \
    --num-nodes 3 \
    --machine-type e2-standard-4 \
    --enable-autoscaling --min-nodes 2 --max-nodes 20

# Get credentials
gcloud container clusters get-credentials production --region us-central1
```

### AKS (Azure)

```bash
# Create AKS cluster
az aks create \
    --resource-group myResourceGroup \
    --name production \
    --node-count 3 \
    --node-vm-size Standard_D4s_v3 \
    --enable-cluster-autoscaler \
    --min-count 2 \
    --max-count 20 \
    --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myResourceGroup --name production
```

---

## 10. Serverless & Managed Services

### AWS Lambda

```python
# Lambda function example
import json
import boto3

def handler(event, context):
    """Process S3 upload event — triggered when a file is uploaded."""
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Process the uploaded file
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=bucket, Key=key)
    content = response['Body'].read()
    
    # Do something with the content
    processed = process_image(content)
    
    # Upload result
    s3.put_object(
        Bucket=bucket,
        Key=f"processed/{key}",
        Body=processed
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({'message': f'Processed {key}'})
    }
```

### When to Use Serverless vs Containers

| Factor | Serverless (Lambda) | Containers (EKS/ECS) |
|--------|--------------------|-----------------------|
| Execution time | < 15 minutes | Unlimited |
| Scale-to-zero | Yes (pay nothing when idle) | No (min 1 instance) |
| Cold start | 100ms – 10s | N/A (always running) |
| Concurrent executions | 1000 default (requestable) | Based on node capacity |
| State | Stateless only | Stateful possible |
| Networking | Limited VPC control | Full VPC control |
| Best for | Event-driven, sporadic, short tasks | Long-running services, complex apps |
| Cost at scale | Expensive at high RPS | Cheaper at high RPS |

### Key Managed Services for DevOps

```
DATABASES (don't manage your own in production):
  AWS RDS / Aurora     — PostgreSQL, MySQL (automated backups, failover, scaling)
  DynamoDB             — NoSQL key-value (serverless, auto-scaling)
  ElastiCache          — Redis/Memcached (managed, multi-AZ)

QUEUES & EVENTS:
  SQS                  — Message queue (decouples services)
  SNS                  — Pub/sub notifications (fan-out to multiple subscribers)
  EventBridge          — Event bus (route events between AWS services)

SECRETS:
  AWS Secrets Manager  — Rotate secrets automatically, audit access
  SSM Parameter Store  — Simple key-value config (free tier available)

CI/CD:
  CodeBuild            — Build service (like a hosted Jenkins agent)
  CodePipeline         — Orchestrates build → test → deploy

MONITORING:
  CloudWatch           — Metrics, logs, alarms, dashboards
  X-Ray                — Distributed tracing
```

---

## 11. Cloud Cost Optimization

### The Cost Optimization Framework

```
CLOUD COST OPTIMIZATION PILLARS:

1. RIGHT-SIZING
   - Most instances are over-provisioned
   - Use CloudWatch/Metrics to find actual usage
   - Downsize instances using 70% of capacity
   - Use AWS Compute Optimizer recommendations

2. PRICING MODELS
   - Reserved Instances / Savings Plans for steady state (up to 72% off)
   - Spot instances for fault-tolerant workloads (up to 90% off)
   - On-Demand only for unpredictable/temporary

3. TURN OFF UNUSED RESOURCES
   - Development environments off nights/weekends (save 65%)
   - Unattached EBS volumes ($0.10/GB/month for nothing)
   - Old snapshots, unused Elastic IPs, idle load balancers
   - Tag everything → find unowned resources

4. ARCHITECTURE OPTIMIZATION
   - Use serverless for sporadic workloads (pay-per-invocation)
   - Use S3 lifecycle policies (move old data to Glacier)
   - Consolidate small databases
   - Use CloudFront CDN (reduces origin load = smaller instances needed)

5. MONITORING & GOVERNANCE
   - AWS Cost Explorer / Budgets / Anomaly Detection
   - Tag policies (every resource tagged with team/project/environment)
   - Budget alerts (notify when spend exceeds threshold)
```

### Cost Comparison Example

```
SCENARIO: Web API serving 10,000 requests/minute, steady traffic

Option A: On-Demand EC2 (m5.large x 3)
  3 × $0.096/hr × 730 hours = $210/month

Option B: Reserved Instances (1-year, no upfront)
  3 × $0.060/hr × 730 hours = $131/month (38% savings)

Option C: Savings Plan (1-year)
  ~$130/month (similar savings, more flexibility)

Option D: 2 Reserved + 1 Spot (if app handles instance loss)
  2 × $0.060 + 1 × $0.029 = $109/month (48% savings)

BIGGEST WINS:
  - Turn off dev/staging nights+weekends: save 65% on those environments
  - Right-size (most teams over-provision by 40-60%)
  - Use Spot for CI/CD runners, batch processing
```

---

## 12. Multi-Cloud & Hybrid Cloud

### Definitions

```
SINGLE CLOUD: All resources in one provider (AWS only)
  Pro: Simpler, deeper integration, single skill set
  Con: Vendor lock-in, single point of failure

MULTI-CLOUD: Resources spread across 2+ providers (AWS + GCP)
  Pro: No vendor lock-in, leverage best-of-breed services
  Con: Complexity, multiple skill sets, data transfer costs, inconsistent APIs

HYBRID CLOUD: Mix of on-premises + cloud
  Pro: Keep sensitive data on-prem, migrate gradually, regulatory compliance
  Con: Complex networking, operational overhead

REALITY CHECK:
  - Most companies use ONE primary cloud + a few specific services from others
  - True multi-cloud (identical deployments on 2+ clouds) is rare and expensive
  - "Multi-cloud" often means: AWS for compute + GCP for BigQuery + Azure for Office 365
```

### Tools for Multi-Cloud

| Tool | Purpose |
|------|---------|
| **Terraform** | Provision infrastructure on any cloud with same language |
| **Kubernetes** | Same orchestration layer regardless of cloud |
| **Ansible** | Configuration management across any infrastructure |
| **Pulumi** | IaC in real programming languages (Python, Go, TypeScript) |
| **Crossplane** | Manage cloud resources as Kubernetes CRDs |

---

## 13. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: Explain the difference between IaaS, PaaS, and SaaS with examples.**

**Perfect Answer:**

"These represent different levels of abstraction in cloud services — how much the provider manages vs how much you manage:

**IaaS (Infrastructure as a Service):** You get virtual hardware — VMs, storage, networking. You still manage the OS, runtime, and application. Example: AWS EC2, Azure VMs. You'd choose this when you need full control over the environment.

**PaaS (Platform as a Service):** You provide your application code; the platform handles everything below — OS, runtime, scaling, load balancing. Example: Heroku, AWS Elastic Beanstalk, Google App Engine. You'd choose this for faster development when you don't need to customize the infrastructure.

**SaaS (Software as a Service):** A complete application delivered over the internet. You just use it — no infrastructure or code management. Example: Gmail, Salesforce, Slack.

The tradeoff is control vs operational burden. IaaS gives maximum control but most operational work. SaaS gives zero control but zero operations. Most DevOps work happens at the IaaS and CaaS (containers) level, where we have enough control to optimize but use cloud automation to reduce operational burden."

---

**Q2: What is an Availability Zone and why do you deploy across multiple AZs?**

**Perfect Answer:**

"An Availability Zone (AZ) is one or more physically separate data centers within a cloud region. Each AZ has independent power, cooling, and networking, but they're connected to each other via low-latency private fiber (typically < 2ms round-trip).

We deploy across multiple AZs for **high availability**. If one AZ suffers a failure (power outage, network issue, natural disaster), our application continues running in the other AZ(s) without interruption.

The pattern:
- Application servers: at least 2 instances, one per AZ minimum
- Load balancer: spans multiple AZs (distributes traffic)
- Database: primary in one AZ, synchronous replica in another (Multi-AZ RDS)
- If AZ-a fails: load balancer detects unhealthy targets, routes all traffic to AZ-b

This gives us a Recovery Time Objective (RTO) of zero — automatic failover with no manual intervention. The small added cost (running in 2+ AZs) is vastly outweighed by the reliability gain. In production, single-AZ deployments are considered unacceptable for anything customer-facing."

---

**Q3: What is the difference between a Security Group and a Network ACL?**

**Perfect Answer:**

"Both are firewalls, but they operate at different levels and with different behavior:

**Security Group (instance-level):**
- Attached to specific instances/ENIs
- **Stateful** — if you allow inbound traffic, the response is automatically allowed outbound (and vice versa)
- Rules are allow-only (you can't write deny rules)
- Default: deny all inbound, allow all outbound
- Evaluated as a whole (all rules considered together)
- Can reference other security groups ('allow traffic from SG-web')

**Network ACL (subnet-level):**
- Applied to the entire subnet
- **Stateless** — you must explicitly allow both inbound AND outbound traffic (including ephemeral return ports)
- Rules can be both allow AND deny
- Rules are evaluated in numbered order (first match wins)
- Default: allow all (customized for defense-in-depth)

In practice: Security Groups are the primary firewall mechanism. NACLs are a second layer of defense, mainly used to block entire IP ranges at the subnet level. The best practice is to use security groups that reference other security groups — for example, the database SG allows port 5432 only from the application SG, regardless of IP."

---

### Intermediate Interview Questions

---

**Q4: Explain how you would design a VPC for a production web application.**

**Perfect Answer:**

"I'd design a three-tier VPC architecture spread across at least two Availability Zones:

**CIDR:** 10.0.0.0/16 (65,536 addresses — room to grow)

**Subnet layout per AZ:**
1. **Public subnet (10.0.1.0/24, 10.0.2.0/24):** Only the Application Load Balancer and NAT Gateways live here. These need direct internet access.

2. **Private subnet - Application tier (10.0.3.0/24, 10.0.4.0/24):** Application servers (EC2, EKS worker nodes). These can reach the internet outbound via NAT Gateway (for package updates, external API calls) but cannot be reached directly from the internet.

3. **Private subnet - Data tier (10.0.5.0/24, 10.0.6.0/24):** Databases (RDS), caches (ElastiCache). These have no internet access at all — not even outbound. They communicate only with the application tier.

**Routing:**
- Public subnets: route to Internet Gateway
- App private subnets: route to NAT Gateway (for outbound)
- Data private subnets: local only (no internet route)

**Security Groups (layered):**
- ALB SG: allows 443 from 0.0.0.0/0
- App SG: allows 8080 from ALB SG only
- DB SG: allows 5432 from App SG only

**Additional components:**
- VPC Flow Logs enabled (for security auditing)
- VPC Endpoints for S3 and DynamoDB (avoid NAT costs for AWS service traffic)
- DNS: Route 53 private hosted zone for internal service discovery

This design follows the principle of least access — each layer can only communicate with its adjacent layers, and the data tier is completely isolated from the internet."

---

**Q5: How do IAM roles work, and why are they preferred over IAM users for services?**

**Perfect Answer:**

"IAM roles provide **temporary security credentials** that are automatically rotated. Unlike IAM users (which have permanent access keys), roles issue credentials that expire after 1–12 hours.

How they work:
1. You create a role with a trust policy (who can assume it) and a permissions policy (what they can do)
2. A service (EC2, Lambda, EKS pod) or user assumes the role
3. AWS STS (Security Token Service) issues temporary credentials (access key + secret key + session token)
4. These credentials auto-expire and are auto-rotated

Why roles are preferred for services:

**Security:**
- No long-lived credentials to store, rotate, or leak
- If an instance is compromised, credentials expire within hours (limited blast radius)
- With IAM users, a leaked key gives permanent access until manually rotated

**Operations:**
- No credential management overhead (no rotation scripts, no secrets in config)
- Instance metadata automatically provides fresh credentials
- Works with OIDC federation (GitHub Actions, EKS pods) — no AWS credentials stored in CI

**Auditability:**
- CloudTrail logs show exactly which role was used and by whom
- Easier to trace actions to specific services

Practical examples:
- EC2 instance role: instance automatically has permissions to read from S3
- Lambda execution role: function has permissions to write to DynamoDB
- EKS IRSA: specific pods get specific permissions (not the whole node)
- Cross-account role: service in Account A assumes role in Account B (no shared credentials)"

---

**Q6: Explain S3 storage classes and lifecycle policies. How do you optimize S3 costs?**

**Perfect Answer:**

"S3 offers multiple storage classes with different price/access tradeoffs:

- **Standard:** Frequent access, highest cost ($0.023/GB/month), instant retrieval
- **Standard-IA:** Infrequent access (>30 days), cheaper storage ($0.0125/GB) but retrieval fee
- **Glacier Instant:** Archive with instant retrieval, very cheap storage ($0.004/GB)
- **Glacier Flexible:** Archive where 1–12 hour retrieval is acceptable ($0.0036/GB)
- **Deep Archive:** Cheapest ($0.00099/GB), 12–48 hour retrieval (regulatory compliance archives)
- **Intelligent-Tiering:** Automatically moves objects between tiers based on access patterns (small monitoring fee)

**Lifecycle policies** automate transitions:
```
Day 0–30:    Standard (frequently accessed after upload)
Day 30–90:   Standard-IA (still needed occasionally)
Day 90–365:  Glacier Instant (rare access, fast retrieval if needed)
Day 365+:    Deep Archive or delete
```

**Cost optimization strategies:**

1. **Lifecycle policies** — biggest win. Most data is accessed in the first week and never again. Auto-transition to cheaper tiers.

2. **Intelligent-Tiering** — for unpredictable access patterns. It monitors and moves objects automatically.

3. **Multipart upload cleanup** — failed multipart uploads accumulate as incomplete fragments. Add an abort rule for incomplete uploads older than 7 days.

4. **S3 analytics** — use Storage Class Analysis to identify objects that could move to cheaper tiers.

5. **Compression** — gzip/zstd before upload. Reduces storage cost AND transfer cost.

6. **Delete what you don't need** — old versions (if versioning is on), expired objects.

In practice, I've seen S3 costs drop 60-70% just by implementing lifecycle policies on buckets that were using Standard for everything."

---

### Advanced Interview Questions

---

**Q7: Design a disaster recovery strategy for a critical application on AWS. Explain RTO and RPO.**

**Perfect Answer:**

"Two key metrics define disaster recovery requirements:

- **RPO (Recovery Point Objective):** How much data can you afford to lose? (measured in time — e.g., RPO of 1 hour means you can lose up to 1 hour of data)
- **RTO (Recovery Time Objective):** How long can the application be down? (e.g., RTO of 15 minutes means it must be back in 15 minutes)

**DR strategies ranked by cost and recovery speed:**

**1. Backup & Restore (cheapest, slowest: RTO hours, RPO hours)**
- Regular backups to S3/Glacier (DB snapshots, AMIs)
- On disaster: launch infrastructure from backups in another region
- Cost: only storage cost when not in disaster mode
- For: non-critical systems, cost-sensitive applications

**2. Pilot Light (low cost, faster: RTO 10-30 min, RPO minutes)**
- Core infrastructure running in DR region at minimum capacity (DB replica, AMI ready)
- On disaster: scale up compute, switch DNS
- Cost: DB replica + minimal infra running
- For: important systems with moderate recovery requirements

**3. Warm Standby (moderate cost, fast: RTO minutes, RPO seconds)**
- Fully functional copy running at reduced capacity in DR region
- On disaster: scale up to full capacity, switch traffic
- Cost: ~30% of primary region cost
- For: business-critical systems

**4. Multi-Site Active-Active (expensive, instant: RTO ~0, RPO ~0)**
- Full production running in both regions simultaneously
- Traffic distributed via Route 53 latency-based routing
- On disaster: Route 53 detects failure, shifts all traffic to healthy region
- Cost: ~2x primary (two full environments)
- For: mission-critical systems (payments, banking)

**My implementation for a typical production app (Warm Standby):**

Primary region (us-east-1):
- Application: EKS cluster with full replica count
- Database: RDS Multi-AZ (synchronous replica within region)
- Cross-region: Aurora Global Database (async replication, <1s lag)

DR region (us-west-2):
- Application: EKS cluster at 30% capacity (HPA ready to scale)
- Database: Aurora read replica (promoted to primary in DR)
- Infrastructure: Terraform can apply full-scale config

Failover process (automated):
1. Route 53 health check detects primary region failure
2. Route 53 fails over DNS to DR region
3. Aurora Global Database promotes DR replica to primary
4. EKS HPA scales up pods in DR region
5. Total RTO: 2-5 minutes

Testing: We conduct DR drills quarterly — actually fail over to DR region, run there for hours, then fail back. Untested DR is broken DR."

---

**Q8: How would you migrate a legacy on-premises application to AWS? Walk through the strategy.**

**Perfect Answer:**

"I follow AWS's 7 R's migration framework, but the decision tree is what matters:

**Assessment first (2-4 weeks):**
- Inventory all applications, dependencies, and data flows
- Measure current resource usage (CPU, memory, storage, network)
- Identify dependencies between applications
- Classify each application into a migration strategy

**The 7 strategies (in order of complexity):**

1. **Retire** — Turn it off. Is anyone using it? 10-20% of apps in a portfolio are unused.

2. **Retain** — Keep on-premises. Some systems can't move (mainframes, latency-sensitive to on-prem hardware, regulatory restrictions).

3. **Rehost (Lift & Shift)** — Move the VM to EC2 as-is. Fastest migration, no code changes. Use AWS Application Migration Service (MGN) for automated server replication.

4. **Replatform (Lift & Reshape)** — Small optimizations during migration. Move to RDS instead of self-managed MySQL. Containerize with minimal code changes.

5. **Repurchase** — Replace with SaaS. Move custom CRM to Salesforce. Move custom email to SES.

6. **Refactor/Re-architect** — Redesign for cloud-native. Decompose monolith into microservices, use serverless. Most expensive but biggest long-term benefit.

7. **Relocate** — Move VMware to VMware on AWS (hypervisor-level migration).

**My typical approach for a production application:**

Phase 1 (Week 1-2): **Rehost** — Get it running on EC2 quickly
- Use MGN to replicate servers to EC2
- Set up VPC with VPN back to on-prem (hybrid connectivity)
- Cutover during maintenance window
- Validate functionality

Phase 2 (Month 1-2): **Replatform** — Quick wins
- Migrate database to RDS (automated backups, Multi-AZ, no patching)
- Move static assets to S3 + CloudFront
- Containerize the application (Docker)
- Deploy on ECS or EKS

Phase 3 (Month 3+): **Refactor** — if justified
- Decompose components that need independent scaling
- Use managed services (SQS for queues, ElastiCache for caching)
- Implement auto-scaling, multi-AZ, proper CI/CD

**Key risks and mitigations:**
- Data migration: use AWS DMS for database migration with minimal downtime
- Network: AWS Direct Connect or VPN for hybrid connectivity during transition
- Rollback plan: keep on-prem running in parallel until cloud is validated
- Compliance: ensure cloud meets regulatory requirements before moving sensitive data"

---

### Scenario-Based Questions

---

**Q9: Your application on EC2 is experiencing intermittent 504 Gateway Timeout errors. How do you troubleshoot?**

**Perfect Answer:**

"504 means the load balancer timed out waiting for a response from the target. The issue is between the ALB and the application — either the app is too slow or the connection is broken.

**Systematic investigation:**

**1. Check ALB access logs and target group health:**
```bash
# Are targets healthy?
aws elbv2 describe-target-health --target-group-arn arn:aws:...

# Check ALB metrics in CloudWatch:
# - TargetResponseTime (is it increasing?)
# - UnHealthyHostCount (are targets failing health checks?)
# - HTTPCode_ELB_504_Count (how many 504s?)
```

**2. Check if it's all targets or specific ones:**
If specific instances: that instance has a problem (CPU, memory, process hanging).
If all targets intermittently: likely a capacity or resource issue.

**3. Common causes and fixes:**

| Cause | Evidence | Fix |
|-------|----------|-----|
| App too slow (> ALB idle timeout) | TargetResponseTime > 60s | Optimize slow endpoints, increase ALB timeout |
| Target health check failing | UnHealthyHostCount > 0 | Fix health endpoint, check if app is overloaded |
| Security group misconfigured | Targets healthy but 504s | Ensure app SG allows traffic from ALB SG on correct port |
| Connection draining during deploy | 504s during deployments | Increase deregistration delay |
| Instance running out of resources | CPU > 90%, memory exhausted | Scale up (bigger instance) or out (more instances) |
| Too many connections (file descriptors) | 'too many open files' in app logs | Increase ulimits, add connection pooling |
| Network issues between AZs | 504s correlate with one AZ | Check VPC routing, subnet configuration |

**4. Quick checks:**
```bash
# SSH into an EC2 instance behind the ALB
# Check application process
ps aux | grep node
top -c

# Check application logs
tail -f /var/log/app/application.log

# Test locally — does the app respond?
curl -v http://localhost:8080/health

# Check connections
ss -s   # Connection summary
ss -tnp | wc -l  # Total connections

# Check if the app is listening on the expected port
netstat -tlnp | grep 8080
```

**5. Resolution priority:**
- If ALB idle timeout (default 60s) < app response time → increase ALB timeout OR fix the slow endpoint
- If targets failing health checks → fix the app or health endpoint
- If resource exhaustion → scale horizontally (add instances to ASG)"

---

**Q10: Your AWS bill increased by 40% month-over-month unexpectedly. How do you investigate and resolve?**

**Perfect Answer:**

"A 40% unexpected increase is a cost incident. I investigate systematically:

**1. Identify WHAT changed (Cost Explorer):**
```bash
aws ce get-cost-and-usage \
    --time-period Start=2024-01-01,End=2024-02-01 \
    --granularity DAILY \
    --metrics UnblendedCost \
    --group-by Type=DIMENSION,Key=SERVICE
```

Look at cost by service — which service spiked? Top suspects:
- EC2 (instances left running, wrong instance type)
- RDS (unintended Multi-AZ, storage auto-scaling)
- Data transfer (cross-region, internet egress)
- NAT Gateway (very expensive per GB if misconfigured)
- S3 (request costs on public bucket being scraped)

**2. Narrow to specific resources:**
- AWS Cost Explorer → filter by tag (team, project, environment)
- If untagged resources are the culprit → someone deployed without tags
- Check for resources in unexpected regions (someone created in wrong region)

**3. Common culprits and fixes:**

| Culprit | How It Happens | Fix |
|---------|---------------|-----|
| NAT Gateway data processing | Private subnet apps making heavy external calls | Use VPC endpoints for AWS services, review traffic |
| Dev environments left running | Forgot to turn off after testing | Schedule auto-stop (Lambda + EventBridge) |
| Over-provisioned instances | Launched large instances 'just in case' | Right-size using CloudWatch CPU/memory data |
| Unattached EBS volumes | Deleted instances but volumes remain | Script to find/delete unattached volumes |
| Data transfer cross-region | DB replication or service calls across regions | Co-locate services, use caching |
| Spot instance price spike | Spot price exceeded On-Demand (rare) | Diversify instance types in Spot fleet |
| DDoS or scraping | Public endpoint getting hammered | WAF, CloudFront, rate limiting |
| Runaway autoscaling | HPA/ASG scaled up and never down | Check scale-down policies, set max limits |

**4. Immediate actions:**
- Kill any obvious waste (unattached resources, oversized dev instances)
- Set up AWS Budgets with alerts ($X threshold → email/Slack immediately)
- Enable AWS Cost Anomaly Detection (ML-based, catches spikes early)

**5. Long-term governance:**
- Mandatory tagging policy (enforce via SCP or AWS Config)
- Weekly cost review (15 minutes, look at trends)
- Reserved Instances / Savings Plans for steady-state workloads
- Automated shutdown of non-production environments outside business hours

In my experience, the top 3 surprise costs are always: NAT Gateway data processing, forgotten dev instances, and unattached EBS volumes."

---

### FAANG-Level Questions

---

**Q11: Design a globally distributed application architecture on AWS that serves users in North America, Europe, and Asia with <100ms latency.**

**Perfect Answer:**

"For sub-100ms latency globally, we need to bring compute and data close to users. Here's my architecture:

**Global Traffic Management:**
- Route 53 with latency-based routing: directs users to the nearest region
- CloudFront CDN: caches static content at 400+ edge locations (single-digit ms for cached content)

**Compute (3 regions):**
- us-east-1 (North America)
- eu-west-1 (Europe)
- ap-southeast-1 (Asia)

Each region runs:
- EKS cluster (application pods)
- Application-level cache (ElastiCache Redis)
- Regional data store

**Data Strategy (the hard part):**

For reads: each region has a local read replica
For writes: depends on consistency requirements:

*Option A — Single-writer with async replication:*
- One region is the primary writer (us-east-1)
- Aurora Global Database replicates to EU and APAC (<1 second lag)
- Reads are local (fast), writes route to primary (higher latency for non-US users)
- Acceptable for apps where slight write latency is OK

*Option B — Multi-writer with conflict resolution:*
- DynamoDB Global Tables: multi-region, multi-active writes
- Last-writer-wins conflict resolution
- Reads AND writes are local in every region
- Best for: user sessions, preferences, non-financial data

*Option C — CQRS (Command Query Responsibility Segregation):*
- Writes go to a central region via SQS/Kinesis
- Processed asynchronously, replicated to all regions
- Reads are always local
- Best for: event-driven systems, eventual consistency acceptable

**Architecture diagram:**
```
User (Tokyo) → Route 53 (latency routing) → ap-southeast-1
                                              │
                                              ▼
                                        CloudFront (cached)
                                              │
                                        ALB → EKS pods
                                              │
                                        ElastiCache (local)
                                              │
                                        Aurora Read Replica (local reads)
                                              │ (writes)
                                              ▼
                                        Aurora Global (primary in us-east-1)
```

**Latency budget:**
- DNS: ~5ms (Route 53 is anycast)
- CDN (cached): <10ms
- CDN miss → origin: within-region = <5ms to ALB
- Application processing: ~20ms
- Cache hit: <2ms
- DB read (local replica): ~5ms
- Total: 30-50ms for cache hits, 50-80ms for DB reads

**Failure handling:**
- If a region goes down: Route 53 detects (health check fails), routes traffic to next-closest region
- Users experience slight latency increase but no outage
- Aurora Global Database promotes replica to primary if primary region fails

**Key tradeoffs:**
- Multi-region adds significant operational complexity
- Data consistency vs latency: choose based on business requirements
- Cost: roughly 3x single-region (but not exactly, since each region can be sized for its traffic)
- Cross-region data transfer costs are significant — minimize replication payload"

---

**Q12: Explain the Shared Responsibility Model. A customer's EC2 instance is compromised — who is responsible?**

**Perfect Answer:**

"The Shared Responsibility Model defines who secures what:

**AWS is responsible for security OF the cloud:**
- Physical data center security (fences, guards, biometrics)
- Hardware (servers, switches, storage)
- Hypervisor and host OS (patching the underlying infrastructure)
- Network infrastructure (global backbone, DDoS protection at infrastructure level)
- Managed service internals (RDS engine patching, S3 durability)

**Customer is responsible for security IN the cloud:**
- Guest OS patching (your EC2 instance's Ubuntu/Amazon Linux)
- Application code and dependencies
- Network configuration (Security Groups, NACLs, VPC design)
- IAM (who has access, least privilege)
- Data encryption (at rest and in transit)
- Firewall/WAF configuration
- Monitoring and incident response

**For the compromised EC2 instance:**

If the compromise happened via:
- **Unpatched OS vulnerability** → Customer's fault (responsible for patching the guest OS)
- **Application vulnerability (SQLi, RCE)** → Customer's fault (responsible for app security)
- **Stolen SSH key or IAM credentials** → Customer's fault (responsible for credential management)
- **Security group too permissive (port 22 open to 0.0.0.0/0)** → Customer's fault
- **Hypervisor escape or hardware-level breach** → AWS's fault (extremely rare, AWS invests billions in this)

**The model shifts based on service type:**
- EC2 (IaaS): customer manages everything above the hypervisor
- RDS (managed): AWS patches the DB engine, customer manages access and data
- Lambda (serverless): AWS manages everything except function code and IAM
- S3: AWS manages durability/availability, customer manages access policies and encryption

**My approach to shared responsibility:**
1. Use managed services where possible (shifts more responsibility to AWS)
2. Automate OS patching (AWS Systems Manager Patch Manager)
3. Principle of least privilege in IAM
4. Encrypt everything (at rest via KMS, in transit via TLS)
5. Enable GuardDuty for threat detection
6. Regular penetration testing of customer-managed components"

---

### Key Interview Terms

| Term | When to Use |
|------|------------|
| **Shared Responsibility Model** | Security discussions |
| **Availability Zone** | High availability design |
| **Region** | Data sovereignty, latency, DR |
| **Principle of Least Privilege** | IAM and security |
| **Defense in depth** | Multi-layer security (SG + NACL + WAF) |
| **Blast radius** | Impact of a failure or security breach |
| **Well-Architected Framework** | AWS design best practices (5 pillars) |
| **CapEx vs OpEx** | Cloud cost model (on-prem vs cloud) |
| **Egress charges** | Data leaving AWS costs money |
| **Landing Zone** | Multi-account AWS setup (Control Tower) |
| **Service Control Policies** | Org-wide guardrails |
| **VPC Endpoints** | Access AWS services without internet |

---

[⬇️ Download This File](#)

---

*Phase 7 Complete. Ready for Phase 8 — Infrastructure as Code.*