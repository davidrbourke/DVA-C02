# AWS Other Services

## AWS SES

- Simple Email Service
  - SMTP
  - SDK
- Integrates with S3, SNS, Lambda.
- Integrated with IAM

### AWS OpenSearch Service

- Successor to Elastic Search
- DynamoDB can only query on PK, and Indexes.
- With OpenSearch you can search any field, even partial matches.
- Used as a complement to another service
- Modes: managed cluster or serverless cluster
- Has its own query language, a plugin enables using SQL.
- Ingests data: Kinesis Data Firehose, AWS IoT, CloudWatch logs.
- Security: Cognito, IAM, KMS, TLS.
- Has Dashboard to visualise data
- Patterns:
  - CRUD -> DynamoDB - > DynamoDB Stream -> Lambda -> OpenSearch
    - Search via OpenSearch to retrieve an ID
    - Then use ID to retrieve full document from DynamoDB
  - CloudWatch Logs -> Subscription filter -> Lambda -> OpenSearch
  - CloudWatch Logs -> Subscription filter -> Kinesis Data Firehose (near real-time) -> OpenSearch
  - Kinesis Data Streams -> Kinesis Data Firehose (Lambda here to transform data) -> OpenSearch
  - Kinesis Data Streams -> Lambda (real-time) -> OpenSearch

## Amazon Athena

- Serverless query service to analyse data stored in S3
- Uses standard SQL (built on Presto)
- Supports CSV, JSON, ORC, Avro, Parquet.
- Price: $5.00 per TB of data scanned.
- Used with Amazon Quicksight for report/dashboard
- Can use with any logs, e.g. VPC Flow, ELB logs, CloudTrail, etc.

### Athena Performance

- Use Columnar data for cost savings
  - Recommended format: Parquet or ORC.
  - Use **Glue** app to convert data to Parquet or ORC.
- Compress data: bzip, gzip, etc.
- Partition data sets using S3 hierarchy, e.g.
  - S3://your-bucket/table/partition-column=value
  - S3://travel/flight/parquet/year=2024/month=1
  - S3://travel/flight/parquet/year=2024/month=2, etc
- Use larger files >128MB to minimize overheard, so fewer files scanned.

### Athena Federated Query

- Can query data anywhere, e.g. on-prem, using a Lambda function and data source connector.
- Can query multiple data-sources via the Lambda, e.g. SQL Server, DynamoDB, etc.

### Athena Demo Steps

1. Create the S3 with data
2. Create Athena DB with a query
3. Create a table with a query with LOCATION of bucket
4. Select query from the table.

```
CREATE DATABASE IF NOT EXISTS my_demo_db;

CREATE EXTERNAL TABLE IF NOT EXISTS my_demo_db.my_table (
  id INT,
  name STRING,
  email STRING,
  registration_date STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 's3://your-bucket-name/folder-with-data/'
TBLPROPERTIES ('skip.header.line.count'='1');

-- Select all records from the table
SELECT * FROM my_demo_db.my_table LIMIT 10;

-- Select with specific columns and a condition
SELECT name, email
FROM my_demo_db.my_table
WHERE id > 100;
```

## Amazon MSK (Managed Streams for Apache Kafka)

- Alternative to Amazon Kinesis
- Options:
  - Fully Managed (you control scaling)
    - Allows creating/updating/deleting clusters
    - MSK creates and manages Kafka broker nodes, and zookeeper nodes.
    - Deploy MSK to your VPC, multi-AZ (up to 3 zones for High Availability)
    - Automatic recovery from common Kafka failures
    - Data store on EBS volume **for as long as your want**
  - MSK Serverless
    - Runs Apache Kafka, you don't manage scalability or capacity
    - Automatically scales based on compute

### How Kafka works

- **Producers**: push to Kafka brokers, consumers pull from brokers.
  - Example producers: Kinesis, IoT, RDS, etc.
- **Consumers**: Kinesis, RDS, S3, your apps (on ECS, EC2, Lambda, EKS, etc).
  - Also: Data Analytics for Apache Flink, AWS Glue for Streaming ETL.

### Kinesis Data Streams vs MSK

| Kinesis Data Streams        | MSK                                   |
| --------------------------- | ------------------------------------- |
| Message Size limit 1MB      | 1MB default (configurable, e.g. 10MB) |
| Shards                      | Topics with Partitions                |
| Shard splitting and Merging | Can only add (not remove) partitions  |
| TLS                         | TLS & Plaintext                       |
| KMS at rest encryption      | KMS at rest encryption                |

## Amazon Certificate Manager (ACM)

- Service to provision/manage SSL/TLS certificates
- To provide HTTPS in-flight encryption
- Certificates get loaded on to an endpoint service, e.g. ALB, ELB, CloudFront, etc.
- Supports public and private certificates
- Free public TLS certs
- Automatic certificate renewal

### ACM Private Certificate

- You need to create a private certificate authority for all servers in your organisation
- Private certs do not work on the public internet
- Certs can be issues for users, services, devices, with services integrated with ACM.
- Purpose:
  - encrypt traffic in your organisation
  - cryptographic code signing
  - authenticate users, computers, API endpoints, IoT devices
  - enterprise customers building public key infrastructure (PKI)

## Amazon Macie

- A service to scan your S3 buckets for sensitive personally identifiable information (PII)
- Uses machine learning
- Notifies EventBridge of breaches, so you can setup notifications from EventBridge.

## AWS App Config

- Normally you include application config with your app/code
- AppConfig lets you configure application config outside of your application
- Example of use:
  - feature flags
- Dynamically change any configuration for your app in real-time
- Can rollback or gradually deploy a config change
- Sources:
  - Parameter store
  - SSM Docs
  - S3 Buckets
- Applications pull from the sources
- Validate config with JSON schema or a custom Lambda
