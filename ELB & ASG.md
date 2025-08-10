# ELB & ASG

## Scalability vs Availability

- Scalability is adapting to greater workloads, types;
  - Vertical - increasing the size of the instance
  - Horizontal - increasing the number of instances, scaling in or out.
- High availability
  - Goal is survive a data centre loss
  - Runing the application in more than 1 availability zone

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

- Classic - deperacted
- Application Load Balancer: HTTP, HTTPS, Websockets
- Network load balancer: TCP, TLS, USP
- Gateway load balancer: Operates at network layer - IP protocol

### Security Groups

- The Public ELB has a security group with inbound rules to allow traffic from external sources, e.g., 0.0.0.0/0.
- The EC2 instances have a security group to only allow traffic coming from the load balancer.

## Application Load Balancer (ALB)

A Layer 7 load balancer supports HTTP/S and Websockets. Good for microservices, and container based applications.

- Load balancer to multiple applications across target machines/groups.
- Load balance to multiple applications on the same machine (not containers).
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
- Each EC2 instance can be access directly using its public IP.

2. Create a Application Load Balancer

- Schemes: Internet-facing, or Internal (choose Internet-facing)
- IP Address Type: IPv4, Dualstack (choose IPv4)
- Network mapping: Can choose AZ subnet, choose to allow subnet for all 3 availability zones
- Security group: create a new SG to only allow HTTP traffic.
- Listener: a new listener should be created to route traffic from port 80 into the ALB to port 80 on the target group instances.
- Target Group: create a new TG that includes both of the EC2 Instances.
  - Health check has path, set to root /

4. To prevent direct access to the ECS instances, so they can only be access via the ALB, update the EC2 Instances shared Security Group to remove the public access on Port 80.
5. Add the Security Group for the ALB on the Inbound rules for the EC2 instance Security Group.
6. Once the ALB is provisioned, you can access the instances on HTTP 80 using the generated public hostname.

- ALB will switch between the routers.

7. Add a custom rule to route traffic for the ~hostname~/error to an error page:

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
