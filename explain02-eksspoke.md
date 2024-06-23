Let's break down the configuration for the EKS cluster in the spoke cluster of your hub-and-spoke model. We'll go through each part and explain its purpose and functionality.

### Module Definition

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.8.4"
```

- **`source`**: Specifies the source of the EKS module from the Terraform Registry.
- **`version`**: Specifies the version of the module to use, which is `20.8.4` or a compatible later version.

### Cluster Configuration

```hcl
  cluster_name                   = local.name
  cluster_version                = local.cluster_version
  cluster_endpoint_public_access = true
```

- **`cluster_name`**: Sets the name of the EKS cluster using a local variable `local.name`.
- **`cluster_version`**: Specifies the Kubernetes version for the EKS cluster using `local.cluster_version`.
- **`cluster_endpoint_public_access`**: Configures the cluster endpoint to be publicly accessible.

### VPC Configuration

```hcl
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
```

- **`vpc_id`**: Uses the VPC ID from the VPC module (`module.vpc.vpc_id`).
- **`subnet_ids`**: Uses the private subnets created in the VPC module (`module.vpc.private_subnets`) for the EKS cluster.

### Authentication Mode

```hcl
  authentication_mode = local.authentication_mode
```

- **`authentication_mode`**: Sets the authentication mode for the EKS cluster using a local variable `local.authentication_mode`.

### Cluster Access Entry

```hcl
  enable_cluster_creator_admin_permissions = true

  access_entries = {
    example = {
      principal_arn = aws_iam_role.spoke.arn

      policy_associations = {
        argocd = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = {
            type = "cluster"
          }
        }
      }
    }
  }
