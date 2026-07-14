# Project 3 — Production-Style AWS VPC with Terraform Modules

A hands-on Terraform project that builds and validates a complete AWS network using:

- A dedicated S3 remote-state backend
- Native S3 state locking
- A reusable VPC child module
- Public and private subnets across two Availability Zones
- An Internet Gateway
- A NAT Gateway with an Elastic IP
- Public and private route tables
- Two EC2 instances for real connectivity testing
- Separate security groups
- Terraform variables, `tfvars`, locals, outputs, loops, and module communication

> This project is intentionally designed as a learning lab. It follows professional Terraform structure while keeping the architecture small enough to build, test, explain, and destroy in one session.

---

## Project Goal

The goal is not only to create AWS resources. The goal is to understand how a real Terraform project is organized and how data moves through the configuration:

```text
terraform.tfvars
        ↓
Root variables.tf
        ↓
Root main.tf
        ↓
Child module variables.tf
        ↓
Child module main.tf
        ↓
AWS resources
        ↓
Child module outputs.tf
        ↓
Root outputs.tf
```

By the end of the project, the infrastructure is tested end to end:

```text
Local computer
      ↓ SSH
Public EC2
      ↓ SSH
Private EC2
      ↓
Private route table
      ↓
NAT Gateway
      ↓
Internet Gateway
      ↓
Internet
```

---

# Architecture

```text
AWS Region: us-east-1

Internet
   │
   ▼
Internet Gateway
   │
   ▼
Public Route Table
0.0.0.0/0 → Internet Gateway
   │
   ├── Public Subnet A — 10.0.1.0/24
   │      ├── Public EC2 / Bastion
   │      └── NAT Gateway + Elastic IP
   │
   └── Public Subnet B — 10.0.2.0/24

Private Route Table
0.0.0.0/0 → NAT Gateway
   │
   ├── Private Subnet A — 10.0.11.0/24
   │      └── Private EC2
   │
   └── Private Subnet B — 10.0.12.0/24

VPC CIDR — 10.0.0.0/16
```

The public EC2 receives a public IP and can be reached from the administrator's public IP.

The private EC2 receives only a private IP. It is reached through the public EC2 and uses the NAT Gateway for outbound Internet access.

---

# What This Project Teaches

## Terraform fundamentals

- Providers
- Resources
- Data sources
- Variables
- `terraform.tfvars`
- Locals
- Outputs
- Resource references
- Dependency graph
- `count`
- `count.index`
- `length()`
- Lists
- Maps
- `merge()`
- Root modules
- Child modules
- Module inputs
- Module outputs
- Remote state
- State locking
- `terraform fmt`
- `terraform validate`
- `terraform plan`
- `terraform apply`
- `terraform destroy`

## AWS fundamentals

- VPC
- CIDR ranges
- Availability Zones
- Public subnets
- Private subnets
- Internet Gateway
- NAT Gateway
- Elastic IP
- Route tables
- Route-table associations
- Security Groups
- EC2
- SSH bastion flow
- Public and private routing

---

# Repository Structure

This lab uses two separate Terraform projects.

```text
terraform-production-vpc-backend/
├── versions.tf
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── main.tf
└── outputs.tf

terraform-production-vpc/
├── backend.tf
├── versions.tf
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── locals.tf
├── main.tf
├── outputs.tf
└── modules/
    └── vpc/
        ├── variables.tf
        ├── main.tf
        └── outputs.tf
```

## Why two separate Terraform projects?

The main project wants to store its state inside an S3 bucket. However, that S3 bucket must exist before Terraform can use it as a backend.

This creates a bootstrap problem:

```text
Terraform needs the S3 bucket
        ↓
The S3 bucket does not exist yet
        ↓
Terraform must create the bucket first
```

The solution is to separate backend infrastructure from application infrastructure:

```text
Backend project
   └── Creates the S3 state bucket

Main project
   └── Uses that bucket and creates the VPC infrastructure
```

The backend is created first and destroyed last.

---

# Terraform State

Terraform state is Terraform's record of the infrastructure it manages. It maps Terraform resource addresses to real AWS resource IDs.

Example:

```text
Terraform resource address:
aws_instance.public

Real AWS resource:
i-0123456789abcdef0
```

Terraform uses three pieces of information:

```text
Terraform code
    = desired state

Terraform state
    = Terraform's memory and resource mapping

AWS
    = actual infrastructure
```

When `terraform plan` runs, Terraform compares all three and decides whether resources must be created, updated, replaced, or destroyed.

If someone deletes an EC2 instance manually, Terraform detects drift and proposes recreating it because the code still says the instance should exist.

