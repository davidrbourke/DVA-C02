# Elastic Beastalk

A way of deploying infrastructure. Most apps have similar architecture, Elastic Beanstalk is a service to create similar architecture that is configurable.

## Components

- Application (enviroments, versions, configurations)
- Application Version: application code iteration version
- Environment - the AWS resources running the application

  - Tiers:
    - Web server: pattern for a web architecture: ELB, AZs, SGs, EC2 instance web servers
    - Worker: pattern for an application with no web client, uses SQS Queue, AZs, SGs, EC2 instances worker server processing from/to the queue.
    - These are auto-scaling environments
  - Dev/test/prod environments

- Deployment Modes
  - Single instance (for dev)
    - Elastic IP, EC2 Instance, RDS
  - High availability with Load balancer
    - ALB, AZs, Mutliple EC2 Instances, RDS
- Frameworks supported: many, .NET, Go, Node, Docker, etc.

## Demo 1 - Single Instance

In the Elastic Beanstalk AWS Console

- Select Environment tier
  - Worker
  - Web (\*)
    - Set app name
    - Set environment name, e.g. dev/test, etc, not limited to dev/test/prod.
- Platform
  - Language/framework, e.g. Node
  - Custom
  - Managed (\*)
- Application code
  - Sample or existing, can upload .zip of built application
- Presets
  - High availability
  - Single Instance (\*)
- Service Access
  - Create a new service role OR Use an existing service role
  - Add EC2 Instance Profile
    - You need to create this EC2 Instance Profile in the IAM Console
      - Add atleast these policies:
        - `AwsElasticBeanStalkWebTier`
        - `AwsElasticBeanStalkWorkerTier`
        - `AwsElasticBeanStalkMulticontainerDocker`
- This crerates the Elastic Beanstalk environment using CloudFormation (CF), in CF you can see the Events of the various components being created, e.g. Security Group, Single EC2 Instance, Public IP, Auto-scaling Group, etc.
- Configure Options:
  - Upload new version
  - Monitoring
  - Configuration - Modify/Apply
- Applications

  - Within this Elastic Beanstalk instance, more environments can be added (e.g. test, prod, etc), you can deploy to these various environments in the single Elastic Beanstalk instance simultaneously.

  ## Demo 2 - Multi-instance

  Multi-instance is suitable for Prod environments as it creates multiple EC2 Instances. The following is similar to Demo 1, but:

  - Presets: **High Availability**
  - Networking
    - VPC
  - Instance scaling - subnets - all Availability Zones
  - Database - is using one, the life of the DB is tied to the life of the Elastic Beanstalk Instance - deleting the EB Instance will also delete the database.
  - Auto-scaling Group
    - Number of min, max, and desired instances.
  - Instance types, e.g. t3.micro
  - Scaling options - scaling in/out setting
  - Load Balancers
    - Type
      - ALB, NLB
      - Dedicated or shared (cheaper)
    - Listeners
  - Monitoring - health reporting
  - Rolling updates and deployment options
  - Log settings - where to steam logs to
  - Components created: ALB, Listener, Target group, IP, Security Groups (ALB to Port 80 on EC2 instances), EC2 instances, Auto-scaling Group (including Dynamic scaling policies automatically created).

## Deployment Modes

This concerns how updates to the application are rolled out. Note: you will pay for all running EC2 instances.

1. All at once

- All instances are taken down in the ASG, and then the new ones start up.
- Fastest overall deployment
- Has downtime
- Cheap - as you never have more instances than your desired capacity

2. Rolling

- Instances are taken down but never below your minimum capacity
- You will be below your normal operating capacity
- E.g. 4 instances -> 2 stopped (2 running total) -> 2 new ones started (4 running total on old and new app version) -> 2 old ones stopped (2 running total) -> 2 new ones added (4 running all running new app version)
- Longer overall deployment than 'All at once'
- Cost is cheaper than Rolling with additional batch - as never go over capacity

3. Rolling with additional batch

