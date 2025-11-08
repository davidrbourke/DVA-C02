# Relational Database Service (RDS)

- A Managed database service for relational databases that SQL
- Engines:
  - Aurora (AWS Proprietary Database)
  - IBM DB2
  - Oracle
  - MySQL
  - MariaDB
  - Postgres
  - SQL Server (Microsoft)

## Managed Service

- Automated Provision
- Automated patching of underlying operating system
- Continuous backups and point in time restore
- Monitoring dashboards
- Read replicas for performance
- Multi AZ for disaster recovery
- Maintenance windows for upgrades
- Scaling capability (vertical and horizontal)
- EBS backed storage
- **Cannot SSH into managed instances - don't have access**

## Storage auto-scaling

When you create the DB you specify storage space required. As you get closer using up the space, more space is **automatically** made available (if enabled).

- Set a Maximum Storage Threshold
- Rules to automatically modify storage:
  - Free storage is < 10%
  - Low storage lasts >= 5 minutes
  - 6 hours have passed since the last modification in storage size
- Supports all RDS engines, and good for where storage is unpredictable.

## RDS Read Replicas

If you have an application with a high number of reads, a single RDS instance may not be able to handle the reads fast enough. In this instance, you can have **read** replicas, so the data is replicated to them.

- Up to **15** read replicas
- Within AZ, Cross AZ, or Cross region
- Eventually consistent - reads are **asynchronous**.
- A read replica can be promoted out of the read-replicas to become its own write database
- The connection string in the application must include all the read replicas included.
  - Auto-scaling and a reader endpoint is only available in RDS Aurora - for non-Aurora RDS, you would have to manually scale in/out read-replicas and update the connection string with the read replicas (although there are work-arounds).
- Example of a use: A reporting database, that you don't want to slow down the application database, can read data from a read-replica.

### Network Costs

- No **network** cost for RDS read replicas where the availability zones (AZ) used are in the same region.
  - E.g., us-east-1a to us-east-1b
- There is a cost when the read replica is in availability zones across different regions.

  - E.g., us-east-1a to us-west-1b

### RDS Multi AZ

This is **NOT** the same as Read Replicas, it is for Disaster Recovery.

- Replication is **synchronous**, so writes only complete when written to all replicas.
- Application communicates to RDS with a single DNS name
- If the RDS fails, there is automatic failover to the replica, you don't need to do anything during the failure.
- Optional: read replicas can also be setup to be multi-AZ.

#### Converting Single-AZ to Multi-AZ

- A zero downtime operation.
- A snapshot gets taken of the primary DB
- The Snapshot is restored to the new database
- Synchronisation is enabled, so the secondary database catches up with it and is then kept in sync with the primary database.

## Aurora DB

A proprietary database from AWS.

- Postgres and MySQL are both supported as Aurora Databases. The drivers work as if Aurora was either of those DBs. When you create the DB, you have to chose either the Postgres Aurora version or MySQL Aurora version.
- Aurora is cloud optimized, 5x performance of MySQL, 3x over Postgres.
- Storage automatically grows
  - Starts at 10GB
  - Grows automatically up to 128TB in increments of 10GB
- Faster replication to read-replicas than MySQL (sub 10ms replica lag)
- Automatic failover is instantaneous.
- Costs 20% more than RDS
- Managed service; automatic backup and recovery, isolation & security, industry compliance, routine maintenance, auto-patching.
- Backtrack restore: can restore to any point in time without using backups.

### High availability

- Creates 6 copies of data across 3 AZ (2 in each AZ). To allow for failures:
  - Requires 4 copies out of 6 for writes to be completed to return a successful write - for performance.
  - Requires 3 copies of out 6 for reads - it might not be the latest copy you get, but 3 of the 6 replicas must be available/healthy for a read, you only get the data from 1 replica.
  - This Multi-AZ setup can be configured, e.g. no. of copies, same or cross region, etc.
- Self healing - can correct corrupt data with synchronisation
- Automated failover handled in 30 seconds
- Can have the primary instance with up to 15 read replicas
- Can have cross-region replication
- **Write endpoint**: A DNS name is provided that points to the write endpoint (primary)
- **Reader endpoint**: Auto-scaling can happen on read-replicas, there is a reader endpoint that connects to all readers, to handle load balancing. You do not need to update the connection string with each read-replica.
- Auto-scaling: a policy has to be setup to add read replicas based on a target metric: CPU usage or average connections. Configure:
  - Metrics
  - Target value, e.g. % CPU consumption
  - Minimum no. of replicas
  - Maximum no. of replicas

## RDS & Aurora Security

- At-rest encryption for the primary and replica databases can be enabled using AWS KMS.
- If the primary is not encrypted, the replicas cannot be.
- Encrypting a current non-encrypted DB requires saving a snapshot and restoring as encrypted.
- In-flight encryption, by default, uses TLS.
- IAM Authentication: Can use IAM Roles to connect to the database so don't have to use a username/password.
- Security Groups: can be setup to control access to the database ports, etc.
- No SSH available except on RDS Custom for Oracle and SQL Server.
- Audit logs, e.g. of queries can be enabled (optionally sent to CloudWatch for longer retention).

## RDS Proxy

The RDS Proxy sits in front of the RDS database, instead of your application connecting directly the database via the connection string, it can connect to the RDS Proxy.

