# DevOps Phase 8 — Infrastructure as Code

> **Learning Goal:** Master the practice of managing infrastructure through code — understand Terraform deeply, learn Ansible for configuration management, explore Pulumi as a modern alternative, and grasp state management, provisioning patterns, and production workflows.

---

## Table of Contents

1. [What is Infrastructure as Code?](#1-what-is-infrastructure-as-code)
2. [IaC Categories — Provisioning vs Configuration](#2-iac-categories--provisioning-vs-configuration)
3. [Terraform — Complete Deep Dive](#3-terraform--complete-deep-dive)
4. [Terraform State Management](#4-terraform-state-management)
5. [Terraform Modules — Reusable Infrastructure](#5-terraform-modules--reusable-infrastructure)
6. [Terraform in Production — Workflows & Patterns](#6-terraform-in-production--workflows--patterns)
7. [Ansible — Configuration Management](#7-ansible--configuration-management)
8. [Pulumi — IaC with Real Programming Languages](#8-pulumi--iac-with-real-programming-languages)
9. [Provisioning Patterns & Strategies](#9-provisioning-patterns--strategies)
10. [IaC Security Best Practices](#10-iac-security-best-practices)
11. [Comparison — Terraform vs Ansible vs Pulumi vs CloudFormation](#11-comparison--terraform-vs-ansible-vs-pulumi-vs-cloudformation)
12. [Interview Mastery](#12-interview-mastery)

---

## 1. What is Infrastructure as Code?

### Beginner Explanation

Imagine you need to set up a production environment: a VPC with subnets, 3 servers, a load balancer, a database, DNS records, firewall rules, and monitoring. Traditionally, you'd:

1. Log into the AWS console
2. Click through 50+ pages of forms
3. Type in values, select options, click "Create"
4. Repeat for every resource
5. Write down what you did (maybe) in a wiki that goes stale

**Infrastructure as Code means defining all of that in text files** — version-controlled, reviewable, repeatable, and automated. Instead of clicking, you write code that says "create these resources with these settings" and a tool makes it happen.

### The Core Principles

```
INFRASTRUCTURE AS CODE PRINCIPLES:

1. DECLARATIVE DEFINITION
   You describe WHAT you want, not HOW to create it.
   "I want 3 servers" → tool figures out how to create them.

2. VERSION CONTROLLED
   Infrastructure files live in Git.
   Every change is a commit with history, blame, and diff.

3. IDEMPOTENT
   Run the same code 100 times → same result.
   If a resource already exists, don't recreate it.

4. SELF-DOCUMENTING
   The code IS the documentation.
   Want to know what's deployed? Read the code.

5. TESTABLE & REVIEWABLE
   Infrastructure changes go through pull requests.
   Teammates review before applying.

6. REPRODUCIBLE
   Create identical environments (dev, staging, prod) from same code.
   Disaster recovery: rebuild entire infrastructure from code.
```

### Before and After IaC

```
BEFORE IaC:
─────────────────────────────────────────────────────────────
  - "Who changed the security group?" → no one knows
  - "Can you create another environment like staging?" → 3 days of clicking
  - "What's actually deployed in production?" → check 15 AWS console pages
  - "Disaster recovery?" → good luck recreating 200 resources from memory
  - "Audit trail?" → maybe someone updated the wiki (probably not)

AFTER IaC:
─────────────────────────────────────────────────────────────
  - "Who changed the security group?" → git blame
  - "Create another environment?" → terraform apply -var env=staging
  - "What's deployed?" → read main.tf
  - "Disaster recovery?" → terraform apply in new region
  - "Audit trail?" → git log shows every change, who, when, why
```

---

## 2. IaC Categories — Provisioning vs Configuration

### Two Complementary Problems

```
PROVISIONING (Terraform, Pulumi, CloudFormation):
  "Create the infrastructure"
  - Create VPCs, subnets, security groups
  - Launch EC2 instances, RDS databases
  - Create load balancers, DNS records
  - Set up IAM roles, S3 buckets
  → WHAT exists

CONFIGURATION MANAGEMENT (Ansible, Chef, Puppet, Salt):
  "Configure what's on the infrastructure"
  - Install packages (nginx, docker, java)
  - Configure files (/etc/nginx/nginx.conf)
  - Start services (systemctl enable docker)
  - Create users, set permissions
  → HOW it's configured

TYPICAL FLOW:
  Terraform creates EC2 instance → Ansible configures it
  (Or: Terraform creates EC2 with user_data script that self-configures)
  (Or: Terraform creates EC2 with pre-baked AMI — "immutable infrastructure")
```

### Mutable vs Immutable Infrastructure

```
MUTABLE (Configuration Management — Ansible/Chef/Puppet):
  Server exists → Ansible SSHes in → installs/updates software in-place
  
  Over time: servers drift (config drift)
  "Snowflake servers" — each slightly different
  
  Used when: you can't easily replace servers (stateful, pets)

IMMUTABLE (Provisioning + Container/AMI — Terraform + Docker/Packer):
  New version needed → Build new AMI/container → Replace old servers entirely
  
  No drift possible — server IS the image
  "Cattle, not pets" — any server is replaceable
  
  Used when: stateless workloads, containers, modern architecture
  
MODERN PRACTICE:
  Immutable for application servers (containers, AMIs)
  Ansible for initial bootstrapping or legacy systems that can't be containerized
```

---

## 3. Terraform — Complete Deep Dive

### What is Terraform?

Terraform is an **open-source infrastructure provisioning tool** by HashiCorp that lets you define cloud resources in declarative configuration files (HCL — HashiCorp Configuration Language) and manages their lifecycle.

**Key characteristics:**
- **Cloud-agnostic:** Works with AWS, Azure, GCP, Kubernetes, and 3,000+ providers
- **Declarative:** You describe desired state; Terraform figures out how to get there
- **State-based:** Tracks what it created so it knows what to update/destroy
- **Plan before apply:** Shows you what will change before making changes

### How Terraform Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TERRAFORM WORKFLOW                                  │
│                                                                         │
│  1. WRITE                                                               │
│     main.tf, variables.tf, outputs.tf                                   │
│     Define desired infrastructure state                                 │
│              │                                                          │
│              ▼                                                          │
│  2. terraform init                                                      │
│     Downloads provider plugins (AWS, Azure, etc.)                       │
│     Initializes backend (where state is stored)                         │
│              │                                                          │
│              ▼                                                          │
│  3. terraform plan                                                      │
│     Compares desired state (code) vs actual state (state file)          │
│     Shows what will be: created (+), modified (~), destroyed (-)        │
│              │                                                          │
│              ▼                                                          │
│  4. terraform apply                                                     │
│     Executes the plan — creates/modifies/destroys resources             │
│     Updates state file with new actual state                            │
│              │                                                          │
│              ▼                                                          │
│  5. STATE FILE (terraform.tfstate)                                      │
│     JSON file tracking: resource IDs, attributes, dependencies          │
│     Maps your code to real-world resources                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

RECONCILIATION:
  Code says: "3 EC2 instances of type t3.medium"
  State says: "2 EC2 instances exist (i-abc, i-def)"
  Plan: "Will create 1 new EC2 instance"
  Apply: Creates the instance, updates state to show 3 instances
```

### Terraform File Structure

```
my-infrastructure/
├── main.tf              # Primary resource definitions
├── variables.tf         # Input variable declarations
├── outputs.tf           # Output value definitions
├── providers.tf         # Provider configuration (AWS, etc.)
├── terraform.tf         # Terraform settings (version constraints)
├── terraform.tfvars     # Variable values (NOT committed if secrets)
├── backend.tf           # Remote state configuration
├── locals.tf            # Local computed values
│
├── modules/             # Reusable modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── environments/        # Per-environment configurations
    ├── dev/
    │   ├── main.tf
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   └── terraform.tfvars
    └── production/
        ├── main.tf
        └── terraform.tfvars
```

### Complete Terraform Example — Production VPC + EKS

```hcl
# ─── terraform.tf (version constraints) ──────────────────────────────
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/infrastructure.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# ─── providers.tf ────────────────────────────────────────────────────
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project_name
    }
  }
}

# ─── variables.tf ────────────────────────────────────────────────────
variable "aws_region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "project_name" {
  description = "Project name for resource naming and tagging"
  type        = string
  default     = "myapp"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of AZs to use"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "eks_node_instance_types" {
  description = "Instance types for EKS managed node group"
  type        = list(string)
  default     = ["m5.large"]
}

variable "eks_node_desired_size" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 3
}

# ─── locals.tf ───────────────────────────────────────────────────────
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  private_subnets = [for i, az in var.availability_zones :
    cidrsubnet(var.vpc_cidr, 4, i)
  ]

  public_subnets = [for i, az in var.availability_zones :
    cidrsubnet(var.vpc_cidr, 4, i + length(var.availability_zones))
  ]

  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# ─── main.tf (VPC) ──────────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

# Public subnets (one per AZ)
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnets[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                          = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    "kubernetes.io/role/elb"                       = "1"
    "kubernetes.io/cluster/${local.name_prefix}"   = "shared"
  }
}

# Private subnets (one per AZ)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnets[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                              = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    "kubernetes.io/role/internal-elb"                  = "1"
    "kubernetes.io/cluster/${local.name_prefix}"       = "shared"
  }
}

# NAT Gateway (one per AZ for high availability)
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = {
    Name = "${local.name_prefix}-nat-eip-${count.index}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${local.name_prefix}-nat-${var.availability_zones[count.index]}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${local.name_prefix}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${local.name_prefix}-private-rt-${count.index}"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ─── EKS Cluster ──────────────────────────────────────────────────────
resource "aws_eks_cluster" "main" {
  name     = local.name_prefix
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = true
    security_group_ids      = [aws_security_group.eks_cluster.id]
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_service_policy,
  ]
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.name_prefix}-workers"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id
  instance_types  = var.eks_node_instance_types

  scaling_config {
    desired_size = var.eks_node_desired_size
    max_size     = var.eks_node_desired_size * 3
    min_size     = 2
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]
}

# ─── IAM Roles for EKS ───────────────────────────────────────────────
resource "aws_iam_role" "eks_cluster" {
  name = "${local.name_prefix}-eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_iam_role_policy_attachment" "eks_service_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_iam_role" "eks_nodes" {
  name = "${local.name_prefix}-eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "ecr_read_only" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes.name
}

# ─── Security Group for EKS ──────────────────────────────────────────
resource "aws_security_group" "eks_cluster" {
  name_prefix = "${local.name_prefix}-eks-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.name_prefix}-eks-cluster-sg"
  }
}

# ─── outputs.tf ──────────────────────────────────────────────────────
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "eks_cluster_endpoint" {
  description = "EKS cluster API endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.main.name
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}
```

### Terraform Core Concepts

#### Resources

```hcl
# A resource is a single infrastructure object
resource "aws_instance" "web" {    # resource_type.local_name
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.private[0].id

  tags = {
    Name = "web-server"
  }
}

# Reference another resource's attributes
resource "aws_eip" "web" {
  instance = aws_instance.web.id   # References the EC2 instance above
}
```

#### Data Sources (Read Existing Resources)

```hcl
# Look up existing resources (not managed by this Terraform)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["existing-vpc"]
  }
}

# Use data source in a resource
resource "aws_instance" "web" {
  ami       = data.aws_ami.amazon_linux.id   # Uses the looked-up AMI
  subnet_id = data.aws_vpc.existing.id
}
```

#### Variables and Types

```hcl
# String
variable "environment" {
  type    = string
  default = "dev"
}

# Number
variable "instance_count" {
  type    = number
  default = 3
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map
variable "instance_types" {
  type = map(string)
  default = {
    dev        = "t3.small"
    staging    = "t3.medium"
    production = "m5.large"
  }
}

# Object (complex type)
variable "database_config" {
  type = object({
    engine         = string
    instance_class = string
    allocated_storage = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    instance_class = "db.t3.medium"
    allocated_storage = 50
    multi_az       = true
  }
}

# Usage
resource "aws_instance" "web" {
  instance_type = var.instance_types[var.environment]
  count         = var.instance_count
}
```

#### Loops and Conditionals

```hcl
# count — create N copies (simple repetition)
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = var.availability_zones[count.index]
}

# for_each — create from a map/set (better for named resources)
variable "s3_buckets" {
  type = map(object({
    versioning = bool
  }))
  default = {
    "logs"    = { versioning = false }
    "backups" = { versioning = true }
    "assets"  = { versioning = true }
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets
  bucket   = "${var.project_name}-${each.key}-${var.environment}"
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.s3_buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.buckets[each.key].id

  versioning_configuration {
    status = "Enabled"
  }
}

# Conditional creation (create resource only if condition is true)
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0   # 1 if true, 0 if false

  alarm_name          = "${local.name_prefix}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}

# for expression (transform lists/maps)
locals {
  subnet_ids = [for s in aws_subnet.private : s.id]
  
  instance_ips = { for i in aws_instance.web : i.tags["Name"] => i.private_ip }
}
```

#### Lifecycle Rules

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"

  lifecycle {
    # Create new before destroying old (zero-downtime replacement)
    create_before_destroy = true

    # Ignore changes to these attributes (prevent drift-induced replacements)
    ignore_changes = [ami, tags["LastModified"]]

    # Prevent accidental destruction
    prevent_destroy = true
  }
}
```

### Essential Terraform Commands

```bash
# Initialize (download providers, configure backend)
terraform init

# Format code (auto-fixes style)
terraform fmt -recursive

# Validate syntax (no API calls)
terraform validate

# Plan (show what will change)
terraform plan
terraform plan -out=plan.tfplan              # Save plan to file
terraform plan -var="environment=production"  # Override variable

# Apply (make changes)
terraform apply
terraform apply plan.tfplan                  # Apply saved plan (no re-prompt)
terraform apply -auto-approve                # Skip confirmation (CI/CD only)
terraform apply -target=aws_instance.web     # Apply single resource

# Destroy (tear down everything)
terraform destroy
terraform destroy -target=aws_instance.web   # Destroy single resource

# State inspection
terraform state list                    # List all managed resources
terraform state show aws_vpc.main       # Show details of one resource
terraform state mv aws_instance.old aws_instance.new  # Rename in state
terraform state rm aws_instance.orphan  # Remove from state (don't destroy)

# Import existing resource into Terraform management
terraform import aws_instance.web i-1234567890abcdef0

# Output values
terraform output
terraform output eks_cluster_endpoint

# Workspace (multiple states from same code)
terraform workspace new staging
terraform workspace select production
terraform workspace list
```

---

## 4. Terraform State Management

### What is State?

Terraform state is a JSON file (`terraform.tfstate`) that maps your configuration to real-world resources. It records:
- Resource IDs (the actual AWS resource identifiers)
- Resource attributes (IP addresses, ARNs, etc.)
- Dependencies between resources
- Metadata for performance optimization

### Why State Exists

```
WITHOUT STATE:
  Terraform would have to query every possible resource in your cloud account
  to figure out what exists and what matches your configuration.
  With 1000+ resources in an AWS account, this would take minutes and be unreliable.

WITH STATE:
  Terraform knows exactly what it manages and what currently exists.
  Plan operation: compare code ↔ state = instant diff.
  
  State tracks: "resource 'aws_instance.web' = i-abc123 in us-east-1"
  So terraform plan can say: "this resource exists, needs these changes"
  Instead of scanning all EC2 instances trying to find a match.
```

### Remote State (CRITICAL for Teams)

```
LOCAL STATE (default — NEVER use in teams):
  terraform.tfstate file on your laptop
  ❌ No collaboration (two people can't work on same infra)
  ❌ No locking (concurrent applies corrupt state)
  ❌ No backup (laptop dies = state lost = orphaned resources)

REMOTE STATE (required for any real project):
  State stored in S3/GCS/Azure Blob with locking via DynamoDB/GCS
  ✅ Team collaboration (everyone reads same state)
  ✅ Locking (only one apply at a time)
  ✅ Encryption at rest
  ✅ Versioning (recover from corruption)
  ✅ Backup built-in
```

### Remote State Configuration (AWS)

```hcl
# backend.tf — S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"    # Prevents concurrent applies
    encrypt        = true                 # Encrypt state at rest
  }
}
```

```bash
# Create the backend infrastructure (do this once, manually or with a bootstrap script)

# S3 bucket for state
aws s3api create-bucket \
    --bucket my-company-terraform-state \
    --region us-east-1

aws s3api put-bucket-versioning \
    --bucket my-company-terraform-state \
    --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
    --bucket my-company-terraform-state \
    --server-side-encryption-configuration '{
        "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
    }'

# DynamoDB table for locking
aws dynamodb create-table \
    --table-name terraform-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

### State Splitting Strategy

```
DON'T: One giant state file for everything (blast radius = entire infrastructure)
DO:    Split state by lifecycle, team, or risk level

RECOMMENDED STRUCTURE:

terraform-state-bucket/
├── network/terraform.tfstate         # VPC, subnets, NAT (rarely changes)
├── security/terraform.tfstate        # IAM roles, policies (sensitive)
├── eks/terraform.tfstate             # EKS cluster (changes occasionally)
├── databases/terraform.tfstate       # RDS, ElastiCache (critical data)
├── applications/
│   ├── api/terraform.tfstate         # API service infra
│   └── web/terraform.tfstate         # Web service infra
└── monitoring/terraform.tfstate      # CloudWatch, alerts

BENEFITS:
  - Smaller blast radius (bad apply only affects one area)
  - Faster plans (fewer resources to check)
  - Independent team ownership
  - Different apply cadences (network: weekly, apps: daily)

CROSS-STATE REFERENCES (data source):
  # In eks/main.tf — read outputs from the network state
  data "terraform_remote_state" "network" {
    backend = "s3"
    config = {
      bucket = "my-company-terraform-state"
      key    = "network/terraform.tfstate"
      region = "us-east-1"
    }
  }

  # Use outputs from the network state
  resource "aws_eks_cluster" "main" {
    subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
  }
```

---

## 5. Terraform Modules — Reusable Infrastructure

### What is a Module?

A module is a **reusable, self-contained package** of Terraform configuration. It's like a function in programming — you call it with inputs and get outputs.

```
WITHOUT MODULES:
  Copy-paste VPC code for dev, staging, prod
  50 resources × 3 environments = 150 resources to maintain separately
  Fix a bug → must fix in 3 places

WITH MODULES:
  Define VPC once as a module
  Call it 3 times with different parameters
  Fix a bug → fix once, all environments get it
```

### Module Structure

```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "Name prefix for all resources"
  type        = string
}

variable "cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "azs" {
  description = "Availability zones"
  type        = list(string)
}

variable "private_subnets" {
  description = "Private subnet CIDR blocks"
  type        = list(string)
}

variable "public_subnets" {
  description = "Public subnet CIDR blocks"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Create NAT gateways for private subnets"
  type        = bool
  default     = true
}

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.name}-vpc" }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = { Name = "${var.name}-private-${var.azs[count.index]}" }
}

