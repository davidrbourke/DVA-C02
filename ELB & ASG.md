# ELB & ASG

## Scalability vs Availability

- Scalability is adapting to greater workloads, types;
  - Vertical - increasing the size of the instance
  - Horizontal - increasing the number of instances, scaling in or out.
- High availability
  - Goal is survive a data centre loss
  - Running the application in more than 1 availability zone

## Elastic Load Balancer (ELB)

Balances requests across multiple EC2 instances. There is one endpoint of connectivity which is the load balancer.
It is a **Managed** load balancer, AWS are responsible for upgrades, maintenance, availability. It's integrated with other AWS services.

- Health checks prevent sending requests to unhealthy instances.
- Provides SSL termination for your websites.
- Enforce stickiness (to a specific instance) using cookies.
- Separate public from private traffic.
- Can setup public and private load balancers.

### Health checks

The load balancer verifies if the instance is healthy by using a port on the instance, e.g. HTTP port /healthz endpoint.

### Types

- Classic - deprecated
- Application Load Balancer: HTTP, HTTPS, Websockets (OSI Layer 7)
- Network load balancer: TCP (Transmission Control Protocol), TLS, UDP (User Datagram Protocol) (OSI Layer 4)
- Gateway load balancer: Operates at network layer - IP protocol

### Security Groups

- The Public ELB has a security group with inbound rules to allow traffic from external sources, e.g., 0.0.0.0/0.
- The EC2 instances have a security group to only allow traffic coming from the load balancer.

## Application Load Balancer (ALB)

A Layer 7 load balancer supports HTTP/S and Websockets. Good for microservices, and container based applications.

- Load balancer to multiple applications across target machines/groups.
- Load balancer to multiple applications on the same machine (not containers).
- Support redirect from HTTP to HTTPS automatically.
- Routing tables, to route to different target groups, based on:
  - URL
  - Hostname
  - Query strings and headers
- Target Groups:
  - EC2 instances
  - ECS
  - Lambda functions
  - IP Addresses - to private addresses, e.g. to on-prem servers with a private IP.
- A **fixed hostname** is created by AWS when creating the ALB (e.g. ---.region.elb.amazonaws.com).
- Data about the true client (not the ALB) is inserted into the X-Forwarded-For header, e.g. X-Forwarded-For (IP), X-Forwarded-Port, X-Forwarded-Proto (Protocol).

### Practical - creating an ALB

1. Create two EC2 Instances with a shared security group with access from the internet on port 80.

- Include User data to use Apache webserver and have the custom index.html that shows the hostname of the EC2 instance.
- Each EC2 instance can be accessed directly using its public IP.

2. Create a Application Load Balancer

- Schemes: Internet-facing, or Internal (choose Internet-facing)
- IP Address Type: IPv4, Dualstack (choose IPv4)
- Network mapping: Can choose AZ subnet, choose to allow subnet for all 3 availability zones
- Security group: create a new SG to only allow HTTP traffic.
- Listener: a new listener should be created to route traffic from port 80 into the ALB to port 80 on the target group instances.
- Target Group: create a new TG that includes both of the EC2 Instances.
  - Health check has path, set to root /

3. To prevent direct access to the ECS instances, so they can only be accessed via the ALB, update the EC2 Instances shared Security Group to remove the public access on Port 80.
4. Add the Security Group for the ALB on the Inbound rules for the EC2 instance Security Group.
5. Once the ALB is provisioned, you can access the instances on HTTP 80 using the generated public hostname.

- ALB will switch between the EC2 Instances.

6. Add a custom rule to route traffic for the ~hostname~/error to an error page:

- On the listener configuration for the ALB add a new rule
- Add Conditions: filter on **Path** is /error
- Add Action: Return fixed response as returning some error text and 404 HTTP status.
- Set Priority as 5, priority is used to resolve where multiple rules are matched by a single request.

Some notes on rules:

- 100 total rules per ALB
- 5 conditions per rule
- 5 wildcards per rule
- 5 weighted target groups per rule
- Action types on a rule:
  - Forward to target groups
  - Redirect to URL
  - Return a fixed response

## Network Load Balancer (NLB)

A Layer 4 load balancer, for TCP and UDP, handles millions of requests per second, ultra-low latency.

- NLB has one static IP per AZ, and elastic IPs can be assigned to each AZ. Useful for allow listing - e.g., a company wants to access your website hosted on an EC2 but they need to have the IP of it to allow-list it in their firewall.
- An ALB generates a fixed hostname. ALB does also generate a public IP but it is **dynamic** and not static, it can change for example if the ALB is restarted due to AWS maintenance.

### When to use an NLB?

