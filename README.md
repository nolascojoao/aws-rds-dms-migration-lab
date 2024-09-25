# AWS RDS Migration Lab
---

## Overview

- Migrate a MariaDB database from an EC2 instance to a Single-AZ RDS MariaDB with AWS DMS.
- Migration is monitored via CloudWatch.

---
⚠️ Attention:

- All the tasks will be completed via the command line using AWS CLI. Ensure you have the necessary permissions. [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Charges may apply for completing this lab. [AWS Pricing](https://aws.amazon.com/pricing/)
---

## Step 1: Create a VPC and Subnets
#### 1.1. Create a VPC
Create a VPC that will be used to host the EC2 instance and RDS.
  - Replace `<cidr-block>` with the CIDR block you wish to use (e.g., 10.0.0.0/16).
```bash
aws ec2 create-vpc --cidr-block <cidr-block>
```

#### 1.2. Create Subnets
Create subnets for the EC2 instance and RDS. A public subnet for EC2 and a private subnet for RDS.
  - Replace `<vpc-id>` with the ID of the VPC created in the previous step.
  - Replace `<cidr-block>` with the desired CIDR block for each subnet (e.g., 10.0.1.0/24 for EC2 and 10.0.2.0/24 for RDS).

#### Public Subnet for EC2:
```bash
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block <cidr-block>
```

#### Private Subnet for RDS:
```bash
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block <cidr-block>
```

#### 1.3. Set up an Internet Gateway
Attach an Internet Gateway to the VPC to allow the EC2 instance to access the internet.
  - Replace `<gateway-id>` with your Internet Gateway ID.
```bash
aws ec2 create-internet-gateway

aws ec2 attach-internet-gateway --vpc-id <vpc-id> --internet-gateway-id <gateway-id>
```

#### 1.4. Create and Associate a Route Table
Create a route table for the public subnet and associate it with the Internet Gateway.
  - Replace `<public-subnet-id>` with your public subnet ID.
  - Replace `<route-table-id>` with your route table ID.
```bash
aws ec2 create-route-table --vpc-id <vpc-id>

aws ec2 create-route \
  --route-table-id <route-table-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <gateway-id>

aws ec2 associate-route-table --subnet-id <public-subnet-id> --route-table-id <route-table-id>
```

---

## Step 2: Configure the EC2 Instance Simulating On-Premise
#### 2.1. Create a Security Group
Create a security group that allows traffic for MariaDB and SSH access.
  - Replace `<sg-name>` for your security group name.
  - Replace `SG-EC2` for your security group description.
  - Replace `<vpc-id>` for your VPC ID.
```bash
aws ec2 create-security-group \
  --group-name <sg-name> \
  --description "SG-EC2" \
  --vpc-id <vpc-id>
```

#### 2.2. Add Rules to Security Group
Allow inbound SSH (port 22) and MariaDB (port 3306) traffic.
  - Replace `<security-group-id>` for your security group ID.
```bash
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol tcp \
  --port 3306 \
  --cidr 0.0.0.0/0
```

#### 2.3. Launch EC2 Instance
Simulate an on-premise environment running MariaDB.
  - Replace `<ami-id>` for your AMI ID (e.g., ami-0ebfd941bbafe70c6). [Find an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html)
  - Replace `<instance-type>` with your desired instance type (e.g., t2.micro).
  - Replace `<security-group-id>` for your security group ID.one).
  - Replace `<subnet-id>` for your subnet ID.
```bash
aws ec2 run-instances \
  --image-id <ami-id> \
  --instance-type t2.micro \
  --key-name YourKeyPairName \
  --security-group-ids <security-group-id> \
  --associate-public-ip-address \
  --subnet-id <subnet-id> 
```

#### 2.4. Connect to EC2 via SSH
SSH into the EC2 instance to configure MariaDB.
  - Replace `<Public-IP-of-instance>` for the public IP of your EC2 instance.
```bash
ssh -i YourKeyPairName.pem ec2-user@<Public-IP-of-instance>
```

#### 2.5. Install MariaDB
Install and configure MariaDB on the EC2 instance.
```bash
sudo yum update -y
sudo yum install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

#### 2.6. Secure MariaDB Installation
Secure the MariaDB instance by setting a root password, removing anonymous users, etc.
```bash
sudo mysql_secure_installation
```

### 2.7. Create a Sample Database
Create a test database and user for the migration.
```bash
mysql -u root -p
CREATE DATABASE exampledb;
CREATE USER 'user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON exampledb.* TO 'user'@'%';
FLUSH PRIVILEGES;
```

---

## Step 3: Create RDS MariaDB Instance
#### 3.1. Create a Security Group for RDS
Create a security group that allows MariaDB traffic to the RDS instance.
  - Replace `<sg-name>` with your security group name.
  - Replace `SG-RDS` with your security group description.
  - Replace `<vpc-id>` with your VPC ID.
```bash
aws ec2 create-security-group \
  --group-name <sg-name> \
  --description "SG-RDS" \
  --vpc-id <vpc-id>
```

#### 3.2. Launch RDS MariaDB Instance
Create an RDS MariaDB instance which will serve as the destination for the migration.
  - Replace `my-rds-instance` for your RDS instance identifier.
  - Replace `admin` for your master username.
  - Replace `adminpassword` for your master password.
```bash
aws rds create-db-instance \
  --db-instance-identifier my-rds-instance \
  --db-instance-class db.t2.micro \
  --engine mariadb \
  --master-username admin \
  --master-user-password adminpassword \
  --allocated-storage 20 \
  --vpc-security-group-ids <security-group-id>
```

#### 3.3. Note the RDS Endpoint
- After the RDS instance is created note down the RDS Endpoint for use in the migration task.

---

## Step 4: Set Up AWS DMS
#### 4.1. Create an IAM Role for DMS
Create an IAM role that allows DMS to access your VPC resources.
```bash
aws iam create-role --role-name dms-role --assume-role-policy-document file://dms-trust-policy.json
```
#### Content of `dms-trust-policy.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "dms.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### 4.2. Attach Policies to the Role
Attach the necessary AWS DMS policies to the role.
```bash
aws iam attach-role-policy --role-name dms-role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
aws iam attach-role-policy --role-name dms-role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole
aws iam attach-role-policy --role-name dms-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSMigrationHubDMSAccess
```

#### 4.3. Create the Source Endpoint (MariaDB)
Define the source database (MariaDB running on EC2).
  - Replace `user` for your MariaDB username.
  - Replace `password` for your MariaDB password.
  - Replace `<EC2-IP>` for the IP of your EC2 instance.
```bash
aws dms create-endpoint \
  --endpoint-identifier mariadb-endpoint \
  --endpoint-type source \
  --engine-name mariadb \
  --username user \
  --password password \
  --server-name <EC2-IP> \
  --port 3306 \
  --database-name exampledb
```

#### 4.4. Create the Target Endpoint (RDS MariaDB)
Define the target database (RDS MariaDB).
  - Replace `admin` for your RDS username.
  - Replace `adminpassword` for your RDS password.
  - Replace `<RDS-Endpoint>` for your RDS endpoint.
```bash
aws dms create-endpoint \
  --endpoint-identifier rds-endpoint \
  --endpoint-type target \
  --engine-name mariadb \
  --username admin \
  --password adminpassword \
  --server-name <RDS-Endpoint> \
  --port 3306 \
  --database-name exampledb
```

#### 4.5. Create a DMS Replication Instance
Create a replication instance for DMS, which will handle the data migration.
  - Replace `<security-group-id>` for your security group ID.
```bash
aws dms create-replication-instance \
  --replication-instance-identifier dms-instance \
  --replication-instance-class dms.t2.micro \
  --allocated-storage 50 \
  --vpc-security-group-ids <security-group-id>
```

#### 4.6. Create and Start the Migration Task
Create and start the migration task that transfers data from MariaDB to RDS MariaDB.
```bash
aws dms create-replication-task \
  --replication-task-identifier migration-mariadb-to-rds \
  --source-endpoint-arn <source-endpoint-arn> \
  --target-endpoint-arn <target-endpoint-arn> \
  --migration-type full-load \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://task-settings.json
```

---

## Step 5: Monitor the Migration with CloudWatch
#### 5.1. Check DMS Logs in CloudWatch
Monitor the migration process using CloudWatch logs.
  - Replace `<ReplicationInstanceArn>` for your replication instance ARN.
  - Replace `<log-stream>` for your log stream name.
```bash
aws logs describe-log-groups
aws logs get-log-events \
  --log-group-name <log-group-name> \
  --log-stream-name <log-stream-name>
```

---

## Step 6: Verify the Migrated Database
#### 6.1. Connect to RDS and Verify Data
Connect to the RDS MariaDB instance and verify that the data was successfully migrated.
  - Replace `<RDS-Endpoint>` for your RDS endpoint.
```bash
mysql -h <RDS-Endpoint> -u admin -p
SHOW DATABASES;
USE exampledb;
SHOW TABLES;
```

---

## Step 7: Clean Up Resources
#### 7.1. Remove the resources
Once the lab is completed remove the resources to avoid unnecessary charges.
  - Replace `<instance-id>`, `<db-instance-identifier>`, `<vpc-id>` and `<replication-instance-arn>` with your values.
```bash
aws ec2 terminate-instances --instance-ids <instance-id>

aws rds delete-db-instance \
  --db-instance-identifier <db-instance-identifier> --skip-final-snapshot

aws ec2 delete-vpc --vpc-id <vpc-id>

aws dms delete-replication-instance --replication-instance-arn <replication-instance-arn>
```
