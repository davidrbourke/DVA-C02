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
