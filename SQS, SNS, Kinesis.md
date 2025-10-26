# Integration and Messaging

There will be a lot of question on this topic, expecially SQS.
Applications need to share data, to communicate with each other.

## Patterns

- Syncrhonous: app to app direcly, e.g. via HTTP
  - spikes: one service can overwhelm the other with too many requests
- Asynchronous: event based, via some middleware service, e.g. a queue
  - spikes: decouple and scale the middleware with, e.g SQS, SNS, Kinesis

## Simple Queuing Service (SQS)

- Contains a queue of messages
- A buffer to decouple between your producers and consumers
  - **Producer** sends messages to the queue
  - **Consumer** pulls the messages from the queue to process them
  - E.g. Front-end webapp receives request, sends message to queue, consumer backend processing application receives the message, do some processing, e.g. video processing, and insert into the DB.
- SQS is the standard queue
- Fully managed
- Attributes
  - Unlimted throughput
  - Unlimited no. of messages in the queue
  - Short lived messages, default retention is 4 days in the queue, max: 14 days
  - Low latency < 10ms
  - Small messages: limited to 256KB per message
- Can have duplicate messages (at least once delivery)
- Can have out of order messaages (best effort ordering)

### SQS Message Producers

- Send a message to SQS using SDK (SendMessage API)
- Message persisted into queue until Consumer reads and then deletes the message

### SQS Consumers

- Consumers are Applications, e.g. running EC2 instance, on-prem servers, Lambdas
- Polls SQS for messages
- Can received upto 10 messages per request
- Consumer then processes the message, e.g. does some action, write to DB, etc
- Once processed, consumer then deletes the message (DeleteMessage API)
- Can have multiple consumers at a time, each consumer receives a different set of messages
- To increase throughput, scale up Consumers, e.g. with Auto-scaling groups for EC2 instance
  - Metric: ApproximateNumberOfMessage (queue length)
  - Set up CloudWatch alarm based on metric
  - ASG will scale up based on the CloudWatch alarm

## SQS Security

- Encryption
  - In-flight encryption using HTTPS API
  - At-rest encryption using KMS Keys
  - Client-side encryption (app is responsible)
- Access Controls: IAM Policies to control access to SQS API
- SQS Access Policies (similar to S3 bucket policies)
  - For Cross account access to SQS Queues
  - For allowing other services to write to an SQS Queue

### Demo 1

- SQS Console
- Create Queue
  - Standard Queue (\*)
  - FIFO Queue
- Standard Queue
  - Configuration
    - **Visibility timeout**: default 30 secs, set between 0 seconds to 12 hours.
    - Message retention in days
    - Max message size 1KB to 256KB
    - Received message wait time
  - Encryption
    - Disabled
    - Enabled
      - Enrcryption Key (AWS SQS Key, or AWS Key Management)
  - Access Policy, Basic vs Advanced
    - Generates a JSON document for Policy
    - Basic - define who can send/receive messages from the queue:
      - Only the queue owner, or:
      - Only the specified AWS Accounts, IAM users and roles - ultimately generates a policy.
    - Advanced - you define in the JSON itself - like the SQS Access Policy examples below.
- AWS Console has a UI for testing sending and receiving messages
- Access Policy tab - shows who can access the queue and how
- Dead letter queue tab

### SQS Access Policy

- JSON IAM Policies

  1. To allow cross account access, e.g. so an EC2 Instance in another account can access the Queue.

  - Principal: The IAM role in Account B that the EC2 instance assumes (EC2SQSAccessRole). -
  - Resource: The ARN of the SQS queue in Account A.

```
{
    "Version": "2012-10-17",
    "Id": "QueuePolicy",
    "Statement": [
    {
        "Sid": "AllowCrossAccountAccess",
        "Effect": "Allow",
        "Principal": {
           "AWS": "arn:aws:iam::222222222222:role/EC2SQSAccessRole"
        },
        "Action": [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl"
        ],
        "Resource": "arn:aws:sqs:us-east-1:111111111111:MyQueue"
    }]
 }
```

