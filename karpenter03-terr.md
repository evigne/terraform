provider "aws" {
  region = var.region
}

variable "cluster_name" {
  description = "The name of the EKS cluster"
  type        = string
}

variable "karpenter_namespace" {
  default     = "kube-system"
  description = "The namespace where Karpenter will be deployed"
  type        = string
}

variable "aws_partition" {
  default     = "aws"
  description = "AWS partition (e.g., aws, aws-cn, aws-us-gov)"
  type        = string
}

data "aws_region" "current" {}

data "aws_caller_identity" "current" {}

data "aws_eks_cluster" "cluster" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "cluster" {
  name = var.cluster_name
}

data "aws_ssm_parameter" "arm_ami_id" {
  name = "/aws/service/eks/optimized-ami/${data.aws_eks_cluster.cluster.version}/amazon-linux-2-arm64/recommended/image_id"
}

data "aws_ssm_parameter" "amd_ami_id" {
  name = "/aws/service/eks/optimized-ami/${data.aws_eks_cluster.cluster.version}/amazon-linux-2/recommended/image_id"
}

data "aws_ssm_parameter" "gpu_ami_id" {
  name = "/aws/service/eks/optimized-ami/${data.aws_eks_cluster.cluster.version}/amazon-linux-2-gpu/recommended/image_id"
}

resource "aws_iam_role" "karpenter_node_role" {
  name = "KarpenterNodeRole-${var.cluster_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Service = "ec2.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "karpenter_node_policy" {
  role       = aws_iam_role.karpenter_node_role.name
  policy_arn = "arn:${var.aws_partition}:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "karpenter_cni_policy" {
  role       = aws_iam_role.karpenter_node_role.name
  policy_arn = "arn:${var.aws_partition}:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "karpenter_ecr_policy" {
  role       = aws_iam_role.karpenter_node_role.name
  policy_arn = "arn:${var.aws_partition}:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "karpenter_ssm_policy" {
  role       = aws_iam_role.karpenter_node_role.name
  policy_arn = "arn:${var.aws_partition}:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role" "karpenter_controller_role" {
  name = "KarpenterControllerRole-${var.cluster_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Federated = "arn:${var.aws_partition}:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${replace(data.aws_eks_cluster.cluster.identity[0].oidc.issuer, "https://", "")}"
      },
      Action = "sts:AssumeRoleWithWebIdentity",
      Condition = {
        StringEquals = {
          "${replace(data.aws_eks_cluster.cluster.identity[0].oidc.issuer, "https://", "")}:aud" = "sts.amazonaws.com",
          "${replace(data.aws_eks_cluster.cluster.identity[0].oidc.issuer, "https://", "")}:sub" = "system:serviceaccount:${var.karpenter_namespace}:karpenter"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "karpenter_controller_policy" {
  name   = "KarpenterControllerPolicy-${var.cluster_name}"
  role   = aws_iam_role.karpenter_controller_role.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "ssm:GetParameter",
          "ec2:DescribeImages",
          "ec2:RunInstances",
          "ec2:DescribeSubnets",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeLaunchTemplates",
          "ec2:DescribeInstances",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeInstanceTypeOfferings",
          "ec2:DescribeAvailabilityZones",
          "ec2:DeleteLaunchTemplate",
          "ec2:CreateTags",
          "ec2:CreateLaunchTemplate",
          "ec2:CreateFleet",
          "ec2:DescribeSpotPriceHistory",
          "pricing:GetProducts"
        ],
        Resource = "*",
        Sid      = "Karpenter"
      },
      {
        Effect = "Allow",
        Action = "ec2:TerminateInstances",
        Condition = {
          StringLike = {
            "ec2:ResourceTag/karpenter.sh/nodepool" = "*"
          }
        },
        Resource = "*",
        Sid      = "ConditionalEC2Termination"
      },
      {
        Effect = "Allow",
        Action = "iam:PassRole",
        Resource = "arn:${var.aws_partition}:iam::${data.aws_caller_identity.current.account_id}:role/KarpenterNodeRole-${var.cluster_name}",
        Sid      = "PassNodeIAMRole"
      },
      {
        Effect = "Allow",
        Action = "eks:DescribeCluster",
        Resource = "arn:${var.aws_partition}:eks:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:cluster/${var.cluster_name}",
        Sid      = "EKSClusterEndpointLookup"
      }
    ]
  })
}

