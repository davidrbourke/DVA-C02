# EC2 Instance Storage

## Elastic Block Storage (EBS)

1. EBS is a network drive you can attach to your EC2 instance while it runs.
2. Allows your instances to persist data even after termination.
3. They can only be mounted to one EC2 instance at a time **(io1/io2 are the only exception to this, up to 16 EC2 Instances can be connected to)**.
4. Bound to a specific availability zone.
5. Free tier: 30GB free EBS of General Purpose (SSD) or Magnetic per month.

## EBS Volume

1. It's a network drive, so not attached physically to the EC2, therefore there is network latency.
2. There is a one-to-many relationship between the EC2 (one) and the EBS (many), an EBS cannot be attached to more than one EC2 at a time.
3. It's locked to an availability zone, to move a volume to a different AZ, a snapshot can be taken and moved across.
4. You pay for provisioned capacity (not used capacity).
5. You can increase the provisioned capacity if needed.
6. The EBS does not need to be attached to an EC2, it can be left and attached when needed.
7. When the EBS volume is attached to the EC2, it must be formatted with a file system (if a snapshot has not be loaded onto it), and mounted: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html

## Delete on termination

When creating an EBS volume during EC2 creation, there is an option to delete on termination. Default selection (can be overridden):

- The root volume is selected for delete on termination
- Any other attached EBS volume is **not selected** by default for deletion on termination. It will remain, unless the option is overridden.

### EBS Volume Types

https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html

Only **gp2/gp3** and **io1/io2** can be used as boot volumes.

- gp2/gp3 SSD: General purpose SSD, balanced performance and workloads.
- io1/io2 Block Express SSD: Highest performance SSD for mission critical low-latency high-throughput workloads.
- st1 HDD: Low cost HDD for frequently accessed, high-throughput workloads.
- sc1 HDD: Lowest cost HDD for less frequently accessed workloads.

#### gp2/gp3

- Cost effective
- Uses: System boot volumes, virtual desktops, dev and test environments.
- gp3 is the newer version, can increase IOPS and throughput independently of storage size.
  - Max throughput p/s: 1000 MiB/s
  - Max IOPS 16,000
- gp2 is the older version, IOPS and throughput are linked to storage size.
  - Max throughput p/s: 250 MiB/s
  - Max IOPS 16,000
- Volume size: 1GiB to 16 TiB.

#### io1/io2 Provisioned IOPS

- Critical business applications with sustained IOPS performance. E.g.,
  - Applications that need more than 16,000 IOPS.
  - Database workloads with high requirement for consistent performance.
- io1
  - Volume size: 4 GiB to 16 TiB
  - Max throughput p/s: 1000 MiB/s
  - Max PIOPS (Provisioned IOPS): 64,000 for Nitro EC2, 32,000 for others.
  - Can increase IOPS independently to storage size.
- io2 Block Express
  - Volume size: 4 GiB to 64 TiB
  - Max throughput p/s: 4000 MiB/s
  - Max PIOPS: 256,000, sub-millisecond latency.
- Supports **EBS Multi-attach**
  - io1/io2 are the **only** EBS storage types that support multi-attach.
  - They can be attached to up to **16** EC2 Instances within the same AZ.

#### Hard Disk Drives

- Cannot be a boot volume
- Volume size: 125 GiB to 16 TiB
- st1: throughput optimised, 500 MiB/s throughput, max IOPS 500
- st2: 250 MiB/s throughput, max IOPS 250

#### ⚙️ Amazon EBS Volume Type Comparison

