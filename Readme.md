# AWS Microservice Architecture Setup

This document provides an overview of the AWS infrastructure set up by the provided CloudFormation template. It outlines the purpose of each resource and provides guidance on post-deployment steps to fully operationalize the microservices architecture.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Networking Components](#networking-components)
    - [VPC (Virtual Private Cloud)](#vpc-virtual-private-cloud)
    - [Internet Gateway](#internet-gateway)
    - [Subnets](#subnets)
    - [Route Tables and Routes](#route-tables-and-routes)
    - [NAT Gateway and Elastic IP](#nat-gateway-and-elastic-ip)
    - [VPC Endpoints](#vpc-endpoints)
- [Security Groups](#security-groups)
- [Compute Resources](#compute-resources)
    - [Amazon ECS Cluster](#amazon-ecs-cluster)
    - [EC2 Instances](#ec2-instances)
- [Load Balancers](#load-balancers)
    - [Application Load Balancer (ALB)](#application-load-balancer-alb)
    - [Classic Load Balancer (CLB)](#classic-load-balancer-clb)
- [Storage](#storage)
    - [Amazon S3 Buckets](#amazon-s3-buckets)
- [Databases](#databases)
    - [Amazon DocumentDB Cluster](#amazon-documentdb-cluster)
    - [Redis](#redis)
- [Message Broker](#message-broker)
    - [RabbitMQ Broker](#rabbitmq-broker)
- [Container Registry](#container-registry)
    - [Amazon ECR Repository](#amazon-ecr-repository)
- [Outputs](#outputs)
- [Post-Deployment Steps](#post-deployment-steps)
    - [1. Build and Push Docker Images](#1-build-and-push-docker-images)
    - [2. Create Task Definitions](#2-create-task-definitions)
    - [3. Deploy Services in ECS Cluster](#3-deploy-services-in-ecs-cluster)
    - [4. Update Load Balancer](#4-update-load-balancer)
    - [5. Update Amplify Environment Variables](#5-update-amplify-environment-variables)
    - [6. Attach Custom Domains](#6-attach-custom-domains)

## Architecture Overview

![AWS ECS Architecture Diagram](AWS%20ECS%20Diagram.drawio.svg)

The CloudFormation template sets up a microservices architecture on AWS, including:

- A Virtual Private Cloud (VPC) with public and private subnets across multiple Availability Zones.
- Security groups for controlling inbound and outbound traffic.
- An ECS cluster for running containerized applications.
- Load balancers for distributing traffic.
- Amazon DocumentDB cluster for database services.
- RabbitMQ broker for messaging.
- Amazon S3 buckets for storage and logging.
- EC2 instances for Redis and an Admin Panel.
- VPC endpoints for private connectivity to AWS services.

The architecture ensures high availability, scalability, and security for microservices applications.

## Networking Components

### VPC (Virtual Private Cloud)

- **Resource:** `VPC`
- **Purpose:** Creates an isolated network environment where you can launch AWS resources in a virtual network that you define.
- **Details:**
    - **CIDR Block:** `10.0.0.0/16`
    - **DNS Support and Hostnames:** Enabled

### Internet Gateway

- **Resource:** `InternetGateway`, `VPCGatewayAttachment`
- **Purpose:** Enables communication between the VPC and the internet.
- **Details:**
    - Attaches the Internet Gateway to the VPC.

### Subnets

- **Public Subnets:**
    - **Resources:** `PublicSubnet1`, `PublicSubnet2`
    - **Purpose:** Hosts public-facing resources like load balancers.
    - **Details:**
        - **CIDR Blocks:** `10.0.1.0/24`, `10.0.2.0/24`
        - **Availability Zones:** Spread across different zones for redundancy.
        - **Auto-assign Public IP:** Enabled
- **Private Subnets:**
    - **Resources:** `PrivateSubnet1`, `PrivateSubnet2`
    - **Purpose:** Hosts private resources like EC2 instances and databases.
    - **Details:**
        - **CIDR Blocks:** `10.0.3.0/24`, `10.0.4.0/24`
        - **Availability Zones:** Spread across different zones for redundancy.
        - **Auto-assign Public IP:** Disabled

### Route Tables and Routes

- **Public Route Table:**
    - **Resource:** `PublicSubnetRouteTable`
    - **Purpose:** Directs internet-bound traffic from public subnets through the Internet Gateway.
    - **Associations:** `PublicSubnet1RouteAssociation`, `PublicSubnet2RouteAssociation`
- **Private Route Table:**
    - **Resource:** `PrivateSubnetRouteTable`
    - **Purpose:** Directs internet-bound traffic from private subnets through the NAT Gateway.
    - **Associations:** `PrivateSubnet1RouteAssociation`, `PrivateSubnet2RouteAssociation`
- **Routes:**
    - **Public Route to Internet Gateway:** `PublicSubnetRouteInternetGateway`
    - **Private Route to NAT Gateway:** `PrivateSubnetRouteNAT`

### NAT Gateway and Elastic IP

- **Resources:** `NatGateway`, `ElasticIP`
- **Purpose:** Allows instances in private subnets to access the internet for updates and patches while preventing inbound internet traffic.
- **Details:**
    - **NAT Gateway** is placed in a public subnet.
    - **Elastic IP** is associated with the NAT Gateway.

### VPC Endpoints

- **Gateway Endpoint for S3:**
    - **Resource:** `VPCGatewayEndpointS3`
    - **Purpose:** Enables private access to Amazon S3 without using an internet gateway or NAT device.
- **Interface Endpoints:**
    - **Resources:** Various `VPCInterfaceEndpoints` for services like ECS, RDS, ECR, CloudWatch Logs.
    - **Purpose:** Enables private connectivity to AWS services over the Amazon network.

## Security Groups

- **ECSSecurityGroup:**
    - **Purpose:** Controls access to ECS tasks.
    - **Inbound Rules:** Allows HTTP (80) and HTTPS (443) from anywhere.
- **ALBSecurityGroup:**
    - **Purpose:** Controls access to the Application Load Balancer.
    - **Inbound Rules:** Allows HTTP (80) and HTTPS (443) from anywhere.
- **RMQSecurityGroup:**
    - **Purpose:** Controls access to the RabbitMQ broker.
    - **Inbound Rules:** Allows TCP port 5671 from anywhere.
- **DocDBSecurityGroup:**
    - **Purpose:** Controls access to the DocumentDB cluster.
    - **Inbound Rules:** Allows TCP port 27017 from anywhere.
- **RedisSecurityGroup:**
    - **Purpose:** Controls access to the Redis instance.
    - **Inbound Rules:** Allows TCP port 6379 and SSH (22) from anywhere.
- **AdminSecurityGroup:**
    - **Purpose:** Controls access to the Admin Panel EC2 instance.
    - **Inbound Rules:** Allows SSH (22), HTTP (80), and HTTPS (443) from anywhere.

## Compute Resources

### Amazon ECS Cluster

- **Resource:** `ECSCluster`
- **Purpose:** Hosts Docker containers for microservices.
- **Details:**
    - **Cluster Name:** Derived from the stack name.

### EC2 Instances

- **Admin EC2 Instance:**
    - **Resource:** `AdminEC2Instance`
    - **Purpose:** Runs the Admin Panel application.
    - **Details:**
        - **Instance Type:** `t3.medium`
        - **AMI:** Amazon Linux 2
        - **Subnet:** PrivateSubnet1
- **Redis EC2 Instance:**
    - **Resource:** `RedisEC2Instance`
    - **Purpose:** Hosts Redis for caching.
    - **Details:**
        - **Instance Type:** `t3.medium`
        - **AMI:** Amazon Linux 2
        - **Subnet:** PrivateSubnet1

## Load Balancers

### Application Load Balancer (ALB)

- **Resource:** `LoadBalancerALB`
- **Purpose:** Distributes incoming traffic across ECS tasks.
- **Details:**
    - **Type:** Internet-facing
    - **Subnets:** PublicSubnet1, PublicSubnet2
    - **Security Group:** `ALBSecurityGroup`
- **Target Group:**
    - **Resource:** `ECSLoadBalancerTargetGroup`
    - **Purpose:** Routes traffic to ECS tasks.
- **Listener:**
    - **Resource:** `ECSLoadBalancerListener`
    - **Purpose:** Listens on port 80 and forwards requests to the target group.

### Classic Load Balancer (CLB)

- **Resource:** `ClassicLoadBalancer`
- **Purpose:** Distributes traffic to the Admin Panel EC2 instance.
- **Details:**
    - **Subnets:** PublicSubnet1, PublicSubnet2
    - **Security Group:** `AdminSecurityGroup`
    - **Listeners:** Listens on port 80.

## Storage

### Amazon S3 Buckets

- **Secure Storage Bucket:**
    - **Resource:** `S3Bucket`
    - **Purpose:** Stores application data securely.
    - **Features:**
        - **Versioning:** Enabled
        - **Access Logging:** Enabled, logs stored in the Logging Bucket.
        - **Public Access Blocked:** Yes
- **Logging Bucket:**
    - **Resource:** `LoggingBucket`
    - **Purpose:** Stores access logs for the Secure Storage Bucket.
    - **Features:**
        - **Versioning:** Enabled
        - **Public Access Blocked:** Yes

## Databases

### Amazon DocumentDB Cluster

- **Cluster:**
    - **Resource:** `DocumentDBCluster`
    - **Purpose:** Provides a MongoDB-compatible database.
    - **Details:**
        - **Master Username/Password:** From parameters.
        - **Security Group:** `DocDBSecurityGroup`
        - **Subnet Group:** `DocDBSubnetGroup`
- **Instances:**
    - **Resources:** `DocumentDBInstance1`, `DocumentDBInstance2`, `DocumentDBInstance3`
    - **Purpose:** Provides high availability and read scaling.
    - **Details:**
        - **Instance Class:** `db.t3.medium`
- **Subnet Group:**
    - **Resource:** `DocDBSubnetGroup`
    - **Purpose:** Specifies subnets for DocumentDB instances.

### Redis

- **Resource:** `RedisEC2Instance`
- **Purpose:** Provides in-memory data store for caching.
- **Details:**
    - **Instance Type:** `t3.medium`
    - **AMI:** Amazon Linux 2
    - **Subnet:** PrivateSubnet1
    - **Security Group:** `RedisSecurityGroup`

## Message Broker

### RabbitMQ Broker

- **Resource:** `RabbitMQBroker`
- **Purpose:** Facilitates asynchronous communication between microservices.
- **Details:**
    - **Engine:** RabbitMQ 3.13
    - **Deployment Mode:** Single Instance
    - **Instance Type:** `mq.t3.micro`
    - **Subnets:** PrivateSubnet1
    - **Security Group:** `RMQSecurityGroup`
    - **Users:** Configured with admin username and password from parameters.

## Container Registry

### Amazon ECR Repository

- **Resource:** `ECRRepository`
- **Purpose:** Stores Docker images for microservices.
- **Details:**
    - **Repository Name:** Derived from the stack name.
    - **Lifecycle Policy:** Retains the 10 most recent images.

## Outputs

- **ALBDNS:** DNS name of the Application Load Balancer.
- **CLBDNS:** DNS name of the Classic Load Balancer.
- **DocumentDBClusterEndpoint:** Endpoint to connect to the DocumentDB cluster.
- **RabbitMQBrokerEndpoint:** Endpoint to connect to the RabbitMQ broker.
- **S3BucketName:** Name of the Secure Storage Bucket.
- **ECRRepositoryURI:** URI of the ECR repository for pushing Docker images.

## Post-Deployment Steps

After the infrastructure is successfully deployed, follow these steps to deploy your microservices and finalize the setup.

### 1. Build and Push Docker Images

- **Build Docker Images:**
    - For each microservice, create a Dockerfile and build the Docker image.
    - Example:
      ```bash
      docker build -t my-microservice .
      ```
- **Tag Images:**
    - Tag each image with the ECR repository URI.
    - Example:
      ```bash
      docker tag my-microservice:latest <ECRRepositoryURI>:latest
      ```
- **Authenticate Docker to ECR:**
    - Obtain login command:
      ```bash
      aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <ECRRepositoryURI>
      ```
- **Push Images:**
    - Push the Docker images to the ECR repository.
    - Example:
      ```bash
      docker push <ECRRepositoryURI>:latest
      ```

### 2. Create Task Definitions

- **Define Task Definitions:**
    - In the ECS console, create a new task definition for each microservice.
    - Specify:
        - **Container Definitions:** Image URI, port mappings, environment variables.
        - **Task Role:** IAM role for task permissions.
        - **Execution Role:** IAM role for ECS tasks to pull images and send logs.
        - **Resource Requirements:** CPU units, memory.

### 3. Deploy Services in ECS Cluster

- **Create Services:**
    - In the ECS console, create a new service for each task definition.
    - Specify:
        - **Cluster:** Select the created ECS cluster.
        - **Service Name:** Name of the service.
        - **Number of Tasks:** Desired number of task instances.
        - **Load Balancing:** Configure to use the ALB and target groups.

### 4. Update Load Balancer

- **Create Target Groups:**
    - For each service, create a target group if needed.
    - Specify health check paths and protocols.
- **Update Listener Rules:**
    - In the ALB settings, update listener rules to direct traffic to the appropriate target groups based on URL paths or host headers.
- **Register Targets:**
    - Ensure that ECS tasks are registered with the target groups.

### 5. Update Amplify Environment Variables

- **Access Amplify Console:**
    - Navigate to the AWS Amplify Console in the AWS Management Console.
- **Update Environment Variables:**
    - For each Amplify application, update the environment variables to point to the new backend services (e.g., API endpoints, authentication details).
- **Redeploy Applications:**
    - Trigger a new build and deploy to apply the changes.

### 6. Attach Custom Domains

- **For Amplify Applications:**
    - In the Amplify Console, navigate to Domain Management.
    - Add your custom domain and follow the verification steps.
- **For Load Balancers:**
    - **Obtain SSL Certificates:**
        - Use AWS Certificate Manager (ACM) to request SSL certificates for your domains.
    - **Configure ALB and CLB:**
        - Update the load balancer listeners to use HTTPS and attach the SSL certificates.
    - **Update DNS Records:**
        - In Route 53 or your DNS provider, create A or CNAME records pointing your custom domains to the load balancer DNS names.

---

By following the steps above, you will have a fully functional microservices architecture on AWS, ready to handle production traffic with high availability and scalability.

---
