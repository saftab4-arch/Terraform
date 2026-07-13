
# Project 2: Custom VPC Networking + Modules (Terraform + AWS)

> The follow-up to Lab 1 — this time building a real, custom network from scratch (no default VPC), fetching AMIs dynamically instead of hardcoding them, and learning modules hands-on by actually proving their value with a live "one change, multiple servers update" test.

**What we built:** A custom VPC with a public subnet, Internet Gateway, and route table; a security group allowing SSH + HTTP; an EC2 instance placed inside that custom network; a reusable Terraform module for spinning up servers; and remote state in S3 using the newer `use_lockfile` approach (no DynamoDB needed).

---

## Table of Contents
1. [What's Different From Lab 1](#whats-different)
2. [Step 1: New Project + Dedicated Remote State](#step-1-remote-state)
3. [Step 2: Variables](#step-2-variables)
4. [Step 3: The VPC](#step-3-vpc)
5. [Step 4: The Public Subnet](#step-4-subnet)
6. [Step 5: Internet Gateway](#step-5-igw)
7. [Step 6: Route Table + Association](#step-6-route-table)
8. [Step 7: Security Group (SSH + HTTP)](#step-7-security-group)
9. [Step 8: The `data` Block — Always-Current AMI](#step-8-data-block)
10. [Step 9: Key Pair + EC2 Instance](#step-9-ec2)
11. [Step 10: Outputs + SSH Verification](#step-10-outputs-ssh)
12. [Step 11: Modules — Building & Proving the Value](#step-11-modules)
13. [Concepts Cheat Sheet](#cheat-sheet)
14. [Real-World Deep Dive: Modules in an EKS Setup](#eks-deep-dive)
15. [Cleaning Up](#cleaning-up)

---

## What's Different From Lab 1 <a name="whats-different"></a>

| Lab 1 | Project 2 |
|---|---|
| Used AWS's **default** VPC | Built a **custom** VPC from scratch |
| Hardcoded AMI ID | Used a `data` block to always fetch the current AMI |
| DynamoDB table for state locking | Newer `use_lockfile = true` — no DynamoDB needed |
| Flat `main.tf`, no modules | Built and used a reusable module |
| `t2.micro` | `t3.micro` |

---

## Step 1: New Project + Dedicated Remote State <a name="step-1-remote-state"></a>

Same "bootstrap" pattern as Lab 1 — a separate, one-time project just to create the S3 bucket that will hold this project's state (can't create the bucket inside the same project that needs to use it — chicken-and-egg problem).

```bash
mkdir ~/terraform-project2-backend
cd ~/terraform-project2-backend
nano main.tf
```

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "syed-project2-tfstate-90712"
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

```bash
terraform init && terraform plan && terraform apply
```

**Then, in the real project folder** (`terraform-project2`), connect to it:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket       = "syed-project2-tfstate-90712"
    key          = "terraform-project2/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}

provider "aws" {
  region = var.region
}
```

### What changed vs. Lab 1's backend: `use_lockfile` instead of DynamoDB

In Lab 1, we needed a separate DynamoDB table just for locking (preventing two people from running `apply` at the same time). AWS's provider flagged that as deprecated. The modern replacement:

```hcl
use_lockfile = true
```

Instead of a whole separate database table, Terraform creates a small lock file **directly inside the S3 bucket**, right next to the state file, whenever someone runs `apply`. Same purpose, one less AWS resource to manage.

---

## Step 2: Variables <a name="step-2-variables"></a>

```hcl
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "The size of the EC2 instance"
  type        = string
  default     = "t3.micro"
}
```

---

## Step 3: The VPC <a name="step-3-vpc"></a>

### The concept
A **VPC (Virtual Private Cloud)** is your own private, isolated section of AWS's network — like renting an entire empty office floor. In Lab 1, we relied on AWS's automatically-created **default** VPC. This time, we build our own.

### CIDR blocks
A VPC needs an IP address range, written in CIDR notation:
```
10.0.0.0/16   →   65,536 possible addresses
```

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "project2-vpc"
  }
}
```

- `enable_dns_support` → lets AWS's internal DNS work inside this VPC
- `enable_dns_hostnames` → lets instances get automatic DNS hostnames

---

## Step 4: The Public Subnet <a name="step-4-subnet"></a>

### The concept
If the VPC is the whole office floor, a **subnet** is one room within it — a smaller IP range carved out of the VPC's total range, pinned to a specific Availability Zone (physical data center location).

```hcl
resource "aws_subnet" "public" {
  vpc_id                   = aws_vpc.main.id
  cidr_block                = "10.0.1.0/24"
  availability_zone         = "us-east-1a"
  map_public_ip_on_launch   = true

  tags = {
    Name = "project2-public-subnet"
  }
}
```

- `cidr_block = "10.0.1.0/24"` → a smaller slice (256 addresses) taken from within the VPC's `10.0.0.0/16` range
- `map_public_ip_on_launch = true` → the key setting that makes this a "public" subnet — any instance launched here automatically gets a public IP

**Note:** this setting alone does NOT give internet access yet — it just means instances get a public IP. The actual internet route comes from the next two steps.

---

## Step 5: Internet Gateway <a name="step-5-igw"></a>

### The concept
An **Internet Gateway (IGW)** is the literal "door" between your VPC and the public internet. Without it, your subnet is fully isolated no matter what other settings you configure — like having a street address but no road connecting your neighborhood to the highway.

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "project2-igw"
  }
}
```

Just attaching this isn't enough on its own — traffic still needs to be told to actually use it. That's the route table.

---

## Step 6: Route Table + Association <a name="step-6-route-table"></a>

### The concept
A **route table** is a set of rules: *"if traffic is headed to this destination, send it this way."* By default, a VPC's main route table only knows how to route traffic *within* the VPC. We need an explicit rule sending internet-bound traffic through the Internet Gateway.

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "project2-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

- `cidr_block = "0.0.0.0/0"` → special notation meaning **"any/all destinations"** — literally every possible IP on the internet
- `gateway_id = aws_internet_gateway.main.id` → send that traffic out through the IGW

**The association is the easy-to-forget piece:** creating a route table doesn't automatically apply it to your subnet — you have to explicitly link the two. Without this, your subnet would silently keep using the VPC's default route table (no internet access), even with a perfectly good custom one sitting unused.

---

## Step 7: Security Group (SSH + HTTP) <a name="step-7-security-group"></a>

```hcl
resource "aws_security_group" "web" {
  name        = "project2-allow-ssh-http"
  description = "Allow SSH and HTTP inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### New vs. Lab 1: `vpc_id` is now required
In Lab 1, no `vpc_id` was specified — AWS silently used the default VPC. Now that we have our own VPC, we must explicitly tell the security group which VPC it belongs to.

### What does `protocol = "-1"` mean?
It's AWS's special code for **"all protocols"** — TCP, UDP, ICMP, everything. Combined with `from_port = 0, to_port = 0`, the egress rule reads as: *"let this server send traffic out to anywhere, using any method, no restrictions."* Inbound (ingress) rules are deliberately restricted (only 22 and 80); outbound (egress) is typically left open since servers need to freely reach out for updates, APIs, DNS, etc.

---

## Step 8: The `data` Block — Always-Current AMI <a name="step-8-data-block"></a>

### The concept
A `resource` block tells Terraform "create this." A **`data` block** tells Terraform **"look up information about something that already exists"** — no creation involved. Instead of hardcoding an AMI ID (which goes stale and is risky if copied from an untrusted source), this asks AWS directly, every run, for the current official image.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

- `most_recent = true` → if multiple AMIs match, grab the newest
- `owners = ["amazon"]` → **critical for security** — only official Amazon-published images, never a random third party
- Filters narrow down to Amazon Linux 2023, 64-bit, HVM virtualization

**Usage elsewhere:** `data.aws_ami.amazon_linux.id` — same reference pattern as always, just prefixed with `data.` instead of nothing, since it's a lookup, not a resource.

---

## Step 9: Key Pair + EC2 Instance <a name="step-9-ec2"></a>

### New key pair (fresh per project)
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/project2-key -N ""
```

```hcl
resource "aws_key_pair" "project2_key" {
  key_name   = "project2-key"
  public_key = file("~/.ssh/project2-key.pub")
}
```

### The instance — where everything connects
```hcl
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.project2_key.key_name

  tags = {
    Name = "project2-web-server"
  }
}
```

**The key new line vs. Lab 1:** `subnet_id = aws_subnet.public.id`. In Lab 1, we never specified a subnet — AWS silently placed the instance in the default VPC's default subnet. This time, the instance is explicitly placed inside the custom subnet built in this project — proof that the whole network chain (VPC → subnet → route table → IGW) is what's actually delivering internet access, not an AWS default.

---

## Step 10: Outputs + SSH Verification <a name="step-10-outputs-ssh"></a>

```hcl
output "instance_public_ip" {
  value = aws_instance.web_server.public_ip
}

output "instance_public_dns" {
  value = aws_instance.web_server.public_dns
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_id" {
  value = aws_subnet.public.id
}
```

```bash
ssh -i ~/.ssh/project2-key ec2-user@<PUBLIC_IP>
```

**Proof it worked:** the SSH prompt showed `ec2-user@ip-10-0-1-46` — that private IP (`10.0.1.46`) falls inside `10.0.1.0/24`, the exact subnet CIDR range defined in Step 4. Not an AWS default — a real, custom-built network working end to end.

---

## Step 11: Modules — Building & Proving the Value <a name="step-11-modules"></a>

### The concept
A **module** is a reusable package of Terraform config — its own folder with `main.tf`, `variables.tf`, `outputs.tf` — that you can call multiple times with different inputs, instead of copy-pasting the same resource blocks repeatedly.

### Honest framing first
For a single instance, called once, a module provides **no real benefit** — the "calling" code is roughly the same length as just writing the resource directly. The value only shows up when the same pattern is called **more than once**, and something later needs to change across all of them.

### Folder structure
```
terraform-project2/
    main.tf              ← root config, calls the module
    variables.tf
    outputs.tf
    modules/
        ec2-server/
            main.tf        ← reusable resource definition
            variables.tf   ← the module's own "empty boxes" to be filled
            outputs.tf
```

### The module's `variables.tf` — declares what it needs (no values yet)
```hcl
variable "instance_type"     { type = string }
variable "subnet_id"          { type = string }
variable "security_group_id"  { type = string }
variable "key_name"           { type = string }
variable "ami_id"             { type = string }
variable "server_name"        { type = string }
```

### The module's `main.tf` — uses `var.X` instead of hardcoded/direct references
```hcl
resource "aws_instance" "server" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [var.security_group_id]
  key_name               = var.key_name

  tags = {
    Name = var.server_name
  }
}
```

**Important:** a module is isolated — it does NOT automatically see or reuse resources from the root `main.tf`. It only knows about what's explicitly passed in as variables.

### The module's `outputs.tf`
```hcl
output "public_ip"    { value = aws_instance.server.public_ip }
output "instance_id"  { value = aws_instance.server.id }
```

> ⚠️ **Common mistake:** do NOT add `[*]` to these values (e.g. `aws_instance.server[*].public_ip`) unless the resource actually uses `count`. Without `count`, `[*]` incorrectly turns a single value into a list.

### Calling the module from root `main.tf`
```hcl
module "app_server" {
  source             = "./modules/ec2-server"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t3.micro"
  subnet_id          = aws_subnet.public.id
  security_group_id  = aws_security_group.web.id
  key_name           = aws_key_pair.project2_key.key_name
  server_name        = "project2-app-server"
}

module "small_test_server" {
  source             = "./modules/ec2-server"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t3.nano"
  subnet_id          = aws_subnet.public.id
  security_group_id  = aws_security_group.web.id
  key_name           = aws_key_pair.project2_key.key_name
  server_name        = "project2-small-server"
}
```

Notice every value on the right side (`data.aws_ami.amazon_linux.id`, `aws_subnet.public.id`, etc.) is a **real, already-existing resource** from this same project — explicitly wired in by hand. Nothing is automatic.

### Referencing a module's output from the root
```hcl
output "app_server_ip" {
  value = module.app_server.public_ip
}
```
Pattern: `module.<NICKNAME>.<OUTPUT_NAME>` — pulling a value out of a module you called.

### The actual proof of value — a live experiment
1. Created 2 servers via the module (`app_server`, `small_test_server`), plus the original 1 built directly (not via module)
2. Edited **one file** — `modules/ec2-server/main.tf` — adding a single line: `monitoring = true`
3. Ran `terraform plan`

**Result:** both `module.app_server.aws_instance.server` and `module.small_test_server.aws_instance.server` showed `monitoring: false -> true`. The original, non-module `aws_instance.web_server` showed **zero changes** — completely unaffected, since it doesn't run through the module.

**This is the entire point of modules, proven concretely:** one edit, one file, every instance built through that module picks up the change automatically on the next `apply`. Instances built outside the module are untouched.

---

## Concepts Cheat Sheet <a name="cheat-sheet"></a>

| Concept | What it means |
|---|---|
| VPC | Your own private, isolated network in AWS |
| CIDR block (`10.0.0.0/16`) | The IP address range assigned to a VPC or subnet |
| Subnet | A smaller IP range carved from the VPC, pinned to one Availability Zone |
| Internet Gateway | The "door" connecting a VPC to the public internet |
| Route Table | Rules for where traffic should go based on destination |
| Route Table Association | Explicitly links a route table to a specific subnet |
| `0.0.0.0/0` | Special CIDR meaning "any/all destinations" |
| `protocol = "-1"` | AWS's code for "all protocols" |
| `data` block | Looks up existing info (e.g., current AMI) — creates nothing |
| Module | A reusable folder of Terraform config, callable with different inputs |
| `module.NICKNAME.OUTPUT` | How you reference a value coming out of a module |
| `var.NAME` | How a module (or any file) reads a variable's value |
| `use_lockfile = true` | Modern S3-backend locking — no DynamoDB table needed |

---

## Real-World Deep Dive: Modules in an EKS Setup <a name="eks-deep-dive"></a>

*(Notes from a deeper discussion on how this scales to a real production Kubernetes setup — background context before starting Project 3.)*

### Where modules actually pay off in the real world
Not from "10 identical things" (like `count` handles) — but from:
1. **Same pattern, reused across environments** (dev/staging/prod) — each needs its own VPC/cluster, structurally identical, values differ
2. **Consuming public, pre-built modules** (e.g., `terraform-aws-modules/eks/aws` from the Terraform Registry) instead of hand-writing hundreds of lines of complex, easy-to-get-wrong infrastructure

### A realistic structure
```
company-infra/
    modules/
        vpc/                    ← company's own hand-written module
    environments/
        dev/main.tf             ← calls modules/vpc + public EKS module
        staging/main.tf
        prod/main.tf            ← same modules, different sizes/settings
```

Each environment has its **own state file** (different `key` in the backend), so a mistake in dev can never accidentally touch prod.

### Terraform's job vs. Kubernetes' job
| Terraform provisions | Kubernetes (kubectl/Helm) manages |
|---|---|
| VPC, subnets, networking | Pods, Deployments |
| EKS cluster (control plane) | Services, Ingress |
| Node groups (the underlying EC2s) | ConfigMaps, Secrets |
| IAM roles, security groups | HPA, Karpenter NodePools |

### `min_size` / `max_size` / `desired_size` vs. HPA / Karpenter
- **HPA (Horizontal Pod Autoscaler)** scales the number of **pods** — doesn't create new servers, just decides how many app copies run on *existing* nodes
- **Karpenter** (or Cluster Autoscaler) is what actually adds/removes **EC2 nodes** dynamically based on real demand
- Terraform's `min/max_size` on a node group acts as a **safety boundary/baseline** — not a replacement for dynamic scaling. In modern setups, Karpenter often runs its own separate, independent provisioning (its own `NodePool` config, applied via `kubectl`) outside the Terraform-managed group entirely — meaning it can scale well beyond what Terraform's node group defines, because it's a genuinely separate pool of nodes
- Karpenter itself runs as a regular **pod**, scheduled onto one of the baseline nodes Terraform created — the small, fixed baseline exists so there's always somewhere reliable for the cluster (and Karpenter itself) to run, even if dynamic scaling has an issue

### When NOT to modularize
- A resource that's genuinely one-off, never repeated, even across environments (e.g., one specific DNS record)
- Premature abstraction adds real cost: more files, more indirection, harder debugging — for something used only once, keeping it flat and readable is often the better call

---

## Cleaning Up <a name="cleaning-up"></a>

```bash
cd ~/terraform-project2
terraform destroy    # type 'yes' — removes VPC, subnet, IGW, route table, SG, key pair, all instances

# Optional — only if not reusing this backend bucket for a future project:
cd ~/terraform-project2-backend
terraform destroy    # type 'yes'
```

---

