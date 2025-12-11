# Terraform Basics

## What is Terraform?

**Terraform** is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp. It allows you to define and provision infrastructure using a declarative configuration language.

### Key Features:
- **Infrastructure as Code**: Define infrastructure in configuration files
- **Multi-Cloud**: Works with AWS, Azure, GCP, and 100+ providers
- **Declarative**: Describe what you want, not how to create it
- **State Management**: Tracks infrastructure state
- **Plan & Apply**: Preview changes before applying
- **Resource Graph**: Determines resource dependencies

## Why Terraform?

1. **Version Control**: Infrastructure changes tracked in Git
2. **Reusable**: Create modules for common patterns
3. **Consistent**: Same configuration produces same result
4. **Collaboration**: Teams can work together
5. **Automation**: Integrate with CI/CD pipelines
6. **Documentation**: Code serves as infrastructure documentation

## Terraform vs Other Tools

| Tool | Type | Language | Cloud |
|------|------|----------|-------|
| Terraform | IaC | HCL | Multi-cloud |
| CloudFormation | IaC | JSON/YAML | AWS only |
| Pulumi | IaC | Programming languages | Multi-cloud |
| Ansible | Configuration Management | YAML | Agnostic |

## Installation

### macOS:
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify installation
terraform version
```

### Linux:
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### Windows:
```bash
choco install terraform
```

## Terraform Workflow

```
Write → Initialize → Plan → Apply → Destroy
  ↓         ↓          ↓       ↓        ↓
.tf files  init     preview  create   cleanup
```

## Basic Terraform Configuration

### main.tf:
```hcl
# Configure the provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0"
}

provider "aws" {
  region = "us-east-1"
}

# Create an S3 bucket
resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name"
  
  tags = {
    Name        = "My Bucket"
    Environment = "Dev"
  }
}

# Create an EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

## Terraform Commands

### Initialize:
```bash
# Initialize working directory
terraform init

# Upgrade providers
terraform init -upgrade
```

### Validate:
```bash
# Validate configuration
terraform validate
```

### Format:
```bash
# Format configuration files
terraform fmt

# Format and show changes
terraform fmt -diff
```

### Plan:
```bash
# Preview changes
terraform plan

# Save plan to file
terraform plan -out=tfplan

# Show saved plan
terraform show tfplan
```

### Apply:
```bash
# Apply changes
terraform apply

# Apply without confirmation
terraform apply -auto-approve

# Apply saved plan
terraform apply tfplan
```

### Destroy:
```bash
# Destroy all resources
terraform destroy

# Destroy without confirmation
terraform destroy -auto-approve

# Destroy specific resource
terraform destroy -target=aws_instance.web
```

### Other Commands:
```bash
# Show current state
terraform show

# List resources in state
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Output values
terraform output

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

## HCL Syntax Basics

### Variables:

```hcl
# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "allowed_ports" {
  description = "List of allowed ports"
  type        = list(number)
  default     = [80, 443, 22]
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {
    Project = "MyProject"
    Owner   = "DevOps"
  }
}
```

### Using Variables:

```hcl
# main.tf
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  
  tags = merge(
    var.tags,
    {
      Name = "WebServer"
      Environment = var.environment
    }
  )
}
```

### Passing Variables:

```bash
# Command line
terraform apply -var="region=us-west-2" -var="environment=prod"

# Variable file (terraform.tfvars)
region       = "us-west-2"
environment  = "prod"
instance_type = "t2.small"

# Apply with variable file
terraform apply -var-file="prod.tfvars"

# Environment variables
export TF_VAR_region="us-west-2"
terraform apply
```

### Outputs:

```hcl
# outputs.tf
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.example.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.example.arn
  sensitive   = true  # Hide in output
}
```

### Data Sources:

```hcl
# Fetch existing VPC
data "aws_vpc" "default" {
  default = true
}

