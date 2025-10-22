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
  - Task role (permission if container needs to use AWS services, e.g. you app needs to access an S3 bucket)
  - Task execution role (auto-created, used by container agent to make AWS API requests on your behalf)
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

## ECS Task Definitions

- Tasks are defined with a JSON file (UI helper in the AWS Console) to tell ECS how to run the Docker container.
- 10 contatiners max per Task definition
- Json definition contains:
  - Image name
    - Essential container (Yes/no) - at least 1 container must be an essential container, if an essential container stops, the Task will be stopped.
    - Registry - can be a public or private repository
  - Port binding for container and host, e.g. container running Apache on port 80, host port 8080 can map to 80. Via the internet you would use Port 8080.
  - Memory and CPU
    - You can configure limits per container on how much of the Task Cpu and memory it can use.
  - Environment varibles
  - Networking
  - IAM Role
  - Logging configuration

### Load balancing - EC2 Launch Type

- If you don't define host ports and only container ports for the ECS Tasks, Dynamic Host Port Mapping is used where a host port number is assigned. E.g. 36789, etc.
- Using ALB (doesn't work for Classic LB), the ALB can find the Task ports on the EC2 Instances.
- The Security Group for the EC2 Instance running the Tasks must allow **any** ports from the ALB, as it doesn't know what they will be.

### Load balancing - Fargate Launch Type

- Each Task has a unique IP
- For Fargate definitions, you don't define a host port, only a container port, e.g. 80
- An Elastic Network Interface (ENI) is automatically created for each Task (with unique private IP) - you need to allow the container port on the ENI Security group from the ALB
- ALB Security group: Allow port 80/443 from the web.

### Task IAM Role

- The IAM Role for the Task's permissions (e.g. Access S3 bucket) are defined at the **Task Definition** level.
- All the containers/services covered by the Task definition will use this same IAM Role.
- To use a different role for other containers/services, they will need to be configured with a different task definition.

### Environment Variables

- Environment variable, options;
  - Hardcode in the Task Definition, e.g. for fixed insecure data, e.g. URLs
  - Pull from a secrets manager on Task startup:
    - SSM Parameter Store
    - Secrets Manager
- Environment files - you can load a full set of environment file from an S3 bucket, called Bulk load.

### Data Volumes (Bind Mounts)

- Share data between multiple containers
- You can mount from an EFS for persistent storage, for temporary storage you can use a file system:
  - EC2 Tasks - using EC2 Instance Storage, data is tied to the lifecyle of the EC2 instance.
  - Fargate Tasks - using ephemeral storage, data is tied to the container lifecyle, size: 20 GiB (default) to 200 GiB.
- Use cases:
  - Share temporary data between containers
  - Side car container pattern, e.g. to send logging data to some other destination, main container writes to the storage, sidecar container reads from the storage and pushes the logs elsewhere.

## ECS Task Placements

Task placement concerns which EC2 Instance the Task is placed on, this is for EC2 Instance Launch types only. Fargate Launch Types, the instance placement is handled by Fargate. You can define a Placement Strategy and Placement Constraints, ECS will make a best effort to place the Task within the Strategy and Constraints.

### Task Placement Process

1. Identify the instances that satisfy CPU, memory, and port requirements in Task Definition.
2. Identify the instances that satisfy the Placement Constraints
3. Identify the instances that satisfy the Placement Strategies.
4. Select the instances for task placement.

### Task Placement Strategies

How tasks are distributed:

1. Binpack - place the task on the container with the least amount of available CPU and or memory. Minimises the no. of instances required/used and saves cost.

```
"placementStrategy": [
  {
    "field": "memory",
    "type": "binpack"
  }
]
```

2. Random - places the task randomly.

```
"placementStrategy": [
  {
    "type": "random"
  }
]
```

3. Spread - place the task evenly across instances based on a specified value, e.g., availabiliy-zone - the tasks would be placed evenly across Instances in different availability zones. Can have multiple spread values. For exmaple with the one below: Tasks will first be spread across zones and instances, then packed within those constraints to optimize memory usage.

```
"placementStrategy": [
  {
    "field": "attribute:ecs.availability-zone",
    "type": "spread"
  },
  {
    "field": "instanceId",
    "type": "spread"
  },
  {
    "field": "memory",
    "type": "binpack"
  }
]
```

### Task Placement Constraints

Where tasks are allowed to run.

- distinctInstance - place each task on different container instance

```
"placementConstraints": [
  {
    "type": "distinctInstance"
  }
]
```

- memberOf - places tasks based on expression using Cluster Query Language

```
"placementConstraints": [
  {
    "expression": "attribute:ecs.instance-type =~ t2.*,
    "type": "memberOf"
  }
]
```

## Elastic Container Registry (ECR)

- Can store docker images in AWS in ECR
- Private and Public repos, public gallery is https://gallery.erc.aws
- Access from ECS to ECR controller via IAM
- ECR supports various admin tasks; image vulnerability scanning, versioning, image tagging, etc.

### ECR Commands to pull/push images to ECR

You need to login Docker on the command line with the AWS ECR credentials and account details (vars: aws_account_id, region).
`aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.eu-west-1.amazonaws.com`
Docker commands
`docker push aws_account_id.dkr.ecr.region.amazonaws.com/imagename:latest`
`docker pull aws_account_id.dkr.ecr.region.amazonaws.com/imagename:latest`
`docker build -t imagename .`
`docker tag image/name:latest aws_account_id.dkr.ecr.eu-west-1.amazonaws.com/new-image-name:latest` - the tag needs the repository name

## AWS CoPilot

This is a CLI tool to help with building and releasing containerised apps on AppRunner, ECS and Fargate.

- Provisions all the required infrastructure
- Simplifies setup of environments so dev can focus on the app
- Can create automated pipelines using CodePipline
- CoPilot assumes various defaults/best practice, you can set specific configuration in manifest.yml files that get generated for the application's CoPilot configuration.
- Flow:
  - YAML or CLI of application architecture ->
  - AWS CoPilot -> Prepares well-architectured infrastructure setup config files ->
  - **CloudFormation** deploys in ECS, Fargate or App Runner.

```
https://github.com/aws-samples/aws-copilot-sample-service
To deploy this app, clone this repo and then run:

copilot init --app demo \
  --name api \
  --type "Load Balanced Web Service" \
  --dockerfile "./Dockerfile" \
  --deploy
Copilot will set up the following resources in your account:

A VPC
Subnets/Security Groups
Application Load Balancer
Amazon ECR Repositories
ECS Cluster & Service running on AWS Fargate
```

## AWS EKS

Elastic Kubernetes Service (EKS), open source alternative to EKS.

- Supports EC2 Instance and Fargate Launch Types
- Use if you are migrating from Kubernetes on-prem or another cloud provider, it is cloud agnostic.
- EKS Nodes are EC2 Instances, running EKS Pods
- Pods are EKS equivalent of Tasks

### EKS Node Types

- Managed Node Groups
  - EC2 instances (Nodes) are created and managed for you
  - Nodes are part of an ASG, support On-Demand and Spot instances
- Self Managed Nodes
  - You create the Nodes and register them to the EKS cluster
  - Nodes are also part of an ASG, support On-Demand and Spot instances
- AWS Fargate
  - You don't manage the nodes or instances

### EKS Data Volumes

- For persistent data storage from EKS
- Assign a Storage Class manifest YAML on your EKS Cluster
- Uses Container Storage Interface (CSI) complaint driver, so the following storage can be used:
  - EBS
  - EFS (for Fargate)
  - FSx for Lustre
  - FSx for NetApp ONTAP