# ... (NAT gateways, route tables, etc.)

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### Using Modules

```hcl
# environments/production/main.tf
module "vpc" {
  source = "../../modules/vpc"

  name            = "production"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
}

module "vpc_dev" {
  source = "../../modules/vpc"

  name            = "dev"
  cidr            = "10.1.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.1.1.0/24", "10.1.2.0/24"]
  public_subnets  = ["10.1.101.0/24", "10.1.102.0/24"]
  enable_nat_gateway = false   # Save cost in dev
}

# Reference module outputs
resource "aws_eks_cluster" "main" {
  subnet_ids = module.vpc.private_subnet_ids
}
```

### Public Module Registry

```hcl
# Use community modules from the Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.4.0"   # Always pin versions!

  name = "production"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false   # One per AZ for HA
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.21.0"

  cluster_name    = "production"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    workers = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
    }
  }
}
```

---

## 6. Terraform in Production — Workflows & Patterns

### CI/CD Pipeline for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'
  push:
    branches:
      - main
    paths:
      - 'infrastructure/**'

env:
  TF_WORKING_DIR: infrastructure/production

jobs:
  # ─── On PR: Plan and comment ────────────────────────────────
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    permissions:
      id-token: write      # OIDC
      contents: read
      pull-requests: write  # Comment on PR

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-terraform
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.6

      - name: Terraform Init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init

      - name: Terraform Plan
        working-directory: ${{ env.TF_WORKING_DIR }}
        id: plan
        run: terraform plan -no-color -out=plan.tfplan
        continue-on-error: true

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pushed by: @${{ github.actor }}*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  # ─── On merge to main: Apply ──────────────────────────────
  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production    # Requires approval

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-terraform
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init

      - name: Terraform Apply
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform apply -auto-approve
```

### GitOps Workflow for Terraform

```
DEVELOPER WORKFLOW:

