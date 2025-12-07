# IAC
Infrastructure as Code means managing and provisioning your cloud infrastructure through code, instead of doing it manually through a console or UI.

- ***Automation*** : No manual clicks — infrastructure is deployed automatically through scripts.
- ***Consistency*** :	Same code → same infrastructure → no configuration drift.
- ***Speed*** :	You can create or destroy full environments (Dev, QA, Prod) in minutes.
- ***Version Control*** :	IaC files are stored in Git, so every infrastructure change is tracked.
- ***Reusability*** :	You can reuse and modularize your code for multiple projects.
- ***Disaster Recovery*** :	Rebuild entire environments quickly if something fails.
- ***ollaboration*** :	Multiple teams can work on the same codebase using Git workflows.
  
# What is terraform
Terraform is an open-source tool created by HashiCorp that helps you manage your cloud infrastructure using code. It uses a simple language called HCL (HashiCorp Configuration Language) to define what resources you need, like servers, networks, or databases.

Terraform can automatically create and manage these resources across different cloud providers such as AWS, Azure, and Google Cloud, all from a single set of configuration files.

# Why Terraform is Popular
- ***Multi-cloud support***	Works with AWS, Azure, GCP, VMware, etc.
- ***Declarative syntax***	You describe what you want, not how to do it.
- ***State management***	Tracks resources in a .tfstate file.
- ***Idempotent***	Running the same code again won’t recreate unchanged resources.
- ***Modules***	Lets you reuse infrastructure components (like functions).
# Terraform lifecycle
Terraform lifecycle defines the complete process of how Terraform initializes, plans, applies, and manages infrastructure as code from setup to destory or deletion.

Terraform workflow follows this sequence:

> ***Init → Fmt → Validate → Plan → Apply → Show/State → Destroy***

| Phase | Description | Command Example |
|--------|--------------|----------------|
| **1️⃣ Init (Initialize)** | Sets up your Terraform working directory. It downloads provider plugins (like AWS, Azure), initializes modules, and configures backend (like S3 & DynamoDB). Must be run first before any other command. | `terraform init` |
| **2️⃣ Fmt (Format)** | Formats your Terraform `.tf` files to a standard readable format. It ensures clean, consistent code style across your project. | `terraform fmt` |
| **3️⃣ Validate (Syntax Check)** | Checks if your Terraform code is syntactically and logically correct. Ensures no typos, missing variables, or invalid blocks before planning. | `terraform validate` |
| **4️⃣ Plan (Dry Run)** | Creates a detailed execution plan by comparing desired configuration with the current state. Shows which resources will be created, changed, or destroyed. | `terraform plan -var-file="dev.tfvars"` |
| **5️⃣ Apply (Deploy Infrastructure)** | Executes the plan — actually creates, updates, or deletes resources in your cloud. Updates the Terraform state file once done. | `terraform apply -auto-approve` |
| **6️⃣ Show (Review State)** | Displays the details of your deployed infrastructure from the Terraform state file. Helps verify what resources exist. | `terraform show` |
| **7️⃣ State (Manage State File)** | Allows you to inspect, move, or remove resources manually from the Terraform state. Used for advanced troubleshooting or imports. | `terraform state list` |
| **8️⃣ Destroy (Tear Down)** | Deletes all infrastructure created by Terraform safely. Used when you want to clean up environments like dev/test. | `terraform destroy -auto-approve` |

# Terraform Variables — Types and Precedence 
Terraform variables make your configuration **dynamic, reusable, flexible, and environment-independent**.  
Instead of hardcoding values, you can define variables and pass values dynamically from files, environment variables, or CLI to  manage different environments (like dev, stage, prod) using the same `.tf` files
# Example
<pre>
variable "region" {
  description = "AWS region"
  default     = "ap-south-1"
} 

provider "aws" {
  region = var.region
} </pre>

# Terraform Variables — Types and Loading Order
Terraform supports different variable **data types** such as **strings**, **numbers**, **booleans**, **lists**, **maps**, and **objects**.

| Type | Example | Description |
|-------|----------|-------------|
| **String** | `"ap-south-1"` | Simple text |
| **Number** | `3`, `10`, `2.5` | Numeric values |
| **Bool** | `true`, `false` | True or False values |
| **List** | `["t2.micro", "t2.small"]` | Ordered list of values |
| **Map** | `{ env = "dev", owner = "teamA" }` | Key-value pairs |
| **Object / Tuple** | Complex structures | Used in advanced modules |

# Example
<pre>variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "allowed_ports" {
  type    = list(number)
  default = [22, 80, 443]
}