---

# Remote State and Locking

The main project's state is stored in S3 instead of only on the local computer.

Benefits include:

- Centralized state
- Version history
- Recovery options
- Team access
- State locking

Native S3 state locking is enabled using:

```hcl
use_lockfile = true
```

State locking prevents two engineers or pipelines from modifying the same state at the same time.

```text
Engineer A runs terraform apply
        ↓
State becomes locked
        ↓
Engineer B cannot apply
        ↓
Engineer A finishes
        ↓
Lock is removed
```

---

# Part 1 — Backend Bootstrap Project

Create the folders:

```bash
mkdir terraform-production-vpc-backend
mkdir terraform-production-vpc
```

Enter the backend project:

```bash
cd terraform-production-vpc-backend
```

Create the files:

```bash
touch versions.tf provider.tf variables.tf terraform.tfvars main.tf outputs.tf
```

## Backend `versions.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

`required_version` defines the supported Terraform CLI version.

`required_providers` tells Terraform to download the official AWS provider.

`version = "~> 6.0"` allows compatible AWS provider 6.x updates while preventing an automatic jump to provider 7.x.

## Backend `provider.tf`

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project   = "terraform-production-vpc"
      ManagedBy = "Terraform"
      Purpose   = "Remote State Backend"
    }
  }
}
```

The provider configures the AWS region. The actual region is supplied through a variable rather than being hardcoded in the provider block.

`default_tags` automatically applies common tags to supported AWS resources.

## Backend `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region where backend resources are created"
  type        = string
}

variable "state_bucket_name" {
  description = "Globally unique S3 bucket name for Terraform remote state"
  type        = string
}
```

`variables.tf` declares what information the project needs. It does not supply the actual values.

## Backend `terraform.tfvars`

```hcl
aws_region       = "us-east-1"
state_bucket_name = "syed-terraform-production-vpc-state-2026"
```

```text
variables.tf
    = declares the inputs

terraform.tfvars
    = supplies the input values
```

S3 bucket names must be globally unique.

## Backend `main.tf`

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket        = var.state_bucket_name
  force_destroy = true

  tags = {
    Name = var.state_bucket_name
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Why `force_destroy = true`?

This lab intentionally destroys the backend after testing.

A versioned S3 bucket may contain current objects, previous object versions, delete markers, and Terraform lock files.

Without `force_destroy`, AWS refuses to delete a non-empty bucket.

For a permanent company backend, the safer pattern is normally:

```hcl
force_destroy = false

lifecycle {
  prevent_destroy = true
}
```

A production backend is usually created once and retained.

## Backend `outputs.tf`

```hcl
output "state_bucket_name" {
  description = "Name of the S3 bucket used for Terraform state"
  value       = aws_s3_bucket.terraform_state.id
}

output "state_bucket_arn" {
  description = "ARN of the S3 bucket used for Terraform state"
  value       = aws_s3_bucket.terraform_state.arn
}
```

## Deploy the Backend

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply
```

Verify versioning:

```bash
aws s3api get-bucket-versioning \
  --bucket syed-terraform-production-vpc-state-2026
```

---

# Part 2 — Main Terraform Project

Enter the main project:

```bash
cd ../terraform-production-vpc
```

Create the root files and VPC module:

```bash
touch backend.tf versions.tf provider.tf variables.tf terraform.tfvars locals.tf main.tf outputs.tf

mkdir -p modules/vpc

touch modules/vpc/variables.tf \
      modules/vpc/main.tf \
      modules/vpc/outputs.tf
```

---

# Root Module vs Child Module

## Root module

The top-level project is the root module. It is responsible for backend configuration, provider configuration, environment values, locals, calling child modules, connecting resources, and exposing final outputs.

## Child module

`modules/vpc` is a child module.

Its responsibility is:

```text
Take networking inputs
        ↓
Create the network
        ↓
Return networking outputs
```

A reusable child module does not contain its own backend or environment-specific `tfvars` file.

---

## Root `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket       = "syed-terraform-production-vpc-state-2026"
    key          = "production-vpc/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

The `key` is the path and filename of this project's state object inside the S3 bucket:

```text
S3 bucket
└── production-vpc/
    └── terraform.tfstate
```

The key is chosen by the engineer.

One bucket can store multiple independent state files:

```text
production-vpc/terraform.tfstate
eks/terraform.tfstate
cicd/terraform.tfstate
```

Backend configuration is processed before normal variable evaluation, so regular variables are not used directly inside this block.

## Root `versions.tf`

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

