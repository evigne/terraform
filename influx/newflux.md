Switching from EFS to EBS requires only a few adjustments to the Terraform configuration. With EBS, you’ll attach a block storage volume directly to the EC2 instance for storing InfluxDB data, providing high-performance storage on a single instance.

Here’s how to modify the configuration to use EBS instead of EFS:

Step 1: Define an EBS Volume

Add an EBS volume resource to allocate storage specifically for InfluxDB data.

# Create an EBS volume for InfluxDB data storage
'''
resource "aws_ebs_volume" "influxdb_data" {
  availability_zone = aws_instance.influxdb_instance.availability_zone  # Same AZ as the EC2 instance
  size              = 20  # Size in GB; adjust as needed
  type              = "gp2"  # General Purpose SSD
  encrypted         = true   # Enable encryption for security
  tags = {
    Name = "InfluxDB-Data"
  }
}
'''
Step 2: Attach the EBS Volume to the EC2 Instance

Create an attachment to link the EBS volume to the EC2 instance.

# Attach the EBS volume to the EC2 instance
resource "aws_volume_attachment" "influxdb_data_attachment" {
  device_name = "/dev/xvdf"  # Device name where the volume will attach
  volume_id   = aws_ebs_volume.influxdb_data.id
  instance_id = aws_instance.influxdb_instance.id
}

Step 3: Update the User Data Script to Mount EBS Volume

In the user data script, include commands to format and mount the EBS volume. The data directory for InfluxDB will be placed on this volume.
'''
resource "aws_instance" "influxdb_instance" {
  ami                    = "ami-024a64a6685d05041"  # Modify based on region
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.influxdb_sg.id]
  key_name               = "your_key_pair"  # Ensure you have this key locally
  associate_public_ip_address = true

  # User Data to Install InfluxDB and Configure Nginx
  user_data = <<-EOF
              #!/bin/bash
              # Update packages
              sudo apt-get update
              sudo apt-get -y upgrade

              # Format and mount the EBS volume
              sudo mkfs.ext4 /dev/xvdf
              sudo mkdir -p /mnt/influxdb/data
              sudo mount /dev/xvdf /mnt/influxdb/data
              echo "/dev/xvdf /mnt/influxdb/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

              # Install InfluxDB
              wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
              echo "deb https://repos.influxdata.com/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
              sudo apt-get update
              sudo apt-get install -y influxdb

              # Configure InfluxDB to use the mounted volume for data storage
              sudo sed -i 's|/var/lib/influxdb|/mnt/influxdb/data|' /etc/influxdb/influxdb.conf

              # Start InfluxDB and enable it to run on boot
              sudo systemctl start influxdb
              sudo systemctl enable influxdb

              # Install Nginx
              sudo apt-get install -y nginx

              # Configure Nginx as a reverse proxy
              cat <<NGINX_CONF | sudo tee /etc/nginx/sites-available/influxdb
              server {
                  listen 80;
                  server_name ${var.domain_name};  # Use variable for Route 53 domain

                  location / {
                      proxy_pass http://localhost:8086;
                      proxy_set_header Host \$host;
                      proxy_set_header X-Real-IP \$remote_addr;
                      proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto \$scheme;
                  }
              }
              NGINX_CONF

              # Enable the Nginx configuration and restart Nginx
              sudo ln -s /etc/nginx/sites-available/influxdb /etc/nginx/sites-enabled/
              sudo systemctl restart nginx
              EOF

  tags = {
    Name = "InfluxDB-Instance"
  }
}
'''
Explanation of Updates

	1.	EBS Volume (aws_ebs_volume): Creates an EBS volume specifically for InfluxDB data, with encryption enabled.
	2.	Volume Attachment (aws_volume_attachment): Attaches the EBS volume to the EC2 instance at /dev/xvdf.
	3.	User Data Script:
	•	Formats the EBS volume as ext4 and mounts it to /mnt/influxdb/data.
	•	Configures the InfluxDB data directory to use /mnt/influxdb/data, where the EBS volume is mounted.
	•	Configures Nginx as a reverse proxy for InfluxDB.

Summary of Resources

	1.	Security Group (influxdb_sg): Allows HTTP, HTTPS, and SSH access.
	2.	EC2 Instance (influxdb_instance): Runs InfluxDB and Nginx on Linux with EBS for data storage.
	3.	EBS Volume (influxdb_data): Provides persistent storage for InfluxDB data.
	4.	Elastic IP (influxdb_eip): Assigns a static IP to the EC2 instance.
	5.	Route 53 DNS Record (influxdb_dns): Points a domain name to the instance’s Elastic IP.

This setup provides persistent storage for InfluxDB data on EBS, accessible via Nginx and Route 53. Let me know if you need any further customization!