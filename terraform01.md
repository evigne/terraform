Certainly! We'll create a new structure with Terraform workspaces and remote state management for the control plane (AppCluster) and spoke clusters (staging, development, and production).

### Updated Folder Structure

```
terraform-project/
  ├── infrastructure/
  │   ├── main.tf
  │   ├── variables.tf
  │   ├── outputs.tf
  │   └── backend.tf
  ├── kubernetes/
  │   ├── app-cluster/
  │   │   ├── main.tf
  │   │   ├── variables.tf
  │   │   ├── outputs.tf
  │   │   └── backend.tf
  │   ├── spoke-clusters/
  │   │   ├── staging/
  │   │   │   ├── main.tf
  │   │   │   ├── variables.tf
  │   │   │   ├── outputs.tf
  │   │   │   └── backend.tf
  │   │   ├── development/
  │   │   │   ├── main.tf
  │   │   │   ├── variables.tf
  │   │   │   ├── outputs.tf
  │   │   │   └── backend.tf
  │   │   ├── production/
  │   │   │   ├── main.tf
  │   │   │   ├── variables.tf
  │   │   │   ├── outputs.tf
  │   │   │   └── backend.tf
  │   ├── monitoring/
  │   │   ├── main.tf
  │   │   ├── variables.tf
  │   │   ├── outputs.tf
  │   │   └── backend.tf
  │   └── argocd/
  │       ├── main.tf
  │       ├── variables.tf
  │       ├── outputs.tf
  │       └── backend.tf
```

### Infrastructure Setup

#### infrastructure/main.tf

```hcl
provider "aws" {
  region = var.region
}

# VPC Module
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name = "eks-vpc"
  cidr = var.vpc_cidr

  azs             = ["us-west-2a", "us-west-2b"]
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  enable_nat_gateway = true
  single_nat_gateway = true
}

# Security Group for RDS
resource "aws_security_group" "rds_sg" {
  name        = "rds-security-group"
  description = "Allow access to RDS"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS Cluster for PostgreSQL
resource "aws_rds_cluster" "this" {
  cluster_identifier      = "app-db-cluster"
  engine                  = "aurora-postgresql"
  engine_version          = "13.6"
  database_name           = "appdb"
  master_username         = "admin"
  master_password         = "admin123"
  backup_retention_period = 5
  preferred_backup_window = "07:00-09:00"
  vpc_security_group_ids  = [aws_security_group.rds_sg.id]
  db_subnet_group_name    = aws_db_subnet_group.this.name

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_rds_cluster_instance" "this" {
  count                = 2
  identifier           = "app-db-instance-${count.index}"
  cluster_identifier   = aws_rds_cluster.this.id
  instance_class       = "db.r5.large"
  engine               = aws_rds_cluster.this.engine
  engine_version       = aws_rds_cluster.this.engine_version
  publicly_accessible  = false
  db_subnet_group_name = aws_db_subnet_group.this.name
}

resource "aws_db_subnet_group" "this" {
  name       = "rds-subnet-group"
  subnet_ids = module.vpc.private_subnets

  tags = {
    Name = "rds-subnet-group"
  }
}

# S3 Bucket
resource "aws_s3_bucket" "this" {
  bucket = "my-app-bucket"
  acl    = "private"

  tags = {
    Name        = "my-app-bucket"
    Environment = "Dev"
  }
}
```

#### infrastructure/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "private_subnets" {
  description = "Private subnets for the VPC"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "public_subnets" {
  description = "Public subnets for the VPC"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}
```

#### infrastructure/outputs.tf

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "private_subnets" {
  value = module.vpc.private_subnets
}

output "public_subnets" {
  value = module.vpc.public_subnets
}
```

#### infrastructure/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### Kubernetes Cluster Setup

#### kubernetes/app-cluster/main.tf

```hcl
provider "aws" {
  region = var.region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "app-cluster"
  cluster_version = "1.21"
  subnets         = var.private_subnets
  vpc_id          = var.vpc_id

  node_groups = {
    app-cluster-nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_type = "t3.medium"
    }
  }
}
```

#### kubernetes/app-cluster/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "Private subnets"
  type        = list(string)
}
```

#### kubernetes/app-cluster/outputs.tf

```hcl
output "cluster_name" {
  value = module.eks.cluster_id
}

output "kubeconfig" {
  value = module.eks.kubeconfig
}
```

#### kubernetes/app-cluster/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/app-cluster/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### kubernetes/spoke-clusters/staging/main.tf

```hcl
provider "aws" {
  region = var.region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

module "eks_spoke" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "staging-cluster"
  cluster_version = "1.21"
  subnets         = var.private_subnets
  vpc_id          = var.vpc_id

  node_groups = {
    staging-cluster-nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_type = "t3.medium"
    }
  }
}
```

#### kubernetes/spoke-clusters/staging/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "Private subnets"
  type        = list(string)
}
```

