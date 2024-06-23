Let's break down and explain the `module "eks"` configuration in your Terraform code step by step.

### Module Definition

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.8"
```

- **`source`**: Specifies the source of the module, which in this case is the `terraform-aws-modules/eks/aws` module from the Terraform Registry.
- **`version`**: Specifies the version of the module to use, which is version `20.8` or later compatible versions.

### Cluster Configuration

```hcl
  cluster_name                   = local.name
  cluster_version                = local.cluster_version
  cluster_endpoint_public_access = true
```

- **`cluster_name`**: Sets the name of the EKS cluster using a local variable `local.name`.
- **`cluster_version`**: Specifies the Kubernetes version for the EKS cluster using `local.cluster_version`.
- **`cluster_endpoint_public_access`**: Sets the cluster endpoint to be publicly accessible.

### VPC Configuration

```hcl
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
```

- **`vpc_id`**: References the VPC ID from the VPC module (`module.vpc.vpc_id`).
- **`subnet_ids`**: Uses the private subnets created in the VPC module (`module.vpc.private_subnets`) for the EKS cluster.

### Authentication Mode

```hcl
  authentication_mode = local.authentication_mode
```

- **`authentication_mode`**: Sets the authentication mode for the EKS cluster using `local.authentication_mode`.

### Cluster Creator Admin Permissions

```hcl
  enable_cluster_creator_admin_permissions = true
```

- **`enable_cluster_creator_admin_permissions`**: Grants admin permissions to the user who created the cluster.

### Node Groups

```hcl
  eks_managed_node_groups = {
    initial = {
      instance_types = ["t3.medium"]

      min_size     = 3
      max_size     = 10
      desired_size = 3
    }
  }
```

- **`eks_managed_node_groups`**: Defines managed node groups for the EKS cluster.
  - **`initial`**: The name of the node group.
  - **`instance_types`**: Specifies the EC2 instance type for the nodes (e.g., `t3.medium`).
  - **`min_size`**: The minimum number of nodes in the node group.
  - **`max_size`**: The maximum number of nodes in the node group.
  - **`desired_size`**: The desired number of nodes in the node group.

### EKS Addons

```hcl
  cluster_addons = {
    eks-pod-identity-agent = {
      most_recent = true
    }
    vpc-cni = {
      before_compute = true
      most_recent    = true
      configuration_values = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"
          WARM_PREFIX_TARGET       = "1"
        }
      })
    }
  }
