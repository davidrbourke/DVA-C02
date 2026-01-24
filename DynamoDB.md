# DynamoDB

- NoSQL Serverless Database
- RDS uses Relational databases, mostly uses vertical scaling, and some horizontal scaling with read replicas, limited. And no horizontal write scaling.

## Dynamo

- Scales horizontally, by adding more instances.
- NoSQL, doesn’t have RDS structure
- Available across AZs
- Scales – distributed database
- Millions of requests per second, trillions of rows, 100s TB of storage.
- Fast, low latency.
- Fully integrated with IAM
- Low cost
- Standard and Infrequent Access tiers

## Basics

- Made of Tables, Primary Key decided at creation.
- Infinite no. of rows
- Items have attributes
- Max item size is 400KB
- Data types:
  - Scalar – string, number, binary, Boolean, null.
  - Document Types – List, Map.
  - Set Types – String set, Number set, Binary set.

## Primary Keys

(Exam will test on this)

Option 1: Partition Key (Hash)

- Must be unique for each item
- Diverse – so data is distributed
- Example: User_ID, other table attributes are not included in the key

Option 2: Partition Key + Sort Key (HASH + RANGE)

- Could be two fields, e.g, USER_ID, and GAME_ID
- Partition key doesn’t have to be unique but the combination of both must be unique.
- Diverse partition key – so data is distributed enough.

## How to choose a partition key, example of these fields

- Movie_id – good, all unique, high cardinality
- Producer_name
- Lead_actor_name
- Movie_language – very poor, single language, e.g. English could skew partition, so data not evenly distributed.

## Demo

- Table name (don’t need to specify a DB name, just Tables).
- Primary key, Sort key (optional)
- Table class: Standard, Standard-IA (Infrequently Accessed)
- Capacity calculator
- Read write mode: On-demand, or Provisioned (in the free tier)
- Read and Write capacity (fixed or auto-scaling), up to 10 within the free tier (e.g. 2 for read, 2 for writes)
  - Can turn on auto-scaling, set min, max capacity and target utilisation percentage.
  - Auto-scaling is be set separately for both Read and Write capacity
- Secondary index
- Can encrypt at rest

## Read & Write Capacity Modes

Can switch between modes every 24 hours.

### Provisioned Mode

- Specify number of read/writes per second
- Plan capacity before hand
- Pay for provisioned read/write capacity units
- Cheaper

### On-Demand Mode (default)

- Read/write scales automatically in/out with workloads
- Pay for what you use
- More expensive

## Provisioned Mode

- Read Capacity Units (RCU) – throughput for reads
- Write Capacity Units (WCU) - throughput for writes
- Table must have provisioned RCU and WCU
- Can setup up optional auto-scaling of throughput
- **Burst capacity**: temporary exceeding of throughput allowed.
  - Exhausting this will throw exception: ProvisionedThroughputExceededException
  - Strategy is to adopt exponential backoff retry
- Formulas **(exam will ask)**
  - Calculating WCUs
    - 1 WCU is 1 write per second up to 1KB in size, examples:
    - 10 items per second at 2KB = 20 WCUs
    - 6 items per second at 4.5KB = 30 WCUs (upper rounding up to 5KB)
    - 120 items per minute at 2KB = 4 WCUs (2 per second x 2KB = 4)
  - Calculating RCUs
    - 1 RCU is 1 strongly consistent read per second up to 4KB in size, OR
    - 1 RCU is 2 eventually consistent read per second up to 4KB in size
    - > 4KB is rounded up to nearest full 4KB
    - 10 strongly consistent reads at 4KB = 10 RCUs
    - 16 eventually consistent reads at 12KB = 24 RCUs
      - (16/2) x (12/4) = 8x3 = 24
    - 10 strongly consistent reads p/s at 6KB = 20 RCUs
      - Rounded up from 6KB to 8KB
      - 10 x (8/4) = 10 x 2 = 20

## Partitioning Internals

- ID Key goes through a hash function to determine which partition the data should be sent to.
- WCUs and RCUs are divided and spread evenly across partitions.
- Throttling
  - Causes
    - Hot keys: one partition key is being read too many times.
    - Hot partitions
    - Very large items
  - To resolve
    - Exponential backoff
    - Distribute partition key better
    - Accelerator (?)

## R/W Capacity On-Demand

- Read/Writes scales automatically, so no need to calculate or specify RCU/WCUs
- Unlimited RCU/WCU, no throttling
- Expensive
  - Charged for all read/writes
    - RUU Read Request Units
    - WRU Write Request Units
    - Costs 2.5x more than Provisioned
    - Use cases: unknown unpredictable workloads