2. Allow an S3 bucket to publish events to an SQS Queue

- In the S3 Bucket, you set up an Event notification to send to the SQS Queue.

```
{
    "Version": "2012-10-17",
    "Id": "S3ToSQSAccessPolicy",
    "Statement": [
        {
            "Sid": "AllowS3BucketToSendMessages",
            "Effect": "Allow",
            "Principal": {
                "Service": "s3.amazonaws.com"
            },
            "Action": "sqs:SendMessage",
            "Resource": "arn:aws:sqs:us-east-1:111111111111:MyQueue",
            "Condition": {
                "ArnEquals": {
                    "aws:SourceArn": "arn:aws:s3:::my-source-bucket"
                }
            }
        }
    ]
}
```

The EC2 example uses `"AWS"` as the principal because it's an IAM role (a human-managed identity), while the S3 example uses `"Service"` because it's an AWS-managed service acting on your behalf. The EC2 role is uniquely identified by its ARN, so no condition is needed. But S3 is a shared service, so the policy includes a `Condition` to restrict access to a specific bucket, preventing any other bucket from sending messages to your queue.

### Message Visibility Timeout

When a message is read by a consumer, this is the timeout value for how long the message cannot be read by another consumer, the consumer must delete the message as 'processed' before this timeout ends, else another consumer will receive the message.

- A message not processed fully during the timeout, will be read twice or more.
- If a consumer needs more time, it should call the `ChangeMessageVisibility` API to get more time.
- Setting this value right is important, to avoid too long between a message being processed again if the consumer crashes, and too short means too many duplicate reads.

### Dead Letter Queue (DLQ)

If a consumer keeps processing a message and it fails and keeps going back into the queue, you can set a threshold for how many times the message can go back into the queue.

- Set a **MaximumReceives** threshold, if it is exceeded, the message goes into the DLQ and it not processed again.
- Allows debugging of something that might be wrong with this message.
- Set a long retention, e.g. 14 days before messages expire from the DLQ so you have time to investigate them and they are not lost.
- DLQ of SQS Queue must be the same type, e.g. Standard queue -> Standard DLQ, FIFO -> FIFO.

#### Demo DLQ

1. You create a new Queue for the DLQ.
2. In your source Queue config, you can set the DLQ created as the target for the DQL and set the **Maximum Receives** value.
3. In AWS Console, in DLQ, there is an option **Start DQL Redrive**
   - You have to select the source queue to push the messages back to.

#### DQL - Redrive to Source

After debugging, you may need to update your application code to handle messages in the DLQ. You can use **redrive to source** to push your DLQ messages back into the SQS Queue to reprocess them after deploying application code updates.

### Delay Queue

Delay messages so that consumer don't see them immediately

- Default is 0 seconds
- Enabled - two ways:
  - Can set default at queue level
  - Or set a per message delay using the **DelaySeconds** parameter

### Long Polling

When polling, when the Consumer requests messages, you can wait for a period if there are no messages.

- Results in less API calls
- Will get messages a soon as one arrives (decreasing latency)
- Set between 1 to 20 seconds
- Prefereable over Short polling
- Enabled - two ways:
  - At Queue Level
  - At the API level using **ReceiveMessageWaitTimeSeconds** when Consumer requests to read.

### SQS Extended Client

For sending large messages beyond 256KB.

- Uses an AWS S3 bucket as a repo for the large data
- Use an AWS SDK library for your programming language, e.g. SQS Extended Client Java Library
- Producer application:
  - 1. Large message/data gets sent to an S3 bucket
  - 2. Write message to the SQS with a reference to the S3 bucket object
- Consumer application:
  - 1. Reads the message from the SQS Queue
  - 2. Uses the data from the message to request the large data object from the S3 bucket
- Example: you wouldn't actually upload a video for processing to the SQS Queue, just to the bucket.

