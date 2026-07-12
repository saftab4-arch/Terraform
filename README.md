# Lab 1: My First Terraform + AWS Project (EC2, from Zero to Hero)

A complete, beginner-friendly walkthrough of provisioning AWS
infrastructure with Terraform - written so that literally anyone can
follow along and replicate it, step by step, with zero prior Terraform
experience.

**What we built:** An EC2 server (eventually 10 of them!), created
entirely through code - including a key pair for SSH access, a firewall
rule (security group), remote state storage in S3, and hands-on
debugging of a real state-file mismatch.

**Tools used:** Ubuntu (via WSL2 on Windows), VS Code, AWS CLI,
Terraform, an AWS account.

## Table of Contents

-   [What is Terraform?](#what%20is-terraform)\
-   [Prerequisites](#prerequisites)\
-   [Step 1: Create an IAM User (Terraform's AWS
    Login)](#step-1-create-an-iam-user-terraforms-aws-login)\
-   [Step 2: Install AWS CLI](#step-2-install-aws-cli)\
-   [Step 3: Connect AWS CLI to Your
    Account](#step-3-connect-aws-cli-to-your-account)\
-   [Step 4: Install Terraform](#step-4-install-terraform)\
-   [Step 5: Your First Terraform Project (an S3
    Bucket)](#step-5-your-first-terraform-project-an-s3-bucket)\
-   [Step 6: Understanding the Terraform
    Workflow](#step-6-understanding-the-terraform-workflow)\
-   [Step 7: Building an EC2 Instance with
    Variables](#step-7-building-an-ec2-instance-with-variables)\
-   [Step 8: SSH Key Pairs - Getting Access to Your
    Server](#step-8-ssh-key-pairs--getting-access-to-your-server)\
-   [Step 9: Security Groups - Opening the Door for
    SSH](#step-9-security-groups--opening-the-door-for-ssh)\
-   [Step 10: Outputs - Getting Useful Info
    Back](#step-10-outputs--getting-useful-info-back)\
-   [Step 11: SSH-ing Into Your Real
    Server](#step-11-ssh-ing-into-your-real-server)\
-   [Step 12: Remote State - Storing State in
    S3](#step-12-remote-state--storing-state-in-s3)\
-   [Step 13: Scaling with count](#step-13-scaling-with-count)\
-   [Key Concepts Cheat Sheet](#key-concepts-cheat-sheet)\
-   [Cleaning Up](#cleaning-up)\
-   [What's Next (Project 2)](#whats-next-project-2)

------------------------------------------------------------------------

## What is Terraform?

Terraform is a tool that lets you describe the cloud infrastructure you
want (servers, storage, networking) in simple text files, and it builds
it for you automatically. This is called **"Infrastructure as Code"
(IaC)**.

Think of it like an IKEA order form, not a recipe: you don't write
step-by-step instructions (*"click here, then here"*) - you just
describe the end result you want (*"1 server, size small, in this
region"*), and Terraform figures out how to make AWS match that
description.

### Why use it instead of clicking around the AWS Console?

-   **Version Control:** Your infrastructure becomes versioned
    (trackable in Git, like code).\
-   **Repeatable:** The same files can build identical dev/staging/prod
    environments effortlessly.\
-   **Predictable:** Changes are previewed before they happen
    (`terraform plan`), ensuring no surprises.\
-   **Scalable:** Large teams can build once and reuse everywhere
    instead of manually clicking through the console for every server.

------------------------------------------------------------------------

## Prerequisites

-   An AWS account (aws.amazon.com)\
-   Ubuntu (native, or via WSL2 on Windows) - or Mac/Linux\
-   VS Code (optional but recommended)\
-   Zero Terraform experience needed - that's the point of this guide!

------------------------------------------------------------------------

## Step 1: Create an IAM User (Terraform's AWS Login)

Terraform needs its own AWS "identity" to work with - not your personal
root login. This is called an IAM user.

\> 💡 *IAM users, policies, and access keys are all 100% free. You only
ever pay for the actual resources (servers, storage) they create.*

**In the AWS Console:**\
1. Search for **IAM** → click into it.\
2. Left sidebar → **Users** → **Create user**.\
3. Name it `terraform-user`.\
4. Do **NOT** check "console access" - this user only needs programmatic
(CLI) access.\
5. Click **Next**.\
6. Choose **Attach policies directly** → search and check
**AdministratorAccess** *(fine for learning; you'd scope this down for
real production use)*.\
7. Click **Next** → **Create user**.\
8. Click into your new user → **Security credentials** tab → scroll to
**Access keys** → **Create access key**.\
9. Choose **Command Line Interface (CLI)** → confirm → **Create access
key**.\
10. Download the `.csv` file or copy both the **Access Key ID** and
**Secret Access Key** somewhere safe - AWS will never show you the
secret key again after this screen.

------------------------------------------------------------------------

## Step 2: Install AWS CLI

On Ubuntu/WSL2, `apt install awscli` often fails or gives an outdated
version. Use AWS's official installer instead:

``` bash
sudo apt update  
sudo apt install curl unzip -y  

curl "<https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip>" -o "awscliv2.zip"  
unzip awscliv2.zip  
sudo ./aws/install  
```

Verify it worked:

``` bash
aws --version  
```

------------------------------------------------------------------------

## Step 3: Connect AWS CLI to Your Account

``` bash
aws configure  
```

It will ask for 4 things - paste in what you saved from Step 1:

``` text
AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID  
AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY  
Default region name [None]: us-east-1  
Default output format [None]: json  
```

This saves your credentials securely in `~/.aws/credentials`. Terraform
automatically reads from this file - you never type your raw API keys
inside a `.tf` file.

Verify it worked:

``` bash
aws sts get-caller-identity  
```

\> 💡 *`sts` = Security Token Service. This command just asks AWS "who
am I?" - a safe, free way to confirm your credentials are valid. You
should see your Account ID and the ARN of `terraform-user`.*

------------------------------------------------------------------------

## Step 4: Install Terraform

``` bash
# Add HashiCorp's official repository  
wget -O- <https://apt.releases.hashicorp.com/gpg> | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg  
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] <https://apt.releases.hashicorp.com> \$(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list  
sudo apt update && sudo apt install terraform -y  
```

Verify the installation:

``` bash
terraform -version  
```

------------------------------------------------------------------------

## Step 5: Your First Terraform Project (an S3 Bucket)

``` bash
mkdir ~/terraform-demo  
cd ~/terraform-demo  
nano main.tf  
```

Paste this in (an S3 bucket is a safe, free first resource to build):

``` hcl
terraform {  
required_providers {  
aws = {  
source = "hashicorp/aws"  
version = "~> 5.0"  
}  
}  
}  

provider "aws" {  
region = "us-east-1"  
}  

resource "aws_s3_bucket" "example" {  
bucket = "your-unique-bucket-name-12345" # must be globally unique across ALL of AWS  
}  
```

*Save and exit nano: `Ctrl+O`, `Enter`, `Ctrl+X`*

### Understanding this file, block by block:

-   **Block 1 (`terraform {}`):** Tells Terraform which "plugin"
    (provider) it needs. Like installing an app dependencies from an App
    Store before using it.\
-   **Block 2 (`provider {}`):** Configures how to use that plugin -
    here, we default to the `us-east-1` AWS region.\
-   **Block 3 (`resource {}`):** The actual infrastructure piece being
    created. The pattern is always:\

``` hcl
resource "<TYPE>" "<YOUR_NICKNAME>" {  
<CONFIG_OPTION> = <VALUE>  
}  
```

-   `TYPE` = What kind of thing (e.g., `aws_s3_bucket`)\
-   `YOUR_NICKNAME` = A name only you use to reference this resource
    elsewhere in your configuration files (not seen in AWS itself).

------------------------------------------------------------------------

## Step 6: Understanding the Terraform Workflow

Every Terraform project follows the exact same 3-command loop:

``` bash
terraform init # Downloads the provider plugin (one-time per project folder)  
terraform plan # PREVIEW what will be created/changed - 100% safe, modifies nothing  
terraform apply # ACTUALLY creates/changes real resources in AWS - type 'yes' to confirm  
```

Run all three now. After running `terraform apply`, check your AWS
Console under **S3** - you should see your new bucket!

Clean up when done testing:

``` bash
terraform destroy # Deletes everything Terraform created in this project - type 'yes' to confirm  
```

### The State File - Terraform's Memory

After your first `apply`, Terraform creates a file called
`terraform.tfstate` in your project folder. This is not something you
edit directly - Terraform manages it automatically.

**Why it matters:** `main.tf` is your wish list. `terraform.tfstate` is
the receipt proving what was actually built along with its real AWS
details (IDs, ARNs, etc.). Without it, Terraform wouldn't know what it
already created, and couldn't safely update or delete things later.

------------------------------------------------------------------------

## Step 7: Building an EC2 Instance with Variables

Let's build a real server. First, create a variables file so we're not
hardcoding configurations:

``` bash
nano variables.tf  
```

``` hcl
variable "region" {  
description = "AWS region to deploy into"  
type = string  
default = "us-east-1"  
}  

variable "instance_type" {  
description = "The size of the EC2 instance"  
type = string  
default = "t2.micro" # free-tier eligible  
}  
```

### Finding a safe, current AMI ID

An AMI (Amazon Machine Image) is the OS template your server boots from.
Never copy an AMI ID from a random old blog post - they are
region-specific, expire, and using an untrusted one can introduce
malware risks. Instead, fetch it dynamically directly from AWS using a
**data block**.

Now update your `main.tf`:

``` hcl
terraform {  
required_providers {  
aws = {  
source = "hashicorp/aws"  
version = "~> 5.0"  
}  
}  
}  

provider "aws" {  
region = var.region  
}

data "aws_ami" "amazon_linux" {  
most_recent = true  
owners = ["amazon"]

filter {  
name = "name"  
values = ["al2023-ami-*-x86_64"]  
}  
}

resource "aws_instance" "my_server" {  
ami = data.aws_ami.amazon_linux.id  
instance_type = var.instance_type

tags = {  
Name = "MyFirstTerraformServer"  
}  
}  
```

**Step 8: SSH Key Pairs - Getting Access to Your Server**

To log into your new server securely, you need an SSH Key Pair. Let's
create an SSH key on your local machine and tell Terraform to register
it with AWS.

Run this in your terminal to generate a key pair:\
bash ssh-keygen -t ed25519 -f \~/.ssh/tf-key -N ""

Now append the key pair resource to your main.tf and link it to your EC2
instance:

``` hcl
resource "aws_key_pair" "deployer" {  
key_name = "terraform-lab-key"  
public_key = file("~/.ssh/tf-key.pub")  
}

**Update your aws_instance block to include this key:**

resource "aws_instance" "my_server" {  
ami = data.aws_ami.amazon_linux.id  
instance_type = var.instance_type  
key_name = aws_key_pair.deployer.key_name

tags = {  
Name = "MyFirstTerraformServer"  
}  
}  
```

**Step 9: Security Groups - Opening the Door for SSH**

By default, AWS blocks all incoming traffic to an EC2 instance. We need
to create a Virtual Firewall rule (a Security Group) to open port 22 for
SSH.

Add the following configuration to your main.tf:

``` hcl
resource "aws_security_group" "allow_ssh" {  
name = "allow_ssh_traffic"  
description = "Allow inbound SSH traffic"

ingress {  
description = "SSH from anywhere"  
from_port = 22  
to_port = 22  
protocol = "tcp"  
cidr_blocks = ["0.0.0.0/0"] # For prod, restrict this to your personal IP  
}

egress {  
from_port = 0  
to_port = 0  
protocol = "-1" # Allows all outbound internet traffic  
cidr_blocks = ["0.0.0.0/0"]  
}  
}

**Link this group back into your aws_instance block:**

resource "aws_instance" "my_server" {  
ami = data.aws_ami.amazon_linux.id  
instance_type = var.instance_type  
key_name = aws_key_pair.deployer.key_name  
vpc_security_group_ids = [aws_security_group.allow_ssh.id]

tags = {  
Name = "MyFirstTerraformServer"  
}  
}  
```

**Step 10: Outputs - Getting Useful Info Back**

Instead of searching around the AWS console to find your server's new
public IP address, we can make Terraform output it directly to our
terminal window.

Create a new file called outputs.tf:\
bash nano outputs.tf

hcl output "server_public_ip" { description = "The public IP address of
our web server" value = aws_instance.my_server.public_ip }

**Step 11: SSH-ing Into Your Real Server**

Deploy your updated configuration:

bash terraform plan terraform apply\
*(Type yes when prompted)*

Once the deployment completes, look at the bottom of your terminal
screen. Your custom output will display something like:\
server_public_ip = "54.210.34.11"

Now, test your connection:\
bash ssh -i \~/.ssh/tf-key ec2-user@`<YOUR_OUTPUTTED_IP>`{=html}\
Type yes to confirm the host signature. Congratulations! You are inside
a live Linux server running on AWS that you provisioned entirely through
code. Type exit to close the connection.

**Step 12: Remote State - Storing State in S3**

Right now, your terraform.tfstate file is on your local machine. If your
computer crashes or a teammate wants to work on this project, they won't
know what infrastructure exists! Let's store our state securely inside
an AWS S3 bucket.

Create a new file named backend.tf:\
bash nano backend.tf

hcl terraform { backend "s3" { bucket = "your-unique-bucket-name-12345"
\# Use the bucket created in Step 5 key = "global/s3/terraform.tfstate"
region = "us-east-1" encrypt = true } }

Because we are changing where state files live, we must re-run
initialization:\
bash terraform init\
Terraform will detect you are moving to a remote system and ask if you
want to copy your local state file contents to S3. Type **yes**. Your
state is now safe and stored in the cloud!

**Step 13: Scaling with count**

What if your application grows and you suddenly need 10 servers instead
of 1? You don't have to copy-paste your code 10 times. Simply add the
count meta-argument to your resource block.

Modify your server resource in main.tf:

``` hcl
resource "aws_instance" "my_server" {  
count = 3 # Changes 1 server into a fleet of 3  
ami = data.aws_ami.amazon_linux.id  
instance_type = var.instance_type  
key_name = aws_key_pair.deployer.key_name  
vpc_security_group_ids = [aws_security_group.allow_ssh.id]

tags = {  
Name = "MyFirstTerraformServer-\${count.index}"  
}  
}  
```

*(Note: Because we added count, you will also need to update your
outputs.tf to handle multiple values using splat expression: value =
aws_instance.my_server\[\*\].public_ip)*

Run terraform apply and watch Terraform seamlessly scale out your
infrastructure.

**Key Concepts Cheat Sheet**

-   **Infrastructure as Code (IaC):** Writing configuration code to
    manage physical/virtual hardware resources.
-   **Providers:** Plugins that connect Terraform to specific platforms
    (AWS, Google Cloud, GitHub).
-   **Resources:** The foundational building blocks of your system (VM
    instances, databases, subnets).
-   **State File:** A JSON snapshot used by Terraform to track real
    deployed environments relative to local files.

**Cleaning Up**

Cloud resources cost money if you leave them running indefinitely. To
tear down everything we created during this lab, execute:

bash terraform destroy\
Review the plan to verify it is removing your infrastructure, then type
yes. Within a few minutes, your cloud environment is completely clean
and zeroed out!
