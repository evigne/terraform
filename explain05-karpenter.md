The empty result for `addons` in the `us-gov-west-1` region indicates that the `eks-pod-identity-agent` add-on might not be supported in this region or for your specific EKS cluster version. To resolve this, you have a few options:

### Option 1: Check AWS Documentation

Verify if the `eks-pod-identity-agent` add-on is available for `us-gov-west-1` and your EKS cluster version in the AWS documentation. Some add-ons might not be available in all regions or for all EKS versions.

### Option 2: Remove or Replace the Add-on

If the add-on is not available, you can either remove it from your configuration or replace it with another add-on that is supported in your region.

### Updating the Terraform Configuration

Here’s how you can update your `eks.tf` to remove or replace the `eks-pod-identity-agent` add-on.

#### Removing the `eks-pod-identity-agent` Add-on

If you decide to remove the add-on, update your `cluster_addons` block to exclude it:

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.11"

  cluster_name    = local.name
  cluster_version = "1.30"

  enable_cluster_creator_admin_permissions = true
  cluster_endpoint_public_access           = true

  cluster_addons = {
    coredns = {
      configuration_values = jsonencode({
        tolerations = [
          {
            key    = "karpenter.sh/controller"
            value  = "true"
            effect = "NoSchedule"
          }
        ]
      })
    }
    # eks-pod-identity-agent = {} # Remove this line
    kube-proxy = {
      addon_version = "1.30.0-eksbuild.1"
    }
    vpc-cni = {
      addon_version = "1.12.0"
    }
  }

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    karpenter = {
      instance_types = ["m5.large"]

      min_size     = 2
      max_size     = 3
      desired_size = 2

      labels = {
        "karpenter.sh/controller" = "true"
      }

      taints = {
        karpenter = {
          key    = "karpenter.sh/controller"
          value  = "true"
          effect = "NoSchedule"
        }
      }
    }
  }

  tags = merge(local.tags, {
    "karpenter.sh/discovery" = local.name
  })
}
```

### Applying the Updated Configuration

1. **Initialize Terraform**:
   ```sh
   terraform init
   ```

2. **Apply the Configuration**:
   ```sh
   terraform apply
   ```

### Verifying the Configuration

Ensure that your EKS cluster is functioning correctly without the `eks-pod-identity-agent` add-on. You can check the status of your cluster and the running add-ons using the AWS Management Console or the AWS CLI.

By following these steps, you can address the issue related to the `eks-pod-identity-agent` add-on not being available in the `us-gov-west-1` region and successfully apply your Terraform configuration.







===================


To modify the Karpenter module in your Terraform configuration, you need to ensure that the Karpenter module does not rely on the `eks-pod-identity-agent` add-on since it is not available in the `us-gov-west-1` region. Here are the steps to adjust the configuration:

### Modifying the Karpenter Module

Ensure that the Karpenter module configuration aligns with the available services and permissions.

#### Step 1: Update the IAM Role for Karpenter

Make sure the IAM role has all necessary permissions without relying on `eks-pod-identity-agent`. You can use the following example for the IAM role and policies:

```hcl
resource "aws_iam_role" "karpenter_role" {
  name = "${local.name}-karpenter-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "eks.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_policy" "karpenter_policy" {
  name        = "${local.name}-karpenter-policy"
  description = "Policy for Karpenter to interact with AWS services"
  policy      = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "ec2:DescribeInstances",
          "ec2:RunInstances",
          "ec2:TerminateInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeVpcs",
          "iam:PassRole",
          "iam:CreateServiceLinkedRole",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "autoscaling:DescribeAutoScalingGroups",
          "eks:DescribeCluster",
          "sts:AssumeRole"
        ],
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "karpenter_policy_attachment" {
  role       = aws_iam_role.karpenter_role.name
  policy_arn = aws_iam_policy.karpenter_policy.arn
}
```

#### Step 2: Update the Karpenter Module Configuration

Ensure the Karpenter module uses the updated IAM role and does not assume the existence of the `eks-pod-identity-agent`:

```hcl
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.11"

  cluster_name = module.eks.cluster_name

  node_iam_role_use_name_prefix   = false
  node_iam_role_name              = aws_iam_role.karpenter_role.name
  create_pod_identity_association = false  # Ensure this is set to false as eks-pod-identity-agent is not available

  tags = local.tags
}
```

### Applying the Updated Configuration

1. **Initialize Terraform**:
   ```sh
   terraform init
   ```

2. **Apply the Configuration**:
   ```sh
   terraform apply
   ```

### Full Updated Example

Here is a consolidated view of the necessary updates in your `eks.tf` file:

```hcl
################################################################################
# Cluster
################################################################################

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.11"

  cluster_name    = local.name
  cluster_version = "1.30"

  enable_cluster_creator_admin_permissions = true
  cluster_endpoint_public_access           = true

  cluster_addons = {
    coredns = {
      configuration_values = jsonencode({
        tolerations = [
          {
            key    = "karpenter.sh/controller"
            value  = "true"
            effect = "NoSchedule"
          }
        ]
      })
    }
    # eks-pod-identity-agent is removed as it is not available
    kube-proxy = {
      addon_version = "1.30.0-eksbuild.1"
    }
    vpc-cni = {
      addon_version = "1.12.0"
    }
  }

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    karpenter = {
      instance_types = ["m5.large"]

      min_size     = 2
      max_size     = 3
      desired_size = 2

      labels = {
        "karpenter.sh/controller" = "true"
      }

      taints = {
        karpenter = {
          key    = "karpenter.sh/controller"
          value  = "true"
          effect = "NoSchedule"
        }
      }
    }
  }

  tags = merge(local.tags, {
    "karpenter.sh/discovery" = local.name
  })
}