### SQS API

Must know for exam:

- Queue management:
  - CreateQueue
    - MessageRetentionPeriod - how long message is kept in queue before being discarded
  - DeleteQueue delete the queue and all messages in the queue
  - PurgeQueue: delete all messages in the queue
- Producer:
  - SendMessage
    - DelaySeconds - wait before message becomes visisble
- Consumer:

  - ReceiveMesage
  - DeleteMessage
  - MaxNumberOfMessages: default 1, max 10 - for Consumer to get in one request
  - ReceivedMessageWaitTimeSecond: for Long Polling

- ChangeMessageVisisbility: length of time after reading message to wait before it becomes visible again for another Consumer, e.g. if consumer needs longer to process a message.

- Batch API calls to reduce number of requests into the API.
  - SendMessage
  - DeleteMessage
  - ChangeMessageVisibility

### SQS - FIFO Queues

- First In First Out
- Queue must be named \*.fifo (also specifically create the FIFO queue type)
- Guarantees that messages will be consumed in the order they produced into the queue.
- Limited throughput:
  - 300 msgs per second without batch
  - 3000 msgs per second with batching

#### Deduplication

**Exactly-once** send

- Optional parameter
- You add a Deduplication ID to the message sent to the queue
- It is for a set time period (5 mins)
- Duplicate Messages with the same Deduplication ID within a set time period are removed from the queue.
- Two methods:
  - Content based de-duplication - sha-256 hash of messsage body is used as ID
  - Explicitly provide an ID when sending the message

#### Message Grouping

- **Message Group ID**
- Mandatory parameter when sending a message
- - If need ordering across all messags, then use the same parameter
- If you need ordering within specific groups, then provide unique parameter for each group of messages.
- E.g. if you want ordering for each customer's messages, then add the customer ID as the message grouping ID.
- For SQS FIFO, the maximum number of consumers that can simultaneously process messages from the SQS FIFO queue is **equal** to the number of message groups you defined. Each consumer can only handle messages from a single message group at a time, so with 10 message groups, you can support up to 10 consumers concurrently.

## Simple Notification Service (SNS)

To avoid an application having to send messages to many consumer by knowing about those consumers, the **publisher subscriber** pattern is used with SNS. An application publishes notifications to SNS **topic**, and other applications can subscribe to the SNS instance to pick up the messages.

- Event producer only sends one message
- Each subscriber gets messages from the topic - SNS pushes the messages out to the subscribers, they do not pull the message.
- Up to 12,500,000 subscriptions per topic
- Up to 100,000 topic limits per account

- SNS can publish messages to:
  - Email
  - Email-json
  - SMS & Mobile notifications
  - HTTP & HTTPS endpoints
  - SQS
  - Lamdba
  - Kinesis Data Firehose
- SNS receives data from AWS services
  - CloudWatch alarms
  - CloudFormation
  - Etc

### Publish to Topic

- Use the SDK from your application:
  - Create the topic
  - Create the subscription
  - Publish to the topic
- Direct Publish (for mobile apps SDK)
  - Create a platform application
  - create a platform endpoint
  - Publish to the platform endpoint
  - Supported by Google GCM, Apple APNS, Amazong ADM

### SNS Security

- Encryption
  - In-flight encryption using HTTPS API
  - At-rest encryption using KMS Keys
  - Client-side encryption
- Access Controls: IAM Policies to control access to SNS API
- SNS Access Policies (similar to S3 bucket policies)
  - For Cross account access to SNS topics
  - For allowing other services to write to an SNS topics

## Architectures

### Fan Out pattern architecture

- Scenario: your application wants to write to multiple SQS queues
- Pattern is application publishes to a single SNS
- The SQS Queues can subscribe to the single SNS
- Each application consumer then consumes from it's own SQS Queue.
- Avoids the single application having to know about all the SQS queues, and also failing to send a message to an SQS queue, (e.g. if app crashes) leaving an inconsistent update across queues.
- SQS Queue access policy must allow for SNS to write to it.
- Cross region delivery: SNS can send to SQS in other regions.