resource "null_resource" "tag_subnets" {
  provisioner "local-exec" {
    command = <<EOF
for NODEGROUP in $(aws eks list-nodegroups --cluster-name "${var.cluster_name}" --query 'nodegroups' --output text); do
  aws ec2 create-tags --tags "Key=karpenter.sh/discovery,Value=${var.cluster_name}" --resources $(aws eks describe-nodegroup --cluster-name "${var.cluster_name}" --nodegroup-name "${NODEGROUP}" --query 'nodegroup.subnets' --output text )
done
EOF
  }
}

resource "null_resource" "tag_security_groups" {
  provisioner "local-exec" {
    command = <<EOF
NODEGROUP=$(aws eks list-nodegroups --cluster-name "${var.cluster_name}" --query 'nodegroups[0]' --output text)
LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name "${var.cluster_name}" --nodegroup-name "${NODEGROUP}" --query 'nodegroup.launchTemplate.{id:id,version:version}' --output text | tr -s "\t" ",")
SECURITY_GROUPS=$(aws ec2 describe-launch-template-versions --launch-template-id "${LAUNCH_TEMPLATE%,*}" --versions "${LAUNCH_TEMPLATE#*,}" --query 'LaunchTemplateVersions[0].LaunchTemplateData.[NetworkInterfaces[0].Groups||SecurityGroupIds]' --output text)
aws ec2 create-tags --tags "Key=karpenter.sh/discovery,Value=${var.cluster_name}" --resources "${SECURITY_GROUPS}"
EOF
  }
}

resource "null_resource" "update_aws_auth_configmap" {
  provisioner "local-exec" {
    command = <<EOF
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth.yaml
cat <<EOT >> aws-auth.yaml
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:${var.aws_partition}:iam::${data.aws_caller_identity.current.account_id}:role/KarpenterNodeRole-${var.cluster_name}
  username: system:node:{{EC2PrivateDNSName}}
EOT
kubectl apply -f aws-auth.yaml
EOF
  }
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter/karpenter"
  version    = "0.37.0"
  namespace  = var.karpenter_namespace

  values = [
    {
      settings = {
        clusterName = var.cluster_name
      },
      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = "arn:${var.aws_partition}:iam::${data.aws_caller_identity.current.account_id}:role/KarpenterControllerRole-${var.cluster_name}"
        }
      },
      controller = {
        resources = {
          requests = {
            cpu    = "1",
            memory = "1Gi"
          },
          limits = {
            cpu    = "1",
            memory = "1Gi"
          }
        }
      }
    }
  ]
}

resource "kubernetes_namespace" "karpenter" {
  metadata {
    name = var.karpenter_namespace
  }
}

resource "kubernetes_custom_resource_definition" "node_pool" {
  metadata {
    name = "karpenter.sh_nodepools"
  }
  spec {
    group   = "karpenter.sh"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "nodepools"
      singular = "nodepool"
      kind     = "NodePool"
    }
  }
}

resource "kubernetes_custom_resource_definition" "ec2_node_class" {
  metadata {
    name = "karpenter.k8s.aws_ec2nodeclasses"
  }
  spec {
    group   = "karpenter.k8s.aws"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "ec2nodeclasses"
      singular = "ec2nodeclass"
      kind     = "EC2NodeClass"
    }
  }
}