# Fetch latest Amazon Linux AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use data source
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  subnet_id     = data.aws_vpc.default.id
}
```

## Resource Dependencies

### Implicit Dependencies:

```hcl
# EC2 instance depends on security group
resource "aws_security_group" "web" {
  name = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.web.id]  # Implicit dependency
}
```

### Explicit Dependencies:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  depends_on = [aws_s3_bucket.example]  # Explicit dependency
}
```

## Terraform State

### What is State?
- Mapping between configuration and real-world resources
- Stores resource metadata
- Performance optimization
- Collaboration mechanism

### Local State:
```bash
# Default: terraform.tfstate file
# Not recommended for teams
```

### Remote State:

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

### State Commands:

```bash
# List resources
terraform state list

# Show resource
terraform state show aws_instance.web

# Remove resource from state
terraform state rm aws_instance.web

# Move resource
terraform state mv aws_instance.web aws_instance.web_server

# Pull remote state
terraform state pull

# Push local state
terraform state push
```

## Terraform Modules

### What are Modules?
Reusable Terraform configurations.

### Module Structure:
```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
  ec2/
    main.tf
    variables.tf
    outputs.tf
```

### Creating a Module:

```hcl
# modules/ec2/main.tf
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
  
  tags = var.tags
}

# modules/ec2/variables.tf
variable "ami" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "tags" {
  type    = map(string)
  default = {}
}

# modules/ec2/outputs.tf
output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```

### Using a Module:

```hcl
# main.tf
module "web_server" {
  source = "./modules/ec2"
  
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.small"
  
  tags = {
    Name        = "WebServer"
    Environment = "Production"
  }
}

# Access module outputs
output "web_server_ip" {
  value = module.web_server.public_ip
}
```

### Using Public Modules:

```hcl
# From Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = "dev"
  }
}
```

## Complete Example

### Project Structure:
```
terraform-project/
  ├── main.tf
  ├── variables.tf
  ├── outputs.tf
  ├── terraform.tfvars
  ├── backend.tf
  └── modules/
      └── ec2/
          ├── main.tf
          ├── variables.tf
          └── outputs.tf
```

### main.tf:
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
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "${var.project_name}-web-server"
  }
}

# Data source for latest AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

### variables.tf:
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "admin_cidr" {
  description = "CIDR block for admin access"
  type        = string
}
```

### outputs.tf:
```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Public IP of web server"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS of web server"
  value       = aws_instance.web.public_dns
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.web.id
}
```

### terraform.tfvars:
```hcl
project_name  = "myproject"
aws_region    = "us-east-1"
instance_type = "t2.micro"
admin_cidr    = "203.0.113.0/24"
```

## Best Practices

1. **Version Control**: Store .tf files in Git
2. **Remote State**: Use remote backend for teams
3. **State Locking**: Enable locking (DynamoDB for S3 backend)
4. **Modules**: Create reusable modules
5. **Variables**: Use variables for flexibility
6. **Outputs**: Export useful information
7. **Naming**: Use consistent naming conventions
8. **Comments**: Document complex logic
9. **Formatting**: Use `terraform fmt`
10. **Validation**: Run `terraform validate`
11. **Plan**: Always review plan before apply
12. **.gitignore**: Exclude sensitive files
```gitignore
# .gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl
```

## Common Issues

### Issue 1: State Locking
```bash
# If state is locked
terraform force-unlock <LOCK_ID>
```

### Issue 2: Resource Already Exists
```bash
# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

### Issue 3: Dependency Cycle
```bash
# Review dependencies
terraform graph | dot -Tpng > graph.png
```

## Summary

- Terraform is an Infrastructure as Code tool
- Uses declarative HCL language
- Supports multiple cloud providers
- State management tracks infrastructure
- Modules enable code reuse
- Plan before apply to preview changes
- Remote state for team collaboration
- Version control infrastructure code
- Essential for modern cloud infrastructure
- Integrates with CI/CD pipelines