### S3 Fan Out pattern architecture

For an event type in S3, e.g. object created event, only 1 rule can be set up to push the event somewhere. If you want to send it to multiple places, you can use the fan out pattern.

- S3 single event rule pushed to SNS topic
- Multiple SNS topic subscribers received the message

### SNS to Amazon S3 though Kinesis Data Firehose

Application -> SNS Topics -> Kinesis DF -> S3 (or any KDF supported destination).

### SNS FIFO Topic

Similar to SQS FIFO.
This is needed if you are using FIFO on the SQS queues, you would have an SNS FIFO instance, it will fan out to SQS FIFO queues.

### SNS Message filtering

- JSON policy to filter messages sent from SNS topic to subscribers
- This is so a subscriber doesn't receive **every** message
- Filter policy goes onto subscriber configuration in SNS, this allows using a single SNS topic, but sending specific messages only to specific SQS queues, e.g. order status messages.

Example policy:

```
{
  "orderStatus": ["shipped", "delivered"]
}
```

Example message with orderStatus:

```
{
  "Message": "Order #12345 has been shipped.",
  "MessageAttributes": {
    "orderStatus": {
      "Type": "String",
      "Value": "shipped"
    }
  }
}
```

## Kinesis Data Streams

Collect and store streaming **real-time** data.

- Click-streams initiated by a user
- IoT Devices
- Metrics and Logs

### Producers

- Applications - take data from app and send in real-time into Kinesis
- Kinesis Agent can do this for some services

- Because we want something to react to the data in real-time: Consumers

### Consumers

- An Application
- Lambda
- Amazon Data Firehose
- Managed service for Apache Flink

### Features

- Data can be retained for up to 365 days
- Can reprocess (replay) data by consumers
- Data can't be deleted from Kinesis until it expires
- Data up to 1MB
- Typically lots of smaller data
- Data order guarantee for data with same **Partition ID**
- Encryption:
  - KMS for at-rest
  - HTTPS for in-flight
- Code Libraries:
  - Kinesis Producer Libray (KPL) - for Producer application
  - Kinesis Client Library (KCL) - for Consumer application

### Capacity Modes

- Provisioned mode:

  - Choose number of shards (related to size of stream)
  - 1 MB/s or 1,000 records p/s per shard
  - 2 MB/s out traffic per shard
  - Scale up to more shards to increase performance
  - Scale in/out number of shards manually
  - Pay per shard per hour

- On-demand mode:
  - You don't provision or manage capacity
  - Default capacity is 4 MB/s, or 4,000 records per second (equivalent for 4 shards)
  - 8 MB/s read/out
  - Scales automatically - using throughput metric peak within last 30 days
    - Max scale is equivalent of 10 shards
  - Pay per stream per hour AND data in/out per GB

### Demo

- Kinesis > Data Streams
- Name
- Capacity modes:
  - On-demand
  - Provisioned
    - Select no. of shards - can use shard estimator tool
- Create - once created:
- Applications menu:

  - Producers:
    - Amazon Kinesis Agent - uses a standalone Java application on application servers to send data to the stream
    - AWS SDK - use Java to develop producers at low level
    - Amazon Kinesis Producer Library (KPL) - to develop producers at a high level with better API
  - Consumers:
    - Amazon Kinesis Data Analytics - process and analyse using SQL or Java
    - Amazon Kinesis Data Firehose - process and store records in a destination
    - Amazon Kinesis Client Libray (KPL) - client library to develop consumer applications
      - Note: When using Kinesis Client Library, each shard is to be read-only by one KCL instance (on an EC2 instance). So if you have 10 shards, the the maximum KCL instances you can have is 10 (so 10 EC2 instances).
  - Monitoring
  - Configuration - Scale
  - Enhanced fan-out - consumers can use the enhanced fan out facility