## Root `provider.tf`

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.common_tags
  }
}
```

## Root `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "availability_zones" {
  description = "Availability Zones"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDRs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "Private subnet CIDRs"
  type        = list(string)
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}
```

A `string` stores one value. A `list(string)` stores multiple string values.

## Root `terraform.tfvars`

```hcl
aws_region  = "us-east-1"
environment = "dev"

vpc_cidr = "10.0.0.0/16"

availability_zones = [
  "us-east-1a",
  "us-east-1b"
]

public_subnet_cidrs = [
  "10.0.1.0/24",
  "10.0.2.0/24"
]

private_subnet_cidrs = [
  "10.0.11.0/24",
  "10.0.12.0/24"
]

instance_type = "t3.micro"
```

The lists are intentionally aligned by index:

```text
Index 0:
us-east-1a
10.0.1.0/24
10.0.11.0/24

Index 1:
us-east-1b
10.0.2.0/24
10.0.12.0/24
```

## Root `locals.tf`

```hcl
locals {
  project_name = "terraform-production-vpc"

  common_tags = {
    Project     = local.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Variables are external inputs. Locals are reusable or calculated values defined inside the configuration.

---

# VPC Child Module

## `modules/vpc/variables.tf`

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "environment" {
  description = "Environment name used for resource naming"
  type        = string
}

variable "availability_zones" {
  description = "Availability Zones used by the subnets"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
}

variable "common_tags" {
  description = "Common tags applied to module resources"
  type        = map(string)
}
```

These are input slots. The root module must supply their values.

## How Root Variables Reach the Child Module

The root `main.tf` contains:

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = var.vpc_cidr
  environment          = var.environment
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  common_tags          = local.common_tags
}
```

Read this line:

```hcl
vpc_cidr = var.vpc_cidr
```

as:

```text
Child module input
vpc_cidr

receives its value from

Root variable
var.vpc_cidr
```

Complete flow:

```text
terraform.tfvars
vpc_cidr = "10.0.0.0/16"
        ↓
Root variables.tf
variable "vpc_cidr"
        ↓
Root main.tf
vpc_cidr = var.vpc_cidr
        ↓
Child variables.tf
variable "vpc_cidr"
        ↓
Child main.tf
cidr_block = var.vpc_cidr
        ↓
AWS VPC
```

The left side of a module call belongs to the child module. The right side comes from the root module.

## `modules/vpc/main.tf`

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-vpc"
    }
  )
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-igw"
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-public-${count.index + 1}"
    }
  )
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-private-${count.index + 1}"
    }
  )
}

resource "aws_eip" "nat" {
  domain = "vpc"

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-nat-eip"
    }
  )
}

resource "aws_nat_gateway" "this" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-nat"
    }
  )

  depends_on = [
    aws_internet_gateway.this
  ]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-public-rt"
    }
  )
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this.id
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-private-rt"
    }
  )
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

---

# Understanding `count`, `length()`, and `count.index`

This block creates multiple public subnets:

```hcl
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
}
```

The list contains two CIDRs:

```hcl
public_subnet_cidrs = [
  "10.0.1.0/24",
  "10.0.2.0/24"
]
```

Therefore:

```text
length(var.public_subnet_cidrs) = 2
```

Terraform creates:

```text
aws_subnet.public[0]
aws_subnet.public[1]
```

First loop:

```text
count.index = 0
CIDR = public_subnet_cidrs[0]
AZ   = availability_zones[0]
```

Second loop:

```text
count.index = 1
CIDR = public_subnet_cidrs[1]
AZ   = availability_zones[1]
```

Only one `count` is needed because one loop controls the whole resource block. The same `count.index` retrieves matching values from the aligned lists.

---

# Resource Dependency Graph

Terraform determines resource creation order from references.

Example:

```hcl
vpc_id = aws_vpc.this.id
```

The Internet Gateway references the VPC ID, so Terraform creates the VPC first.

The NAT Gateway references the Elastic IP and public subnet:

```hcl
allocation_id = aws_eip.nat.id
subnet_id     = aws_subnet.public[0].id
```

The private route table references the NAT Gateway:

```hcl
nat_gateway_id = aws_nat_gateway.this.id
```

Terraform builds this dependency graph automatically.

---

# Why `domain = "vpc"` for the Elastic IP?

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"
}
```

`domain = "vpc"` tells AWS that the Elastic IP is allocated for VPC networking. It does not identify one specific VPC.

The actual relationship is indirect:

```text
Elastic IP
    ↓ attached to
NAT Gateway
    ↓ placed in
Public subnet
    ↓ belongs to
