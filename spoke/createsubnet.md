Sure, I can help you create 3 public and 3 private subnets in an existing VPC, along with the route tables, internet gateway, and NAT gateway for the private subnets. Here's a step-by-step guide using Terraform:

1. **Define Variables:** First, let's define variables for your CIDR values and the existing VPC ID.

2. **Create Subnets:** Then, we will create the public and private subnets.

3. **Create an Internet Gateway:** Attach this to the VPC for public subnets.

4. **Create a NAT Gateway:** Place this in one of the public subnets for private subnet internet access.

5. **Create Route Tables:** Create separate route tables for public and private subnets and associate the subnets with these route tables.

### Step-by-Step Terraform Configuration

1. **Define Variables**

```hcl
variable "vpc_id" {
  description = "The ID of the existing VPC"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for the public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for the private subnets"
  type        = list(string)
}
```

2. **Create Subnets**

```hcl
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = var.vpc_id
  cidr_block              = element(var.public_subnet_cidrs, count.index)
  availability_zone       = element(data.aws_availability_zones.available.names, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet ${count.index + 1}"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = var.vpc_id
  cidr_block        = element(var.private_subnet_cidrs, count.index)
  availability_zone = element(data.aws_availability_zones.available.names, count.index)

  tags = {
    Name = "Private Subnet ${count.index + 1}"
  }
}
```

3. **Create an Internet Gateway**

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = var.vpc_id

  tags = {
    Name = "Internet Gateway"
  }
}
```

4. **Create a NAT Gateway**

```hcl
resource "aws_eip" "nat" {
  vpc = true
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "NAT Gateway"
  }
}
```

5. **Create Route Tables**

```hcl
resource "aws_route_table" "public" {
  vpc_id = var.vpc_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "Public Route Table"
  }
}

resource "aws_route_table" "private" {
  vpc_id = var.vpc_id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Name = "Private Route Table"
  }
}
```

6. **Associate Subnets with Route Tables**

```hcl
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

### Example Values for Variables

You can define your variables in a separate `.tfvars` file or directly in your Terraform configuration:

```hcl
variable "vpc_id" {
  default = "vpc-12345678"
}

variable "public_subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "private_subnet_cidrs" {
  default = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
}
```

### Running the Configuration

1. **Initialize Terraform:**

   ```sh
   terraform init
   ```

2. **Plan the Configuration:**

   ```sh
   terraform plan -var-file="variables.tfvars"
   ```

3. **Apply the Configuration:**

   ```sh
   terraform apply -var-file="variables.tfvars"
   ```

Feel free to replace the variable values with your actual CIDR blocks and VPC ID. Let me know if you need any further adjustments or assistance!