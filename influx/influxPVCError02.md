The error message `Failed to provision volume with SG “efs-sg”: rpc error: code = InvalidArgument desc = missing provisioning mode parameters` suggests that there are missing parameters required for provisioning the volume using the EFS CSI driver. This typically indicates that the StorageClass configuration is missing some necessary parameters.

### Ensure Correct StorageClass Configuration

Here’s how to ensure your StorageClass is correctly configured with all necessary parameters for the EFS CSI driver:

1. **Create a ConfigMap for EFS CSI Driver**
   The EFS CSI driver requires a ConfigMap with the necessary configuration. Ensure you have a ConfigMap in place.

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: efs-provisioner
     namespace: kube-system
   data:
     provisioner: efs.csi.aws.com
   ```

   Apply the ConfigMap:

   ```sh
   kubectl apply -f efs-provisioner-configmap.yaml
   ```

2. **Correct StorageClass Configuration**
   Here’s an example StorageClass configuration for the EFS CSI driver:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: efs-sc
   provisioner: efs.csi.aws.com
   parameters:
     provisioningMode: efs-ap
     fileSystemId: <file-system-id>
     directoryPerms: "700"
   reclaimPolicy: Retain
   volumeBindingMode: Immediate
   ```

   - Replace `<file-system-id>` with your actual EFS file system ID.

   Apply the StorageClass:

   ```sh
   kubectl apply -f storageclass.yaml
   ```

3. **PersistentVolumeClaim Configuration**
   Ensure your PVC references the correct StorageClass:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: influxdb-pvc
     namespace: default
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 10Gi
     storageClassName: efs-sc
   ```

   Apply the PVC:

   ```sh
   kubectl apply -f pvc.yaml
   ```

### Updated Terraform Configuration

Here’s the updated Terraform configuration incorporating these changes:

#### StorageClass Configuration in Terraform

```hcl
resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "efs-sc"
  }

  provisioner = "efs.csi.aws.com"
  reclaim_policy = "Retain"
  volume_binding_mode = "Immediate"
  
  parameters = {
    provisioningMode = "efs-ap"
    fileSystemId     = aws_efs_file_system.influxdb.id
    directoryPerms   = "700"
  }
}
```

#### ConfigMap for EFS CSI Driver in Terraform

```hcl
resource "kubernetes_config_map" "efs_provisioner" {
  metadata {
    name      = "efs-provisioner"
    namespace = "kube-system"
  }

  data = {
    provisioner = "efs.csi.aws.com"
  }
}
```

#### Complete Example with Dependencies

```hcl
provider "aws" {
  region = "us-west-2"  # Update with your region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
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

resource "kubernetes_config_map" "efs_provisioner" {
  metadata {
    name      = "efs-provisioner"
    namespace = "kube-system"
  }

  data = {
    provisioner = "efs.csi.aws.com"
  }
}

resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "efs-sc"
  }

  provisioner = "efs.csi.aws.com"
  reclaim_policy = "Retain"
  volume_binding_mode = "Immediate"
  
  parameters = {
    provisioningMode = "efs-ap"
    fileSystemId     = aws_efs_file_system.influxdb.id
    directoryPerms   = "700"
  }
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
  depends_on = [
    kubernetes_persistent_volume_claim.influxdb_pvc
  ]

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

### Deployment Steps