1. Create feature branch
2. Modify Terraform code
3. Open Pull Request
4. CI runs: terraform fmt check + terraform validate + terraform plan
5. Plan output posted as PR comment
6. Team reviews the plan
7. Merge to main
8. CI runs: terraform apply (with required approval in GitHub Environment)
9. Infrastructure updated

RULES:
  - No one runs terraform apply locally
  - All changes go through PRs
  - Plan is visible and reviewed before apply
  - Apply requires manual approval for production
  - State is locked during apply (no concurrent changes)
```

### Handling Drift

```bash
# Drift = someone changed infrastructure outside Terraform (console, CLI)

# Detect drift
terraform plan
# Shows: "Resource has been changed outside of Terraform"

# Option 1: Overwrite manual change (make reality match code)
terraform apply   # Reverts the manual change

# Option 2: Import the change into state (make code match reality)
# Update your .tf file to match the current state, then:
terraform plan   # Should show no changes

# Option 3: Refresh state (update state to match reality without changing infra)
terraform apply -refresh-only

# PREVENTION:
# - All changes MUST go through Terraform (enforce via SCPs/policies)
# - Use read-only AWS console access for most users
# - Periodic drift detection in CI (terraform plan → alert if changes detected)
```

---

## 7. Ansible — Configuration Management

### What is Ansible?

Ansible is an **agentless configuration management tool** that automates:
- Software installation and configuration
- System administration tasks
- Application deployment
- Orchestration of multi-step workflows

**Key differentiator:** Agentless — uses SSH (Linux) or WinRM (Windows). No agent software to install on targets.

### Ansible Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  CONTROL NODE (your laptop or CI server)                    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Playbook    │  │  Inventory   │  │   Roles      │      │
│  │  (tasks.yml) │  │  (hosts)     │  │  (reusable)  │      │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘      │
│         │                  │                                │
│         └──────────────────┘                                │
│                    │                                        │
└────────────────────┼────────────────────────────────────────┘
                     │ SSH (no agent needed on targets)
          ┌──────────┼──────────────┐
          │          │              │
          ▼          ▼              ▼
     ┌─────────┐ ┌─────────┐ ┌─────────┐
     │ Web-1   │ │ Web-2   │ │ DB-1    │
     │ (target)│ │ (target)│ │ (target)│
     └─────────┘ └─────────┘ └─────────┘
     
     No agent installed — Ansible SSHes in,
     runs Python commands, exits.
```