- Use Terminal or CLI in AWS Console Cloudshell

  - `aws --version` Use version 2
  - Write to stream using `put-record`:
  - V1:

  ```
    aws kinesis put-record \
    --stream-name my-stream-name \
    --partition-key my-partition-key \
    --data "my-payload-data"
  ```

  - V2:

  ```
  aws kinesis put-record \
  --stream-name my-stream-name \
  --partition-key my-partition-key \
  --data "my-payload-data" \
  --cli-binary-format raw-in-base64-out
  ```

  - To Consume, need to get Shard Id when using CLI (handled by SDK if that is used instead):

  ```
  aws kinesis describe-stream \
    --stream-name my-stream-name
  ```

  - To consumer data, create a shard interator:

  ```
  aws kinesis get-shard-iterator \
  --stream-name my-stream-name \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON
  ```

  - Fetch data using shard iterator:

  ```
  aws kinesis get-records \
    --shard-iterator <your-shard-iterator-token> \
    --limit 100
  ```

  - Other --shard-iterator-type options:

    - TRIM_HORIZON: start from the oldest record
    - LATEST: start from the newest record
    - AT_TIMESTAMP: start from a specific timestamp
    - AFTER_SEQUENCE_NUMBER: start after a specific record

  - Example of data returned:

  ```
  {
    "Records": [
      {
        "SequenceNumber": "49590338271490256608559692538361571095921575989136588898",
        "ApproximateArrivalTimestamp": 1633024800.123,
        "Data": "eyJ1c2VySWQiOiAiMTIzIiwgImV2ZW50IjogInNpZ25pbiJ9",
        "PartitionKey": "user-123"
      },
      {
        "SequenceNumber": "49590338271490256608559692538361571095921575989136588899",
        "ApproximateArrivalTimestamp": 1633024801.456,
        "Data": "eyJ1c2VySWQiOiAiNDU2IiwgImV2ZW50IjogInNpZ25vdXQifQ==",
        "PartitionKey": "user-456"
      }
    ],
    "NextShardIterator": "AAAAAAAAAAH6J5Wv...truncated...",
    "MillisBehindLatest": 0
  }
  ```

  - Can contiue to read using created shard iterator but use different shard-iterator type to get continuing data.

## Amazon Data Firehose

A service to send data from sources into target destinations.

- Various producers can push data to Firehose
- Or some services Firehose can pull data (Kinesis Data Streams, CloudWatch, AWS IoT)
- 1 MB per record max
- Data accumulated into a buffer in Firehouse
- Buffer is flushed to send batches of data into target destinations:
  - AWS First:
    - S3, Redshift, OpenSearch **(need to know these)**
  - 3rd party:
    - Datadog, splunk, mongoDB, new relic
  - HTTP Endpoints
    - For when a service is not supported.
- Firehose can use Lamda functions if data transformation is needed
- Option available to write all or failed data into an S3 bucket for a backup.

### Features

- Fully managed
- Automatic scaling, serverless, pay for what you use
- Near real-time with buffering based on size/time, there is a delay because of filling the buffer and waiting for it to be flushed.
- Data formats: CSV, JSON, Parquet, Avro, Raw Text, Binary data
- Convert data to Parquet/ORC, compression with gzip/snappy
- Custom transformation: using Lambda, e.g. change from CSV to JSON, etc

### 📊 Kinesis Data Streams vs Kinesis Data Firehose

