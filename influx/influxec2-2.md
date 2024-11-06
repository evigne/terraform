To enhance security, we can tighten the Security Group rules to limit access to only trusted IPs and reduce open ports. Here’s an updated configuration with better security practices, including restricting SSH and InfluxDB access.

Enhanced Security Configuration

In this configuration:

	•	Only specific IP ranges are allowed to access the InfluxDB port (8086) and SSH (22).
	•	Ingress rules are restricted to trusted IPs, and we add additional security for outbound traffic by limiting it to essential traffic only.

# Define a Security Group with Restricted Access for InfluxDB
resource "aws_security_group" "influxdb_sg" {
  name        = "secure_influxdb_security_group"
  description = "Restricted access for InfluxDB and SSH traffic"

  # InfluxDB port restricted to trusted IPs
  ingress {
    from_port   = 8086
    to_port     = 8086
    protocol    = "tcp"
    cidr_blocks = var.allowed_influxdb_cidrs  # List of trusted CIDRs allowed for InfluxDB access
  }

  # SSH port restricted to a specific IP for remote management
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs  # List of trusted CIDRs for SSH access
  }

  # Outbound traffic restricted to essential CIDR ranges, such as 0.0.0.0/0 for simplicity
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # Modify this to narrow outbound access as needed
  }
}

# Create an EC2 Instance for InfluxDB
resource "aws_instance" "influxdb" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.influxdb_sg.id]
  ebs_optimized          = true

  # User Data for InfluxDB installation
  user_data = <<-EOF
              #!/bin/bash
              sudo apt-get update
              sudo apt-get install -y wget
              wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
              echo "deb https://repos.influxdata.com/ubuntu focal stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
              sudo apt-get update && sudo apt-get install -y influxdb
              sudo systemctl enable influxdb
              sudo systemctl start influxdb
              EOF

  tags = {
    Name = "InfluxDB-Instance"
  }
}

# Create an EBS Volume for Data Storage
resource "aws_ebs_volume" "influxdb_data" {
  size              = var.data_disk_size  # Specify the disk size in GB
  encrypted         = true
  type              = "gp2"               # General Purpose SSD
  availability_zone = aws_instance.influxdb.availability_zone
}

# Attach EBS Volume to the InfluxDB EC2 Instance
resource "aws_volume_attachment" "influxdb_data_attachment" {
  device_name = "/dev/xvdf"  # Modify based on OS and preference
  volume_id   = aws_ebs_volume.influxdb_data.id
  instance_id = aws_instance.influxdb.id
  force_detach = true
}

# Route 53 DNS Record for InfluxDB
resource "aws_route53_record" "influxdb_dns" {
  zone_id = var.zone_id  # ID of your Route 53 Hosted Zone
  name    = "${var.name}"  # Domain name for the InfluxDB instance, e.g., "influxdb.example.com"
  type    = "A"
  ttl     = 300
  records = [aws_instance.influxdb.public_ip]
}

# Output the public IP and DNS of the instance
output "influxdb_public_ip" {
  value = aws_instance.influxdb.public_ip
}

output "influxdb_dns" {
  value = aws_route53_record.influxdb_dns.fqdn
}

Additional Security Variables

Add the following variables to variables.tf for controlling allowed IPs for InfluxDB and SSH access:

# variables.tf

variable "allowed_influxdb_cidrs" {
  description = "List of trusted CIDR blocks allowed for InfluxDB access"
  type        = list(string)
  default     = ["192.0.2.0/24"]  # Replace with your trusted IPs for InfluxDB access
}

variable "allowed_ssh_cidrs" {
  description = "List of trusted CIDR blocks allowed for SSH access"
  type        = list(string)
  default     = ["198.51.100.0/32"]  # Replace with your specific IP for SSH access
}

variable "zone_id" {
  description = "The Route 53 Zone ID for DNS records"
  type        = string
}

variable "name" {
  description = "The DNS name for the InfluxDB instance"
  type        = string
}

Explanation of Enhanced Security

	•	Allowed CIDR Blocks: Limits access to the InfluxDB and SSH ports to only the specified IP ranges. Replace the CIDR values with trusted IP ranges for your setup.
	•	Ingress and Egress Rules: The egress rule allows all outbound traffic by default but can be customized further to limit outgoing traffic.
	•	Encrypted EBS Volume: Ensures data on the EBS volume is encrypted.

This setup enhances security by ensuring that only trusted IPs can access InfluxDB and SSH, reducing the risk of unauthorized access.