output "configure_kubectl" {
  description = "Configure kubectl: make sure you're logged in with the correct AWS profile and run the following command to update your kubeconfig"
  value       = "aws eks --region ${local.region} update-kubeconfig --name ${module.eks.cluster_name}"
}

################################################################################
# IAM Role and Policies for Karpenter
################################################################################

resource "aws_iam_role" "karpenter_role" {
  name = "${local.name}-karpenter-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "eks.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_policy" "karpenter_policy" {
  name        = "${local.name}-karpenter-policy"
  description = "Policy for Karpenter to interact with AWS services"
  policy      = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "ec2:DescribeInstances",
          "ec2:RunInstances",
          "ec2:TerminateInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeVpcs",
          "iam:PassRole",
          "iam:CreateServiceLinkedRole",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "autoscaling:DescribeAutoScalingGroups",
          "eks:DescribeCluster",
          "sts:AssumeRole"
        ],
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "karpenter_policy_attachment" {
  role       = aws_iam_role.karpenter_role.name
  policy_arn = aws_iam_policy.karpenter_policy.arn
}

################################################################################
# Kubernetes Namespace
################################################################################

resource "kubernetes_namespace" "karpenter" {
  metadata {
    name = "karpenter"
  }
}

################################################################################
# Helm charts
################################################################################

resource "helm_release" "karpenter" {
  namespace           = kubernetes_namespace.karpenter.metadata[0].name
  name                = "karpenter"
  repository          = "oci://public.ecr.aws/karpenter"
  repository_username = data.aws_ecrpublic_authorization_token.token.user_name
  repository_password = data.aws_ecrpublic_authorization_token.token.password
  chart               = "karpenter"
  version             = "0.36.2"
  wait                = false

  values = [
    <<-EOT
    nodeSelector:
      karpenter.sh/controller: 'true'
    tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - key: karpenter.sh/controller
        operator: Exists
        effect: NoSchedule
    settings:
      clusterName: ${module.eks.cluster_name}
      clusterEndpoint: ${module.eks.cluster_endpoint}
      interruptionQueue: ${module.karpenter.queue_name}
    EOT
  ]

  lifecycle {
    ignore_changes = [
      repository_password
    ]
  }
}

module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.11"

  cluster_name = module.eks.cluster_name

  node_iam_role_use_name_prefix   = false
  node_iam_role_name              = aws_iam_role.karpenter_role.name
  create_pod_identity_association = false  # Ensure this is set to false

  tags = local.tags
}
```

### Apply the Configuration

1. **Initialize Terraform**:
   ```sh
   terraform init
   ```

2. **Apply the Configuration**:
   ```sh
   terraform apply
   ```

By following these steps, you will ensure that the Karpenter module is correctly configured without relying on the `eks-pod-identity-agent` add-on, which is not available in the `us-gov-west-1` region. This should resolve the issues you are encountering.