variable "tags" {
  type = map(string)
  default = {
    env   = "dev"
    owner = "ragha"
  }
} </pre>

# Terraform Variable Precedence

Terraform can take variable values from multiple sources.  If the same variable is defined in more than one place, Terraform follows a **specific order of precedence**.  
The **last (highest priority)** value always wins.

***Terraform Variable Loading Order***

| Step | Source | Example | Priority |
|------|----------|----------|-----------|
| **1** | **Default** | `default = "ap-south-1"` |  Lowest |
| **2** | **terraform.tfvars** | `region = "us-east-1"` | ⬆️ |
| **3** | **Custom tfvars file** | `terraform apply -var-file="prod.tfvars"` | ⬆️⬆️ 
| **4** | **Environment variable** | `export TF_VAR_region="ap-northeast-1"` | ⬆️⬆️⬆️ |
| **5** | **CLI variable** | `terraform apply -var="region=ca-central-1"` | **Highest** |

Terraform loads variables in this order:  
**Default → tfvars → custom tfvars → environment → CLI**

When duplicates exist —  The **last defined source (highest priority)** always overwrites the previous ones.
# Provider 
A provider in Terraform is the plugin that allows Terraform to connect with a specific platform or service like AWS, Azure, GCP, Kubernetes, etc.
Sometimes, you may want to use more than one provider in a single Terraform project. This is called a multi-provider setup.
**Multiple Cloud Providers**
```hcl
provider "aws" {
  region = "ap-south-1"
}

provider "azurerm" {
  features {}
}
```
**Multiple Accounts or Regions for the Same Provider**
```hcl
provider "aws" {
  region = "ap-south-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}
```
# What Is a Resource Block and Resource Dependencies ?

A resource block defines what you want to create in your infrastructure.
```hcl resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "MyWebServer"
  }
}
```
Sometimes one resource depends on another.  Exampl You must create a VPC before creating a Subnet and you must create a Subnet before creating an EC2 inside it.
Terraform automatically handles most dependencies but if you want explicitily metion **depends_on** in resource block.
The below exmpale first create S3 bucket and then create app server.
```hcl resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  depends_on    = [aws_s3_bucket.my_bucket]
}
```
# What Is a Data Source ?
A data source is used to read existing information from the cloud, not to create new resources. In this example first fetch Prod-VPC and th vpc related subnets.

```hcl # --- Get the Prod VPC by Tag ---
data "aws_vpc" "prod" {
  filter {
    name   = "tag:Name"
    values = ["Prod-VPC"]
  }
}

# --- Get All Subnets Inside the Prod VPC ---
data "aws_subnets" "prod_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.prod.id]
  }
}
```
# Terraform State
Terraform state is a file (terraform.tfstate) that keeps track of everything in the Terraform has created in your cloud. By default terraform is stateless it doesn’t automatically know what already exists in your cloud.

Terrraform state file by default it stores on local where you are executing the project. 

Using Remote Backend in Terraform is a centralized storage (like AWS S3 or Azure Blob) where the state file is stored securely instead of keeping it locally. Also it is useful to share with multiple team members, enable locking to prevent conflicts, and ensure safe, consistent infrastructure management.
## For AWS
```bash
   aws s3 mb s3://my-terraform-state-bucket
```
```bash
   aws dynamodb create-table \
  --table-name terraform-lock-table \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```
```hcl terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock-table"
    encrypt        = true
  }
}
```
## For Azure
```bash
az storage account create \
  --name mytfstateaccount \
  --resource-group terraform-rg \
  --location eastus \
  --sku Standard_LRS

az storage container create \
  --name tfstate \
  --account-name mytfstateaccount
```
```hcl terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "mytfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```
## Meta-Argument in Terraform
In Terraform, a meta-argument is a special setting that can be used inside any resource block. It controls how Terraform manages that resource — not what the resource is