## Read Consistency across distributed DBs

- Eventually Consistent Reads (Default)
  - May get stale data as read not fully replicated across all replicas
- Strongly Consistent Reads
  - No stale data on read
  - Request this in the Table query, set `ConsistentRead` parameter to true, Not set for entire table.
  - Consumes twice RCU

## Operations

| Operation | Name               | Description                                                                                                  |
| --------- | ------------------ | ------------------------------------------------------------------------------------------------------------ |
| Write     | PutItem            | Create or replace item with same partition key                                                               |
|           | UpdateItem         | Edit or add, can use **Atomic counters**                                                                     |
|           | Conditional Writes | Allow write only if conditions are met. For concurrency, no performance impact.                              |
| Read      | GetItem            | Read based on PK, Hash or Hash+Range, Eventually or strongly consistent, can use `ProjectionExpression`      |
|           | Query              | KeyConditionExpression: Partition key (requried), sort key (optional)                                        |
|           |                    | FilterExpression: Additional to KeyCondition, uses only non-PK attributes.                                   |
|           |                    | Can query a table, local secondary index, or global secondary index                                          |
|           | Scan               | Export entire table, returns up to 1MB of data, use Limit to reduce RCUs, Use parallel scan for performance. |
|           |                    | Can use with filter and project expressions                                                                  |
|           |                    | Limit of items to return, or up to 1 MB, use pagination to exceed.                                           |
| Delete    | DeleteItem         | Delete single item, can be conditional                                                                       |
|           | DeleteTable        | Delete whole table and all items, quicker than calling DeleteItem on all items.                              |
| Batch     |                    | Can save latency by batching items, operations are parallel to speed up, need to retry for failed items      |
|           | BatchWriteItem     | Up to 25 PutItem or DeleteItem per call                                                                      |
|           |                    | Up to 16MB of write data, 400KB max per item                                                                 |
|           |                    | Cannot batch **Update**                                                                                      |
|           |                    | UnprocessedItems: failed writes, use exponential backoff or increase WCU, and retry.                         |
|           | BatchGetItem       | Returns items from 1+ tables. Items retrieved in parallel for performance.                                   |
|           |                    | Up to 100 items, or 16MB data max                                                                            |
|           |                    | **UnprocessedKeys**: for failed reads returned, using exponential backoff, or increase RCUs.                 |

### PartiQL

This is a SQL style query language for CRUD operations, doesn't not enhance main operations, only runs against a single table, **no table joins**.

Example:

```
UPDATE "Users"
SET Email = 'unique@example.com'
WHERE UserId = '123'
IF Email <> 'unique@example.com'

SELECT * FROM "Orders"
WHERE OrderDate BETWEEN '2025-01-01' AND '2025-11-22'
AND Status = 'SHIPPED'

# Example reading from index
SELECT * FROM "Users"."EmailIndex"
WHERE Email = 'alice@example.com'
```

## Conditional Writes

For Put, Update, Delete Item & BatchWriteItem.

### Conditions

- attribute_exists
- attribute_not_exists
- attribute_type
- contains (for string)
- begins_with (for string)
- ProductCategory IN (:cat1, :cat2) and Price between :low and :high
- size (string length)

### Examples

The example is to create an item, only where the email address does not already exist.

```
aws dynamodb put-item \
    --table-name Users \
    --item file://item.json \
    --condition-expression "attribute_not_exists(Email) OR Email <> :e" \
    --expression-attribute-values file://condition-values.json
```

Files

```
# item.json
{
  "UserId": {"S": "123"},
  "Email": {"S": "test@example.com"}
}


# conditional-values.json
{
  ":e": {"S": "test@example.com"}
}

```

## Indexes

### Local Secondary Index (LSI)

- Alternative sort key for table, same partition key as base table.
- **Define ONLY at table creation time**
- Up to - must be scalar type: sting, number, binary
- Can contain some or all attributes of the base table (KEYS_ONLY, INCLUDE, ALL)
  - Example, if you want to query without LSI, you query on primary and sort, and then filter based on the attributes. But add LSI for the attributes, and you can query on that LSI.
  - When creating it, you specify the **sort key** for the index and **index name**, and attributes to project. You will query using the base table partition key + new secondary key (LSI) and index name.

### Global Secondary Index (GSI)