resource "kubernetes_custom_resource_definition" "node_claim" {
  metadata {
    name = "karpenter.sh_nodeclaims"
  }
  spec {
    group   = "karpenter.sh"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "nodeclaims"
      singular = "nodeclaim"
      kind     = "NodeClaim"
    }
  }
}

resource "kubernetes_manifest" "default_node_pool" {
  manifest = {
    apiVersion = "karpenter.sh/v1beta1"
    kind       = "NodePool"
    metadata = {
      name      = "default"
      namespace = var.karpenter_namespace
    }
    spec = {
      template = {
        spec = {
          requirements = [
            {
              key      = "kubernetes.io/arch"
              operator = "In"
              values   = ["amd64"]
            },
            {
              key      = "kubernetes.io/os"
              operator = "In"
              values   = ["linux"]
            },
            {
              key      = "karpenter.sh/capacity-type"
              operator = "In"
              values   = ["spot"]
            },
            {
              key      = "karpenter.k8s.aws/instance-category"
              operator = "In"
              values   = ["c", "m", "r"]
            },
            {
              key      = "karpenter.k8s.aws/instance-generation"
              operator = "Gt"
              values   = ["2"]
            }
          ]
          nodeClassRef = {
            apiVersion = "karpenter.k8s.aws/v1beta1"
            kind       = "EC2NodeClass"
            name       = "default"
          }
        }
      }
      limits = {
        cpu = 1000
      }
      disruption = {
        consolidationPolicy = "WhenUnderutilized"
        expireAfter         = "720h"
      }
    }
  }
}