- When an application can only access within 1-3 IPs.
- When extremely high performance is required.

### Target Groups

- EC2 Instances
- IP Addresses (must be private)
  - Server in AWS
  - Service in private data centre
- An Application Load Balancer: you might do this when a static IP is required (NLB) but you also want the rules of an ALB.
- Health checks performed on the target groups by an NLB support: TCP, HTTP, HTTPS protocols.

### Demo

1. Create a Network Load Balancer

- Schemes: Internet-facing, or Internal (choose Internet-facing)
- IP Address Type: IPv4, Dualstack (choose IPv4)
- Network mapping: Can choose AZ subnet, choose to allow subnet for all 3 availability zones
- Security group: create a new SG to only allow HTTP traffic.
- Listener: a new listener should be created to route traffic from port TCP 80 into the NLB to TCP port 80 on the target group instances.
- Target Group: create a new TG that includes both of the EC2 Instances.

  - Health check has path, set to root HTTPS /
    - Advanced settings: healthy threshold, unhealthy threshold, timeout interval.

- On the EC2 instances, allow the NLB security group in the inbound traffic of the Security Group the EC2 instances are using.

This is very similar to setting up an ALB.

## Gateway Load Balancer (GWLB)

This is when you want all traffic to go through 3rd party network inspection tools, e.g., firewalls, detection and prevention systems, deep packet inspection, payload manipulation, etc.

- It operates at the network level (Layer 3 for IP Packets).
- Uses the GENEVE protocol on port 6081

### How it works

- All user traffic goes through a GWLB
- Traffic is distributed across a set of virtual appliances with 3rd party tools (e.g. EC2 instances)
- Accepted traffic goes back to the GWLB, rejected traffic is dropped.
- GWLB then routes traffic to your application

### Target Groups

- EC2 Instances
- IP Address (must be private - if running the virtual appliances on your own network/data centre)

## Sticky Sessions

This is when the client is always routed back to the same instance behind the load balancer. For ALB, Classic and NLB, a cookie is attached with the response/request with an expiration date, that is used to enable stickiness. NLB doesn't need a cookie.

- Application based cookie
  - Custom cookie
    - generated by the target
    - any custom attributes can be added
    - cookie name must be specific individually for each target group
    - cannot use reserved cookie names: AWSALB, AWSALBAPP, AWSALBTG
  - Application cookie
    - Generated by the ELB
    - Uses name AWSALBAPP
- Duration based cookie

  - Generated by the ELB
  - Uses name AWSALB (or AWSELB for Classic Load Balancer)

  ### Configuration

  This is configured at the Target Group level, on the target deregistration management attributes, options are:

  - Round robin (default)
  - Least outstanding requests
  - Weighted
  - Target selection configuration
    - Turn on stickiness option
    - Stickiness type can be configured

## Cross Zone Load Balancing (CZLB)

When using multiple Availability Zones, each Zone has an ELB Node, within each AZ, there may be an unequal number of EC2 instances.

- With CZLB, the traffic is distributed evenly across the total number of instances in all AZs.
- Without CZLB, the traffic is distributed evenly across the number of AZs, which means that traffic may not be evenly spread among total instances across all zones.
- ALB - CZLB is enabled by default, can be **disabled** at the **Target group level**.
  - No charges for inter AZ data.
- NLB & GWLB - disabled by default.
  - If enabled, there are charges for inter AZ data.
  - **Enabled** at the **Load balancer level**.

## SSL & TLS Certificates

Allow traffic between load balancer and clients to be encrypted.

- SSL Secure Sockets Layer, used to encrypt connections.
- TLS Transport Layer Security, is a newer version of SSL.
- Public certificates are issued by Certificate Authorities (CA).
- Using the certificate the traffic between the client and load balancer can be encrypted.
- Certificates have an expiration date and must be renewed.

### SSL In Load Balancer

- Client users -> HTTPS Encrypted -> Load Balancer
- Load Balancer -> HTTP Over private VPC -> EC2 Instance
- The Load Balancer performs SSL Termination
- The Load Balancer use X.509 certificates
- AWS Certificate Manager (ACM) is the place to manage certificates, you can upload your own certificates there.
- In the Load Balancer Listener:
  - You must specify a default certificate
  - Optional: add multiple certificates to support multiple domains
  - Clients use Server Name Indication (SNI) to specify the hostname they reach
  - Can configure a security policy to support older versions of SSL/TLS.

### Server Name Indication (SNI)

Allows loading multiple certificates for multiple websites onto one server.

- The client will specify which website/hostname to request, and the server will load the correct certificate for the website.
- Works on ALB, NLB, or Cloudfront
- Does not work on older generations, e.g. Classic LB

