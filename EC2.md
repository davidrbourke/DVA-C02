# EC2

## Basics

Elastic Compute - Infrastructure as a service

- VMs (EC2)
- Storing data on Virtual Drives (EBS)
- Distributing load balancing (ELB)
- Auto-scaling Groups (ASG)
- Bootstrapping - run scripts on start up, e.g. installing software

IP is dynamic by default. A new IP is generated for the VM between restarts.

## Types of instances

https://aws.amazon.com/ec2/instance-types/

### Naming convention

Example: M6.2xlarge

#### Explained

- Class, e.g. M
- Generation, e.g. 6
- Size, e.g. 2xlarge

### Classes

General Purpose - balanced, e.g. web servers, diverse workloads (M class)
C- Compute Optimized, e.g. gaming, machine learning, intensive CPU usage, etc. (C class)
R & X - Memory Optimized (R for RAM), e.g. in memory databases, etc. (R & X class)
Storage Optimized - high frequency OLTP systems, cache for in memory, data warehouses, etc. (I & D class)

## Firewalls

### Security Groups

- Security Groups control traffic allowed in and out of EC2 instances.
- Only contains **allow** rules.
- Can references by IP, or by other Security Groups.
- Regulate

  - Access to ports
  - Authorised IP ranges
  - control of inbound network
  - control of outbound network

- Can be attached to multiple instances
- Locked down to specific regions
- Live outside of the EC2, EC2 won't see blocked traffic
- Maintain one separate security group for SSH access (best practice)
- Timeout errors - means security group blocked
- Connection refused, or other error - means application was reached by there is some issue.
- Referencing other security groups allows EC2 instances that share the same SG to communicate with each other.

  E.g.
  Type: HTTP
  Protocol: TCP
  Port Range: 80
  Source: 0.0.0.0/0
  Description: ...

#### Common Ports

- SSH 22
- FTP 21
- SFTP 22
- HTTP 80
- HTTPS 443
- RDP 3389 (Remote Desktop for a Window instance)