resource "kubernetes_manifest" "default_ec2_node_class" {
  manifest = {
    apiVersion = "karpenter.k8s.aws/v1beta1"
    kind       = "EC2NodeClass"
    metadata = {
      name      = "default"
      namespace = var.karpenter_namespace
    }
    spec = {
      amiFamily = "AL2"
      role      = "KarpenterNodeRole-${var.cluster_name}"
      subnetSelectorTerms = [
        {
          tags = {
            karpenter.sh/discovery = var.cluster_name
          }
        }
      ]
      securityGroupSelectorTerms = [
        {
          tags = {
            karpenter.sh/discovery = var.cluster_name
          }
        }
      ]
      amiSelectorTerms = [
        {
          id = data.aws_ssm_parameter.arm_ami_id.value
        },
        {
          id = data.aws_ssm_parameter.amd_ami_id.value
        }
      ]
    }
  }
}
```

### Instructions to Use the Script

1. Save the above script to a file named `main.tf`.
2. Create a `variables.tf` file to define your variables:
    ```hcl
    variable "region" {
      description = "The AWS region to create resources in"
      type        = string
    }

    variable "cluster_name" {
      description = "The name of the EKS cluster"
      type        = string
    }

    variable "karpenter_namespace" {
      description = "The namespace where Karpenter will be deployed"
      type        = string
      default     = "kube-system"
    }

    variable "aws_partition" {
      description = "AWS partition (e.g., aws, aws-cn, aws-us-gov)"
      type        = string
      default     = "aws"
    }
    ```

3. Initialize Terraform:
    ```sh
    terraform init
    ```

4. Apply the Terraform configuration:
    ```sh
    terraform apply
    ```

Make sure to review and adjust the script according to your specific requirements and environment details. This script automates the steps required for migrating from Cluster Autoscaler to Karpenter, setting up IAM roles, updating ConfigMap, deploying Karpenter, and configuring necessary resources.





#####========####===###=#=#===#####
provider "aws" {
  region = var.region
}

variable "cluster_name" {
  description = "The name of the EKS cluster"
  type        = string
}

variable "karpenter_namespace" {
  default     = "kube-system"
  description = "The namespace where Karpenter will be deployed"
  type        = string
}

variable "aws_partition" {
  default     = "aws"
  description = "AWS partition (e.g., aws, aws-cn, aws-us-gov)"
  type        = string
}

data "aws_region" "current" {}

data "aws_caller_identity" "current" {}

data "aws_eks_cluster" "cluster" {
  name = var.cluster_name
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter/karpenter"
  version    = "0.37.0"
  namespace  = var.karpenter_namespace

  values = [
    {
      settings = {
        clusterName = var.cluster_name
      },
      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = "arn:${var.aws_partition}:iam::${data.aws_caller_identity.current.account_id}:role/KarpenterControllerRole-${var.cluster_name}"
        }
      },
      controller = {
        resources = {
          requests = {
            cpu    = "1",
            memory = "1Gi"
          },
          limits = {
            cpu    = "1",
            memory = "1Gi"
          }
        }
      }
    }
  ]
}

resource "kubernetes_namespace" "karpenter" {
  metadata {
    name = var.karpenter_namespace
  }
}

resource "kubernetes_custom_resource_definition" "node_pool" {
  metadata {
    name = "karpenter.sh_nodepools"
  }
  spec {
    group   = "karpenter.sh"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "nodepools"
      singular = "nodepool"
      kind     = "NodePool"
    }
  }
}

resource "kubernetes_custom_resource_definition" "ec2_node_class" {
  metadata {
    name = "karpenter.k8s.aws_ec2nodeclasses"
  }
  spec {
    group   = "karpenter.k8s.aws"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "ec2nodeclasses"
      singular = "ec2nodeclass"
      kind     = "EC2NodeClass"
    }
  }
}

resource "kubernetes_custom_resource_definition" "node_claim" {
  metadata {
    name = "karpenter.sh_nodeclaims"
  }
  spec {
    group   = "karpenter.sh"
    version = "v1beta1"
    scope   = "Namespaced"
    names {
      plural   = "nodeclaims"
      singular = "nodeclaim"
      kind     = "NodeClaim"
    }
  }
}

resource "kubernetes_manifest" "default_node_pool" {
  manifest = {
    apiVersion = "karpenter.sh/v1beta1"
    kind       = "NodePool"
    metadata = {
      name      = "default"
      namespace = var.karpenter_namespace
    }
    spec = {
      template = {
        spec = {
          requirements = [
            {
              key      = "kubernetes.io/arch"
              operator = "In"
              values   = ["amd64"]
            },
            {
              key      = "kubernetes.io/os"
              operator = "In"
              values   = ["linux"]
            },
            {
              key      = "karpenter.sh/capacity-type"
              operator = "In"
              values   = ["spot"]
            },
            {
              key      = "karpenter.k8s.aws/instance-category"
              operator = "In"
              values   = ["c", "m", "r"]
            },
            {
              key      = "karpenter.k8s.aws/instance-generation"
              operator = "Gt"
              values   = ["2"]
            }
          ]
          nodeClassRef = {
            apiVersion = "karpenter.k8s.aws/v1beta1"
            kind       = "EC2NodeClass"
            name       = "default"
          }
        }
      }
      limits = {
        cpu = 1000
      }
      disruption = {
        consolidationPolicy = "WhenUnderutilized"
        expireAfter         = "720h"
      }
    }
  }
}

resource "kubernetes_manifest" "default_ec2_node_class" {
  manifest = {
    apiVersion = "karpenter.k8s.aws/v1beta1"
    kind       = "EC2NodeClass"
    metadata = {
      name      = "default"
      namespace = var.karpenter_namespace
    }
    spec = {
      amiFamily = "AL2"
      role      = "KarpenterNodeRole-${var.cluster_name}"
      subnetSelectorTerms = [
        {
          tags = {
            karpenter.sh/discovery = var.cluster_name
          }
        }
      ]
      securityGroupSelectorTerms = [
        {
          tags = {
            karpenter.sh/discovery = var.cluster_name
          }
        }
      ]
      amiSelectorTerms = [
        {
          id = data.aws_ssm_parameter.arm_ami_id.value
        },
        {
          id = data.aws_ssm_parameter.amd_ami_id.value
        }
      ]
    }
  }
}