#### kubernetes/spoke-clusters/staging/outputs.tf

```hcl
output "cluster_name" {
  value = module.eks_spoke.cluster_id
}

output "kubeconfig" {
    value = module.eks_spoke.kubeconfig
}
```

#### kubernetes/spoke-clusters/staging/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/spoke-clusters/staging/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### kubernetes/spoke-clusters/development/main.tf

```hcl
provider "aws" {
  region = var.region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

module "eks_spoke" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "development-cluster"
  cluster_version = "1.21"
  subnets         = var.private_subnets
  vpc_id          = var.vpc_id

  node_groups = {
    development-cluster-nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_type = "t3.medium"
    }
  }
}
```

#### kubernetes/spoke-clusters/development/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "Private subnets"
  type        = list(string)
}
```

#### kubernetes/spoke-clusters/development/outputs.tf

```hcl
output "cluster_name" {
  value = module.eks_spoke.cluster_id
}

output "kubeconfig" {
  value = module.eks_spoke.kubeconfig
}
```

#### kubernetes/spoke-clusters/development/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/spoke-clusters/development/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### kubernetes/spoke-clusters/production/main.tf

```hcl
provider "aws" {
  region = var.region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

module "eks_spoke" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "production-cluster"
  cluster_version = "1.21"
  subnets         = var.private_subnets
  vpc_id          = var.vpc_id

  node_groups = {
    production-cluster-nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_type = "t3.medium"
    }
  }
}
```

#### kubernetes/spoke-clusters/production/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "Private subnets"
  type        = list(string)
}
```

#### kubernetes/spoke-clusters/production/outputs.tf

```hcl
output "cluster_name" {
  value = module.eks_spoke.cluster_id
}

output "kubeconfig" {
  value = module.eks_spoke.kubeconfig
}
```

#### kubernetes/spoke-clusters/production/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/spoke-clusters/production/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### kubernetes/monitoring/main.tf

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "prometheus" {
  name       = "prometheus"
  namespace  = "monitoring"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  version    = "16.12.2"
}

resource "helm_release" "grafana" {
  name       = "grafana"
  namespace  = "monitoring"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"
  version    = "6.17.4"

  set {
    name  = "adminPassword"
    value = "admin"
  }
}

resource "helm_release" "loki" {
  name       = "loki"
  namespace  = "monitoring"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "loki-stack"
  version    = "2.5.0"
}

resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}
```

#### kubernetes/monitoring/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}
```

#### kubernetes/monitoring/outputs.tf

```hcl
output "prometheus_url" {
  value = helm_release.prometheus.status.endpoint[0]
}

output "grafana_url" {
  value = helm_release.grafana.status.endpoint[0]
}

output "loki_url" {
  value = helm_release.loki.status.endpoint[0]
}
```

#### kubernetes/monitoring/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/monitoring/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### kubernetes/argocd/main.tf

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "argo_cd" {
  name       = "argo-cd"
  namespace  = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "3.23.0"

  set {
    name  = "server.service.type"
    value = "LoadBalancer"
  }

  set {
    name  = "installCRDs"
    value = "true"
  }
}

resource "kubernetes_namespace" "argocd" {
  metadata {
    name = "argocd"
  }
}
```

#### kubernetes/argocd/variables.tf

```hcl
variable "region" {
  description = "AWS region"
  default     = "us-west-2"
}
```

#### kubernetes/argocd/outputs.tf

```hcl
output "argo_cd_url" {
  value = helm_release.argo_cd.status.endpoint[0]
}
```

#### kubernetes/argocd/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-states"
    key            = "kubernetes/argocd/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### Summary

This structure separates the infrastructure setup from the Kubernetes cluster setup and uses Terraform workspaces and remote state management to handle different environments (staging, development, production).

To deploy the infrastructure, navigate to the `infrastructure` directory and run:

```sh
terraform init
terraform apply
```

To deploy the Kubernetes clusters, navigate to the appropriate directories (`app-cluster`, `spoke-clusters/staging`, `spoke-clusters/development`, `spoke-clusters/production`) and run:

```sh
terraform init
terraform workspace new <workspace-name>  # e.g., "staging", "development", "production"
terraform apply
```

Finally, navigate to the `monitoring` and `argocd` directories to deploy the monitoring tools and Argo CD:

```sh
terraform init
terraform apply
```

This approach ensures that your infrastructure and Kubernetes clusters are managed independently, allowing for better organization and maintainability. If you need further adjustments or additional details, let me know!
 