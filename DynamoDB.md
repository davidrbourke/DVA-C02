# DynamoDB

- NoSQL Serverless Database
- RDS uses Relational databases, mostly uses vertical scaling, and some horizontal scaling with read replicas, limited. And no horizontal write scaling.

## Dynamo

- Scales horizontally, by adding more instances.
- NoSQL, doesn’t have RDS structure
- Available across AZs
- Scales – distributed database
- Millions of requests per second, trillions on rows, 100s TB of storage.
- Fast, low latency
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

Option 1: Partition Key (Has)

- Must be unique for each time
- Diverse – so data is distributed
- Example: User_ID, other table attributes are not included in the key

Option 2: Partition Key + Sort Key (HASH + RANGE)

- Could be two fields, e.g, USER_ID, and GAME_ID
- Partition key doesn’t have to be unique but the combination of both must be unique.
- Diverse partition key – so data is distributed enough.

## How to choose a partition key, example of these fields

Movie_id – good, all unique, high cardinality
Producer_name
Lead_actor_name
Movie_language – very poor, single language, e.g. English could skew partition, so data not evenly distributed.

## Demo

- Table name (don’t need to specify a DB name, just Tables).
- Primary key, Sort key (optional)
- Table class: Standard, Standard-IA (Infrequently Accessed)
- Capacity calculator
- Read write mode: On-demand, or Provisioned (in the free tier)
- Read and Write capacity (fixed or auto-scaling), up to 10 within the free tier (e.g. 2 for read, 2 for writes)
- Secondary index
- Can encrypt at rest

## Read & Write Capacity Modes

Can switch between modes every 24 hours.

Provisioned Mode

- Specify number of read/writes per second
- Plan capacity before hand
- Pay for provisioned read/write capacity units
- Cheaper

On-Demand Mode (default)

- Read/write scales automatically in/out with workloads
- Pay for what you use
- More expensive

## Provisioned Mode

- Read Capacity Units (RCU) – throughput for reads
- Write Capacity Units (WCU) - throughput for writes
- Table must have provisioned RCU and WCU
- Can setup up optional auto-scaling of throughput
- **Burst capacity**: temporary exceeding of throughput allowed.
  - Exhausting this will through exception: ProvisionedThroughputExceededException
  - Strategy is to adopt exponential backoff retry
- Formulas (exam will ask)
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
      - (16/2) * (12/4) = 8*3 = 24
    - 10 strongly consistent reads p/s at 6KB = 20 RCUs
      - Rounded up from 6KB to 8KB
      - 10 _ (8/4) = 10 _ 2 = 20

## Partitioning Internal

- ID Key goes through a hash function to determine which partition the data should be sent to.
- WCUs and RCUs are divided and spread evenly across partitions.
- Throttling
  - Causes
    - Hot keys: one partition key is being read too many times.
    - Hot partitions
    - Very large items
  - To resolve
    - Exponential backup
    - Distrbute partition key better
    - Accelerator

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
  - Request this in the Table query, set ConsistentRead parameter to true, Not set for entire table.
  - Consumes twice RCU
