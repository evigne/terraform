When configuring EKS clusters, the networking setup, including the subnets and NAT gateways, can significantly impact the creation and functionality of your nodes. Here are a few key points to check:

### Key Areas to Check:

1. **Subnet Configuration**:
   - Ensure that your EKS nodes are being launched in subnets that are tagged correctly for EKS.
   - Public subnets should have the tag `kubernetes.io/role/elb` set to `1`.
   - Private subnets should have the tag `kubernetes.io/role/internal-elb` set to `1`.

2. **Route Tables**:
   - Public subnets should have a route to an Internet Gateway (IGW).
   - Private subnets should have a route to a NAT Gateway (NGW) to enable instances in private subnets to communicate with the internet for updates, etc.

3. **NAT Gateway**:
   - Ensure your NAT Gateway is properly configured and attached to a public subnet.
   - Confirm that private subnets route outbound traffic through the NAT Gateway.

4. **Security Groups**:
   - Verify that the security groups associated with your nodes allow the necessary traffic (e.g., SSH access, traffic between nodes).

5. **EKS Cluster IAM Role**:
   - Make sure your EKS cluster has the necessary IAM roles and permissions.

6. **VPC Configuration**:
   - Verify that your VPC has the appropriate CIDR blocks and does not conflict with other networks.

### Example Terraform Configuration for Subnets and NAT Gateway:

Here's an example of a Terraform configuration to ensure you have the correct setup:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.${count.index}.0/24"
  map_public_ip_on_launch = true
  availability_zone = element(local.azs, count.index)

  tags = {
    "kubernetes.io/role/elb" = "1"
    Name                     = "eks-public-subnet-${count.index}"
  }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.${count.index}.0/24"
  availability_zone = element(local.azs, count.index)

  tags = {
    "kubernetes.io/role/internal-elb" = "1"
    Name                              = "eks-private-subnet-${count.index}"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = element(aws_subnet.public.*.id, count.index)
  route_table_id = aws_route_table.public.id
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = element(aws_subnet.private.*.id, count.index)
  route_table_id = aws_route_table.private.id
}

resource "aws_eip" "nat" {
  vpc = true
}
```

### Steps to Debug:

1. **Check the AWS Management Console**:
   - Navigate to the VPC dashboard and ensure all subnets, route tables, and NAT gateways are correctly configured.

2. **Review EKS Node Logs**:
   - Look at the CloudFormation stack events for your EKS nodes to find any specific errors.

3. **EKS Cluster Networking**:
   - Verify the cluster's VPC and subnet configurations in the EKS console.

If you can provide specific error messages or logs, I can offer more targeted advice.