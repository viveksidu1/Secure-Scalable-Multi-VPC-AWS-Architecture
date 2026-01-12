📘 Master Implementation Guide: Secure AWS Hybrid Cloud Architecture
Project: Secure 3-Tier Architecture with Transit Gateway & Automated Scaling
Level: Beginner to Advanced
Author: [Vivek Sidu]
Date: January 9, 2026
________________________________________
Introduction & Architecture Overview
This document serves as a comprehensive step-by-step guide to deploying a production-grade AWS infrastructure. The project bridges a secure Management Network (for admins) with a Production Network (for applications) using advanced security protocols.
Key Technologies Used:
•	Networking: VPC, Transit Gateway, NAT Gateway, Subnets.
•	Compute: EC2, Auto Scaling Groups (ASG), Launch Templates.
•	Security: IAM Roles, Security Groups, S3 Object Lock, SSH Agent Forwarding.
•	Database & Storage: RDS (PostgreSQL), EFS (Shared File System).
•	Load Balancing: Application Load Balancer (ALB).

## 🏗️ Architecture Diagram
Click the diagram below to watch the DEMO VIDEO.....

[![AWS Architecture](images/diagram.png)](https://youtu.be/Q4YCyA1KFXE)

________________________________________
Phase 1: Compliance & Audit Storage (S3 Setup) 🗄️
Objective: Create a secure, immutable storage location for network logs.
1.	Navigate to S3 Console > Click Create bucket.
2.	Bucket Name: secure-pay-audit-logs-portfolio (Names must be unique).
3.	Region: Select your working region (e.g., ap-south-1).
4.	Object Ownership: Keep "ACLs disabled".
5.	Block Public Access: Ensure "Block all public access" is Checkmarked.
6.	Bucket Versioning: Select Enable (Required for Object Lock).
7.	Object Lock: Select Enable.
o	Note: This prevents logs from being deleted by hackers or accidental clicks.
8.	Click Create Bucket.
9.	Set Retention: Open the bucket > Properties > Object Lock > Edit.
o	Default retention: Enable.
o	Mode: Compliance.
o	Period: no. of Days as you like.
o	Click Save changes.
________________________________________
Phase 2: Identity & Access Management (IAM Roles) 🆔
Objective: Give our servers permission to talk to AWS services without using secret keys.
1.	Navigate to IAM Console > Roles > Create role.
2.	Trusted Entity Type: Select AWS Service.
3.	Service or Use Case: Select EC2.
4.	Add Permissions: Search for and select the following policies:
o	AmazonSSMManagedInstanceCore (Allows patching & session manager access).
o	AmazonS3ReadOnlyAccess (Allows accessing S3 buckets).
o	AmazonEC2ReadOnlyAccess
o	AmazonElasticFileSystemClientReadWriteAccess
5.	Click Next.
6.	Role Name: SSM-Role-Admin.
7.	Click Create role.
________________________________________
Phase 3: Network Infrastructure (VPC & Subnets) 🌐
Objective: Build two separate networks—one for Management and one for Production.
3.1 Create Management VPC (For Bastion)
1.	Go to VPC Console > Your VPCs > Create VPC.
2.	Name: Mgmt-VPC.
3.	IPv4 CIDR: 10.0.0.0/16.
4.	Create Subnet:
o	Name: Mgmt-Public-Subnet.
o	AZ: ap-south-1a.
o	CIDR: 10.0.1.0/24.
5.	Internet Gateway (IGW):
o	Create IGW named Mgmt-IGW.
o	Select IGW > Actions > Attach to VPC > Select Mgmt-VPC.
6.	Route Table:
o	Find the Main Route Table for Mgmt-VPC. Name it Mgmt-Public-RT.
o	Edit Routes: Add 0.0.0.0/0 -> Target: Mgmt-IGW.
o	Subnet Associations: Associate Mgmt-Public-Subnet.



3.2 Create Production VPC (For App & DB)
1.	Create VPC: Name Prod-VPC with CIDR 10.1.0.0/16.
2.	Create Subnets (Total 4):
o	Public (For Load Balancer & NAT):
	Prod-Pub-Sub-1 (10.1.1.0/24)
	Prod-Pub-Sub-2 (10.1.2.0/24)
o	Private (For Database):
	Prod-Pvt-Sub-1 (10.1.10.0/24)
	Prod-Pvt-Sub-2 (10.1.20.0/24)
3.	Internet Gateway: Create Prod-IGW and attach to Prod-VPC.
4.	NAT Gateway (For Private Internet):
o	Go to NAT Gateways > Create Prod-NAT-1 & Prod-NAT-2
o	Subnet: Prod-Pub-Sub-1 & Prod-Pub-Sub-2.
o	Connectivity: Public.
o	Elastic IP: Click "Allocate Elastic IP".
o	Click Create.
5.	Route Tables (Prod):
o	Public RT: Create Mgmt-Public-RT. Add route 0.0.0.0/0 -> Mgmt-IGW & Prod-Public-RT. Add route 0.0.0.0/0 -> Prod-IGW
o	 Associate both Public Subnets.
o	Private RT: Create Prod-Private-RT-1. Add route 0.0.0.0/0 -> NAT Gateway & Prod-Private-RT-2. Add route 0.0.0.0/0 -> NAT Gateway. Associate all 2 Private Subnets (App & DB).
________________________________________
Phase 4: Connectivity (Transit Gateway) 🌉
Objective: Connect the two isolated VPCs securely.
1.	Create Transit Gateway: Go to VPC Dashboard > Transit Gateways > Create.
o	Name: Secure-Hub-TGW.
o	Auto-accept shared attachments: Enable.
2.	Create Attachments:
o	Attachment 1: (Management Side)
	TGW ID: Secure-Hub-TGW.
	VPC ID: Mgmt-VPC.
	Subnet: Mgmt-Public-Subnet.
o	Attachment 2: (Production Side)
	TGW ID: Secure-Hub-TGW.
	VPC ID: Prod-VPC.
	Subnets: Prod-Pvt-Sub-1 & Prod-Pvt-Sub-2.
3.	Update Route Tables (Crucial Step):
o	Go to Mgmt-Public-RT > Routes > Add Route:
	Destination: 10.1.0.0/16 (Prod CIDR).
	Target: Transit Gateway.
o	Go to Prod-Private-RT-1 > Routes > Add Route:
	Destination: 10.0.0.0/16 (Mgmt CIDR).
	Target: Transit Gateway.
o	Go to Prod-Private-RT-2 > Routes > Add Route:
	Destination: 10.0.0.0/16 (Mgmt CIDR).
	Target: Transit Gateway.

VPC Flow Logs Setup
1.	Select Prod-VPC.
2.	Actions > Create Flow Log.
3.	Name: Prod-Network-Audit
4.	Filter: All.
5.	Destination: Send to S3 bucket.
6.	S3 Bucket ARN: Paste ARN of project-vpc-flowlogs-secure (Created in Phase 1).
________________________________________
Phase 5: Security Groups (The Firewall) 🛡️
Objective: Define strict inbound/outbound rules.
Create the following Security Groups (SGs):
SG Name	VPC	Inbound Rules	Description
Mgmt-Bastion-SG	Mgmt-VPC	SSH (22): My IP (Your Public IP)	Only allows you to enter.
Prod-ALB-SG	Prod-VPC	HTTP (80): 0.0.0.0/0 (Anywhere)


HTTPS (443): 0.0.0.0/0	For Public Load Balancer.
Prod-App-SG	Prod-VPC	HTTP (80): From Prod-ALB-SG


SSH (22): From 10.0.0.0/16 (Mgmt CIDR)
HTTP (80): From 10.0.0.0/16
ICMP: From 10.0.0.0/16	Allows ALB traffic & Bastion access.
Prod-DB-SG	Prod-VPC	PostgreSQL (5432): From Prod-App-SG


PostgreSQL (5432): From 10.0.0.0/16	Allows App & Admin access.
Prod-EFS-SG	Prod-VPC	NFS (2049): From Prod-App-SG	Allows Storage Access.
________________________________________
Phase 6: Database & Storage Layer 💾
6.1 RDS PostgreSQL Setup
1.	Subnet Group: RDS > Subnet groups > Create.
o	VPC: Prod-VPC.
o	Name: Prod-DB-Subnet-group
o	Subnets: Prod-Pvt-Sub-1 & Prod-Pvt-Sub-2
2.	Create Database:
o	Standard Create > PostgreSQL.
o	Template: Free Tier or Production.
o	Identifier: prod- paymentapp-db.
o	Master Username: postgres. Password: 12345678.
o	Connectivity:
	VPC: Prod-VPC.
	Public Access: No.
o	Security Group: Prod-DB-Subnet-group
o	Additional Configuration:
	Intial Database Name: paymentapp
3.	Click Create. Note down the Endpoint URL once active.
6.2 EFS Shared Storage
1.	Go to EFS > Create file system.
2.	Name: Prod-Shared-Storage.
3.	VPC: Prod-VPC.
4.	Customize:
o	Mount Targets: Ensure targets are in Prod-Pvt-Sub-1 and Prod-Pvt-Sub-2.
o	Security Group: Detach default, Attach Prod-EFS-SG.
5.	Click Create. Note down the File System ID (fs-xxxx).
________________________________________





Phase 7: Compute & Automation ⚙️
7.1 Create Launch Template
1.	Go to EC2 > Launch Templates > Create.
2.	Name: Prod-App-Template.
3.	AMI: Amazon Linux 2023.
4.	Instance Type: t2.micro.
5.	Key Pair: Projectkey.
6.	Network Settings: Select Security Group Prod-App-SG.
7.	Advanced Details:
o	IAM Instance Profile: SSM-Role-Admin.
o	User Data: (Copy-Paste the script below).
Bash
#!/bin/bash
sudo yum update -y
sudo yum install -y amazon-efs-utils
sudo dnf install -y python3-pip
sudo pip3 install botocore
sudo yum install -y php
sudo dnf install postgresql15-server -y

#Install Postgre SQL
sudo postgresql-setup --initdb
sudo systemctl start postgresql
sudo systemctl enable postgresql

#Install Apache web server (httpd)
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd

#Create a simple HTML file to verify the web server is running, including dynamic hostname
sudo sh -c 'echo "<?php echo \"<h1>Welcome to (PRIVATE EC2)which is inside(PRIVATE SUBNET)of PRODUCTION VPC: \" . gethostname() . \"</h1>\"; ?>" > /var/www/html/index.php'

7.2 Create Target Group & Load Balancer
1.	Target Group: EC2 > Target Groups > Create.
o	Target type: Instances.
o	Protocol: HTTP (80).
o	VPC: Prod-VPC.
o	Health Check: /
o	Register Targets: Select all Instances
2.	Load Balancer (ALB): EC2 > Load Balancers > Create ALB.
o	Name: Prod-Public-ALB
o	Scheme: Internet-facing.
o	Network: Prod-VPC -> Prod-Pub-Sub-1 & Prod-Pub-Sub-1.
o	Security Group: Prod-ALB-SG.
o	Listeners: HTTP (80) -> Forward to Target Group created above (Prod-App-TG).
7.3 Create Auto Scaling Group (ASG)
1.	ASG Name: Prod-ASG.
2.	Template: Select Prod-App-Template.
3.	Network: Prod-VPC -> Select both Private Subnets (Prod-Pvt-Sub-1 & Prod-Pvt-Sub-2).
4.	Load Balancing: Attach to existing Load Balancer -> Select your Target Group.
5.	Scaling: Min: 2, Desired: 2, Max: 4.
________________________________________
Phase 8: Management Access (Bastion Host) 🔑
1.	Launch EC2 Instance in Mgmt-VPC.
2.	Name: Mgmt-Bastion-Server
3.	Subnet: Mgmt-Public-Subnet.
4.	Auto-assign Public IP: Enable.
5.	Security Group: Mgmt-Bastion-SG.
6.	Advance: IAM Instance Profile -> SSM-Role-Admin
7.	Key Pair: Create new key ProjectKey.pem and download it.
________________________________________
Phase 9: Secure Operations (How to Connect) 🚀
9.1 Enable SSH Agent Forwarding (Local PC)
This allows you to jump from Bastion to Private servers without copying the key file.
For Windows (PowerShell Administrator):
PowerShell
Set-Service ssh-agent -StartupType Manual
Start-Service ssh-agent
Get-Service ssh-agent

#goto the path where .pem file is located
ssh-add ProjectKey.pem
ssh -A -I ProjectKey.pem ec2-user@<Bastions-Public-IP>
For Mac/Linux:
Bash
chmod 400 projectkey.pem
ssh-add -K projectkey.pem
9.2 Connect to Bastion
Bash
ssh -A -i projectkey.pem ec2-user@<BASTION-PUBLIC-IP>
9.3 Jump to Private App Server
Once inside Bastion, type:
Bash
ssh ec2-user@<PRIVATE-IP-OF-ProductionVPC-SERVER>
________________________________________

Phase 10: Database Security Implementation (RBAC) 🔐
Enforce Principle of Least Privilege.
1.	Connect to RDS (From Bastion):
Bash
psql -h <RDS-ENDPOINT> -U postgres -d paymentapp
2.	Run SQL Commands:
SQL
-- Create User for Application (Read/Write)
CREATE USER admin WITH PASSWORD 'admin123';
GRANT ALL PRIVILEGES ON DATABASE paymentapp TO admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin;

-- Create User for Humans (Read-Only)
CREATE USER monitor WITH PASSWORD 'monitor123';
GRANT CONNECT ON DATABASE paymentapp TO monitor;
GRANT USAGE ON SCHEMA public TO monitor;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO monitor;
3.	Validation: Try deleting data using monitor. It should fail with "Permission Denied".

### 🎥 Project Implementation Demo
Click the image below to watch the COMPLETE WALKTHROUGH.....

[![Watch the video](images/diagram.png)](https://youtu.be/k1VrlAx0SBk)

________________________________________
Final Verification Checklist ✅
1.	ALB DNS Check: Copy ALB DNS name in browser -> Should show "Production Server: ip-xx-xx".
2.	SSH Jump Check: Can you login to Private IP from Bastion without a .pem file?
3.	DB Check: Can you connect to RDS from Bastion using monitor?
4.	Scaling Check: Terminate one App server; ASG should launch a new one automatically.
Congratulations! You have successfully deployed a secure, enterprise-grade AWS architecture.