| Volume Type | Max Size | Max IOPS                     | Max Throughput               | Notes                                                         |
| ----------- | -------- | ---------------------------- | ---------------------------- | ------------------------------------------------------------- |
| **gp2**     | 16 TiB   | 16,000 (scales with size)    | 250 MiB/s (scales with size) | Baseline 3 IOPS/GB; burst to 3,000 IOPS for <1 TiB            |
| **gp3**     | 16 TiB   | 16,000 (provisionable)       | 1,000 MiB/s (provisionable)  | Performance decoupled from size; cheaper than gp2             |
| **io1**     | 16 TiB   | 64,000 Nitro (provisionable) | 1,000 MiB/s                  | High-performance; supports multi-attach                       |
|             |          | 32,000 others                |                              |                                                               |
| **io2**     | 64 TiB   | 256,000 (provisionable)      | 4,000 MiB/s                  | Higher durability (99.999%); ideal for critical workloads     |
| **st1**     | 16 TiB   | 500 (Not guaranteed)         | 500 MiB/s (burst)            | Throughput-optimized HDD; best for large sequential workloads |
| **sc1**     | 16 TiB   | 250 (Not guaranteed)         | 250 MiB/s (burst)            | Cold HDD; lowest cost; infrequent access                      |

### EBS Description

- Size of volume
- IOPS: Input/Output Operations per second - how many read/write operations the EBS volume can handle per second.
- Throughput: The amount of data transferred per second.

## EBS Snapshots

A backup of your EBS, and can be copied across regions and availability zones.
Don't need to detach the EBS to take a snapshot.

### EBS Snapshot Archive

1. For cheaper storage of the snapshot (75% cheaper) it can be moved to an archive.
2. Restoring the snapshot from the archive takes 24 to 72 hours.

### EBS Recycle Bin

1. A rule can be setup to keep deleted snapshots in a recycle bin in case of accidental deletion, etc.
2. The length the snapshot is retained in the recycle bin is from 1 day to 1 year.

### Fast Snapshot Restore (FSR)

FRS enables the snapshot to be reinitialised on first use with no latency.

## Amazon Machine Image (AMI)

An AMI is a template that is used to create EC2 Instances. It includes the OS, applications, configuration, and monitoring. Creating your own AMI can have a faster boot time as software is pre-packaged.

- AMIs are built for a specific region, and must be copied to other regions to use on an EC2 in another region.
- There are public AMIs.
- You can create your own AMI.
- You can buy and sell AMIs on the Amazon Marketplace.

### Process to create an AMI

1. Start an EC2 Instance and customise it.
2. Stop the instance (recommended for stability).
3. Build an AMI from the Instance.
4. Launch a new EC2 Instance from the customised AMI.

In the demo of this:

- Created an EC2 instance with the User data script setup to install Apache webserver, but did not create an index.html in /var/www/html.

```
#!/bin/bash
# install httpd (Linux 2 version)
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

- Created an AMI from the EC2 instance.
- Created a new EC2 instanced from the AMI, and in the User data script, did not install the Apache webserver, as it is already in the AMI, but did create a custom /var/www/html/index.html

```
#!/bin/bash
# Customise index.html
ble httpd
echo "<h1>Hello world from $(hostname -f)</h1>" > /var/www/html/index.html
```

## EC2 Instance Store

The Elastic Block Storage (EBS) is a network drive, so there is network latency when accessing it. To have faster access to storage from an EC2 instance, an **EC2 Instance Store** can use the hard drive attached to the physical server.

### Ephemeral storage

- When terminating the EC2 instance, the storage will be lost.
- Failure of the physical server will also lose the disk.
- Use it only for temporary storage.

## Elastic File System (EFS)

A Managed network file system.

- Can be mounted on an EC2
- Works on EC2s across multiple availability zones.
- Multiple EC2s across AZs can connect to the same EFS.
- Uses NFS internally.
- Accessed by security groups, subnets must be configured.
- Can only be used by Linux based EC2s, uses POSIX file system.
- Pay per GiB - expensive.

### EFS Scale

- 1000s of concurrent NFS clients, 10GB+ throughput
- Can grow to petabtyes
- Encryption at Rest (configurable)
- You only pay for the storage you use (not provisioned, EFS scales automatically).

### Performance mode

- General Purpose (default) - latency-sensitive, e.g., webserver, CMS.
- Max I/O - higher latency & throughput, designed for highly-parallelised workloads, e.g., big data, media processing (not available with Elastic throughput mode).

### Throughput mode

- Bursting - this is when the throughput grows as you use more storage.
- Enhanced:
  - Provisioned - reserved throughput regardless of storage size.
  - Elastic - automatically scales based on workload, for unpredictable workloads.
