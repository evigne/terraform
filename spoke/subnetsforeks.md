To create subnets for an EKS cluster using Terraform, you need to define VPC, subnet, route tables, and necessary internet gateways and NAT gateways. Here's a detailed example of how to create public and private subnets for an EKS cluster:

### Step-by-Step Guide:

1. **Define the VPC**:
   Create a new VPC for your EKS cluster.
   
```hcl
resource "aws_vpc" "eks_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "eks-vpc"
  }
}
```

2. **Create Internet Gateway**:
   Create an internet gateway for your VPC.

```hcl
resource "aws_internet_gateway" "eks_igw" {
  vpc_id = aws_vpc.eks_vpc.id
  tags = {
    Name = "eks-igw"
  }
}
```

3. **Create Subnets**:
   Create public and private subnets in different availability zones.

```hcl
resource "aws_subnet" "eks_public_subnet" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = element(var.public_subnet_cidrs, count.index)
  availability_zone = element(var.availability_zones, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "eks-public-subnet-${count.index}"
  }
}

resource "aws_subnet" "eks_private_subnet" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = element(var.private_subnet_cidrs, count.index)
  availability_zone = element(var.availability_zones, count.index)

  tags = {
    Name = "eks-private-subnet-${count.index}"
  }
}
```

4. **Create Route Tables**:
   Create route tables for public and private subnets.

```hcl
resource "aws_route_table" "eks_public_rt" {
  vpc_id = aws_vpc.eks_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.eks_igw.id
  }

  tags = {
    Name = "eks-public-rt"
  }
}

resource "aws_route_table" "eks_private_rt" {
  vpc_id = aws_vpc.eks_vpc.id
  tags = {
    Name = "eks-private-rt"
  }
}
```

5. **Associate Route Tables**:
   Associate the public and private subnets with their respective route tables.

```hcl
resource "aws_route_table_association" "public_subnet_association" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = element(aws_subnet.eks_public_subnet.*.id, count.index)
  route_table_id = aws_route_table.eks_public_rt.id
}

resource "aws_route_table_association" "private_subnet_association" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = element(aws_subnet.eks_private_subnet.*.id, count.index)
  route_table_id = aws_route_table.eks_private_rt.id
}
```

6. **Create NAT Gateway**:
   Create a NAT gateway for the private subnets.

```hcl
resource "aws_eip" "nat" {
  count = length(var.public_subnet_cidrs)
  vpc   = true

  tags = {
    Name = "eks-nat-eip-${count.index}"
  }
}

resource "aws_nat_gateway" "eks_nat_gateway" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = element(aws_eip.nat.*.id, count.index)
  subnet_id     = element(aws_subnet.eks_public_subnet.*.id, count.index)

  tags = {
    Name = "eks-nat-gateway-${count.index}"
  }
}

resource "aws_route" "private_subnet_nat_gateway" {
  count                   = length(var.private_subnet_cidrs)
  route_table_id          = aws_route_table.eks_private_rt.id
  destination_cidr_block  = "0.0.0.0/0"
  nat_gateway_id          = element(aws_nat_gateway.eks_nat_gateway.*.id, count.index)
}
```

### Variables File (`variables.tf`):

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "List of public subnet CIDRs"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "List of private subnet CIDRs"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b"]
}
```

This setup provides you with a VPC containing public and private subnets in multiple availability zones, which is a best practice for an EKS cluster deployment. Adjust the CIDR blocks and availability zones as needed for your specific requirements.