```

- **`cluster_addons`**: Defines the addons to be installed in the EKS cluster.
  - **`eks-pod-identity-agent`**: Installs the EKS Pod Identity Agent with the latest version (`most_recent = true`).
  - **`vpc-cni`**: Installs the VPC CNI addon.
    - **`before_compute`**: Ensures the addon is deployed before compute resources.
    - **`most_recent`**: Ensures the latest version of the addon is used.
    - **`configuration_values`**: Specifies configuration values for the addon in JSON format.
      - **`env`**: Environment variables for the addon configuration.
        - **`ENABLE_PREFIX_DELEGATION`**: Enables prefix delegation.
        - **`WARM_PREFIX_TARGET`**: Sets the warm prefix target to `1`.

### Tags

```hcl
  tags = local.tags
}
```

- **`tags`**: Applies the tags defined in `local.tags` to the EKS cluster and its resources.

### Summary

This Terraform configuration defines an EKS cluster using the `terraform-aws-modules/eks/aws` module. It sets up the cluster with specified parameters like name, version, VPC configuration, authentication mode, managed node groups, and addons. It also applies tags to the cluster resources. The configuration ensures that the EKS cluster is deployed within the specified VPC and subnets, with managed node groups for compute resources and necessary addons like the VPC CNI for networking.





Sure, let's break down each part of the `cluster_addons` configuration to understand what it does.

### `cluster_addons`

The `cluster_addons` block is used to define various addons that can be installed on your EKS cluster. Addons are pre-configured software components that enhance the functionality of your cluster. In this case, two addons are specified: `eks-pod-identity-agent` and `vpc-cni`.

#### `eks-pod-identity-agent`

```hcl
eks-pod-identity-agent = {
  most_recent = true
}
```

- **`most_recent`**: This parameter ensures that the most recent version of the `eks-pod-identity-agent` addon is installed. The `eks-pod-identity-agent` addon is used to enable IAM roles for Kubernetes service accounts, allowing for more fine-grained permissions for pods.

#### `vpc-cni`

```hcl
vpc-cni = {
  before_compute = true
  most_recent    = true
  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"
      WARM_PREFIX_TARGET       = "1"
    }
  })
}
```

- **`before_compute`**: This parameter specifies that the VPC CNI addon should be deployed before any compute resources (like node groups). This ensures that the networking setup is ready before the nodes are created, which is crucial for network configuration.
- **`most_recent`**: Ensures that the most recent version of the VPC CNI addon is installed. The VPC CNI (Container Network Interface) plugin is responsible for managing network interfaces in your Kubernetes pods and nodes.
- **`configuration_values`**: This block allows you to pass specific configuration settings to the addon in JSON format. 

#### Configuration Values

The `configuration_values` block contains environment variables for configuring the VPC CNI addon:

- **`env`**: This is a dictionary of environment variables passed to the VPC CNI plugin.

  - **`ENABLE_PREFIX_DELEGATION`**: When set to `"true"`, this enables prefix delegation in the VPC CNI. Prefix delegation allows for more efficient use of IP addresses by allocating prefixes (subnets) instead of individual IP addresses. This can help to reduce IP address exhaustion in your VPC.
  
  - **`WARM_PREFIX_TARGET`**: This sets the warm prefix target to `1`. The warm prefix target determines the number of free prefixes that should be maintained. When the number of free prefixes drops below this target, the CNI plugin will automatically request additional prefixes to ensure that there are always free IP addresses available for new pods.

### Summary

- **`eks-pod-identity-agent`**: Installs the EKS Pod Identity Agent to enable IAM roles for Kubernetes service accounts, ensuring the latest version is used.
- **`vpc-cni`**: Installs the VPC CNI plugin with the latest version and specific configurations:
  - **`before_compute`**: Ensures the addon is installed before any compute resources.
  - **`ENABLE_PREFIX_DELEGATION`**: Enables efficient IP address management through prefix delegation.
  - **`WARM_PREFIX_TARGET`**: Configures the warm prefix target to maintain a certain number of free prefixes.

These addons enhance the functionality and efficiency of your EKS cluster, ensuring proper network configuration and security practices.





Sure, let's dive into the `authentication_mode` setting in your EKS module configuration.

### Authentication Mode

```hcl
authentication_mode = local.authentication_mode
```

### Explanation

The `authentication_mode` setting in the EKS module defines how authentication is managed for your EKS cluster. There are typically a few modes of authentication that you can choose from when setting up an EKS cluster:

1. **IAM Roles for Service Accounts (IRSA)**: This allows Kubernetes service accounts to assume IAM roles. This is useful for granting fine-grained permissions to specific pods running in your cluster.

2. **AWS IAM**: This allows you to use AWS IAM identities (users, roles) to authenticate to the Kubernetes API server.

3. **AWS IAM Authenticator**: This uses the `aws-iam-authenticator` to enable authentication using AWS IAM identities. This mode allows you to map AWS IAM users and roles to Kubernetes RBAC roles.

### Setting in Context

In your configuration, the `authentication_mode` is set using a local variable `local.authentication_mode`. The value of `local.authentication_mode` would determine the mode of authentication for your EKS cluster. Here’s how different values might be used:

#### Example Values for `authentication_mode`

- **`"aws"`**: This would indicate that AWS IAM roles are used for authentication.
- **`"iam"`**: This could indicate that IAM roles for service accounts (IRSA) are used.
- **`"oidc"`**: This might indicate the use of an OpenID Connect provider for authentication.
- **`"eks"`**: This would typically indicate using the default EKS authentication method, which involves mapping IAM users and roles to Kubernetes RBAC roles.

### Practical Use

Here’s an example of how you might set this variable in your `locals` block:

```hcl
locals {
  authentication_mode = "aws"
}
```

In this case, your EKS cluster would be configured to use AWS IAM for authentication. This means that IAM users and roles can be mapped to Kubernetes RBAC roles and used to authenticate to the Kubernetes API server.

### Importance

Choosing the right authentication mode is critical for the security and management of your EKS cluster. Here’s why:

- **Security**: Using IAM roles and policies ensures that you can control access to your cluster using AWS's robust security features.
- **Granular Permissions**: With IRSA, you can grant specific permissions to pods, ensuring that they only have the access they need.
- **Ease of Management**: AWS IAM allows for centralized management of user and role permissions, making it easier to manage access to your EKS cluster.

### Summary

The `authentication_mode` setting in your EKS module configuration specifies how authentication is handled for your EKS cluster. It leverages AWS IAM capabilities to provide secure and manageable access control, enabling the mapping of IAM users and roles to Kubernetes RBAC roles. The specific value of `local.authentication_mode` determines which method of authentication is used, enhancing the cluster's security and manageability.