| Meta Argument  | Purpose                                | Example                               | Use Case                               |
| -------------- | -------------------------------------- | ------------------------------------- | -------------------------------------- |
| **depends_on** | Force order between resources          | `depends_on = [aws_s3_bucket.bucket]` | Create one resource only after another |
| **count**      | Create multiple identical resources    | `count = 3`                           | Launch 3 EC2 servers                   |
| **for_each**   | Create multiple unique resources       | `for_each = var.instances`            | Different configs for each environment |
| **provider**   | Select specific provider configuration | `provider = aws.us`                   | Multi-region or multi-account setup    |
| **lifecycle**  | Control create/destroy/update behavior | `prevent_destroy = true`              | Protect critical resources             |
```hcl  #############################################
#  PROVIDERS (Default + Secondary Region)
#############################################

provider "aws" {
  region = "ap-south-1"  # Primary region (Mumbai)
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"   # Secondary region (Virginia)
}

#############################################
#  SHARED RESOURCE - S3 BUCKET (with lifecycle)
#############################################

resource "aws_s3_bucket" "logs" {
  bucket = "company-app-logs-bucket"

  lifecycle {
    prevent_destroy = true        # Protect from accidental deletion
    ignore_changes  = [tags]      # Ignore tag changes made manually
    #create_before_destroy = true # When replacing, create new one first, then delete old
  }

  tags = {
    Environment = "prod"
    Owner       = "DevOps-Team"
  }
}

#############################################
#  LOCAL VALUE - Define Regions for EC2
#############################################

locals {
  regions = {
    india = aws
    us    = aws.us
  }
}

#############################################
#  EC2 INSTANCES (for_each + count + depends_on)
#############################################

# Define instance types for different environments
variable "instance_types" {
  default = {
    india = "t2.micro"
    us    = "t2.small"
  }
}

resource "aws_instance" "app_servers" {
  for_each = local.regions          # Create instances for both regions
  provider = each.value             # Use correct provider per region
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_types[each.key]

  # Create 2 EC2 instances per region
  count = 2

  # Ensure EC2 creation only after S3 bucket exists
  depends_on = [aws_s3_bucket.logs]

  lifecycle {
    create_before_destroy = true    # Avoid downtime during replacements
  }

  tags = {
    Name = "app-server-${each.key}-${count.index + 1}"
    Region = each.key
  }
}

#############################################
#  OUTPUTS
#############################################

output "s3_bucket_name" {
  value = aws_s3_bucket.logs.bucket
}

output "instance_ids" {
  value = [for instance in aws_instance.app_servers : instance.id]
}
```
## Terraform Taint
Terraform taint is a command used to mark a resource for recreation during the next terraform apply. When something in your infrastructure becomes corrupted, misconfigured, or outdated, you can mark it as tainted so Terraform automatically recreates it in the next terraform apply. After new resource is successfully created, Terraform automatically removes the taint flag from the state file.
```bash
terraform taint aws_instance.web
```
If you marked something as tainted by mistake, you can remove the taint using:
```bash
terraform taint aws_instance.web
```
# Terraform debugging
Debugging is useful when something breaks or doesn’t work as expected and you need to troubleshoot deeply.
Log levels are ***ERROR***, ***WARN***, ***INFO***, ***DEBUG***,***TRACE***
```bash
export TF_LOG=TRACE
export TF_LOG_PATH=terraform-debug.log
terraform apply
```
# Conditions and Functions in Terraform
Conditions and functions make Terraform configurations smart and dynamic,   allowing you to perform logic, calculations, or data transformations.
- Make decisions (if-else logic)
- Reuse code across environments
- Format or manipulate values (like strings, lists, maps)
- Simplify complex resource configurations
```hcl
variable "environment" {
  type = string
  default = "dev"

  validation {
    condition     = contains(["dev", "qa", "prod"], var.environment)
    error_message = "Environment must be one of: dev, qa, or prod."
  }
}

variable "instance_count" {
  type = number
  default = 2

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 5
    error_message = "Instance count must be between 1 and 5."
  }
}

locals {
  instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = local.instance_type
  tags = {
    Name = "web-${var.environment}-${count.index + 1}"
  }
}
```
# Provisioners in Terraform
Provisioners in Terraform are used to execute scripts or commands on a resource (like a virtual machine) after it is created or before it is destroyed
Provisioners are used for:
- Installing packages or software (e.g., installing Nginx or Docker on an EC2)
- Running configuration scripts
- Copying files from local to remote machines
- Performing initial setup commands.

Terraform has two main types of provisioners based on where the execution happens

***Local Provisioner (local-exec)***

Runs the command on the machine where Terraform is executed (your laptop or Jenkins agent).
Usecase: You want to trigger something locally after infrastructure is deployed

***Remote Provisioner (remote-exec)***

Runs the command on the created resource itself (like inside the EC2 or VM).
Use case: You want to configure the server right after it’s created — e.g., install packages, configure files.

***Connection block***

The connection block defines how Terraform connects to a remote resource, usually a VM or EC2 instance, so that it can run provisioners like remote-exec or file

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = "my-ec2-key"   # This is the key pair name in AWS console

  provisioner "local-exec" {
    command = "echo EC2 instance ${self.public_ip} created >> instance_log.txt"
  }
}

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("C:/Users/Raghavendra/my-ec2-key.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum install -y nginx",
      "sudo systemctl start nginx"
    ]
  }
}
```