### Inventory (What Hosts to Manage)

```ini
# inventory/production.ini

[webservers]
web-1.example.com ansible_host=10.0.1.10
web-2.example.com ansible_host=10.0.1.11
web-3.example.com ansible_host=10.0.1.12

[databases]
db-primary.example.com ansible_host=10.0.2.10
db-replica.example.com ansible_host=10.0.2.11

[monitoring]
prometheus.example.com ansible_host=10.0.3.10

[webservers:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key
app_port=8080

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# inventory/production.yml (YAML format — more flexible)
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3
  children:
    webservers:
      hosts:
        web-1:
          ansible_host: 10.0.1.10
        web-2:
          ansible_host: 10.0.1.11
      vars:
        app_port: 8080
    databases:
      hosts:
        db-primary:
          ansible_host: 10.0.2.10
          db_role: primary
        db-replica:
          ansible_host: 10.0.2.11
          db_role: replica
```

### Playbook (What to Do)

```yaml
# playbooks/setup-webserver.yml
---
- name: Configure web servers
  hosts: webservers
  become: yes              # Run as root (sudo)
  
  vars:
    app_name: myapp
    app_version: "2.1.0"
    nginx_worker_processes: auto
  
  tasks:
    # ─── System Updates ─────────────────────────────────────
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install required packages
      apt:
        name:
          - nginx
          - docker.io
          - python3-pip
          - curl
          - jq
        state: present

    # ─── Docker Setup ───────────────────────────────────────
    - name: Start and enable Docker
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Add deploy user to docker group
      user:
        name: deploy
        groups: docker
        append: yes

    # ─── Nginx Configuration ────────────────────────────────
    - name: Deploy nginx configuration
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
        mode: '0644'
      notify: Reload nginx              # Trigger handler on change

    - name: Enable site
      file:
        src: /etc/nginx/sites-available/{{ app_name }}
        dest: /etc/nginx/sites-enabled/{{ app_name }}
        state: link
      notify: Reload nginx

    # ─── Application Deployment ─────────────────────────────
    - name: Pull application Docker image
      docker_image:
        name: "registry.example.com/{{ app_name }}"
        tag: "{{ app_version }}"
        source: pull

    - name: Run application container
      docker_container:
        name: "{{ app_name }}"
        image: "registry.example.com/{{ app_name }}:{{ app_version }}"
        state: started
        restart_policy: unless-stopped
        ports:
          - "{{ app_port }}:8080"
        env:
          NODE_ENV: production
          DATABASE_URL: "{{ vault_database_url }}"

    # ─── Health Check ───────────────────────────────────────
    - name: Wait for application to be healthy
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health_check
      until: health_check.status == 200
      retries: 30
      delay: 5

  # ─── Handlers (run once at end, only if notified) ─────────
  handlers:
    - name: Reload nginx
      systemd:
        name: nginx
        state: reloaded
```

### Ansible Roles (Reusable Components)

```
roles/
├── common/                    # Base configuration for ALL servers
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   ├── templates/
│   ├── files/
│   └── defaults/main.yml     # Default variables (overridable)
│
├── nginx/                     # Nginx installation and config
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   ├── templates/nginx.conf.j2
│   └── defaults/main.yml
│
└── docker/                    # Docker installation
    ├── tasks/main.yml
    ├── handlers/main.yml
    └── defaults/main.yml
```

```yaml
# Using roles in a playbook
- name: Configure web servers
  hosts: webservers
  become: yes
  
  roles:
    - common
    - docker
    - nginx
    - role: app_deploy
      vars:
        app_version: "2.1.0"
```

### Ansible Vault (Secrets)

```bash
# Encrypt a file
ansible-vault encrypt secrets.yml

# Decrypt
ansible-vault decrypt secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Run playbook with vault password
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Encrypt a single variable
ansible-vault encrypt_string 'super-secret-password' --name 'db_password'
```

### Essential Ansible Commands

```bash
# Run a playbook
ansible-playbook -i inventory/production.yml playbooks/deploy.yml

# Check mode (dry run — no changes made)
ansible-playbook playbook.yml --check --diff

# Limit to specific hosts
ansible-playbook playbook.yml --limit web-1

# Run ad-hoc command on all servers
ansible all -i inventory.yml -m ping
ansible webservers -m shell -a "df -h"
ansible databases -m service -a "name=postgresql state=restarted"

# List hosts that would be affected
ansible-playbook playbook.yml --list-hosts
```

---

## 8. Pulumi — IaC with Real Programming Languages