## Connection Draining or Deregistration Delay

When an EC2 instance is set to draining:

1. Current connections are given time to complete, default 300 seconds (1-3600 seconds, 0 to disable).
2. New connections are directed to other instances while the instance is draining.

A scenario where draining will occur is during auto-scaling when a instance is being allocated for termination.

## Auto-Scaling Groups (ASG)

Horizontal scaling: scale-out and scale-in new instance based on load. ASG automates the scaling.

1. Set number of instances, configure:
   - Minimum
   - Desired
   - Maximum
2. Unhealthy instances can be terminated and recreated using the health check.
3. A Launch Template is needed - for configuring the instance template, so the ALB knows how to create the new Instance, all the normal information required for creating a new EC2 Instance:

- AMI + Instance type
- EC2 User Data
- EBS Volumes
- Security Groups
- SSH Key pair
- IAM Roles
- Network and subnets information
- Load Balancer information

4. Scaling Policies

- Scale in and out based on Cloud Watch alarms
  - This would be a metric, e.g. average CPU, etc.
  - If the alarm fires, a scaling activity is triggered in the ASG.

### Scaling Policies

- Dynamic Scaling

  - **Target tracking Scaling**
    - Simple to set-up
    - Example: tracking an average ASG CPU to stay around a consistent percentage, e.g. 50%.
    - Cloud Watch alarms are automatically created in the background when setting up Target tracking.
  - **Simple Scaling**
    - Scale out or in, based on Cloud Watch alarms, e.g. CPU > 75%, CPU < 30%.
    - Configure the action to take when the alarm triggers, e.g.
      - Units of instances to add or remove, or
      - Percentage of ASG instances to increase/decrease
      - Cool down period (default 300 seconds)
  - **Step Scaling**
    - Similar to Simple scaling, scale out or in, based on Cloud Watch alarms, e.g. CPU > 75%, CPU < 30%.
    - Steps: you can add various levels to scale out/in if the Cloud Watch alarm breach value is higher or lower.
    - E.g. if a low breach, add 1 unit, if higher, add 2 units, etc.

- Scheduled Scaling
  - When you can anticipate times when there will be higher load, and more instance will be needed.
- Predictive Scaling
  - Automatically ASG is analysing historical load, and adjusting automatically by predicting future load.
  - Machine Learning driven.
  - Need to configure a Metric, and a Target utilisation.

#### Cool Down Period

After a scaling event is triggered, there is a period of time to see what the impact of the scaling event is, before making any decisions to auto-scale again.

### Good Scaling Metrics

- CPU usage
- RequestCountPerTarget - if requests are increasing/decreasing, can scale-out/in.
- Average Network In/Out
- Any custom metrics that can be pushed to CloudWatch.

### ASG Instance Refresh

For when the AMI is updated, instead of manually going through each instance and restarting with the new AMI, an Instance Refresh allows:

- Setting a minimum health percentage.
- Specify a warm up time, for how long before the instance should be ready to use.
- Applying a new launch template with the new AMI.
- Instance Refresh will go through and restart each Instance with the new AMI, but won't let the total number of healthy instances drop below the minimum healthy percentage.

### ASG Demo

1. Create an ASG launch template

- Select an image - Quick Start - Amazon Linux
  - Free tier eligible AMI
- Instance Type T2 Micro
- Include a key pair (new or existing)
- Subnet - don't include in the template
- Security Group - can select an existing one or create one
- Configure EBS volume
- User data: use the start up script to create the webserver

2. Create the ASG Instance

Configure Launch options:

- Instance requirements - to override template
- Network - VPC for the scaling group
- Select all Availability Zones
  - Distribution: Balanced Best efforts
- Load Balancer - if you don't do this, instance will be created, but won't be added to the Target groups for the ALB automatically.

  - Attach to an existing ALB, NLB, GWLB (or Classic)
  - Select Target Groups from existing Load Balancer Target Groups
  - Enable Load Balancer EC2 Health checks

  3. Configure Group size and scaling

  - Desired Capacity 1
  - Min Scaling 1
  - Max Scaling 3

  - Add the scaling settings later

  4. Review options and create the ASG

  - The initial EC2 instance will be created.
  - The ASG Activity tab will show the instances activity for adding/removing EC2 Instances.

  5. Scaling the ASG

  - In the ASG, go to the Automatic Scaling tab
  - Create a Dynamic scaling policy
  - Policy type: Target tracking scaling
  - Metric: Average CPU utilisation
  - Target Value: 40

  6. Test the policy by generating load on the running EC2 instance (e.g. using stress.sh)
