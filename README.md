# AWS RDS Migration Lab

## Overview

This lab demonstrates the process of migrating a MariaDB database running on an EC2 instance (simulating an on-premise environment) to an RDS MySQL instance using AWS DMS and monitoring the migration with CloudWatch. The entire process will be executed via AWS CLI.

---
⚠️ Attention:

- All the tasks will be completed via the command line using AWS CLI. Ensure you have the necessary permissions. [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Charges may apply for completing this lab. [AWS Pricing](https://aws.amazon.com/pricing/)
---

## Step 1: Configure the EC2 Instance Simulating On-Premise
#### 1.1. Create a Security Group
```bash
aws ec2 create-security-group \
  --group-name <sg-name> \
  --description "SG-EC2" \
  --vpc-id <vpc-id>
```
- **Purpose:** Create a security group that allows traffic for MariaDB and SSH access.
- **Explanation:**
  - `--group-name`: The name of the security group, used to identify it for managing security rules.
  - `--description`: A brief description of the security group’s purpose, which helps clarify its role (e.g., "Security group for EC2 instance running MariaDB").
  - `--vpc-id`: The VPC ID where the security group will be created.

#### 1.2. Add Rules to Security Group
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
- **Purpose:** Allow inbound SSH (port 22) and MariaDB (port 3306) traffic.
- **Explanation:**
  - `--port 22`: Enables SSH access to the instance.
  - `--port 3306`: Enables MariaDB connections.
  - `--cidr 0.0.0.0/0`: Opens these ports to all IP addresses (for testing purposes, restrict as needed).

#### 1.3. Launch EC2 Instance
```bash
aws ec2 run-instances \
  --image-id <ami-id> \
  --count 1 \
  --instance-type t2.micro \
  --key-name YourKeyPairName \
  --security-group-ids <security-group-id> \
  --subnet-id <subnet-id>
```
- **Purpose:** Simulate an on-premise environment running MariaDB.
- **Explanation:**
  - `--security-group-ids`: The security group allowing SSH and MariaDB traffic.
  - `--subnet-id`: The subnet ID where the EC2 instance will reside (use an existing public subnet or create a new one).

#### 1.4. Connect to EC2 via SSH
```bash
ssh -i YourKeyPairName.pem ec2-user@<Public-IP-of-instance>
```
- **Purpose:** SSH into the EC2 instance to configure MariaDB.

#### 1.5 Install MariaDB
```bash
sudo yum update -y
sudo yum install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```
- **Purpose:** Install and configure MariaDB on the EC2 instance.

#### 1.6. Secure MariaDB Installation
```bash
sudo mysql_secure_installation
```
- **Purpose:** Secure the MariaDB instance by setting a root password, removing anonymous users, etc.

### 1.7. Create a Sample Database
```bash
mysql -u root -p
CREATE DATABASE exampledb;
CREATE USER 'user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON exampledb.* TO 'user'@'%';
FLUSH PRIVILEGES;
```
- **Purpose:** Create a test database and user for the migration.

---

## Step 2: Create RDS MySQL Instance
#### 2.1. Launch RDS MySQL Instance
```bash
aws rds create-db-instance \
  --db-instance-identifier my-rds-instance \
  --db-instance-class db.t2.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password adminpassword \
  --allocated-storage 20
```
- **Purpose:** Create an RDS MySQL instance, which will serve as the destination for the migration.

#### 2.2. Note the RDS Endpoint
- After the RDS instance is created, note down the RDS Endpoint for use in the migration task.

---

## Step 3: Set Up AWS DMS
#### 3.1. Create an IAM Role for DMS
```bash
aws iam create-role --role-name dms-vpc-role --assume-role-policy-document file://policy.json
aws iam attach-role-policy \
  --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
```
- **Purpose:** Create an IAM role that allows DMS to access your VPC resources.

#### 3.2. Create the Source Endpoint (MariaDB)
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
- **Purpose:** Define the source database (MariaDB running on EC2).

#### 3.3. Create the Target Endpoint (RDS MySQL)
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
- **Purpose:** Define the target database (RDS MySQL).

#### 3.4. Create a DMS Replication Instance
```bash
aws dms create-replication-instance \
  --replication-instance-identifier dms-instance \
  --replication-instance-class dms.t2.micro \
  --allocated-storage 50 \
  --vpc-security-group-ids <security-group-id>
```
- **Purpose:** Create a replication instance for DMS, which will handle the data migration.

#### 3.5. Create and Start the Migration Task
```bash
aws dms create-replication-task \
  --replication-task-identifier migration-mariadb-to-rds \
  --source-endpoint-arn arn:aws:dms:mariadb-endpoint \
  --target-endpoint-arn arn:aws:dms:rds-endpoint \
  --migration-type full-load \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://task-settings.json
```
- **Purpose:** Create and start the migration task that transfers data from MariaDB to RDS MySQL.

---

## Step 4: Monitor the Migration with CloudWatch
#### 4.1. Check DMS Logs in CloudWatch
```bash
aws logs describe-log-groups
aws logs get-log-events --log-group-name /aws/dms/<ReplicationInstanceArn> --log-stream-name <log-stream>
```
- **Purpose:** Monitor the migration process using CloudWatch logs.

---

## Step 5: Verify the Migrated Database
#### 5.1. Connect to RDS and Verify Data
```bash
mysql -h <RDS-Endpoint> -u admin -p
SHOW DATABASES;
USE exampledb;
SHOW TABLES;
```
- **Purpose:** Connect to the RDS MySQL instance and verify that the data was successfully migrated.