### What is Pulumi?

Pulumi is an IaC tool that lets you define infrastructure using **real programming languages** (Python, TypeScript, Go, C#, Java) instead of a domain-specific language like HCL.

### Why Pulumi?

```
TERRAFORM (HCL):
  - DSL — limited constructs (no real loops, functions are basic)
  - Good for simple infrastructure
  - Hard for complex logic (string manipulation, conditionals)
  - Separate testing framework needed

PULUMI (real languages):
  - Use Python/TypeScript/Go — real loops, classes, functions
  - Use your IDE's autocomplete, type checking, linting
  - Test with your language's testing framework (pytest, jest)
  - Import existing libraries (call APIs, generate configs)
  - Same state management concept as Terraform
```

### Pulumi Example (Python)

```python
# __main__.py — equivalent to the Terraform VPC+EKS example
import pulumi
import pulumi_aws as aws
import pulumi_eks as eks

# ─── Configuration ──────────────────────────────────────────
config = pulumi.Config()
environment = config.require("environment")
project_name = config.get("projectName") or "myapp"
name_prefix = f"{project_name}-{environment}"

# ─── VPC ────────────────────────────────────────────────────
vpc = aws.ec2.Vpc(f"{name_prefix}-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={"Name": f"{name_prefix}-vpc"}
)

# Create subnets in a loop (real Python!)
azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
private_subnets = []
public_subnets = []

for i, az in enumerate(azs):
    public_subnet = aws.ec2.Subnet(f"{name_prefix}-public-{az}",
        vpc_id=vpc.id,
        cidr_block=f"10.0.{i + 100}.0/24",
        availability_zone=az,
        map_public_ip_on_launch=True,
        tags={"Name": f"{name_prefix}-public-{az}"}
    )
    public_subnets.append(public_subnet)
    
    private_subnet = aws.ec2.Subnet(f"{name_prefix}-private-{az}",
        vpc_id=vpc.id,
        cidr_block=f"10.0.{i}.0/24",
        availability_zone=az,
        tags={"Name": f"{name_prefix}-private-{az}"}
    )
    private_subnets.append(private_subnet)

# ─── EKS Cluster (using component library) ──────────────────
cluster = eks.Cluster(f"{name_prefix}-cluster",
    vpc_id=vpc.id,
    subnet_ids=[s.id for s in private_subnets],
    instance_type="m5.large",
    desired_capacity=3,
    min_size=2,
    max_size=10,
)

# ─── Outputs ────────────────────────────────────────────────
pulumi.export("vpc_id", vpc.id)
pulumi.export("cluster_name", cluster.core.cluster.name)
pulumi.export("kubeconfig", cluster.kubeconfig)
```

### Pulumi Example (TypeScript)

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as eks from "@pulumi/eks";

const config = new pulumi.Config();
const environment = config.require("environment");
const namePrefix = `myapp-${environment}`;

// VPC
const vpc = new aws.ec2.Vpc(`${namePrefix}-vpc`, {
    cidrBlock: "10.0.0.0/16",
    enableDnsHostnames: true,
    tags: { Name: `${namePrefix}-vpc` },
});

// Subnets with TypeScript logic
const azs = ["us-east-1a", "us-east-1b", "us-east-1c"];
const privateSubnets = azs.map((az, i) =>
    new aws.ec2.Subnet(`${namePrefix}-private-${az}`, {
        vpcId: vpc.id,
        cidrBlock: `10.0.${i}.0/24`,
        availabilityZone: az,
        tags: { Name: `${namePrefix}-private-${az}` },
    })
);

// EKS Cluster
const cluster = new eks.Cluster(`${namePrefix}-cluster`, {
    vpcId: vpc.id,
    subnetIds: privateSubnets.map(s => s.id),
    instanceType: "m5.large",
    desiredCapacity: 3,
    minSize: 2,
    maxSize: 10,
});

export const kubeconfig = cluster.kubeconfig;
export const clusterName = cluster.core.cluster.name;
```

### Pulumi Commands

```bash
# Initialize new project
pulumi new aws-python    # or aws-typescript, aws-go

# Preview changes (like terraform plan)
pulumi preview

# Deploy (like terraform apply)
pulumi up

# Destroy
pulumi destroy

# Stack management (like terraform workspaces)
pulumi stack init staging
pulumi stack select production
pulumi stack ls
```

---

## 9. Provisioning Patterns & Strategies

### Pattern 1: Environment Parity

```
SAME CODE, DIFFERENT VALUES:

modules/          (shared code)
├── vpc/
├── eks/
└── rds/

environments/
├── dev/
│   └── terraform.tfvars    (small instances, 1 AZ, no HA)
├── staging/
│   └── terraform.tfvars    (medium instances, 2 AZs, limited HA)
└── production/
    └── terraform.tfvars    (large instances, 3 AZs, full HA)

# dev/terraform.tfvars
environment         = "dev"
instance_type       = "t3.small"
eks_node_count      = 2
multi_az            = false
rds_instance_class  = "db.t3.small"

# production/terraform.tfvars
environment         = "production"
instance_type       = "m5.large"
eks_node_count      = 6
multi_az            = true
rds_instance_class  = "db.r5.xlarge"
```

### Pattern 2: Terragrunt (DRY Terraform)

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# environments/production/vpc/terragrunt.hcl
terraform {
  source = "../../../modules/vpc"
}

include "root" {
  path = find_in_parent_folders()
}

inputs = {
  name        = "production"
  cidr        = "10.0.0.0/16"
  azs         = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

### Pattern 3: Immutable Infrastructure with Packer + Terraform

```
BUILD AMI (Packer) → PROVISION INFRASTRUCTURE (Terraform)

Step 1: Packer builds a pre-configured AMI
  - Install Docker, nginx, monitoring agent
  - Bake application code
  - Run tests on the image
  - Publish AMI: ami-newversion

Step 2: Terraform uses the new AMI
  - Update launch template with new AMI ID
  - Rolling replacement of instances (ASG)
  - Old instances terminated, new ones launched

NO configuration management at runtime.
Every instance is identical because they boot from the same image.
```

```json
// packer/app.pkr.hcl
source "amazon-ebs" "app" {
  ami_name      = "myapp-{{timestamp}}"
  instance_type = "t3.medium"
  region        = "us-east-1"
  source_ami_filter {
    filters = {
      name = "amzn2-ami-hvm-*-x86_64-gp2"
    }
    owners = ["amazon"]
    most_recent = true
  }
  ssh_username = "ec2-user"
}

build {
  sources = ["source.amazon-ebs.app"]

  provisioner "shell" {
    scripts = [
      "scripts/install-docker.sh",
      "scripts/install-monitoring.sh",
      "scripts/pull-app-image.sh"
    ]
  }
}
```

---

## 10. IaC Security Best Practices

### Secret Management

```hcl
# ❌ NEVER hardcode secrets in Terraform files
resource "aws_db_instance" "bad" {
  password = "super-secret-123"   # This ends up in state file AND git!
}

# ✅ Option 1: Variable with no default (prompted or from tfvars)
variable "db_password" {
  type      = string
  sensitive = true    # Prevents showing in plan output
}

# ✅ Option 2: Read from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "good" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}

# ✅ Option 3: Environment variable (TF_VAR_db_password)
# Set in CI/CD: export TF_VAR_db_password=${{ secrets.DB_PASSWORD }}
```

### State File Security

```
THE STATE FILE CONTAINS SECRETS IN PLAINTEXT.
Even if you use "sensitive = true", the value is in the state file.

MITIGATIONS:
1. Remote backend with encryption (S3 + encrypt=true)
2. Restrict access to state bucket (IAM policy)
3. Enable versioning on state bucket (recover from corruption)
4. Never commit terraform.tfstate to git
5. Use terraform_remote_state data source for cross-state references
   (instead of sharing the actual state file)
```

### Policy as Code (Sentinel / OPA)

```
# Enforce policies BEFORE terraform apply:
# - No public S3 buckets
# - All resources must be tagged
# - No overly permissive security groups
# - Instance types must be from approved list

# Example with OPA (Open Policy Agent):
# policy/no_public_s3.rego
package terraform

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.acl == "public-read"
  msg := sprintf("S3 bucket '%s' must not be public", [resource.name])
}
```

---

## 11. Comparison — Terraform vs Ansible vs Pulumi vs CloudFormation

| Feature | Terraform | Ansible | Pulumi | CloudFormation |
|---------|-----------|---------|--------|----------------|
| **Type** | Provisioning | Configuration Mgmt | Provisioning | Provisioning |
| **Language** | HCL | YAML | Python/TS/Go/C# | YAML/JSON |
| **Cloud support** | Multi-cloud (3000+ providers) | Multi-cloud | Multi-cloud | AWS only |
| **State** | State file (remote) | Stateless (checks each run) | State file (remote) | Managed by AWS |
| **Agent** | Agentless | Agentless (SSH) | Agentless | N/A (AWS service) |
| **Execution** | Plan → Apply | Push (SSH into targets) | Preview → Up | Change Sets → Execute |
| **Idempotent** | Yes (declarative) | Yes (most modules) | Yes (declarative) | Yes (declarative) |
| **Learning curve** | Medium (HCL is simple) | Low (YAML) | Depends on language | Medium |
| **Testing** | terraform validate, terratest | molecule, testinfra | pytest, jest (native) | cfn-lint, taskcat |
| **Drift detection** | terraform plan | Re-run playbook | pulumi preview | Stack drift detection |
| **Best for** | Cloud infrastructure | Server configuration | Complex logic, devs | AWS-only shops |
| **Community** | Massive | Massive | Growing | AWS-focused |

### When to Use What

```
TERRAFORM:
  ✅ Creating cloud resources (VPC, EC2, RDS, EKS)
  ✅ Multi-cloud environments
  ✅ Infrastructure that needs state tracking
  ✅ Team needs plan/apply workflow with approvals

