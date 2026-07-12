Lab 1: My First Terraform + AWS Project (EC2, from Zero to Hero)


A complete, beginner-friendly walkthrough of provisioning AWS infrastructure with Terraform — written so that literally anyone can follow along and replicate it, step by step, with zero prior Terraform experience.



What we built: An EC2 server (eventually 10 of them!), created entirely through code — including a key pair for SSH access, a firewall rule (security group), remote state storage in S3, and hands-on debugging of a real state-file mismatch.

Tools used: Ubuntu (via WSL2 on Windows), VS Code, AWS CLI, Terraform, an AWS account.


Table of Contents


What is Terraform?
Prerequisites
Step 1: Create an IAM User (Terraform's AWS Login)
Step 2: Install AWS CLI
Step 3: Connect AWS CLI to Your Account
Step 4: Install Terraform
Step 5: Your First Terraform Project (an S3 Bucket)
Step 6: Understanding the Terraform Workflow
Step 7: Building an EC2 Instance with Variables
Step 8: SSH Key Pairs — Getting Access to Your Server
Step 9: Security Groups — Opening the Door for SSH
Step 10: Outputs — Getting Useful Info Back
Step 11: SSH-ing Into Your Real Server
Step 12: Remote State — Storing State in S3
Step 13: Scaling with count
Key Concepts Cheat Sheet
Cleaning Up
What's Next (Project 2)



What is Terraform? <a name="what-is-terraform"></a>

Terraform is a tool that lets you describe the cloud infrastructure you want (servers, storage, networking) in simple text files, and it builds it for you automatically. This is called "Infrastructure as Code."

Think of it like an IKEA order form, not a recipe: you don't write step-by-step instructions ("click here, then here") — you just describe the end result you want ("1 server, size small, in this region"), and Terraform figures out how to make AWS match that description.

Why use it instead of clicking around the AWS Console?


Your infrastructure becomes versioned (trackable in Git, like code)
It's repeatable — the same files can build identical dev/staging/prod environments
Changes are previewed before they happen (terraform plan), so no surprises
Large teams can build once, reuse everywhere instead of manually clicking through the console for every server



Prerequisites <a name="prerequisites"></a>


An AWS account (aws.amazon.com)
Ubuntu (native, or via WSL2 on Windows) — or Mac/Linux
VS Code (optional but recommended)
Zero Terraform experience needed — that's the point of this guide



Step 1: Create an IAM User (Terraform's AWS Login) <a name="step-1-create-an-iam-user"></a>

Terraform needs its own AWS "identity" to work with — not your personal root login. This is called an IAM user.

IAM users, policies, and access keys are all 100% free. You only ever pay for the actual resources (servers, storage) they create.

In the AWS Console:


Search for IAM → click into it
Left sidebar → Users → Create user
Name it terraform-user
Do NOT check "console access" — this user only needs programmatic (CLI) access
Click Next
Choose "Attach policies directly" → search and check AdministratorAccess (fine for learning; you'd scope this down for real production use)
Click Next → Create user
Click into the new user → Security credentials tab → scroll to Access keys → Create access key
Choose "Command Line Interface (CLI)" → confirm → Create access key
Download the .csv file or copy both the Access Key ID and Secret Access Key somewhere safe — AWS will never show you the secret key again after this screen.



Step 2: Install AWS CLI <a name="step-2-install-aws-cli"></a>

On Ubuntu/WSL2, apt install awscli often fails or gives an outdated version. Use AWS's official installer instead:

bashsudo apt update
sudo apt install curl unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

Verify it worked:

bashaws --version


Step 3: Connect AWS CLI to Your Account <a name="step-3-connect-aws-cli"></a>

bashaws configure

It will ask for 4 things — paste in what you saved from Step 1:

AWS Access Key ID [None]: <your access key>
AWS Secret Access Key [None]: <your secret key>
Default region name [None]: us-east-1
Default output format [None]: json

This saves your credentials in ~/.aws/credentials. Terraform automatically reads from this file — you never type your keys inside a .tf file.

Verify it worked:

bashaws sts get-caller-identity

sts = Security Token Service. This command just asks AWS "who am I?" — a safe, free way to confirm your credentials are valid. You should see your Account ID and the ARN (unique identity string) of terraform-user.


Step 4: Install Terraform <a name="step-4-install-terraform"></a>

bash# Add HashiCorp's official repository
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y

Verify:

bashterraform -version


Step 5: Your First Terraform Project (an S3 Bucket) <a name="step-5-first-project"></a>

bashmkdir ~/terraform-demo
cd ~/terraform-demo
nano main.tf

Paste this in (an S3 bucket is a safe, free first thing to build):

hclterraform {
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

resource "aws_s3_bucket" "example" {
  bucket = "your-unique-bucket-name-12345"   # must be globally unique across ALL of AWS
}

Save and exit nano: Ctrl+O, Enter, Ctrl+X

Understanding this file, block by block

Block 1 — the terraform block: tells Terraform which "plugin" (provider) it needs. Like installing an app before using it.

Block 2 — the provider block: configures how to use that plugin — here, which AWS region to default to.

Block 3 — the resource block: the actual thing being created. The pattern is always:

hclresource "<TYPE>" "<YOUR_NICKNAME>" {
  <setting> = <value>
}


TYPE = what kind of thing (e.g., aws_s3_bucket)
YOUR_NICKNAME = a name only you use, to reference this resource elsewhere in your files (not seen in AWS itself)



Step 6: Understanding the Terraform Workflow <a name="step-6-workflow"></a>

Every Terraform project follows the same 3-command loop:

bashterraform init      # Downloads the provider plugin (one-time per project)
terraform plan       # PREVIEW what will be created/changed — 100% safe, changes nothing
terraform apply      # ACTUALLY creates/changes real resources in AWS — type 'yes' to confirm

Run all three now. After apply, check the AWS Console → S3 → you should see your bucket.

Clean up when done testing:

bashterraform destroy    # Deletes everything Terraform created — type 'yes' to confirm

The State File — Terraform's Memory

After your first apply, Terraform creates a file called terraform.tfstate in your project folder. This is not something you write — Terraform manages it automatically.

Why it matters: main.tf is your wish list. terraform.tfstate is the receipt proving what was actually built and its real AWS details (IDs, ARNs, etc.). Without it, Terraform wouldn't know what it already created, and couldn't safely update or delete things later.


Step 7: Building an EC2 Instance with Variables <a name="step-7-ec2-with-variables"></a>

Let's build a real server. First, create a variables file so we're not hardcoding values:

bashnano variables.tf

hclvariable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "The size of the EC2 instance"
  type        = string
  default     = "t2.micro"   # free-tier eligible
}

Finding a safe, current AMI ID

An AMI (Amazon Machine Image) is the OS template your server boots from. Never copy an AMI ID from a random blog or Google search — they're region-specific, expire, and using an untrusted one is a real security risk (could contain malware). Instead, get it directly from AWS:


Easiest for beginners: AWS Console → EC2 → "Launch Instance" → copy the AMI ID shown under "Amazon Linux" (don't actually finish launching — just copy the ID)
More advanced/automatic (a data block):


hcldata "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
# then reference it as: data.aws_ami.amazon_linux.id

Now update main.tf:

hclterraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

resource "aws_instance" "my_server" {
  ami           = "ami-XXXXXXXXXXXXXXXXX"   # your real AMI ID here
  instance_type = var.instance_type

  tags = {
    Name = "terraform-demo-server"
  }
}


Step 8: SSH Key Pairs — Getting Access to Your Server <a name="step-8-ssh-keys"></a>

An EC2 instance with no key pair boots up, but you can never log into it. SSH keys work in pairs:


Private key (no extension) → stays on YOUR machine forever, never shared
Public key (.pub) → shared freely, given to AWS


Analogy: the public key is a padlock you hand to AWS to put on the server's door. The private key is the only key that opens it. When you connect, your machine proves it holds the matching private key — without ever transmitting it.

Generate a key pair locally (best practice — private key never leaves your machine):

bashssh-keygen -t rsa -b 4096 -f ~/.ssh/terraform-key -N ""

Register the public key with AWS, via Terraform. Add to main.tf:

hclresource "aws_key_pair" "my_key" {
  key_name   = "terraform-key"
  public_key = file("~/.ssh/terraform-key.pub")
}

file(...) is a Terraform function that reads a file's contents directly — no manual copy-pasting of the long key string.


Step 9: Security Groups — Opening the Door for SSH & HTTP <a name="step-9-security-groups"></a>

By default, AWS blocks ALL inbound traffic to an instance. Even with the right key, you can't connect without a rule allowing it. Add to main.tf:

hclresource "aws_security_group" "allow_ssh_http" {
  name        = "allow-ssh-http"
  description = "Allow SSH and HTTP inbound traffic"

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


⚠️ cidr_blocks = ["0.0.0.0/0"] means "allow from anywhere on the internet" — fine for learning, but in real production setups you'd restrict this to specific/known IPs.



Now wire the key pair and security group into your instance:

hclresource "aws_instance" "my_server" {
  ami                    = "ami-XXXXXXXXXXXXXXXXX"
  instance_type          = var.instance_type
  key_name               = aws_key_pair.my_key.key_name
  vpc_security_group_ids = [aws_security_group.allow_ssh_http.id]

  tags = {
    Name = "terraform-demo-server"
  }
}

Understanding the reference pattern: TYPE.NICKNAME.ATTRIBUTE

This is one of the most important patterns in Terraform:

aws_key_pair . my_key . key_name
    ↑            ↑         ↑
  TYPE        NICKNAME  ATTRIBUTE

Why .id for the security group but .key_name for the key pair? Because each field on each resource expects whatever AWS's own underlying API expects for that specific field — not a universal rule. Security groups are referenced by ID in AWS's API; key pairs are referenced by name. There's no way to guess this — you check the Terraform provider docs (registry.terraform.io) for each resource type.

Why brackets [...] on vpc_security_group_ids but not key_name? Because vpc_security_group_ids is defined as a list (an instance can have multiple security groups), while key_name is a single value (an instance can only have one key pair). Brackets appear whenever the field's schema type is a list.


Step 10: Outputs — Getting Useful Info Back <a name="step-10-outputs"></a>

Instead of digging through the AWS Console for your server's IP every time, have Terraform print it for you:

bashnano outputs.tf

hcloutput "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.my_server.public_ip
}

Now run terraform apply — it'll print the IP directly in your terminal after creating the resources.


Step 11: SSH-ing Into Your Real Server <a name="step-11-ssh-in"></a>

bashterraform init
terraform plan      # should show 3 to add: key pair, security group, instance
terraform apply     # type 'yes'

Once it finishes, grab the IP from the output, then:

bashssh -i ~/.ssh/terraform-key ec2-user@<PUBLIC_IP>

First connection will ask to confirm trust — type yes. You should land on a prompt like:

[ec2-user@ip-172-31-xx-xx ~]$

You are now inside a real, live AWS server, provisioned entirely by your code. 🎉

Verify you're on a different machine:

bashwhoami
hostname
cat /etc/os-release
exit    # returns to your local machine


Step 12: Remote State — Storing State in S3 <a name="step-12-remote-state"></a>

Right now, terraform.tfstate lives only on your laptop. If it's lost or corrupted, Terraform "forgets" everything it built (even though the resources still exist in AWS) — a real problem for teams, and a good habit to fix even solo.

The chicken-and-egg problem

You can't create the S3 bucket for storing state inside the same project that needs to use it — Terraform needs somewhere to save state before it can even create the bucket meant to hold that state. Solution: create the state-storage bucket in a separate, one-time "bootstrap" project.

Create a separate folder:

bashcd ~
mkdir terraform-backend-setup
cd terraform-backend-setup
nano main.tf

hclterraform {
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
  bucket = "your-unique-tfstate-bucket-name"   # must be globally unique
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

What each piece does:


S3 bucket → holds the actual terraform.tfstate file remotely
Versioning → keeps old versions of the state file, so you can recover if it ever gets corrupted
DynamoDB table → prevents two people from running apply at the same time and corrupting the state file (a "locking" mechanism)


How the DynamoDB lock actually works

DynamoDB supports a conditional write: "insert this row ONLY IF it doesn't already exist." When you run apply, Terraform tries to insert a row with a LockID. If it succeeds, you hold the lock and can proceed. If someone else already holds it, the insert fails and you get an error telling you to wait. When apply finishes, the row is deleted, freeing the lock. Note: this table on its own does nothing — it only becomes a "lock" once a project's backend config explicitly points at it (see below).

bashterraform init
terraform plan
terraform apply    # type 'yes'

Now connect your EC2 project to use this bucket

Go back to terraform-demo/main.tf and merge the backend config into your existing terraform { } block (you can only have ONE terraform { } block per file):

hclterraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "your-unique-tfstate-bucket-name"
    key            = "terraform-demo/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
  }
}


bucket → which bucket holds the state
key → the "filename/path" inside the bucket for THIS project's state (a bucket can hold state for many projects, each under a different key)
dynamodb_table → which table to use for locking


bashterraform init

Terraform will ask to migrate/copy your existing local state into the new backend — confirm with yes.

Verify:

bashterraform plan    # should show 0 to add, 0 to change, 0 to destroy


💡 Lesson learned the hard way: if you ever run terraform apply -refresh-only and confirm it, make sure you understand what it changes in your state before running a regular apply next — refreshing can remove resources from state if AWS reports them as gone, which can cascade into replacements on the next apply. Always run terraform plan and read it carefully before typing yes. If your local and remote state ever disagree, terraform state list and terraform state show <resource> are your best tools for seeing the truth.




Step 13: Scaling with count <a name="step-13-count"></a>

Want 10 servers instead of 1? Change one line:

hclresource "aws_instance" "my_server" {
  count                  = 10
  ami                    = "ami-XXXXXXXXXXXXXXXXX"
  instance_type          = var.instance_type
  key_name               = aws_key_pair.my_key.key_name
  vpc_security_group_ids = [aws_security_group.allow_ssh_http.id]

  tags = {
    Name = "terraform-demo-server-${count.index}"
  }
}


count = 10 → create 10 of this resource
${count.index} → a counter (0 through 9) so each instance gets a unique name


Update your output since my_server is now a list, not a single instance:

hcloutput "instance_public_ip" {
  value = aws_instance.my_server[*].public_ip
}

[*] means "give me this attribute from every instance in the list."

bashterraform plan     # should show 9 to add (your existing instance becomes my_server[0])
terraform apply    # type 'yes'


Key Concepts Cheat Sheet <a name="cheat-sheet"></a>

ConceptWhat it meansresource "TYPE" "nickname" {}Defines a real thing to create in AWSvariable "name" {}A reusable input value, defined in variables.tfvar.nameHow you reference a variable's valueTYPE.nickname.attributeHow you reference another resource's value.idMost resources' unique AWS identifier.arnAmazon Resource Name — used mainly for permissions/policies[ ] around a valueMeans the field expects a list, not a single valueoutput "name" {}Prints a useful value after apply (defined in outputs.tf)terraform.tfstateTerraform's memory of what it actually builtterraform initDownloads provider plugins; sets up/connects backendterraform planSafe preview of changes — nothing is touchedterraform applyActually creates/changes real AWS resourcesterraform destroyDeletes everything Terraform manages in this projectterraform state listLists every resource currently tracked in stateterraform state show <resource>Shows full details of one tracked resourcecount = NCreates N copies of a resourceRemote backend (S3)Stores state in the cloud instead of your laptopDynamoDB lock tablePrevents two people applying at the same time


Cleaning Up <a name="cleaning-up"></a>

Always destroy the project using the backend first, before destroying the backend itself:

bashcd ~/terraform-demo
terraform destroy    # type 'yes'

# Only if you also want to remove the remote state storage itself:
cd ~/terraform-backend-setup
terraform destroy    # type 'yes'


Note: if the state bucket has versioning enabled, terraform destroy may fail with BucketNotEmpty even if the bucket looks empty — old versions are still technically stored. Use the AWS Console → bucket → Empty button (which handles all versions), then delete the bucket.




What's Next (Project 2) <a name="whats-next"></a>


Proper networking — build a custom VPC/subnet instead of relying on AWS's default one
Modules — package the EC2 + Security Group + Key Pair pattern into a reusable module you can call with different variables, instead of copy-pasting the same blocks into every new project
Block storage — attach additional EBS volumes to an instance
Exploring state further — terraform state list, state show, state mv, state rm, and safely handling drift



Built hands-on, mistakes and all — including a real state-file recovery. That debugging experience taught more than a clean run ever could.


Appendix: Final, Complete Working Code

This is the exact, final state of every file in both projects, as they ended up after everything above. Copy these directly to replicate the finished result in one go.

terraform-backend-setup/main.tf

(the one-time bootstrap project — creates the S3 bucket + DynamoDB table used for remote state)

hclterraform {
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
  bucket = "syed-aftab-tfstate-8823"
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

terraform-demo/main.tf

(the real project — 10x EC2 instances, key pair, security group, remote state backend)

hclterraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "syed-aftab-tfstate-8823"
    key            = "terraform-demo/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
  }
}

provider "aws" {
  region = var.region   # match whatever region you set in aws configure
}

resource "aws_key_pair" "my_key" {
  key_name   = "terraform-key"
  public_key = file("~/.ssh/terraform-key.pub")
}

resource "aws_security_group" "allow_ssh" {
  name        = "allow-ssh"
  description = "Allow SSH inbound traffic"

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
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

resource "aws_instance" "my_server" {
  count         = 10
  ami           = "ami-01edba92f9036f76e"   # Amazon Linux 2023, us-east-1 - free tier eligible
  instance_type = var.instance_type
  key_name      = aws_key_pair.my_key.key_name
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]

  tags = {
    Name = "terraform-demo-server"
  }
}

terraform-demo/variables.tf

hclvariable "instance_type" {
  description = "The size of the EC2 instance"
  type        = string
  default     = "t2.micro"   # free-tier eligible
}

variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

terraform-demo/outputs.tf

hcloutput "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.my_server[*].public_ip
}


Note: ami-01edba92f9036f76e was the current Amazon Linux 2023 AMI in us-east-1 at the time this lab was built. AMI IDs change over time — always fetch a current one for your region (see Step 7) rather than reusing this ID as-is.




Note on naming: since count = 10 is used, every instance shares the same Name tag (terraform-demo-server) rather than unique names like terraform-demo-server-0, -1, etc. To tell instances apart in the console, update the tag to Name = "terraform-demo-server-${count.index}".
