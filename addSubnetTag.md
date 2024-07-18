To tag private subnets for multiple EKS clusters in the same VPC using Terraform, you can define your subnets and apply the necessary tags. Below is an example of how you can achieve this in Terraform:

### Terraform Configuration

1. **Define Variables:**
   Define variables for your subnets and cluster names.

   ```hcl
   variable "clusters" {
     description = "Map of cluster names to their private subnet IDs"
     type        = map(list(string))
     default     = {
       cluster1 = ["subnet-12345678", "subnet-23456789"]
       cluster2 = ["subnet-34567890", "subnet-45678901"]
     }
   }
   ```

2. **Tagging Subnets:**
   Use the `aws_subnet` resource to tag the subnets. Here's how you can loop through the clusters and apply the tags.

   ```hcl
   provider "aws" {
     region = "us-west-2"
   }

   resource "aws_vpc" "main" {
     cidr_block = "10.0.0.0/16"
   }

   resource "aws_subnet" "private" {
     count = length(flatten(values(var.clusters)))
     vpc_id     = aws_vpc.main.id
     cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)

     tags = {
       Name                           = "private-subnet-${count.index}"
       "karpenter.sh/discovery" = lookup({for k, v in var.clusters : v[count.index] => k}, aws_subnet.private[count.index].id, "default")
     }
   }
   ```

3. **Applying the Configuration:**
   Initialize Terraform and apply the configuration.

   ```sh
   terraform init
   terraform apply
   ```

### Complete Example

Here is the complete Terraform script to tag the subnets:

```hcl
variable "clusters" {
  description = "Map of cluster names to their private subnet IDs"
  type        = map(list(string))
  default     = {
    cluster1 = ["subnet-12345678", "subnet-23456789"]
    cluster2 = ["subnet-34567890", "subnet-45678901"]
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  count = length(flatten(values(var.clusters)))
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)

  tags = {
    Name                           = "private-subnet-${count.index}"
    "karpenter.sh/discovery" = lookup({for k, v in var.clusters : v[count.index] => k}, aws_subnet.private[count.index].id, "default")
  }
}
```

### Explanation:
1. **Variable Definition:**
   - The `clusters` variable holds a map of cluster names to their respective private subnet IDs.

2. **AWS Provider Configuration:**
   - Configures the AWS provider to use a specific region.

3. **VPC Resource:**
   - Creates a VPC with a CIDR block.

4. **Subnet Resource:**
   - Creates subnets and tags them with the `karpenter.sh/discovery` key, using the cluster name as the value.

5. **Tag Lookup:**
   - The `lookup` function is used to dynamically assign the correct cluster name to each subnet based on the provided subnet IDs.

### Applying the Configuration:
Run the following commands to apply the Terraform configuration:

```sh
terraform init
terraform apply
```

This configuration ensures that each private subnet is tagged with the correct `karpenter.sh/discovery` key for each cluster, allowing Karpenter to discover and use them appropriately.

            ##################

You're correct that you can only assign one value to the `karpenter.sh/discovery` tag for a given subnet. This means each private subnet can only be associated with one cluster for Karpenter's discovery purposes.

To manage multiple EKS clusters within the same VPC but using separate private subnets, you'll need to ensure that each cluster has its own dedicated set of subnets. Here's how you can achieve this with Terraform:

### Terraform Configuration for Multiple Clusters

1. **Define Variables:**
   Define variables for your clusters and their respective private subnet CIDR blocks.

   ```hcl
   variable "clusters" {
     description = "Map of cluster names to their private subnet CIDR blocks"
     type        = map(list(string))
     default     = {
       clusterA = ["10.0.1.0/24", "10.0.2.0/24"]
       clusterB = ["10.0.3.0/24", "10.0.4.0/24"]
     }
   }
   ```

2. **Create VPC:**
   Create a VPC where your clusters will reside.

   ```hcl
   provider "aws" {
     region = "us-west-2"
   }

   resource "aws_vpc" "main" {
     cidr_block = "10.0.0.0/16"
   }
   ```

3. **Create Subnets and Apply Tags:**
   Loop through the clusters and create subnets for each, applying the necessary tags.

   ```hcl
   resource "aws_subnet" "private" {
     for_each = {for k, v in var.clusters : k => v}

     count    = length(each.value)
     vpc_id   = aws_vpc.main.id
     cidr_block = each.value[count.index]

     tags = {
       Name                           = "private-subnet-${each.key}-${count.index}"
       "karpenter.sh/discovery"       = each.key
     }
   }
   ```

### Complete Example

Here's the full Terraform script:

```hcl
variable "clusters" {
  description = "Map of cluster names to their private subnet CIDR blocks"
  type        = map(list(string))
  default     = {
    clusterA = ["10.0.1.0/24", "10.0.2.0/24"]
    clusterB = ["10.0.3.0/24", "10.0.4.0/24"]
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  for_each = {for k, v in var.clusters : k => v}

  count    = length(each.value)
  vpc_id   = aws_vpc.main.id
  cidr_block = each.value[count.index]

  tags = {
    Name                     = "private-subnet-${each.key}-${count.index}"
    "karpenter.sh/discovery" = each.key
  }
}
```

### Explanation:

1. **Variable Definition:**
   - The `clusters` variable holds a map of cluster names to their respective private subnet CIDR blocks.

2. **AWS Provider Configuration:**
   - Configures the AWS provider to use a specific region.

3. **VPC Resource:**
   - Creates a VPC with a CIDR block.

4. **Subnet Resource:**
   - Loops through each cluster and creates the specified subnets, tagging each subnet with the appropriate `karpenter.sh/discovery` value.

### Applying the Configuration:
Run the following commands to apply the Terraform configuration:

```sh
terraform init
terraform apply
```

This setup ensures that each cluster has its own dedicated subnets within the VPC, tagged appropriately for Karpenter's discovery, avoiding conflicts and allowing each cluster to manage its resources independently.