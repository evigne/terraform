You don't necessarily need to create two separate security groups (SGs) for EFS and EKS workers, but it is a common best practice to create distinct security groups to clearly define and manage different types of traffic.

Here’s why you might choose to create separate security groups for EFS and EKS workers:

1. **Separation of Concerns**:
   - **EFS SG (efs_sg)**: This security group is specifically for controlling access to the EFS mount targets. It only needs to allow NFS traffic (port 2049) from the EKS worker nodes.
   - **EKS Worker SG (eks_worker_sg)**: This security group is for EKS worker nodes. It can include rules for multiple types of traffic required by the nodes, including communication with EFS, other cluster components, and external services.

2. **Simplified Management**:
   - By having separate security groups, you can manage and modify the rules more easily and without risk of accidentally impacting unrelated traffic.

3. **Enhanced Security**:
   - Ensuring that only necessary traffic is allowed can enhance security by reducing the attack surface.

### Example Configuration

Let's revisit the example with these considerations:

#### EFS Security Group (efs_sg)

```hcl
resource "aws_security_group" "efs_sg" {
  name        = "efs_sg"
  description = "Allow NFS traffic to EFS"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_worker_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "efs_sg"
  }
}
```

#### EKS Worker Security Group (eks_worker_sg)

```hcl
resource "aws_security_group" "eks_worker_sg" {
  name        = "eks_worker_sg"
  description = "Allow traffic for EKS worker nodes"
  vpc_id      = module.vpc.vpc_id

  # Allow NFS traffic to EFS
  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = module.vpc.private_subnets_cidr_blocks
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "eks_worker_sg"
  }
}
```

### Integrating with the Rest of the Infrastructure

Combine these security groups with the rest of your infrastructure setup as previously shown. Here’s the full context:

#### Complete Example with Security Groups

```hcl
provider "aws" {
  region = "us-west-2"  # Update with your region
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = "my-vpc"
  cidr   = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.3.0/24", "10.0.4.0/24"]
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "my-cluster"
  cluster_version = "1.23"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id

  cluster_addons = {
    aws-efs-csi-driver = {
      most_recent = true
    }
  }
  
  node_groups = {
    example = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_type = "m5.large"

      key_name = "my-key"

      additional_security_group_ids = [aws_security_group.eks_worker_sg.id]
    }
  }
}

resource "aws_security_group" "eks_worker_sg" {
  name        = "eks_worker_sg"
  description = "Allow traffic for EKS worker nodes"
  vpc_id      = module.vpc.vpc_id

  # Allow NFS traffic to EFS
  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = module.vpc.private_subnets_cidr_blocks
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "eks_worker_sg"
  }
}

resource "aws_security_group" "efs_sg" {
  name        = "efs_sg"
  description = "Allow NFS traffic to EFS"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_worker_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "efs_sg"
  }
}

resource "aws_efs_file_system" "influxdb" {
  creation_token = "influxdb-efs"
  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }
  tags = {
    Name = "influxdb-efs"
  }
}

resource "aws_efs_mount_target" "influxdb" {
  count             = length(module.vpc.private_subnets)
  file_system_id    = aws_efs_file_system.influxdb.id
  subnet_id         = element(module.vpc.private_subnets, count.index)
  security_groups   = [aws_security_group.efs_sg.id]
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"  # Update if necessary
  }
}

resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "efs-sc"
  }

  provisioner = "efs.csi.aws.com"
  reclaim_policy = "Retain"
}

resource "kubernetes_persistent_volume_claim" "influxdb_pvc" {
  metadata {
    name      = "influxdb-pvc"
    namespace = "default"
  }

  spec {
    access_modes = ["ReadWriteMany"]
    resources {
      requests = {
        storage = "10Gi"
      }
    }
    storage_class_name = "efs-sc"
  }
}

resource "helm_release" "influxdb" {
  name       = "influxdb"
  repository = "https://helm.influxdata.com/"
  chart      = "influxdb"
  namespace  = "default"

  set {
    name  = "persistence.enabled"
    value = "true"
  }

  set {
    name  = "persistence.existingClaim"
    value = kubernetes_persistent_volume_claim.influxdb_pvc.metadata[0].name
  }

  set {
    name  = "persistence.storageClass"
    value = "efs-sc"
  }
}
```

This setup ensures clear separation and management of security group rules for different purposes, enhancing security and manageability.