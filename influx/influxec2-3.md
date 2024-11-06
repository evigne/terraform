Based on the provided AWS CloudFormation template references, I’ll modify our Terraform configuration to:

	1.	Use a specific AMI and instance type as specified.
	2.	Create a security group allowing HTTP (port 80) and SSH (port 22).
	3.	Include a UserData script to install Docker, pull the InfluxDB image, and run it on port 80.

Modified Terraform Configuration

# Define Security Group for App Node with HTTP and SSH Access
resource "aws_security_group" "app_node_sg" {
  name        = "app_node_sg"
  description = "Security group for the app nodes that allow HTTP and SSH access"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Open HTTP to all IPs
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Open SSH to all IPs (consider restricting this for security)
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create an EC2 Instance for InfluxDB with Docker Installed
resource "aws_instance" "app_node" {
  ami                    = "ami-024a64a6685d05041"  # Specified AMI
  instance_type          = "t2.micro"              # Specified instance type
  key_name               = "Influx20"              # Specified key pair
  vpc_security_group_ids = [aws_security_group.app_node_sg.id]
  ebs_optimized          = true

  # User Data to Install Docker and Run InfluxDB
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -qq
              apt-get install -y apt-transport-https ca-certificates
              apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
              apt-get update -qq
              apt-get purge lxc-docker || true
              curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
              add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
              apt-get -y install linux-image-extra-$(uname -r) linux-image-extra-virtual
              apt-get -y install docker-ce
              usermod -aG docker ubuntu
              docker image pull quay.io/influxdb/influxdb:2.0.0-alpha
              docker container run -p 80:9999 quay.io/influxdb/influxdb:2.0.0-alpha
              EOF

  tags = {
    Name = "App-Node-Instance"
  }
}

# Create an EBS Volume for Data Storage
resource "aws_ebs_volume" "influxdb_data" {
  size              = var.data_disk_size  # Specify the disk size in GB
  encrypted         = true
  type              = "gp2"               # General Purpose SSD
  availability_zone = aws_instance.app_node.availability_zone
}

# Attach EBS Volume to the App Node EC2 Instance
resource "aws_volume_attachment" "influxdb_data_attachment" {
  device_name = "/dev/xvdf"  # Modify based on OS and preference
  volume_id   = aws_ebs_volume.influxdb_data.id
  instance_id = aws_instance.app_node.id
  force_detach = true
}

# Route 53 DNS Record for InfluxDB
resource "aws_route53_record" "influxdb_dns" {
  zone_id = var.zone_id  # ID of your Route 53 Hosted Zone
  name    = "${var.name}"  # Domain name for the InfluxDB instance, e.g., "influxdb.example.com"
  type    = "A"
  ttl     = 300
  records = [aws_instance.app_node.public_ip]
}

# Output the public IP and DNS of the instance
output "influxdb_public_ip" {
  value = aws_instance.app_node.public_ip
}

output "influxdb_dns" {
  value = aws_route53_record.influxdb_dns.fqdn
}

Explanation of Modifications

	•	AMI and Instance Type: Set to ami-024a64a6685d05041 and t2.micro as specified.
	•	Security Group: app_node_sg allows HTTP (80) and SSH (22) from all IPs (0.0.0.0/0), replicating the CloudFormation configuration.
	•	UserData Script: The script installs Docker, pulls the specified InfluxDB image, and runs it on port 80 (mapped from container port 9999).
	•	EBS Volume: An EBS volume is attached for data persistence, matching the earlier setup.
	•	Route 53 Record: Adds a DNS record to access the instance via a domain name.

This setup runs InfluxDB in a Docker container and exposes it on port 80. For security, consider restricting SSH access to a specific IP range rather than allowing all (0.0.0.0/0). Let me know if you need further customization!