- RDS Proxy is a managed proxy for RDS
- Allows DB connections to be pooled at the proxy level instead of the database, so connections can be pooled and shared across multiple applications. Useful also when using AWS Lambdas that can temporarily increase connections drastically.
- Reduces load on the database, as fewer overall connections may be needed, and minimizes how long the connections may be open for.
- RDS Proxy is Serverless, autoscaling and Multi-AZ
- Reduces RDS & Aurora failover time by up to 66%
- Supports RDS: MySQL, Postgres, MariaDB, SQL Server, Aurora. Oracle is not supported.
- Application just needs a different connection string to the proxy instead of the database.
- Can be used to enforce IAM Authentication for DB connection, and stores credentials in AWS Secrets Manager.
  - For non-RDS Proxy IAM Authentication, an IAM User or Role must be given permission to connect to the database. Using AWS CLI or SDK, a token is requested for the User/Role, and the token is used as the password to connect to the database.
  - For RDS Proxy IAM Authentication, IAM Authenticates the client (App or user), then RDS proxy authenticates to the DB, with a password/credentials stored in Secrets Manager.
- RDS Proxy is not publicly accessible from the internet, it must be accessed from a VPC.

## ElastiCache

Elasticache is managed cache (Redis or Memcached), in-memory databases for high-performance and low latency. Helps reduce the load off of the database for read intensive workloads. Makes the application stateless.

- It is managed, so AWS are responsible for maintenance, patching, failover, recovery, etc.
- Requires coding your application specifically to use ElastiCache patterns.

### ElastiCache Solution Architectures

- DB cache
  1. Application attempts to read data from the cache, a cache-hit will return the data.
  2. A cache-miss, will load the data from the database back to the application.
  3. The application will write the data to the cache for future requests to use.
  4. An invalidation strategy is required, to keep data roughly in sync with the DB.
- User Session store, to make application Stateless
  1. User session is written to cache by one instance of the application
  2. Another instance of the application (e.g. on a different EC2 instance) for the User, loads state from the cache. So the user state is not tied to the instance.

### Redis vs MemCached

#### Redis

- Redis has read replicas
- Data persistence using Append Only File (AOF) - to recover from failure, it stores a log of the SET, DEL events, and re-runs them all on failure recovery.
- Backup and restore features
- Supports sets and sorted sets

#### Memcached

- Supports partitioning/sharding of data (not the same as replication, different data on different shards)
- Not durable, e.g. no recovery from failure
- Has backup and restore features
- Supports multi-threaded architecture

### Redis Demo

1. Configuration

- Deployment options
  - Serverless - auto-scales, managed
  - Design your own - select node type, size, and count

2. Design your own (options):

- Creation method:
  - Easy create
  - Cluster cache - set all options
  - Restore from backup
- Cluster mode - to scale dynamically with no downtime
  - Enabled or Disabled
  - ElastiCache Redis Cluster with Cluster-Mode Disabled allows for a maximum of five Read Replicas
- Cluster name
- Cluster location
  - AWS
  - On-prem
- Multi-AZ
  - Enabled or Disabled
- Auto-failover (yes/no)
- Cluster settings
  - Engine version
  - Port
  - Parameter groups
  - Node type, e.g. cache.t2.micro (0.5GiB)
  - Number of replicas
  - Subnet groups and VPC
  - AZ Placements, e.g. placing nodes in different AZs for better durability
- Advanced
  - Encryption at rest (enabled/disabled)
    - Encryption key, etc
  - Encryption in transit
    - Access control configuration:
      - Using a Redis auth token, or
      - Using user group ACL
  - Security groups - to configure which applications have network access

3. Once created, endpoints are created

- Primary endpoint: for writing
- Reader endpoint: for reading

### Caching Strategies

https://aws.amazon.com/caching/best-practices/

- Only cache the data if it is effective for that data
  - E.g. data that doesn't change that frequently
- Is it safe? E.g. can it be eventually consistent
- It is structured well for caching, e.g. key-value pair

#### Caching design patterns

- **Lazy loading/lazy population/cache-aside**

  1. First read from the cache, if cache-hit, return it
  2. Cache-miss: read the data from the DB
  3. Application writes data to the cache, so it is available for other requests, to get a cache-hit
  4. Only requested data ends up in the cache
  5. Cache failure is not critical
  6. A cache miss results in 3 network calls, read/read/write
  7. Stale data, data may be updated in the DB, and not in the cache.

  ```
  record = cache.get(key)
  if (record is null)
  {
    record = getFromDatabase(key)
    cache.set(key, record)
    return record
  }
  return record
  ```

- **Write through**
  Add or update the database when the database is updated

1. Write to DB
2. Write to cache
3. Read from cache
4. Data is never stale
5. There is write penalty, write/write
6. There is missing data until the cache is updated - might chose to combine with cache-aside to hydrate the cache, e.g. to recover from failure
7. Cache churn - a lot of data written to the cache may never be read

```
record = updateToDatabase(key, values)
cache.set(key, record)
return record
```

### Cache evictions and Time to live (TTL)

Various methods to remove data from the cache:

- Explicitly delete data
- Cache is full so data is evicted - if this happens a lot, need to think about scaling up the cache, e.g. more memory.
- Setting a TTL - expired data is evicted, good for cache-aside, not for write-through

## MemoryDB for Redis

This is when you want to use Redis as your primary database, not just as a cache, and you need strong durability, and can't tolerate data loss.
“Use ElastiCache when Redis is your performance booster. Use MemoryDB when Redis is your source of truth.”

- Redis compatible, durable, in-memory database service
- Ultra-fast - 160 million requests per second
- Durable in memory, with Multi-AZ transactional log, so allows for fast recovery.
- Scales from 10GB to 100TB+
- Useful for online gaming, streaming services.