VPC
```

---

## `modules/vpc/outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_public_ip" {
  description = "Public Elastic IP assigned to the NAT Gateway"
  value       = aws_eip.nat.public_ip
}
```

Inside the child module:

```hcl
aws_eip.nat.public_ip
```

Outside the child module:

```hcl
module.vpc.nat_gateway_public_ip
```

Inputs go into modules through variables. Values come out through outputs.

---

# Root `main.tf`

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = var.vpc_cidr
  environment          = var.environment
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  common_tags          = local.common_tags
}

resource "aws_key_pair" "lab_key" {
  key_name   = "${var.environment}-terraform-key"
  public_key = file(pathexpand("~/.ssh/id_ed25519.pub"))

  tags = local.common_tags
}

resource "aws_security_group" "public_ec2" {
  name        = "${var.environment}-public-ec2-sg"
  description = "Allow SSH from administrator public IP"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description = "SSH from administrator computer"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["REPLACE_WITH_YOUR_PUBLIC_IP/32"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-public-ec2-sg"
    }
  )
}

resource "aws_security_group" "private_ec2" {
  name        = "${var.environment}-private-ec2-sg"
  description = "Allow SSH only from the public EC2 security group"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "SSH from public EC2"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.public_ec2.id]
  }

  egress {
    description = "Allow outbound traffic through the NAT Gateway"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-private-ec2-sg"
    }
  )
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

resource "aws_instance" "public" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  subnet_id                   = module.vpc.public_subnet_ids[0]
  vpc_security_group_ids      = [aws_security_group.public_ec2.id]
  key_name                    = aws_key_pair.lab_key.key_name
  associate_public_ip_address = true

  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-public-ec2"
    }
  )
}

resource "aws_instance" "private" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  subnet_id                   = module.vpc.private_subnet_ids[0]
  vpc_security_group_ids      = [aws_security_group.private_ec2.id]
  key_name                    = aws_key_pair.lab_key.key_name
  associate_public_ip_address = false

  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-private-ec2"
    }
  )
}
```

Replace `REPLACE_WITH_YOUR_PUBLIC_IP/32` with:

```bash
curl https://checkip.amazonaws.com
```

---

# Why the Root Uses `module.vpc.vpc_id`

The security groups are created in the root module. The VPC resource exists inside the child module.

Therefore, the root cannot use:

```hcl
aws_vpc.this.id
```

The child exposes it:

```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}
```

The root reads it with:

```hcl
module.vpc.vpc_id
```

---

# Root `outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = module.vpc.private_subnet_ids
}

output "public_ec2_public_ip" {
  description = "Public IP of the public EC2 instance"
  value       = aws_instance.public.public_ip
}

output "public_ec2_private_ip" {
  description = "Private IP of the public EC2 instance"
  value       = aws_instance.public.private_ip
}

output "private_ec2_private_ip" {
  description = "Private IP of the private EC2 instance"
  value       = aws_instance.private.private_ip
}

output "nat_gateway_public_ip" {
  description = "Elastic IP used by the NAT Gateway"
  value       = module.vpc.nat_gateway_public_ip
}
```

---

# SSH Key Creation

```bash
ssh-keygen -t ed25519 -C "terraform-production-vpc"
```

Default files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Terraform uploads only the public key to AWS.

---

# Initialize and Deploy

From the main project directory:

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform apply
```

After apply:

```bash
terraform output
```

---

# Verify the Remote State

The bucket should contain:

```text
production-vpc/
└── terraform.tfstate
```

---

# Test the Public EC2

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@PUBLIC_EC2_PUBLIC_IP
```

Verify outbound connectivity:

```bash
curl https://checkip.amazonaws.com
```

The result should be the public EC2's public IP.

---

# Test the Private EC2 Through the Public EC2

## Lab method

Copy the private key to the public EC2:

```bash
scp -i ~/.ssh/id_ed25519 \
  ~/.ssh/id_ed25519 \
  ec2-user@PUBLIC_EC2_PUBLIC_IP:~/.ssh/
```

SSH into the public EC2:

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@PUBLIC_EC2_PUBLIC_IP
```

Secure the copied key:

```bash
chmod 600 ~/.ssh/id_ed25519
```

SSH to the private EC2:

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@PRIVATE_EC2_PRIVATE_IP
```

> Copying a private key to a bastion is acceptable only for a temporary lab. Production environments should use SSH ProxyJump, SSH agent forwarding, or AWS Systems Manager Session Manager.

---

# Validate the NAT Gateway

From the private EC2:

```bash
curl https://checkip.amazonaws.com
```

The result should match:

```bash
terraform output -raw nat_gateway_public_ip
```

Also test package access:

```bash
sudo dnf check-update
sudo dnf install -y docker
```

A successful download proves the private EC2 can initiate outbound Internet connections through the NAT Gateway.

---

# Public vs Private Traffic

## Public EC2

```text
Local computer
    ↓ SSH over Internet