| Feature                 | Kinesis Data Streams                              | Kinesis Data Firehose                               |
| ----------------------- | ------------------------------------------------- | --------------------------------------------------- |
| **Use Case**            | Real-time custom processing                       | Near real-time delivery to storage/analytics        |
| **Latency**             | Sub-second                                        | Typically 60 seconds (buffered)                     |
| **Data Retention**      | Up to 365 days (default 24 hours)                 | No retention; data is delivered immediately         |
| **Replay Capability**   | ✅ Yes — reprocess using shard iterators          | ❌ No — data cannot be replayed once delivered      |
| **Consumer Model**      | Pull-based (apps poll for data)                   | Push-based (auto delivery to destinations)          |
| **Destinations**        | Custom apps, Lambda, Analytics, Firehose          | S3, Redshift, OpenSearch, Splunk, HTTP endpoints    |
| **Transformation**      | Custom via consumer apps or Lambda                | Built-in via Lambda preprocessing                   |
| **Scaling**             | Manual (shard-based) or On-Demand mode            | Automatic                                           |
| **Throughput Limits**   | Shard-based: 1 MB/s or 1,000 records/s per shard  | Up to 5,000 records/s or 5 MB/s per delivery stream |
| **Ordering Guarantees** | Per shard                                         | No strict ordering                                  |
| **Encryption**          | Server-side (KMS), client-side supported          | Server-side (KMS)                                   |
| **Monitoring**          | CloudWatch metrics, enhanced monitoring optional  | CloudWatch metrics                                  |
| **Pricing Model**       | Based on shard hours and PUT payloads             | Based on data volume and transformation             |
| **Setup Complexity**    | Requires stream management and consumer logic     | Simple setup, minimal configuration                 |
| **Ideal For**           | Custom stream processing, analytics, ML pipelines | ETL pipelines, log delivery, data lake ingestion    |

### Demo Firehose

- Amazon Kinesis Data Firehose
- Source:Data streams
- Destination: S3
- Browse and choose Stream (from previous demo)
- Transform: can setup using lamda functions
- Convert record format into Parquet or ORC
- Destination settings:

  - S3 bucket - created or create one
  - Dynamic partitioning
  - S3 bucket prefix (optional) - default is a data format - can be overriden
  - S3 bucket error output prefix (optional) - for errors (/errors)
  - S3 buffer hints - size of buffer before flushing
    - Buffer size, default 5MiB, 1MiB to 128MiB.
    - Buffer interval, how fast to flush if buffer doesn't fill up, default 300 seconds, but 60 to 900 seconds
  - S3 compression and encryption - e.g., gzip message into target to safe space
    - Option to encrypt records
  - Permissions:
    - Create or select IAM role for Data Firehose to write data into S3 bucket

- Menus tabs
- Monitoring
- Config
- Logs
- Send data to the Data Stream from demo 1 using the CLI
  - Data previously send to the Stream will not go into Firehose, only new data while Firehose is active
  - Need to wait for buffer to flush and then object will appear in S3 bucket.

## Apache Flink

- Framework used for processing data streams (Java, Scala or SQL)
- Managed service
- Two sources feed into it:
  - Kinesis Data Streams
  - Amazon MSK (Apache Kafka)
  - **Does NOT read from Amazon Data Firehose**
- Managed service, runs on a managed cluster; compute resource with auto-scaling, parallel computation, AWS manages backups (checkpoints/snapshots)
- Use any Flink feature to transform data

## SQS vs SNS vs Kinesis

| SQS                                             | SNS                                                  | Kinesis                                         |
| ----------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------- |
| Consumer "pull data"                            | Push data to many subscribers                        | Standard: pull data, 2MB per shared             |
| Can have as many workers (consumers) as we want | Up to 12,500,000 subscribers                         | Enhanced-fan out: push data 2MB per shard       |
|                                                 | Up to 100,000 topics                                 |                                                 |
| Data is deleted after being consumed            | Data is not persisted (lost if not delivered)        | Possibilty to replay data                       |
|                                                 |                                                      | Data expires after X days                       |
|                                                 | Pub/Sub                                              | Meant for real-time big data, analytics and ETL |
| No need to provision throughput                 | No need to provision throughput                      | Provided mode or on-demand capacity mode        |
| Ordering guarantees only on FIFO queues         | FIFO capability for SQS FIFO                         | Ordering at the shard level                     |
| Individual message delay capability             | Integrates with SQS for fan-out architecture pattern |                                                 |