ANSIBLE:
  ✅ Configuring servers after creation (install packages, manage files)
  ✅ Legacy systems that can't be containerized
  ✅ One-off operational tasks (restart services, rotate certs)
  ✅ Configuration of network devices (routers, switches)

PULUMI:
  ✅ Development teams who prefer real programming languages
  ✅ Complex infrastructure logic (conditionals, loops, abstractions)
  ✅ Teams that want to test infrastructure with standard test frameworks
  ✅ Kubernetes-heavy environments (strong K8s provider)

CLOUDFORMATION:
  ✅ AWS-only, deep integration needed (StackSets, custom resources)
  ✅ AWS Enterprise support (AWS will help debug CloudFormation)
  ✅ When staying within AWS ecosystem is a priority

COMMON COMBINATION:
  Terraform for infrastructure provisioning
  + Ansible for legacy server configuration
  + Helm for Kubernetes application deployment
```

---

## 12. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: What is Infrastructure as Code and why is it important?**

**Perfect Answer:**

"Infrastructure as Code is the practice of managing and provisioning infrastructure through machine-readable definition files rather than manual processes or interactive configuration tools.

It's important for five reasons:

1. **Reproducibility:** You can recreate identical environments on demand. Disaster recovery becomes 'terraform apply in a new region' instead of 'try to remember what we configured.'

2. **Version control:** Infrastructure changes go through Git — you have full history, code review, and the ability to revert. 'Who changed the security group last Thursday?' becomes a git blame, not a mystery.

3. **Consistency:** Dev, staging, and production use the same code with different parameters. This eliminates 'works in staging, breaks in production' configuration drift.

4. **Automation:** Infrastructure changes happen through CI/CD pipelines — no manual clicking, no human error, no reliance on tribal knowledge.

5. **Documentation:** The code IS the documentation. It's always accurate because it's what's actually deployed. Wiki pages go stale; running code doesn't.

The tradeoff is upfront investment in learning the tools and establishing workflows, but the payoff in reliability, speed, and safety is enormous — especially as team size and infrastructure complexity grow."

---

**Q2: Explain the Terraform workflow — what happens when you run init, plan, and apply?**

**Perfect Answer:**

"The Terraform workflow has three main stages:

**`terraform init`** — Initialization:
- Downloads provider plugins (AWS, Azure, etc.) based on your provider blocks
- Configures the backend (where state is stored — S3, GCS, etc.)
- Downloads modules from registries or local paths
- Creates the `.terraform` directory with downloaded plugins
- This is run once per project, or when providers/backend change

**`terraform plan`** — Planning:
- Reads current state (from remote backend)
- Reads your configuration files (.tf files)
- Compares desired state (code) vs actual state (state file)
- Generates an execution plan showing what will be:
  - `+` created (new resources)
  - `~` modified in-place (attribute changes)
  - `-/+` replaced (destroy and recreate, often due to immutable attributes)
  - `-` destroyed
- No changes are made — this is read-only and safe

**`terraform apply`** — Execution:
- Re-runs the plan (or uses a saved plan file)
- Prompts for confirmation (unless `-auto-approve`)
- Creates, modifies, or destroys resources via cloud API calls
- Respects dependency order (creates VPC before subnet)
- Updates the state file with new resource attributes
- Outputs any defined output values

The plan step is the critical safety mechanism — it lets you and your team review exactly what will change before anything happens. In CI/CD, we save the plan as a file (`-out=plan.tfplan`) and apply that exact plan to avoid any gap between what was reviewed and what's applied."

---

**Q3: What is Terraform state and why does it exist?**

**Perfect Answer:**

"Terraform state is a JSON file that maps your configuration to real-world resources. It tracks which cloud resources Terraform manages, their current attributes, and relationships between them.

It exists for three reasons:

1. **Performance:** Without state, Terraform would need to query every resource in your cloud account to determine what exists. With 500 resources, that could take minutes. State makes `terraform plan` instant because it already knows what exists.

2. **Resource tracking:** State maps your logical names (`aws_instance.web`) to physical IDs (`i-abc123`). This is how Terraform knows which real resource corresponds to which block in your code.

3. **Dependency resolution:** State tracks the dependency graph — which resources depend on others. This enables correct ordering during creation and destruction.

**Critical practices for state:**
- Never store locally in a team environment — use remote state (S3, GCS)
- Always enable locking (DynamoDB) to prevent concurrent applies
- Enable encryption (state contains sensitive values in plaintext)
- Enable versioning (recover from state corruption)
- Split state by component (reduce blast radius)
- Never commit state to Git (contains secrets)
- Never manually edit the state file (use `terraform state` commands)"

---

### Intermediate Interview Questions

---

**Q4: How do you handle secrets in Terraform?**

**Perfect Answer:**

"This is one of the trickiest aspects of IaC because Terraform state stores all values in plaintext, including sensitive ones.

**The problem:** Even if you mark a variable as `sensitive = true`, the actual value is still in the state file. Anyone with state access can read it.

**My approach (layered):**

1. **Never hardcode secrets in .tf files or terraform.tfvars committed to Git.**
Use environment variables (`TF_VAR_xxx`) or a secrets file excluded from Git.

2. **Read secrets from a secrets manager at apply time:**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}
```
The secret value still ends up in state, but it's never in your code.