- Alternative PK (Hash, or Hash + Range) from base table
- Speeds up queries on non-key attributes
- Scalar types
- Can contain some or all attributes of the base table, KEYS_ONLY, INCLUDE, ALL)
- **Can add it after table creation**
- When creating it, need to specify a new **partition key** and optional **sort key** for the index, and index name, and attributes to project.
- It's like a 'new' table, **so must provision RCUs and WCUs**.

### Throttling on Indexes

- GSI
  - If writes are throttled on the GSI, the main table write is impacted - throttled also - need to allow GSI WCUs carefully.
- LSI
  - Uses WCUs and RCUs from base table, no separate throttling considerations.

## Optimistic Locking (aka Optimistic concurrency)

- Conditional writes - DynamoDB has this feature to ensure an item has not updated before you update/delete it.
- Attribute - version number on item.
- Client gets latest version to update, and only updates if version number has not changed when it does the write, then it also increments the version.
- If the version read has changed by the time of the write, the write is rejected.

## DynamoDB Accelerator (DAX)

- In-memory cache for Dynamo DB
- Micro-second latency for cached read & write
- No application logic required, just need to enable it.
- Solves the 'hot key' problem, because the frequently access item will be cached.
- 5 mins TTL
- Up to 10 DAX cluster, can have multi-AZ (3 recommended for Prod).

### DAX vs ElastiCache

- DAX
  - Individual object cache for simple queries.
- ElastiCache
  - Can cache the result of some complex calculation that would normally be done in code repeatedly.
- Can use DAX and ElastiCache in combination.

### DAX Demo

- In DynamoDB > DAX > Clusters - Create Cluster
- Cluster name
- Node families
  - t-type - bursting, for lower throughput.
  - r-type - fixed capacity, for always ready capacity
  - All families
- Node types
  - e.g. dax.t2.small, vCPU and Memory
- Cluster size (3 for multi-AZ)
- Network - Subnets - need to create a subnet in the VPC
- Access control - security group to access DAX cluster, open port 8111 or 9111 (for encrypted in transit),
- AZ location
- IAM Permissions - role to give DAX access to DynamoDB, all, or limited tables.
- Parameter groups, e.g. Item TTL and Query TTL (5 mins default on both)
- Maintenance Windows, no preference or specific time window
- Cluster endpoint - this is what the application should use to access the cluster.

## DynamoDB Streams

- Ordered stream of item-level modifications, e.g. every CRUD operation.
- Can be sent to:
  - Kinesis data streams (receiver)
  - Lambda function (can pull)
    - Enable **Event Source Mapping** to read from Stream
    - Ensure Lambda has permissions to access the Stream
    - Lambda is invoked Synchronously ?
  - Kinesis client library applications
- Data retention 24 hours - need to persist elsewhere
- Use case
  - react to real-time changes in the DB
  - analytics
  - implement cross region replication
  - insert into derivative tables
  - insert into OpenSearch Service (via Kinesis Data Stream -> Firehose => OpenSearch)
- Can choose info appearing in the data stream:
  - KEYS_ONLY - only the key attributes of the modified item
  - NEW_IMAGE - the entire item after modification
  - OLD_IMAGE - entire item before modification
  - NEW_AND_OLD_IMAGES - both of above
- Streams are made of Shards, AWS provisions them automatically
- Records are not retroactive once stream enabled, only new updates.

### Demo

- Uses Lambda Stream
- Trigger: needs to be set on the DynamoDb stream option
  - Choose Lambda function to be invoked when stream is updated
  - Can use a Lambda blueprint: dynamodb-process-stream
  - Lambda needs permissions
    - Role: add policies
      - AmazonDynamoDBReadOnlyAccess
      - AWSLambdaDynamoDBExecutionRole
  - Choose DB Table for Stream
  - Batch window - to batch up item updates before invoking Lambda
  - Starting position

## TTL Time to Live

- Automatically delete items after expiry time.
- No WCUs consumed
- Number datatype: Unix Epoch timestamp
- Items deleted within a few days of expiration
- Items expired but not deleted still show up in read/query/scans - need to filter them out if you don't want to see them.
- Items also deleted from indexes
- A delete operation for expired items appears in DynamoDB Streams
- When creating the item, you set an expire_on attribute (Unix epoch).
- Enable the Time to Live, set the name of the attribute to look for (expire_on).

## DynamoDB CLI

(Exam likely to ask about these).

- `--projection-expression`: one of more attributes to retrieve.
- `--filter-expression`: filter items before returning
- `--page-size`: for pagination, retrieve full list of items, but split into smaller API calls, default: 1000 items.
- `--max-times`: max number of items to show in the CLI, return NextToken
- `--starting-token`: specify the last NextToken to retrieve the next set of items.

