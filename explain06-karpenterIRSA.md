To install Karpenter on your EKS cluster using IAM Roles for Service Accounts (IRSA) instead of eks-pod-identity-agent, you can follow these steps:

1. **Create an IAM Role for Karpenter:**
   
   Define a new IAM role that Karpenter will use. This role should have the necessary policies attached to it.

   ```hcl
   resource "aws_iam_role" "karpenter" {
     name = "karpenter-controller"
     assume_role_policy = data.aws_iam_policy_document.karpenter_assume_role_policy.json
   }

   data "aws_iam_policy_document" "karpenter_assume_role_policy" {
     statement {
       actions = ["sts:AssumeRoleWithWebIdentity"]
       principals {
         type        = "Federated"
         identifiers = [aws_iam_openid_connect_provider.eks.arn]
       }
       condition {
         test     = "StringEquals"
         variable = "${aws_iam_openid_connect_provider.eks.url}:sub"
         values   = ["system:serviceaccount:karpenter:karpenter"]
       }
     }
   }
   ```

2. **Attach Policies to the IAM Role:**
   
   Attach the required policies to the IAM role. These policies should allow Karpenter to manage EC2 instances and other AWS resources.

   ```hcl
   resource "aws_iam_role_policy_attachment" "karpenter_controller" {
     role       = aws_iam_role.karpenter.name
     policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
   }

   resource "aws_iam_role_policy_attachment" "karpenter_instance_profile" {
     role       = aws_iam_role.karpenter.name
     policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
   }
   ```

3. **Create a Kubernetes Service Account:**

   Create a Kubernetes Service Account in the `karpenter` namespace and annotate it with the ARN of the IAM role created above.

   ```hcl
   resource "kubernetes_service_account" "karpenter" {
     metadata {
       name      = "karpenter"
       namespace = "karpenter"
       annotations = {
         "eks.amazonaws.com/role-arn" = aws_iam_role.karpenter.arn
       }
     }
   }
   ```

4. **Deploy Karpenter using Helm:**

   Use Helm to deploy Karpenter, ensuring that it uses the Service Account with the correct IAM role.

   ```bash
   helm repo add karpenter https://charts.karpenter.sh
   helm repo update
   helm install karpenter karpenter/karpenter \
     --namespace karpenter \
     --create-namespace \
     --set serviceAccount.create=false \
     --set serviceAccount.name=karpenter \
     --set controller.clusterName=<your-cluster-name> \
     --set controller.clusterEndpoint=<your-cluster-endpoint> \
     --set controller.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile
   ```

This setup will use IRSA to give Karpenter the necessary permissions to operate within your EKS cluster on GovCloud. Make sure to replace `<your-cluster-name>` and `<your-cluster-endpoint>` with your actual cluster name and endpoint.

Let me know if you need any further assistance!



            $$$$$$$$$$$$$$$$$$$$$$$$$

Let's convert the steps into a Terraform configuration to deploy Karpenter with IRSA on your EKS cluster in AWS GovCloud.

### Step 1: Create IAM Role and Policies for Karpenter

```hcl
resource "aws_iam_role" "karpenter" {
  name = "karpenter-controller"
  assume_role_policy = data.aws_iam_policy_document.karpenter_assume_role_policy.json
}

data "aws_iam_policy_document" "karpenter_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "${aws_iam_openid_connect_provider.eks.url}:sub"
      values   = ["system:serviceaccount:karpenter:karpenter"]
    }
  }
}

resource "aws_iam_role_policy_attachment" "karpenter_controller_policy" {
  role       = aws_iam_role.karpenter.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role_policy_attachment" "karpenter_instance_profile_policy" {
  role       = aws_iam_role.karpenter.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}
```

### Step 2: Create Kubernetes Service Account

```hcl
resource "kubernetes_service_account" "karpenter" {
  metadata {
    name      = "karpenter"
    namespace = "karpenter"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.karpenter.arn
    }
  }
}
```

### Step 3: Deploy Karpenter using Helm Provider in Terraform

```hcl
provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.eks.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.eks.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.eks.token
  }
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "https://charts.karpenter.sh"
  chart      = "karpenter"
  namespace  = "karpenter"
  create_namespace = true

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = kubernetes_service_account.karpenter.metadata[0].name
  }

  set {
    name  = "controller.clusterName"
    value = aws_eks_cluster.eks.name
  }

  set {
    name  = "controller.clusterEndpoint"
    value = aws_eks_cluster.eks.endpoint
  }

  set {
    name  = "controller.aws.defaultInstanceProfile"
    value = "KarpenterNodeInstanceProfile" # Ensure this IAM instance profile exists
  }
}
```

### Full Terraform Script

Combining all the steps, your full Terraform script would look something like this:

```hcl
provider "aws" {
  region = "us-gov-west-1" # Adjust to your AWS GovCloud region
}

provider "kubernetes" {
  host                   = aws_eks_cluster.eks.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.eks.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.eks.token
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.eks.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.eks.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.eks.token
  }
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.aws_eks_cluster.eks.certificate_authority[0].data]
  url             = aws_eks_cluster.eks.identity.oidc.issuer
}

data "aws_eks_cluster" "eks" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "eks" {
  name = var.cluster_name
}

resource "aws_iam_role" "karpenter" {
  name = "karpenter-controller"
  assume_role_policy = data.aws_iam_policy_document.karpenter_assume_role_policy.json
}

data "aws_iam_policy_document" "karpenter_assume_role_policy" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "${aws_iam_openid_connect_provider.eks.url}:sub"
      values   = ["system:serviceaccount:karpenter:karpenter"]
    }
  }
}

resource "aws_iam_role_policy_attachment" "karpenter_controller_policy" {
  role       = aws_iam_role.karpenter.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role_policy_attachment" "karpenter_instance_profile_policy" {
  role       = aws_iam_role.karpenter.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "kubernetes_service_account" "karpenter" {
  metadata {
    name      = "karpenter"
    namespace = "karpenter"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.karpenter.arn
    }
  }
}

resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "https://charts.karpenter.sh"
  chart      = "karpenter"
  namespace  = "karpenter"
  create_namespace = true

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = kubernetes_service_account.karpenter.metadata[0].name
  }

  set {
    name  = "controller.clusterName"
    value = aws_eks_cluster.eks.name
  }

  set {
    name  = "controller.clusterEndpoint"
    value = aws_eks_cluster.eks.endpoint
  }

  set {
    name  = "controller.aws.defaultInstanceProfile"
    value = "KarpenterNodeInstanceProfile" # Ensure this IAM instance profile exists
  }
}
```

Replace `var.cluster_name` with the name of your EKS cluster. This script sets up the necessary IAM roles and policies, creates the Kubernetes service account with IRSA, and deploys Karpenter using Helm within Terraform.