3. **Secure the state file:**
- Encrypted S3 bucket with restrictive IAM policies
- Only CI/CD role can read/write state
- Developers can't access state directly

4. **For truly sensitive resources (like RDS passwords):**
- Use `ignore_changes` lifecycle and let the secrets manager rotate independently
- Or use `random_password` resource with `sensitive = true` and let only the application read from Secrets Manager

5. **In CI/CD:**
- Secrets injected via GitHub Secrets or Vault
- OIDC for AWS authentication (no stored credentials)
- Plan output filtered to not show sensitive values

The bottom line: there's no perfect solution. The state file is the weak point. The best mitigation is strict access control on the state backend and using external secrets managers so that secrets aren't hardcoded anywhere in your codebase."

---

**Q5: Explain Terraform modules. How do you structure a large Terraform project?**

**Perfect Answer:**

"Modules are reusable packages of Terraform configuration — the equivalent of functions or libraries in programming. They accept inputs (variables), create resources, and expose outputs.

**Why modules:**
- DRY principle — define once, use everywhere
- Abstraction — hide complexity behind a simple interface
- Consistency — all VPCs are created the same way
- Versioning — pin module versions, upgrade deliberately

**My project structure for a large organization:**

```
terraform/
├── modules/                    # Internal reusable modules
│   ├── vpc/                   # VPC module
│   ├── eks-cluster/           # EKS with our standard config
│   ├── rds-postgres/          # Postgres with encryption, backups
│   └── monitoring-stack/      # CloudWatch + alerts
│
├── environments/              # Live infrastructure
│   ├── shared/                # Shared services (DNS, IAM)
│   │   ├── main.tf
│   │   └── backend.tf
│   ├── production/
│   │   ├── network/          # Separate state per layer
│   │   ├── compute/
│   │   └── databases/
│   └── staging/
│       ├── network/
│       ├── compute/
│       └── databases/
│
└── terragrunt.hcl            # DRY configuration (optional)
```

**Key principles:**

1. **State isolation:** Each directory has its own state. Network changes can't accidentally destroy databases.

2. **Module versioning:** Internal modules use Git tags. Upgrade by changing the version, not by pulling latest.

3. **Environment parity:** Same modules called with different variables for dev/staging/prod.

4. **Composition over inheritance:** Small, focused modules composed together rather than one giant module.

5. **Remote state references:** Cross-layer communication via `terraform_remote_state` data source."

---

**Q6: Compare Terraform and Ansible. When would you use each?**

**Perfect Answer:**

"They solve different but complementary problems:

**Terraform** excels at provisioning — creating and managing cloud infrastructure (VPCs, EC2 instances, databases, load balancers). It's declarative and stateful: you declare what should exist, and Terraform figures out how to get from current state to desired state.

**Ansible** excels at configuration — installing packages, managing files, starting services, configuring running systems. It's also declarative (idempotent modules) but stateless: it doesn't track what it previously configured, it just converges the system to the desired state each run.

**When I use Terraform:**
- Creating cloud resources (anything with a provider)
- Infrastructure lifecycle management (create, update, destroy)
- When I need to track state and dependencies
- Multi-cloud resource orchestration

**When I use Ansible:**
- Configuring existing servers (install packages, manage configs)
- Operational tasks (restart services, deploy applications)
- Managing network devices (switches, routers)
- One-off procedures (certificate rotation, emergency patching)
- When targets don't have a cloud API (bare metal, legacy systems)

**Common pattern:** Terraform creates the infrastructure, Ansible configures it.
But in modern architectures, Ansible's role shrinks because:
- Containers eliminate the need to configure servers
- Cloud services are pre-configured (RDS, EKS)
- User-data scripts or Packer AMIs handle initial configuration
- Kubernetes manifests configure application deployment

So in a containerized environment, it's often: Terraform for infra + Helm for app deployment, with Ansible reserved for legacy systems."

---

### Advanced Interview Questions

---

**Q7: How do you manage Terraform at scale with 50+ engineers and hundreds of resources?**

**Perfect Answer:**

"At scale, the challenges are: state contention, blast radius, code organization, governance, and developer velocity. Here's my approach:

**1. State splitting strategy:**
Split by risk/lifecycle, not by team. Each state file is a blast radius boundary:
- Network (rarely changes, high blast radius)
- Security/IAM (sensitive, limited access)
- Per-service infrastructure (each service team owns their state)
- Shared services (monitoring, logging)

**2. Module library (internal registry):**
A central Git repo of approved modules with semantic versioning. Services consume modules like library dependencies. Changes to modules go through review, are versioned, and consuming teams upgrade deliberately.

**3. CI/CD pipeline:**
- PR: `terraform plan` runs, output posted as PR comment
- Merge: `terraform apply` with required approvals for production
- Drift detection: scheduled plan that alerts on unexpected differences
- No one runs apply locally (enforced via IAM)

**4. Policy enforcement:**
- Sentinel or OPA policies run during plan phase
- Enforce: mandatory tags, approved instance types, no public S3, encryption everywhere
- Prevents non-compliant infrastructure from being created

**5. Workspace/directory structure:**
Using Terragrunt or a monorepo with per-environment directories. Each team's infrastructure is isolated but uses shared modules.

**6. State locking and access:**
- DynamoDB locking prevents concurrent applies
- State bucket IAM: per-team access to their state files only
- CI/CD service account is the only principal with apply permissions

**7. Developer experience:**
- Pre-commit hooks: `terraform fmt`, `terraform validate`, `tflint`
- Documentation generated from modules (terraform-docs)
- Standardized PR template showing plan output
- Self-service infrastructure: teams create new services from templates

The key insight: at scale, Terraform is not just a technical tool — it requires organizational patterns (team ownership, review processes, governance) as much as technical ones."

---

**Q8: Your Terraform apply failed halfway through. Some resources were created, some weren't. How do you recover?**