```
# Retrieves all items (e.g. 3 items but in 3 API calls)
aws dynamodb scan --table-name UserPosts --page-size 1

# Fetch next item, uses NextToken return from initial --max-times scan
aws dynamodb scan --table-name UserPosts --max-items 1 --starting-token ey........
```

## Transactions

- All or nothing add/update/delete item for multiple items across one or more tables.
- Atomic ACID
- Read Modes - Eventual consistency, strong consistency, **transactional**
- Write Modes - standard, **transactional**
- Consumes 2x WCUs & RCUs, as DB performs 2 operations for transactional ops.
- Two operations:
  - TransactGetItem - one or more GetItem
  - TransactionWriteItem - one or more PutItem, UpdateItem, DeleteItem

- **Important for exam, is calculation of transactional WCUs**
- WCUs example: 3 transactional write p/s with 5KB item
  - (3x1) x (5KB/1KB) x 2 = 30 WCUs
- RCUs example: 5 transactions read p/s, with item 5KB
  - 5 x (8KB/4KB) x 2 = 20 RCUs

## DynamoDB Session State

- Can store session state as a cache, so can be shared across instances, e.g. User log in Session.
- ElastiCache also does this, but ElastiCache is in Memory
- DynamoDB is serverless, can scale better than ElastiCache.
- EFS can also be used for Cache for EC2 instances, but other two options are better.

## Partitioning Strategies

Some scenarios might not have a good distributed Partition key, e.g. natural key is Candidate_ID for voting and there are only 2 Candidates, you could get a hot ket. In this scenario you can add a suffix to the key, e.g.

- Candidate_A, - Candidate_B, becomes
- Candidate\*A**\*20**, - Candidate\*B**\*72**, Candidate\*A**\*100**, - Candidate\*B**\*712**, etc.
- The suffix methods;
  - Sharding using random suffix
  - Sharding using calculated suffix

## Write Types

- Concurrent writes/Optimistic locking
  - To avoid dirty writes, use **Conditional writes**, using a version on the record, or checking the attribute hasn't updated.
- Atomic Writes
  - incrementing on top of previous update (?)
- Batch writes
  - User updates a batch of records

## Large Objects Pattern

Due to 400KB item size limit, use an S3 for large objects, and store the pointer to the object in S3.

- Client will read the DB to get the S3 address, and then client will get the object from the S3 bucket.

## Indexing S3 Object Metadata

- Flow:
  - Write to S3
  - Lambda triggers on event to sent S3 Metadata to store in DynamoDB, this can be indexed.
- This allows you to query the DynamoDB for S3 objet metadata, which you could not do directly or efficiently on S3.

## Operations

- Table clean up, 2 options:
  1. Scan + DeleteItem (Slow, and expensive)
  2. Drop table + Recreate table (fast and cheaper)
- Copy Table, 3 options:
  1. Using AWS Backup - managed service to create point-in-time backups of DynamoDB and other AWS resources.
  2. Using AWS Glue - a serverless ETL service, can read from DynamoDB and write to another.
  3. Scan + PutItem or BatchWriteItem, write your own code to call these operations.

## Security

- Security
  - VPC Endpoints available to access DB without using internet
  - Access to DB can be controlled via IAM
  - Encryption at rest using AWS KMS and in transit using SSL/TLC

## Other features

- Backup and restore
  - Point in time recovery (PITR)
  - No performance impact (?)

- Global Tables
  - Multi-region, fully replicated, high performance - needs to be enabled

- DynamoDB Local
  - Can develop against local DB without using service in AWS
- AWS Database Migration Service (AWS DMS) can be used to migrate DynamoDB from MongoDB, Oracle, MySQL, S3, etc.

- Fine grained access
  - To give public or app use access to DynamoDB specific Table, use Identity providers:
  - Amazon Cognito User Pools
  - Google
  - Facebook
  - OpenID Connect
  - SAML
- Users log in with the ID provider, get temporary AWS credentials, credentials used to obtain IAM role - this role should only allow access to required tables.
  - IAM Role has Conditions

  ```
  {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyTenantTable",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:BatchGetItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/Tenant123Orders",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"],
          "dynamodb:Attributes": [
            "OrderId",
            "OrderDate",
            "Status",
            "TotalAmount"
          ]
        }
      }
    }
  ]
  }

  ```

  - **cognito-identity.amazonaws.com:sub** will get replaced with the users 'tenant' at runtime.
  - Can also limit the attributes.
