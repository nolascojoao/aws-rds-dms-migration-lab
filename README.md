# AWS RDS Migration Lab
---

## Overview

This lab demonstrates the process of migrating a MariaDB database running on an EC2 instance (simulating an on-premise environment) to an RDS MySQL instance using AWS DMS and monitoring the migration with CloudWatch.

---
⚠️ Attention:

- All the tasks will be completed via the command line using AWS CLI. Ensure you have the necessary permissions. [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Charges may apply for completing this lab. [AWS Pricing](https://aws.amazon.com/pricing/)
---

## Step 1: Configure the EC2 Instance Simulating On-Premise
#### 1.1. Create a Security Group
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

#### 1.2. Add Rules to Security Group
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

#### 1.3. Launch EC2 Instance
Simulate an on-premise environment running MariaDB.
  - Replace `<ami-id>` for your AMI ID (e.g., ami-0ebfd941bbafe70c6). [Find an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html) 
  - Replace `<security-group-id>` for your security group ID.one).
  - Replace `<subnet-id>` for your subnet ID.
```bash
aws ec2 run-instances \
  --image-id <ami-id> \
  --count 1 \
  --instance-type t2.micro \
  --key-name YourKeyPairName \
  --security-group-ids <security-group-id> \
  --subnet-id <subnet-id>
```

#### 1.4. Connect to EC2 via SSH
SSH into the EC2 instance to configure MariaDB.
  - Replace `<Public-IP-of-instance>` for the public IP of your EC2 instance.
```bash
ssh -i YourKeyPairName.pem ec2-user@<Public-IP-of-instance>
```

#### 1.5 Install MariaDB
Install and configure MariaDB on the EC2 instance.
```bash
sudo yum update -y
sudo yum install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

#### 1.6. Secure MariaDB Installation
Secure the MariaDB instance by setting a root password, removing anonymous users, etc.
```bash
sudo mysql_secure_installation
```

### 1.7. Create a Sample Database
Create a test database and user for the migration.
```bash
mysql -u root -p
CREATE DATABASE exampledb;
CREATE USER 'user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON exampledb.* TO 'user'@'%';
FLUSH PRIVILEGES;
```

---

## Step 2: Create RDS MySQL Instance
#### 2.1. Launch RDS MySQL Instance
Create an RDS MySQL instance, which will serve as the destination for the migration.
  - Replace `my-rds-instance` for your RDS instance identifier.
  - Replace `admin` for your master username.
  - Replace `adminpassword` for your master password.
```bash
aws rds create-db-instance \
  --db-instance-identifier my-rds-instance \
  --db-instance-class db.t2.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password adminpassword \
  --allocated-storage 20
```

#### 2.2. Note the RDS Endpoint
- After the RDS instance is created, note down the RDS Endpoint for use in the migration task.

---

## Step 3: Set Up AWS DMS
#### 3.1. Create an IAM Role for DMS
Create an IAM role that allows DMS to access your VPC resources.
```bash
aws iam create-role --role-name dms-vpc-role --assume-role-policy-document file://policy.json
aws iam attach-role-policy \
  --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
```

#### 3.2. Create the Source Endpoint (MariaDB)
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

#### 3.3. Create the Target Endpoint (RDS MySQL)
Define the target database (RDS MySQL).
  - Replace `admin` for your RDS username.
  - Replace `adminpassword` for your RDS password.
  - Replace `<RDS-Endpoint>` for your RDS endpoint.
```bash
aws dms create-endpoint \
  --endpoint-identifier rds-endpoint \
  --endpoint-type target \
  --engine-name mysql \
  --username admin \
  --password adminpassword \
  --server-name <RDS-Endpoint> \
  --port 3306 \
  --database-name exampledb
```

#### 3.4. Create a DMS Replication Instance
Create a replication instance for DMS, which will handle the data migration.
  - Replace `<security-group-id>` for your security group ID.
```bash
aws dms create-replication-instance \
  --replication-instance-identifier dms-instance \
  --replication-instance-class dms.t2.micro \
  --allocated-storage 50 \
  --vpc-security-group-ids <security-group-id>
```

#### 3.5. Create and Start the Migration Task
Create and start the migration task that transfers data from MariaDB to RDS MySQL.
```bash
aws dms create-replication-task \
  --replication-task-identifier migration-mariadb-to-rds \
  --source-endpoint-arn arn:aws:dms:mariadb-endpoint \
  --target-endpoint-arn arn:aws:dms:rds-endpoint \
  --migration-type full-load \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://task-settings.json
```

---

## Step 4: Monitor the Migration with CloudWatch
#### 4.1. Check DMS Logs in CloudWatch
Monitor the migration process using CloudWatch logs.
  - Replace `<ReplicationInstanceArn>` for your replication instance ARN.
  - Replace `<log-stream>` for your log stream name.
```bash
aws logs describe-log-groups
aws logs get-log-events --log-group-name /aws/dms/<ReplicationInstanceArn> --log-stream-name <log-stream>
```

---

## Step 5: Verify the Migrated Database
#### 5.1. Connect to RDS and Verify Data
Connect to the RDS MySQL instance and verify that the data was successfully migrated.
  - Replace `<RDS-Endpoint>` for your RDS endpoint.
```bash
mysql -h <RDS-Endpoint> -u admin -p
SHOW DATABASES;
USE exampledb;
SHOW TABLES;
```
