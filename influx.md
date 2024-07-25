To create a Terraform script that deploys InfluxDB via Helm and uses Amazon EFS for storage, you'll need to follow these steps:

1. **Setup the EFS Filesystem**: Create an EFS filesystem and mount targets.
2. **Create an EFS CSI driver**: Install the EFS CSI driver in your EKS cluster.
3. **Create a StorageClass**: Define a StorageClass that uses the EFS CSI driver.
4. **Deploy InfluxDB**: Use the Helm provider in Terraform to deploy InfluxDB with a PersistentVolumeClaim (PVC) that uses the EFS-backed StorageClass.

Here's a full Terraform script that accomplishes these steps:

### Step 1: Setup the EFS Filesystem

First, you'll need to create an EFS filesystem and mount targets.

```hcl
provider "aws" {
  region = "us-west-2"  # Update with your region
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
  count             = length(aws_subnet_ids.eks.ids)
  file_system_id    = aws_efs_file_system.influxdb.id
  subnet_id         = element(aws_subnet_ids.eks.ids, count.index)
  security_groups   = [aws_security_group.efs_sg.id]
}
```

### Step 2: Create the EFS CSI Driver

Next, you'll need to deploy the EFS CSI driver to your EKS cluster.

```hcl
resource "helm_release" "efs_csi_driver" {
  name       = "aws-efs-csi-driver"
  repository = "https://kubernetes-sigs.github.io/aws-efs-csi-driver/"
  chart      = "aws-efs-csi-driver"
  namespace  = "kube-system"

  set {
    name  = "image.repository"
    value = "602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/aws-efs-csi-driver"
  }

  set {
    name  = "controller.serviceAccount.create"
    value = "true"
  }

  set {
    name  = "controller.serviceAccount.name"
    value = "efs-csi-controller-sa"
  }
}
```

### Step 3: Create the StorageClass

Define a StorageClass that uses the EFS CSI driver.

```hcl
resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "efs-sc"
  }

  provisioner = "efs.csi.aws.com"

  reclaim_policy = "Retain"
}
```

### Step 4: Deploy InfluxDB with Helm

Deploy InfluxDB with a PersistentVolumeClaim that uses the EFS-backed StorageClass.

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"  # Update if necessary
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

### Final Notes

- Ensure your EKS cluster is already set up and your `~/.kube/config` is configured to point to your cluster.
- Update the region, EKS cluster details, and any other specific configuration details.
- You might need to install the necessary providers if they aren't already installed by running `terraform init`.

This script sets up an EFS filesystem, installs the EFS CSI driver, defines a StorageClass, and deploys InfluxDB with the necessary configuration to use the EFS storage.