# ECS and ECR

## Docker images

Docker image storage options:

- Docker Hub - public images (https://hub.docker.com)
- Amazon Elastic Container Registery (ECR)
  - Private repository
  - Public repository - Amazon ECR Public Gallery (https://gallery.ecr.aws)

## Docker container management on AWS

- Amazon Elastic Container Service (ECS) - Amazon's own container platform
- Amazon Elastic Kubernetes Service (EKS)
- Amazon Fargate - Amazon's own serverless container platform, works with ECS and EKS

## ECS Launch Types

### ECS - EC2 Launch Type

Docker container launched on ECS - EC2 Launch Type:

- Launch ECS Tasks on ECS Clusters
- EC2 launch type: you provide and maintain the EC2 Instances that the ECS Cluster runs on.
- Each EC2 Instance runs the ECS Agent running Docker.
- AWS starts and stops containers
- Can have multiple EC2 Instances in the ECS Cluster
- When deploying a container, you have already provisioned and created the ECS Cluster and EC2 Instances.

### ECS - Fargate Launch Type

- You don't not provision and manage the infrastructure
- It's 'serverless' (off course using EC2 underneath but managed by AWS)
- You just create task definitions
- AWS run the ECS Tasks for you based on CPU and RAM required
- Scaling - just increase number of tasks

## ECS - IAM Roles for ECS

### EC2 Instance profile (EC2 Launch Type only)

- You need an EC2 instance profile for launching the EC2 instances (see Auto-scaling Groups).
- EC2 Instance profile will have associated IAM Roles for permissions:
  - ECS Agent usage
  - Make API call to ECS Service
  - Send container logs to CloudWatch
  - Pull Docker container from ECR
  - Pull secrets from Secrets Manager, etc

### ECS Task Role

- **Used by both launch types (EC2 and Fargate)**
- Allows ECS Tasks to have specific IAM Roles
- Different ECS Services can use the different roles depending on what permissions they need, and what they need to access.
- Task Role is defined in the task definition.

## Load Balancer Integrations

- Application Load Balancer: supported and works for most cases
- Network Load Balancer: for high throughput, or for AWS Private Link.

## Data Persistence

- Use Elastic File System (EFS) for all Launch Types
- Data is persisted independent of containers
- Data can be shared between container instances
- Mount EFS onto ECS Tasks
- AWS S3 cannot be mounted onto ECS Tasks

## ECS Demo - ECS Cluster

1. ECS > Create **ECS Cluster**
2. Cluster name
3. Infrastructure options

- AWS Fargate (x)
- EC2 Instances (x)
  - Choose/Create ASG
  - Choose EC2 Instance type (e.g. t2.micro)
  - Choose capacity
  - SSH keypair
  - EBS size
  - Network settings:
    - VPC
    - Subnets
    - Security Groups
    - Auto-assign public IP
- External Instances using ECS anywhere

4. In this demo, both AWS Fargate and EC2 Instances are selected.

- When creating the ECS, all the required innfrastructure is created for the EC2 and Fargate providers.
- In the ECS > Infrastructure tab, there will be multiple capacity providers for the ECS where ECS Services can be deployed:
  - FARGATE (Fargate Provider) - Standard On-demand pricing
  - FARGATE_SPOT (Fargate Provider) - Discounted spare capacity
  - Infra-ECS-Cluster... (ASG Provider) - This is the EC2 Instance launch type

## ECS Demo - ECS Service

1. Create a new Task Definition

- Family name
- Prereq: Need a docker image, e.g, nginx image
- Infrastructure requirements:
  - Launch type:
    - AWS Fargate (x)
    - ECS Instance
- AWS Fargate details
  - OS Architecture, e.g. Linux
  - CPU, e.g. .5 vCPU
  - Memory, e.g. 1 GB
  - Task role (permission if container needs to use AWS services)
  - Task execution role (auto-created)
- Container details
  - Name
  - Image URI - to pull from the Docker hub
  - Essential container (yes/no)

2. Launch the service

- Cluster > Go to the previously created cluster
- Services - Create
- Task definition family - choose the one created in step 1.
- Choose revision
- Service name: update or accept default
- Compute Configuration
  - Strategy:
    - Capacity provider strategy (x)
    - Launch Type
  - Capacity provider:
    - Use FARGATE
  - Deployment configuration: Replica
  - Desired tasks: 1
- Networking
  - Auto-generated VPC and Subnets
  - Security Groups
    - Create a new one - Allow HTTP traffic from anywhere on Port 80.
  - Public IP - Yes
- Load Balancer
  - ALB or NLB (Choose ALB)
  - Create a new ALB
  - Listener on Port 80 HTTP
  - Create new Target group (HTTP on Port 80)
- Launch
  - A public url is generated, you can accss the webpage running the Nginx image in ECS.
  - You can see the 'Events' page for a log of all Events

3. Launch more tasks
   - Cluster > {your cluster} > Services > {your service} > Tasks > Update service
   - Update the service to increase the number of desired tasks (e.g. to 3)
   - New tasks will be provisioned automatically on the FARGATE service
   - Load the webpage and refresh, the ALB will load-balance across the mutliple instances of the running containers.

## ECS Service Auto-scaling

- Auto-scaling, increases/decreases the number of tasks (container instances) running, it does not scale the infrastructure (e.g. does not add more EC2 instances).
- Fargate takes care of scaling architecture underneath.
- Setup AWS Application Auto Scaling with the ECS, metrics to scale on:
  - ECS Service Average CPU Utilisation
  - ECS Service Average Memory Utilisation - Scale on RAM
  - ALB Request Count Per Target - ALB metric
- Auto-scaling ECS Tasks in ECS Service Auto-scaling works for both Fargate and EC2 Instance launch types - but for EC2 instance launch types, scaling the underlying EC2 instances running is not included.

### Auto-scaling behaviours

- Target tracking - scale based on target value of the metrics above
- Step scaling - based on CloudWatch alarm
- Scheduled scaling - based on date/time

### EC2 Launch Type - Auto-Scaling EC2 Instances

Fargate does this automatically if using the Fargate launch type, for **EC2 Launch type** options are:

- Auto Scaling Group Scaling
  - Scale your ASG based CPU utilisation, adds EC2 instances over time.
- ECS Cluster Capacity Provider
  - Automatically provisions and scales EC2 instances for your ECS Tasks, when CPU/RAM are needed.
  - It is paired with an ASG.

## ECS Rolling Update

When you want to update the version of your Tasks, e.g. updated image, you can do rolling updates. Containers must be destroyed and new ones created. You specify:

1. Minimum healthy percent: this defines how many tasks should be running at minimum, e.g. 10 tasks, 50%, 5 tasks could be destroyed at once, as long as 5 ar remaning.
2. Maximum percent: this defines how many tasks can be running concurrently, so how many new ones can be created when old ones are deleting. e.g. 200% with 10 tasks means all new tasks could spin up while old ones are deleting, you could temporarily be runing 20 tasks.

100% Minimum healthy percent and 200% Maximum percent would ensure zero downtime. New tasks would be created before any were deleted.

## ECS Solution Architectures

1. ECS tasks invoked by Event Bridge

- This creates something like a serverless architecture
- Flow:
  - Object created in S3 Bucket
  - Event Bridge event has a rule to run an ECS Task
  - ECS Task is created in ECS
  - ECS Task has ECS Task Role with permission to access the S3 Bucket, process the object, and write somewhere else with permission, e.g. a database

2. ECS tasks invoked by Event Bridge Schedule

- This is similar to above, but the ECS Task is created on a scheduled frequency, e.g. every hour and not triggered by a specific event.

3. SQS Queue

- SQS Queue is processing Messages
- ECS Tasks running in ECS are polling for Messages
- ECS Service Auto Scaling can be enabled to add more Tasks when the Queue has more messages

4. Intercept Tasks using Event Bridge
   Event Bridge can be used to watch for Events in ECS, e.g. a stopped Task, Event Bridge can be configured to perform some action, e.g. email an Admistrator.
