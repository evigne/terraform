To get the details of an existing VPC and its subnets using Terraform, you can use the `data` sources to retrieve information about the existing infrastructure. Here's how you can do it:

### Step-by-Step Guide

1. **Retrieve VPC Details:**
   Use the `aws_vpc` data source to fetch the details of your existing VPC.

   ```hcl
   data "aws_vpc" "selected" {
     filter {
       name   = "tag:Name"
       values = ["your-vpc-name"]
     }
   }
   ```

2. **Retrieve Subnet Details:**
   Use the `aws_subnet_ids` data source to get the IDs of the subnets within the VPC, and then use the `aws_subnet` data source to get details of each subnet.

   ```hcl
   data "aws_subnet_ids" "selected" {
     vpc_id = data.aws_vpc.selected.id
   }

   data "aws_subnet" "selected" {
     for_each = toset(data.aws_subnet_ids.selected.ids)
     id       = each.key
   }
   ```

3. **Use Retrieved Details to Create Other Resources:**
   You can now use the retrieved VPC and subnet details to create other resources.

### Example Terraform Script

Here's a complete example Terraform script that retrieves the VPC and subnet details and then uses these details to create an EC2 instance in the retrieved subnets.

```hcl
provider "aws" {
  region = "us-west-2"
}

# Retrieve the VPC details
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Name"
    values = ["your-vpc-name"]
  }
}

# Retrieve the subnet IDs within the VPC
data "aws_subnet_ids" "selected" {
  vpc_id = data.aws_vpc.selected.id
}

# Retrieve details for each subnet
data "aws_subnet" "selected" {
  for_each = toset(data.aws_subnet_ids.selected.ids)
  id       = each.key
}

# Example: Create an EC2 instance in each subnet
resource "aws_instance" "example" {
  for_each = data.aws_subnet.selected

  ami           = "ami-0c55b159cbfafe1f0" # Replace with your AMI ID
  instance_type = "t2.micro"
  subnet_id     = each.value.id

  tags = {
    Name = "example-instance-${each.key}"
  }
}

output "vpc_id" {
  value = data.aws_vpc.selected.id
}

output "subnet_ids" {
  value = data.aws_subnet_ids.selected.ids
}

output "subnet_cidrs" {
  value = { for k, v in data.aws_subnet.selected : k => v.cidr_block }
}
```

### Explanation:

1. **AWS Provider:**
   - Configures the AWS provider to use the specified region.

2. **VPC Data Source:**
   - Retrieves the VPC details by filtering based on the VPC name tag.

3. **Subnet IDs Data Source:**
   - Retrieves the IDs of the subnets within the specified VPC.

4. **Subnet Data Source:**
   - Retrieves the details for each subnet using the subnet IDs.

5. **EC2 Instance Resource:**
   - Creates an EC2 instance in each of the retrieved subnets. This is just an example resource to demonstrate how you can use the retrieved subnet details.

6. **Outputs:**
   - Outputs the VPC ID, subnet IDs, and subnet CIDR blocks for verification and further use.

### Applying the Configuration:

Run the following commands to apply the Terraform configuration:

```sh
terraform init
terraform apply
```

This will retrieve the existing VPC and subnet details and create the specified resources using these details.