**Perfect Answer:**

"A partial apply is one of the trickiest scenarios in Terraform. Here's what happened and how to recover:

**What happens during a partial failure:**
Terraform creates resources in dependency order. If resource C depends on B depends on A, it creates A, then B, then C. If B fails, A exists (in state) and C was never attempted.

The state file reflects reality: A is in state (it was created), B might be partially in state (depending on when it failed), C is not in state.

**Recovery steps:**

**1. Don't panic. Read the error message.**
```bash
terraform plan
```
Run plan immediately — it will show you the current state vs desired state. Terraform is designed to recover from partial failures.

**2. Most common case: just re-run apply.**
Terraform is idempotent. It sees A already exists (no change), attempts B again, then creates C. If the failure was transient (API timeout, rate limit), simply re-running apply fixes it.

**3. If the resource is in a bad state:**
Sometimes a resource is 'half-created' — it exists in cloud but in an error state. Terraform might not be able to update or delete it through normal means.

Options:
- Fix the underlying issue and re-apply
- Manually delete the broken resource in the console, then run `terraform state rm` to remove it from state, then re-apply to recreate
- Use `terraform taint` (deprecated) or `terraform apply -replace=aws_instance.broken` to force recreation

**4. If state is corrupted:**
Rare, but possible if the process was killed during state write.
- Remote state with versioning: restore previous version from S3 versioning
- `terraform state pull` / `terraform state push` for manual fixes

**5. Prevention:**
- Use `-target` sparingly — it can cause partial states
- Always use remote state with locking (prevents concurrent applies)
- Keep resources small and independent (split state)
- Use `create_before_destroy` lifecycle for critical resources
- Save plan files and apply the saved plan (ensures consistency)

**Key principle:** Terraform handles partial failures gracefully by design. The state always reflects what actually exists, and subsequent applies will converge to the desired state. The most dangerous thing you can do is manually intervene without understanding the state."

---

### Scenario-Based Questions

---

**Q9: You have 200 existing AWS resources created manually via the console. How do you bring them under Terraform management?**

**Perfect Answer:**

"Migrating existing infrastructure to Terraform (brownfield adoption) is a phased process:

**Phase 1: Inventory and prioritize**
- Use AWS Config or cloud-nuke's list feature to inventory all resources
- Prioritize by risk and frequency of change:
  - Start with stable, rarely-changing resources (VPC, subnets)
  - Move to more dynamic resources later (EC2, security groups)

**Phase 2: Import resources into Terraform**

For each resource:
1. Write the Terraform configuration that describes it
2. Run `terraform import` to associate the existing resource with your config
3. Run `terraform plan` — if it shows changes, adjust your config until plan shows no changes

```bash
# Import an existing VPC
terraform import aws_vpc.main vpc-abc123

# Import a security group
terraform import aws_security_group.web sg-def456

# Plan should show no changes (config matches reality)
terraform plan
# No changes. Infrastructure is up-to-date.
```

**Phase 3: Tools to accelerate**

- **Terraformer** (by Google): auto-generates .tf files AND imports state for existing resources
```bash
terraformer import aws --resources=vpc,subnet,sg --regions=us-east-1
```

- **Import blocks** (Terraform 1.5+): declarative import in configuration
```hcl
import {
  to = aws_vpc.main
  id = "vpc-abc123"
}
```
Then run `terraform plan -generate-config-out=generated.tf` to auto-generate the resource block.

**Phase 4: Validate and enforce**
- After import: `terraform plan` should show zero changes
- Enable drift detection in CI (scheduled plan)
- Restrict manual console access (read-only IAM for most users)
- All future changes must go through Terraform PRs

**Key lessons from doing this in practice:**
- Don't try to import everything at once — batch by resource type
- Some resources have complex import IDs (check provider docs)
- The hardest part is getting the config exactly right — use `terraform plan` iteratively
- Accept that some legacy resources may not be worth importing (replace them instead)
- Budget 2-5 minutes per resource for simple ones, 15-30 for complex ones (EKS, complex IAM)"

---

**Q10: How do you test Terraform code before applying to production?**

**Perfect Answer:**

"Testing Terraform operates at multiple levels, from fast/cheap to slow/thorough:

**Level 1: Static analysis (seconds, every commit)**
```bash
terraform fmt -check          # Formatting
terraform validate            # Syntax and type checking
tflint                        # Best practices, deprecated usage
checkov / tfsec               # Security misconfigurations
terraform-docs                # Generate docs from code
```

**Level 2: Plan review (seconds, every PR)**
```bash
terraform plan
# Review output: no unintended destroys, expected changes only
# Automated: check plan for destructive operations, alert if found
```

**Level 3: Policy as code (seconds, CI pipeline)**
```
OPA/Sentinel policies:
  - All resources must have required tags
  - No public S3 buckets
  - Security groups must not allow 0.0.0.0/0 on non-web ports
  - Instance types must be from approved list
  - Encryption must be enabled on all storage
```

**Level 4: Integration testing (minutes, pre-merge)**
```python
# Terratest (Go) — actually creates resources, validates, destroys
import testing
import terraform

def test_vpc_creates_correctly():
    options = terraform.Options(
        terraform_dir='./modules/vpc',
        vars={'name': 'test', 'cidr': '10.99.0.0/16'}
    )
    terraform.init_and_apply(options)
    vpc_id = terraform.output(options, 'vpc_id')
    assert vpc_id.startswith('vpc-')
    terraform.destroy(options)
```

**Level 5: Apply to staging first (always)**
Production applies only after the exact same code has been applied to staging without issues.

**Level 6: Canary/progressive apply**
For very large changes, apply to a subset of production first (one region, one account), monitor, then continue.

**My CI pipeline:**
```
PR opened:
  1. fmt + validate + tflint (30s)
  2. terraform plan (1min)
  3. Policy checks (10s)
  4. Plan posted as PR comment

PR merged (staging):
  5. Apply to staging (auto)
  6. Run integration tests against staging

Release to production:
  7. Apply to production (with manual approval)
  8. Monitor for 30 minutes
```"

---

### Key Interview Terms

| Term | When to Use |
|------|------------|
| **Declarative vs Imperative** | Comparing IaC approaches |
| **State file** | Terraform architecture |
| **Idempotent** | Explaining why re-running is safe |
| **Drift / drift detection** | When infra diverges from code |
| **Blast radius** | Justifying state splitting |
| **Modules** | Code reuse and abstraction |
| **Remote state + locking** | Team collaboration |
| **Plan/Apply workflow** | Safe change management |
| **Provider** | Terraform's cloud API abstraction |
| **Immutable infrastructure** | Modern deployment pattern |
| **Configuration drift** | Problem with mutable infrastructure |
| **Policy as code** | Governance enforcement |
| **Terragrunt** | DRY Terraform at scale |
| **Import** | Brownfield adoption |

---

[⬇️ Download This File](#)

---

*Phase 8 Complete. Ready for Phase 9 — Monitoring & Logging.*