Internet Gateway
    ↓
Public route table
    ↓
Public EC2
```

## Private EC2

```text
Internet
    ✕
Private EC2
```

Outbound traffic:

```text
Private EC2
    ↓
Private route table
    ↓
NAT Gateway
    ↓
Internet Gateway
    ↓
Internet
```

---

# Useful Verification Commands

```bash
terraform output
aws ec2 describe-vpcs
aws ec2 describe-subnets
aws ec2 describe-route-tables
aws ec2 describe-nat-gateways
aws ec2 describe-instances
aws ec2 describe-security-groups
aws s3 ls s3://syed-terraform-production-vpc-state-2026
```

---

# Cleanup

## 1. Destroy the main infrastructure first

From `terraform-production-vpc`:

```bash
terraform destroy
```

This removes the EC2 instances, Security Groups, AWS key pair, NAT Gateway, Elastic IP, route tables, associations, subnets, Internet Gateway, and VPC.

The backend bucket must still exist while the main infrastructure is being destroyed.

## 2. Destroy the backend last

```bash
cd ../terraform-production-vpc-backend
terraform destroy
```

Because the lab backend uses `force_destroy = true`, Terraform can remove object versions and then delete the bucket.

```text
Backend = first resource created, last resource destroyed
```

---

# Common Errors and Troubleshooting

## Unsupported module attribute

```text
module.vpc does not have an attribute named nat_gateway_public_ip
```

Add the missing output to `modules/vpc/outputs.tf`:

```hcl
output "nat_gateway_public_ip" {
  value = aws_eip.nat.public_ip
}
```

## BucketNotEmpty

Disabling versioning does not remove versions that already exist.

For this disposable lab, configure:

```hcl
force_destroy = true
```

Then run:

```bash
terraform apply
terraform destroy
```

## Invalid index

The CIDR and Availability Zone lists do not have matching lengths.

Keep the lists aligned.

## SSH timeout to public EC2

Check the public IP, Internet Gateway route, Security Group source IP, and SSH username `ec2-user`.

## Cannot SSH to private EC2

Check the private EC2 Security Group source, private IP, key pair, and private-key permissions.

## Private EC2 has no Internet access

Check the NAT Gateway, Elastic IP, route-table associations, private default route, and outbound Security Group rule.

---

# Final Data Flow Summary

## Inputs

```text
terraform.tfvars
        ↓
Root variables.tf
        ↓
Root main.tf
```

## Module call

```text
Root main.tf
        ↓
module "vpc"
        ↓
modules/vpc/variables.tf
```

## Resource creation

```text
modules/vpc/main.tf
        ↓
VPC
Subnets
IGW
NAT
Routes
```

## Module outputs

```text
modules/vpc/outputs.tf
        ↓
module.vpc.vpc_id
module.vpc.public_subnet_ids
module.vpc.private_subnet_ids
module.vpc.nat_gateway_public_ip
```

## Root resources

```text
VPC module outputs
        ↓
Security Groups
        ↓
EC2 instances
```

## Final outputs

```text
Root outputs.tf
        ↓
terraform output
```

---

# Key Lessons

1. Terraform state maps Terraform resources to real AWS resources.
2. Remote state is stored separately from the main infrastructure.
3. State locking prevents simultaneous state modification.
4. The root module orchestrates the project.
5. Child modules contain reusable infrastructure.
6. Root variables receive environment values.
7. Module variables receive values from the root module.
8. Inputs enter modules through variables.
9. Values leave modules through outputs.
10. `count` creates multiple instances of one resource block.
11. `count.index` selects matching values from aligned lists.
12. Terraform builds a dependency graph from references.
13. Public subnets route directly to an Internet Gateway.
14. Private subnets route outbound traffic through a NAT Gateway.
15. A NAT Gateway allows outbound Internet access without exposing private instances directly.
16. The backend is created first and destroyed last.

---

# Project Result

This project successfully demonstrated:

- A reusable Terraform VPC module
- A professional root-module structure
- S3 remote state with locking
- Public and private AWS networking
- A real public-to-private SSH flow
- Verified outbound NAT connectivity
- Clean Terraform lifecycle management

The most important result was not only that the resources were created. It was understanding how every file, variable, module, resource, output, route, and state object connected together.