- Never runs below normal capacity, you will exceed you normal capacity and have to pay for all running instances while they are running.
- E.g. 4 instances -> 2 added (6 running) -> 2 old instances stopped (4 running) -> 2 new instances added (6 running) -> final 2 old instances stopped (4 running all new app version)
- Longer overall deployment than 'All at once'

4. Immutable

- A new Auto-scaling group (ASG) is created, instances are created in the new ASG.
- Health checks can be run against the instances in the new ASG
- New instances are attached to the existing ASG
- Old instances are terminated
- The new 'temporary' ASG is deleted
- Easy to rollback, if the health checks fail in the new 'temporary' ASG, the instances are not attached to the existing ASG. The new instances are terminated.
- More expensive than 'All at once' or Rolling.

5. Traffic splitting

- Also known as Canary testing, similar to Immutable, but a % of traffic can be routed to the new instances in the new 'temporary' ASG.
- You can test the functionality of the new Instances.
- Finally migrate the instances to the existing ASG, and terminate the old instances and 'temporary' ASG.

6. Swap environments (blue/green)

- Not pure blue/green deployment, but a way to achieve similar.
- Clone an Elastic Beanstalk environment in the EB instance - this creates all components, e.g, Load balancer, environment variables, instances.
- Note: after cloning you can change conifguration, but **you cannot change the Load Balancer type**.
- Deploy the new app version to the cloned environment.
- Test the cloned application
- Swap environment domains, e.g. modify DNS entries, Route 53 can be used to weight traffic to blue or green.

## Elastic Beanstalk CLI

- Specific CLI for EBS (don't need to know command for the exam).
- Commands: eb create, eb status, etc.
- Package code as a .zip and upload onto Beanstalk
- Zip is uploaded to an S3 bucket.
- Deploy code into the environment (code pulled into Elastic Beanstalk from the S3)

## Beanstalk lifecycle policy

- A maximum of 1,000 application versions can be retained.
- Phase out old application version based on
  - time
  - or no. of app versions (space)
- Current 'live' version will not be deleted
- Can **choose** not to delete the source code bundle in S3

## Elastic beanstalk extensions

- All parameters in the UI/CLI can be configured in your source code in a **\*.ebextensions/** directory
  - YAML or Json format as a **\*.config** file
- Everything created by the .ebextentions config file will be deleted if you delete the EB apps, e.g. cache, RDS, etc
- option_settings is for custom settings/env properties

```
# .ebextensions/environment.config

option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    API_KEY: your-api-key-here

packages:
  yum:
    git: []
    nginx: []

files:
  "/etc/nginx/conf.d/custom.conf":
    mode: "000644"
    owner: root
    group: root
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection 'upgrade';
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }

commands:
  restart_nginx:
    command: "service nginx restart"
```

## CloudFormation in Elastic Beanstalk

EB uses CloudFormation to deploy/provision components. .ebextensions + CloudFormation allows you to set up **any** services in AWS. You can look into CloudFormation to see what has been created by the EB deployment.

## EB Migration

After creating the EB environment, you cannot change the Elastic Load Balancer type. To do this follow the steps:

1. Create a new EB environment except for the Load Balancer Type

- Note: you cannot **Clone** because that would also clone the Load Balancer Type.

2. Deploy the app onto the new environment
3. Move traffic to the new environment, e.g. CNAME swap or Route 53

## EB and RDS

Deleting the EB Instance will delete all services created by it, including any RDS instances. To move an RDS away from this risk in production:

1. Snapshot the RDS (as a protection in case this goes wrong)
2. Set setting on the RDS instance itself to protect it from deletion in the AWS RDS Console.
3. Create a new EB instance without the RDS.
4. Point the new EB instance application to the RDS using the connection string
5. Redirect traffic to the new EB instance application
6. Terminate the old EB instance - it will attempt to delete the RDS instance as well but this step will fail (because of step 2).
7. CloudFormation will delete the EB instance but as it fails to delete the RDS instance, the CF stack will still be there. Manually delete the CF stack.
