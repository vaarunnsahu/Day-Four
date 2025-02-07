# Deploying Containerized Flask App on AWS

This guide walks through deploying a containerized Flask application on AWS using ECS (Elastic Container Service) with an application load balancer and RDS database.

---

## **🌐 Network Architecture Setup**

### **1️⃣ VPC and Subnet Configuration**
1. **Create VPC**  
   - CIDR block: `10.0.0.0/16`  
   - Enable DNS hostnames and DNS resolution  

2. **Create Subnets**  
   - **Private Subnets**:  
     - Subnet 1: `10.0.1.0/24` (AZ-1)  
     - Subnet 2: `10.0.2.0/24` (AZ-2)  
   - **Public Subnets**:  
     - Subnet 1: `10.0.3.0/24` (AZ-1)  
     - Subnet 2: `10.0.4.0/24` (AZ-2)  

3. **Route Tables**  
   - Create separate route tables for public and private subnets  
   - Associate subnets with respective route tables  

4. **Internet Gateway**  
   - Create and attach to VPC  
   - Add route in public route table: `0.0.0.0/0` → Internet Gateway  

5. **NAT Gateways**  
   - Create 2 NAT Gateways (one in each public subnet)  
   - Add routes in private route tables: `0.0.0.0/0` → NAT Gateway  

---

## **🔒 Security Groups Configuration**

1. **Application Load Balancer (ALB) Security Group**  
   ```
   Inbound Rules:
   - HTTP (80) from 0.0.0.0/0
   - HTTPS (443) from 0.0.0.0/0
   ```

2. **Flask Application Security Group**  
   ```
   Inbound Rules:
   - Port 8000 from ALB Security Group
   ```

3. **RDS Security Group**  
   ```
   Inbound Rules:
   - PostgreSQL (5432) from Flask App Security Group
   ```

---

## **📦 RDS Setup**

1. **Create RDS Subnets**  
   - Subnet 1: `10.0.5.0/24` (AZ-1)  
   - Subnet 2: `10.0.6.0/24` (AZ-2)  

2. **PostgreSQL RDS Instance**  
   - Engine: PostgreSQL  
   - Instance Configuration:  
     - Multi-AZ deployment  
     - Private subnet placement  
     - Security group attachment  
   - **Note down:**  
     - Endpoint  
     - Username  
     - Password  
     - Database name  
     - Port (5432)  

---

## **📌 Container Registry Setup**

1. **Create ECR Repository**  
   ```bash
   aws ecr create-repository \
       --repository-name my-flask-app \
       --image-scanning-configuration scanOnPush=true
   ```

2. **Authentication**  
   ```bash
   aws ecr get-login-password --region region | \
   docker login --username AWS --password-stdin \
   aws_account_id.dkr.ecr.region.amazonaws.com
   ```

---

## **🐳 Application Containerization**

1. **Application Structure:**
   ```
   .
   ├── app/
   │   ├── __init__.py
   │   ├── routes.py
   │   └── models.py
   ├── Dockerfile
   ├── requirements.txt
   └── wsgi.py
   ```

2. **GitHub Actions Workflow**
   ```yaml
   name: Build-Push

   on:
   push:
       branches:
       - main
       paths:
       - 'src/**'
       - '.github/workflows/build-push.yml'
   workflow_dispatch:

   jobs:
   build:
       runs-on: ubuntu-latest

       steps:
       - name: Checkout
         uses: actions/checkout@v2

       - name: Configure AWS Credentials
         uses: aws-actions/configure-aws-credentials@v1
         with:
           aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
           aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
           aws-region: ap-south-1

       - name: Login to Amazon ECR
         run: aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.ap-south-1.amazonaws.com
       - name: Build, Tag, and Push Docker Image
         run: |
           cd src
           docker build -t aws_account_id.dkr.ecr.ap-south-1.amazonaws.com/bootcampflask:latest .
           docker push aws_account_id.dkr.ecr.ap-south-1.amazonaws.com/bootcampflask:latest
   ```

---

## **🚀 ECS Deployment**

1. **Create ECS Cluster**  
   - Cluster type: Fargate  
   - VPC: Use created VPC  

2. **Task Definition**  
   - Launch type: Fargate  
   - Task role: ECS execution role with ECR access  
   - Container configuration:  
     - Image: ECR image URI  
     - Port mappings: 8000  
     - Environment variables:  
       - DATABASE_URL from AWS Secrets Manager  
     - Log configuration: CloudWatch logs  

3. **ECS Service**  
   - Launch type: Fargate  
   - Network configuration:  
     - VPC: Created VPC  
     - Subnets: Private subnets  
     - Security groups: Flask app security group  
   - Load balancer:  
     - Target group: Port 8000  
     - Health check path: `/health`  

---

## **🔗 DNS and SSL Configuration**

1. **Domain Setup**  
   ```bash
   aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)
   ```

2. **SSL Certificate**  
   ```bash
   aws acm request-certificate --domain-name example.com --validation-method DNS
   ```

3. **ALB Listener Configuration**  
   - HTTP (80): Redirect to HTTPS  
   - HTTPS (443): Forward to target group  

4. **Route53 Record**  
   ```bash
   aws route53 change-resource-record-sets --hosted-zone-id HOSTED_ZONE_ID --change-batch '{ "Changes": [{ "Action": "CREATE", "ResourceRecordSet": { "Name": "example.com", "Type": "A", "AliasTarget": { "HostedZoneId": "ALB_ZONE_ID", "DNSName": "ALB_DNS_NAME", "EvaluateTargetHealth": true }}}]}'
   ```

---

## **📊 Verification**
- Access application: `https://example.com`
- Monitor:  
  - ECS service status  
  - CloudWatch logs  
  - ALB target group health  
  - RDS metrics  

---

### 🎯 **Happy Deploying! 🚀**

