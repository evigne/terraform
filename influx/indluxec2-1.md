Here’s an updated version that includes a Route 53 record to resolve the InfluxDB instance using a domain name. This is useful for accessing the instance through a friendly DNS name rather than directly using the IP address.

Modified Configuration with Route 53

# Define a Security Group for InfluxDB
resource "aws_security_group" "influxdb_sg" {
  name        = "influxdb_security_group"
  description = "Allow InfluxDB and SSH traffic"

  ingress {
    from_port   = 8086
    to_port     = 8086
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Open InfluxDB port to the world (modify for security)
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # SSH port, restrict to your IP for security
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
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

Explanation of the Route 53 Configuration

	•	zone_id: This is the ID of your Route 53 Hosted Zone. You should replace var.zone_id with the actual hosted zone ID or define zone_id as a variable.
	•	DNS Record (name): Specifies the subdomain for your InfluxDB instance. You can use any domain managed in Route 53, e.g., influxdb.example.com.
	•	type: The type is set to “A” for an IPv4 address record.
	•	TTL (ttl): This is the time-to-live for the DNS record, set to 300 seconds by default.

Required Variables

To make this work, you need to define a few additional variables in variables.tf:

# variables.tf

variable "zone_id" {
  description = "The Route 53 Zone ID for DNS records"
  type        = string
}

variable "name" {
  description = "The DNS name for the InfluxDB instance"
  type        = string
}

This configuration will create an EC2 instance with an attached EBS volume and an accessible DNS name through Route 53. Let me know if you need further customization!