```

- **`enable_cluster_creator_admin_permissions`**: Grants admin permissions to the user who created the cluster.
- **`access_entries`**: Defines additional access entries for the cluster.
  - **`example`**: Name of the access entry.
    - **`principal_arn`**: The ARN of the IAM role (`aws_iam_role.spoke.arn`) that will be granted access.
    - **`policy_associations`**: Associates policies with the access entry.
      - **`argocd`**: Name of the policy association.
        - **`policy_arn`**: ARN of the policy to be associated (`AmazonEKSClusterAdminPolicy`).
        - **`access_scope`**: Defines the scope of access.
          - **`type`**: Scope type, in this case, `cluster`.

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
  - **`initial`**: Name of the node group.
    - **`instance_types`**: Specifies the EC2 instance type for the nodes (`t3.medium`).
    - **`min_size`**: Minimum number of nodes in the node group.
    - **`max_size`**: Maximum number of nodes in the node group.
    - **`desired_size`**: Desired number of nodes in the node group.

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

This Terraform configuration sets up an EKS cluster for a spoke cluster in a hub-and-spoke model. Key components include:

1. **Cluster Configuration**: Defines basic settings like name, version, and endpoint access.
2. **VPC Configuration**: Specifies the VPC and subnets for the cluster.
3. **Authentication Mode**: Sets the authentication method for the cluster.
4. **Cluster Access Entry**: Grants specific IAM roles access to the cluster with defined policies.
5. **Node Groups**: Defines managed node groups with specific instance types and sizes.
6. **EKS Addons**: Installs necessary addons like the Pod Identity Agent and VPC CNI with custom configurations.
7. **Tags**: Applies tags to cluster resources for organization and management.

This configuration ensures the spoke cluster is properly integrated within the overall hub-and-spoke model, with appropriate access control, networking, and node management.


### Detailed Explanation: `access_entries` and `cluster_addons`

#### `access_entries`

The `access_entries` block in the EKS module configuration is used to define additional access controls for your EKS cluster. This is particularly useful in a hub-and-spoke model where you might need to grant specific roles or users access to the spoke cluster with fine-grained permissions. Here’s a detailed breakdown:

```hcl
access_entries = {
  example = {
    principal_arn = aws_iam_role.spoke.arn

    policy_associations = {
      argocd = {
        policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
        access_scope = {
          type = "cluster"
        }
      }
    }
  }
}
```

- **`access_entries`**: This block allows you to specify different entities (users, roles, etc.) that should have access to the EKS cluster.
  
  - **`example`**: This is the name of the access entry. You can have multiple entries like this, each with a unique name.

    - **`principal_arn`**: The ARN of the IAM role or user that you want to grant access to. In this example, it’s using `aws_iam_role.spoke.arn`, which refers to an IAM role named `spoke`.

    - **`policy_associations`**: This block allows you to associate one or more policies with the principal specified by `principal_arn`.

      - **`argocd`**: This is the name of the policy association. You can have multiple policy associations under each access entry.

        - **`policy_arn`**: The ARN of the IAM policy that you want to associate with the principal. In this example, it’s using `AmazonEKSClusterAdminPolicy`, which is a predefined policy that grants administrative access to the EKS cluster.

        - **`access_scope`**: This defines the scope of access that the policy provides.
          
          - **`type`**: Specifies the type of access. In this case, it is set to `cluster`, meaning the policy applies to the entire EKS cluster.

##### Why Use `access_entries`?

- **Granular Access Control**: Allows you to grant specific permissions to different roles or users, ensuring that only authorized entities can perform certain actions on the cluster.
- **Security**: By assigning roles with the least privilege necessary, you enhance the security of your cluster.
- **Management**: Simplifies the management of access permissions by centralizing them in your Terraform configuration.

#### `cluster_addons`

The `cluster_addons` block is used to define and configure various addons that enhance the functionality of your EKS cluster. Addons are additional software components that provide essential features and services for your cluster.

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

- **`cluster_addons`**: This block is used to specify the addons that you want to install on your EKS cluster.

  - **`eks-pod-identity-agent`**: Installs the EKS Pod Identity Agent addon.
    
    - **`most_recent`**: Ensures that the most recent version of the addon is installed. The EKS Pod Identity Agent enables the use of IAM roles for Kubernetes service accounts, providing fine-grained permissions for pods.

  - **`vpc-cni`**: Installs the VPC CNI (Container Network Interface) addon.
    
    - **`before_compute`**: Ensures that the VPC CNI addon is deployed before any compute resources (like node groups). This is crucial because the VPC CNI manages the networking setup, and it needs to be ready before nodes are created.
    
    - **`most_recent`**: Ensures that the latest version of the addon is installed.
    
    - **`configuration_values`**: This block allows you to pass specific configuration settings to the addon in JSON format.
      
      - **`env`**: A dictionary of environment variables passed to the VPC CNI plugin.
        
        - **`ENABLE_PREFIX_DELEGATION`**: When set to `"true"`, enables prefix delegation, which allows for more efficient use of IP addresses by allocating prefixes (subnets) instead of individual IP addresses. This helps reduce IP address exhaustion in your VPC.
        
        - **`WARM_PREFIX_TARGET`**: Sets the warm prefix target to `1`. The warm prefix target determines the number of free prefixes that should be maintained. When the number of free prefixes drops below this target, the CNI plugin automatically requests additional prefixes to ensure there are always free IP addresses available for new pods.

##### Why Use `cluster_addons`?

- **Enhanced Functionality**: Addons provide essential features and services that enhance the functionality of your EKS cluster.
- **Configuration Management**: Allows you to configure and manage the settings of each addon directly from your Terraform configuration.
- **Efficiency and Performance**: By configuring addons like VPC CNI with prefix delegation, you can improve the efficiency and performance of your cluster’s networking.
- **Security**: Addons like the EKS Pod Identity Agent enhance the security of your cluster by enabling fine-grained IAM roles for Kubernetes service accounts.

### Summary

- **`access_entries`**: Used to define and manage access control for your EKS cluster. It allows you to grant specific permissions to IAM roles or users, enhancing security and simplifying management.
  
- **`cluster_addons`**: Used to install and configure addons that provide essential features and services for your EKS cluster. Addons like the VPC CNI and EKS Pod Identity Agent enhance networking efficiency and security.

By using `access_entries` and `cluster_addons`, you ensure that your EKS cluster in the spoke of the hub-and-spoke model is secure, well-managed, and equipped with essential functionalities.

