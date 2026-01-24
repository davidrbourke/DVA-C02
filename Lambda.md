# Lambda

## Serverless

You still have servers, but you don't have to manage them.
Anything remotely managed. **You** don't provision them.

- Lambda
- DynamoDB
- AWS Cognito
- API Gateway
- S3
- SNS & SQS
- Kinesis Data Firehose
- Aurora Serverless
- Step functions
- Fargate

## Why Lambda

- Virtual functions - no servers to manage
- Short executions - up to 15 mins
- Runs on demand - not paying when function is not running.
- Scaling - automated - AWS provisions more functions
- Easy pricing
  - pay per request and compute time
  - Free tier:
    - 1 million request, and:
    - 400,000 GB of compute time, e.g., this is 400,000 seconds of per 1 GM of RAM, so using less RAM is more time, e.g. using 500MB == 800,000 seconds.
  - After free used up:
    - $0.20 per 1 million requests
    - $1.00 for 60,000 GB of RAM seconds
- Integrated with whole of AWS suite of services, can trigger the Lambda from events, e.g.:
  - API Gateway - create a REST API to invoke Lambda func
  - Kinesis - data transformations on fly
  - DynamoDB - trigger from DB events
  - S3 - trigger from e.g file created
  - CloudFront
  - CloudWatch Events EventBridge
    - React to AWS infra changes
    - Serverless Cronjobs triggered from EventBridge
  - CloudWatch Logs - stream logs
  - SNS - react to events
  - SQS - process messages
  - Cognito - react to user logging into DB
- Many programming languages:
  - Node.js
  - Python
  - Java
  - C#
  - Ruby
  - Custom Runtime API (community supported, e.g Rust, Golang)
  - Lambda Container Image
    - Container must implement the Lambda Runtime API
    - ECS/Fargate preferred for running arbitrary Docker images
- CloudWatch monitoring
- Can provision up to 10GB of RAM per function
  - Also increased CPU and Network

## Demo 1 - Intro

- Create function
  - Author from scratch
  - Use a blue print (\*)
  - Container image
- Function name
- Runtime, e.g. Python
- Execution role
  - Create new role
  - Use existing
  - Create a new role from AWS policy templates
- Function code
- Create
- Test function - uses sample Json input
  - Can save test event - to re-trigger same test event
- CloudWatch
  - Can view the Log group and logs from the Lambda job
- Configuration
  - Memory
  - Ephemeral storage
  - Timeouts
  - Execution role
    - The basic Lambda role, allows Lambda to write to CloudWatch logs
    - Update this role to add more permission, e.g. connect to S3 bucket
- Trigger configuration
  - Setup all the sources of events that can trigger the Lambda function

## Synchronous Invocations

- Waiting for the result once you invoke the Lambda
- Errors must be handled on the client side, e.g. retry, etc
- Synchronous services
  - User invoked:
    - Elastic Load Balancing, e.g ALB
    - API Gateway
    - CloudFront
    - S3 Batch
    - CLI
  - Service invoked:
    - Cognito
    - Step Functions
    - Lex
    - Alexa
    - Kinesis Data Firehose

### CLI invocation - synchronous

`aws lambda list-functions --region eu-west-1`

`aws lambda invoke --function-name demo-lambda --cli-binary-format raw-in-base64-out --payload '{\"key1\": \"value\"}' --region eu-west-1 response.json`

### Lambda ALB Integration

Expose the Lambda to the internet for HTTP invocation

- ALB or API Gateway
- Register Lambda function into target group
  - Client -> ALB -> TargetGroup (with Lambda)
- HTTP request gets converted to JSON, data available:
  - http method
  - query string parameters
  - headers
  - body
  - isBase64Encoded

```
{
    "requestContext": {
        "elb": {
            "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/lambda-279XGJDqGZ5rsrHC2Fjr/49e9d65c45c6791a"
        }
    },
    "httpMethod": "GET",
    "path": "/lambda",
    "queryStringParameters": {
        "query": "1234ABCD"
    },
    "headers": {
        "accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8",
        "accept-encoding": "gzip",
        "accept-language": "en-US,en;q=0.9",
        "connection": "keep-alive",
        "host": "lambda-alb-123578498.us-east-1.elb.amazonaws.com",
        "upgrade-insecure-requests": "1",
        "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36",
        "x-amzn-trace-id": "Root=1-5c536348-3d683b8b04734faae651f476",
        "x-forwarded-for": "72.12.164.125",
        "x-forwarded-port": "80",
        "x-forwarded-proto": "http",
        "x-imforwards": "20"
    },
    "body": "",
    "isBase64Encoded": False
}
```

- Response gets convert from JSON back to HTTP response:
  - statusCode
  - statusDescription
  - headers
  - body
  - isBase64Encoded

#### ALB Multi-Value Headers

- Allows the querystring parameters in HTTP request to be repeated, instead of keeping just one, all can be kept, e.g.
- `http://example.com/path?name=foo&name=bar` converts to JSON array:
- `"queryStringParameters":{"name":["foo", "bar"]}`
- If ALB Multi-Value headers are not enabled in the ALB, then just one of the parameters with the repeated name will be chosen.
- Can enable this at the Target Group level.

## Demo 2 - ALB + Lambda

- Create the Lambda as per Demo 1
- Create the Load Balancer (ALB)
  - ALB Listener to target a new Target Group
  - Create the new Target Group and choose target Type: **Lambda function**
  - Choose the Lambda function
  - Create the Target Group
  - Apply the Target Group to the Listener in the ALB
- You can request to the URL of the ALB in the browser
  - By default this will return a file to be downloaded with the response from the Lambda
  - To return a response to be displayed in the browser instead, need to return the correct response with correct headers, cannot just return the HTML:

```
{
    "statusCode": 200,
    "statusDescription": "200 OK",
    "isBase64Encoded": False,
    "headers": {
        "Content-Type": "text/html"
    },
    "body": "<h1>Hello from Lambda!</h1>"
}
```

- In CloudWatch you can look at the Lambda invocation history to see the request event parameters, headers, etc.
- The ALB TargetGroup must have permission to access the Lambda, example Resource Policy added to the Lambda:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowALBInvocation",
      "Effect": "Allow",
      "Principal": {
        "Service": "elasticloadbalancing.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:myLambdaFunction",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-target-group/abc123"
        }
      }
    }
  ]
}
```

## Asynchronous Lambda Invocations

Some services must use Asynchronous, or business requirement where you don't need to wait for a response immediately.

- Services, e.g.
  - S3
  - SNS
  - CloudWatch Events / EventBridge
  - _Many others not needed for exam_
- Example:
  - SQS Queue Events
  - Lambda reads
  - On failure Lambda will retry
    - Define a DLQ to prevent endless retry
- Can also be invoked from CLI if not returning result - just add the `--invocation-type Event`:

```
aws lambda invoke \
  --function-name demo-lambda \
  --cli-binary-format raw-in-base64-out \
  --payload '{"key1": "value"}' \
  --invocation-type Event \
  --region eu-west-1 \
  response.json
```

You will get an HTTP Status code to confirm the request submission status, but this is not the result of the Lambda function itself.

### Dead Letter Queue in Lambda

To prevent endless retry on Asynchronous lambda errors, setup a DLQ:

- In Lambda Console
- Edit **Asynchronous Configuration**
  - Retries
    - Maximum age of event: Max amount of time to keep unprocessed events in the queue
    - Retry attempts: number of times to retry the function on error
  - Dead-Letter Queue, choose one of:
    - SNS
    - SQS
- The SQS or SNS must be setup separately
- Lambda Execution Role policy must be updated to be allowed to write into the SQS Queue
  - e.g.`AmazonSQSFullAccess` just for demo
  - This can be edited/setup via the Lambda Console, but the policy is applied to the SQS itself.

## CloudWatch Events/Event Bridge for Lambda Integration

Two ways:

- CRON or Rate EventBridge Rule to trigger on schedule to invoke Lambda
- CodePipeline EventBridge Rule to trigger on state changes to invoke Lambda

### Demo 3

- Create the Lambda
- In EventBridge Console
  - Create a new Rule
  - Name
  - Event Bus - default
  - Rule type
    - Rule with event pattern
    - Schedule (\*)
  - Define Scheduled pattern, e.g. Cron or regular
  - Choose target: Lambda
    - Choose the specific Lambda
- In Lambda Permissions
  - A Resource Policy will have been created to allow the EventBridge to invoke the function
- In the CloudWatch logs, you can view the invocation of the Lambda by the Event Cron schedule

## Lambda & S3 Event Notifications

Get notified whenever an object is created/deleted, etc, in the S3 Bucket.
Some patterns:

- S3 -> Lambda -> DLQ -> SQS (S3 straight to Lambda)
  - Notification can take seconds to a minute
  - Enable versioning on the S3 bucket objects to ensure all writes trigger events, because two writes happening in rapid succession might mean only 1 event is raised instead of 2.
- S3 -> SNS -> SQS
- S3 -> SQS - Lambda

### Demo 4 - S3 to Lambda

- Create a new Lambda function
- Create an S3 bucket
- In Bucket properties
  - Event Notification Properties
  - Create
    - Event name
    - Prefix: /
    - Event types (choose all object create event)
    - Destination
      - Lambda function (\*)
        - Choose from your Lambda functions, or:
        - Enter Lambda function ARN
      - SNS topic
      - SQS queue
- Lambda resource policy will have been updated to allow S3 bucket to invoke the function
- In the CloudWatch logs, you can view the invocation of the Lambda by the S3 object event
  - The event data passed to the Lambda includes data about the object, e.g. name, etc. You can use the data to call the S3 bucket to get the object and do something with it.

## Lambda - Event Source Mapping

Some sources cannot invoke the Lambda directly, so Event Source Mapping is used. This is when the Event Source Mapping is setup to poll the source for items, and trigger the Lambda. Example would be using SQS, SQS cannot trigger the Lambda, so Event Source Mapping polls the SQS Queue for messages, on finding messages, it triggers the Lambda, passing the messages into the Lambda.

- Used when records need to be **polled** from the source
- Lambda invoked **synchronously**
- Uses with services:
  - Kinesis Data Stream
    - Example:
      - Event Source Mapping polling setup internally
      - Polls Kinesis for data
      - When returns a data batch, Lambda is triggered synchronously with an event batch
  - SQS and SQS FIFO Queues
  - DynamoDB Streams
- Two ways to use Event Source mapping; **Streams** and **Queues**
- **Streams**
  - Used with Kinesis & DynamoDB
  - Event source mapping creates an iterator for each shard, processes items in order
  - Iterator starts from beginning or a timestamp
  - Processed items are not removed from the Stream, other consumers can read them
  - Uses:
    - Low traffic: wait for records to build up and batch process them
    - High traffic
      - Process multiple batches in parallel:
        - Parallelisation: Up to 10 batches per shard processed simultaneously
        - Normal: 1 Lambda invocation per shard
      - In-order processing is guaranteed for each partition key
  - Errors
    - An error in a batch, entire batch is reprocessed until success or batch items expire, can become blocking.
    - You can pause processing for the shared on error, by configuring Event source mapping to:
      - discard old events - events can go to another destination
      - restrict the number of retries
      - split the batch on error (e.g. Lambda is timing out processing entire batch)
- **Queues**
  - SQS and SQS FIFO
    - Queue is pulled by Event source mapping
    - When there is data, Lambda is invoked with the batch of data
  - Long Polling is used
  - Configure batch size (1-10 messages)
  - Set the queue visibility timeout to 6x the Lambda function timeout (recommended)
  - You cannot use the DLQ in Lambda for failures, because that is for Asynchronous Lambdas, and Event source mapping Lambdas are Synchronous.
    - So you can setup the DLQ in SQS
  - Failures to go to another destination
  - Lambda supports FIFO processing for FIFO queues
    - SQS FIFO Scaling
      - Messages with the same GroupID will be processed in order, so no parallel execution for messages sharing the same GroupID.
      - Lambda function scales to the number of active message groups - still a max overall of 1000 concurrent processing (e.g. 1000 Message GroupIDs)
  - Standard queue items are not processed in order
  - Lambda scales to process messages as fast as possible in a **standard** queue
    - **Lambda adds 60 more instances per minute to scale up**
    - Max: up to 1000 batches of messages processed simultaneously
  - Error in a batch, entire batch is returned to the Queue
    - Messages may be processed again in a different batch
  - Use Idempotent processing for the Lambda as the same message may be pulled from the queue twice
  - Lambda deletes items from the queue after they're processed successfully

### Demo - Event source mapping - using SQS Queue

- Create Lambda
- Author from Scratch
- Create an SQS Queue separately - type Standard
- Configure Lambda
  - Add trigger
    - SQS - choose the created Queue
    - Configure Batch size: 1 to ?
    - Batch window - amount of time in seconds to gather messages before invoking function
    - Enable trigger
  - Need to configure IAM Role so Lambda has permission to read from the SQS by attaching `AWSLambdaSQSQueueExecutionRole` permission to the IAM Policy for the Lambda.
  - Update the code in the function - print event and return "success"
- To Test
  - Send a message to SQS via the SQS test send message console page
  - Lambda is continuously polling the SQS, so should pick up the message
  - CloudWatch log events will show the data returned and the invocation working
  - Message in Queue will be removed as now processed by the Lambda function
- **Disable the trigger** in the Console - to prevent cost and polling

- Kinesis - if using Kinesis you would choose the trigger as Kinesis
  - Select the Data stream
  - Consumer - if using a fan out consumer mode in kinesis you can choose the consumer to listen for updates on.
  - Batch size: 100 default, number of records to read at once
  - Batch window: wait time to build bigger batch (5 mins max)
  - Starting position - where to start reading the data stream from:
    - Latest
    - Trim horizon
    - At timestamp
  - On-failure destination - destination to send discarded records that could not be read
  - Retry attempts: -1 default, times to retry the function on error.
  - Max age of record: -1 default no limit, Max up to 7 days, max age of record Lambda sends to a function for processing
  - Split batch on error (checkbox) - if function returns an error, split the batch into two and retry.
  - Concurrent batches per shard: default 1 - process batches from the same shard concurrently.
  - Tumbling window duration: 0-900 (15 mins max), time window for an aggregation
  - Report batch item failures (checkbox) - allow function to return a partial successful response for a batch of records.
  - Enable trigger option

## Event and Context Objects

These are the two types of data send to the Lambda.

- Event
  - JSON document contains data the function is going to process
  - E.g. invoking service, SQS, EventBridge, etc
  - Event is converted into an Object in your chosen language
- Context
  - Data and methods that provide information about the Lambda itself, and runtime environment.
  - E.g. aws_request_id, function_name, memory_limit_in_mb

```
import json

def lambda_handler(event, context): # Log context information
print(f"Function name: {context.function_name}")
print(f"Memory limit (MB): {context.memory_limit_in_mb}")
print(f"AWS request ID: {context.aws_request_id}")
print(f"Log stream name: {context.log_stream_name}")

    # Process each record in the event (SQS messages come in batches)
    for record in event['Records']:
        print("Message ID:", record['messageId'])
        print("Message Body:", record['body'])

        # You can parse the message body if it's JSON
        try:
            body = json.loads(record['body'])
            print("Parsed Body:", body)
        except json.JSONDecodeError:
            print("Message body is not valid JSON")

    return {
        'statusCode': 200,
        'body': json.dumps('Messages processed successfully')
    }

```

## Lambda Destinations

For failed result of an Asynchronous invocation or Event mapper Lambda, the event data can be sent to a destination (also can setup for successful Lambda).

- Asynchronous destinations:
  - SQS
  - SNS
  - Another Lambda
  - EventBridge Bus
- Event source mapping destinations:
  - SQS
  - SNS
- Failed events will be sent to the destination **after** the Lambda retry attempts have all failed.

For SQS source, favour Destinations over DLQ, as more target destinations.

### Demo - Destinations

- On Lambda, Add Destination configuration
- Source
  - Asynchronous invocation (use pre-created S3 bucket)
  - Stream invocation (e.g. Kinesis)
- Condition
  - On failure
  - On success
- Destination type, e.g. SQS
- Destination - choose pre-created SQS
- IAM Role of Lambda gets Permission added to policy called `AWSLambdaSQSQueueDestinationExecutionRole`
- Can add multiple destinations to handle success and failure as well.

## Lambda Permissions

When Lambda is executing (read/update/uploading) on other services, it needs an IAM Role with permission to those services, here are some sample policies:

- `AWSLambdaBasicExecutionRole` - upload logs to CloudWatch
- `AWSLambdaKinesisExecutionRole` - read from Kinesis
- etc, other for various services

- When using Event Source Mapping, Lambda uses the execution role to read event data.
- Best practice: 1 Lambda Execution per function
- When the Lambda function is invoked by other services, those services need permission to the Lambda.
  - IAM Principal can access Lambda:
    - if the IAM Policy attached to the Principal authorises it (e.g. User access)
    - or if the resource-based policy authorises it (e.g. service access)

Example of resource policy on the Lambda to allow an S3 bucket events to invoke the function:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3InvokeLambda",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyLambdaFunction",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:s3:::my-source-bucket"
        }
      }
    }
  ]
}
```

Example of a policy for Lambda to access Kinesis data stream - after invocation from Event source mapping:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListStreams"
      ],
      "Resource": "arn:aws:kinesis:us-east-1:123456789012:stream/my-kinesis-stream"
    }
  ]
}
```

## Lambda Environment Variables

- Key value pair (string)
- Lambda service adds its own variables
- Can add your own
- Can add them in KMS to encrypt them
  - Encrypt with own key or Lambda service key

Example of using a environment variable, you can add the key-value pair in the Environment variables Console for the Lambda (if not encrypting).

```
import os

def lambda_handler(event, context):
    # Retrieve the environment variable
    my_env_var = os.environ.get('MY_ENV_VAR', 'default_value')

    print(f"Environment variable MY_ENV_VAR is: {my_env_var}")

    return {
        'statusCode': 200,
        'body': f"Env var value: {my_env_var}"
    }
```

## Lambda Logging and Monitoring

- CloudWatch Logs - as seen - shows data of each event triggering the Lambda, and errors, outputs, console logs.
- CloudWatch Metrics, e.g. invocations, duration, error count, error/success rate, throttles( going over limits), async delivery failures, concurrent executions.
- X-Ray tracing: Enable it as Configuration - Monitoring tools - AWS X-Ray: Active tracing, in the Lambda
  - AWS runs the X-Ray Daemon for you
  - Your code must be using the AWS X-Ray SDK
  - Must have correct policy to write to X-Ray `AWSXRayDaemonWriteAccess`
  - Environment variables to communicate with X-Ray (can be in exam):
    - `_X_AMZN_TRACE_ID` - contains the tracing header
    - `AWS_XRAY_CONTEXT_MISSING` - by default, LOG_ERROR
    - `AWS_XRAY_DAEMON_ADDRESS` - the X-Ray Daemon IP_ADDRESS:PORT

## Customisation at the Edge

- Edge Functions:
  - Cloud that runs at the CloudFront Edge distributions
  - Runs close to users to minimize latency
  - Runs before request is passed onwards to destination
- Two types:
  - **CloudFront Functions**
    - Lightweight function in Javascript
    - High scale latency sensitive customizations, change response or request between CloudFront and the Origin.
    - High performance: Sub millisecond startup, millions of requests per second
    - Native to CloudFront, code managed in CloudFront
    - Max execution time: < 1ms
    - Max memory: 2MB
    - No network/file system access
    - Max package size: 10KB
    - No Access to request body
    - Uses:
      - Cache key transformations, e.g. transform URL, query string, etc.
      - Header manipulation
      - URL redirect/rewrites
      - Auth, create/invalidate JWT, allow/deny
  - **Lambda@Edge**
    - Function in Node.js or Python
    - 1000s of request per second
    - Can change request or response between CloudFront and the Origin
    - **Note: can also change request or response between the Viewer and CloudFront**
    - Author function in one region, and CloudFront replicates it in all locations required
    - Max execution time: 5-10 seconds
    - Max memory: 128MB to 10GB
    - Has network/file system access
    - Max package size: 1MB to 50MB
    - Has Access to request body
    - Uses:
      - Adjustable CPU/Memory
      - When you need network/file access
      - When you have longer processing
      - When you need to access the request body
- Uses, some examples:
  - Customise CDN content
  - Website security and privacy
  - SEO
  - Bot mitigation at the edge
  - Real-time image transformation
  - User authentication and authorisation
- Pay only for what you use
- Serverless

## Lambda in VPC

Lambda is deployed in an AWS managed VPC.

- It can access other AWS Services that can be accessed publicly, e.g. Dynamo DB
- It can access the internet
- It **cannot** by default, access your own VPC and service deployed in it, e.g. Private RDS
  - Solution: deploy Lambda in your Private VPC

### Deploy Lambda in your VPC

- You need to provide the Lambda VPC ID, Subnets and Security Groups
- Lambda will create an ENI in your subnets
  - For this, Lambda needs `AWSLambdaVPCAccessExecutionRole`
- **IMPORTANT**:
  - Lambda function in your VPC does not have access to the internet
  - Deploying a Lambda function in a public subnet does not give it internet access or a public IP
- To access the internet from your VPC:
  - Deploy Lambda in private subnet with Private RDS
  - Create a Public subnet
    - Create NAT Gateway/Instance in the Public subnet
    - Create Internet Gateway in Public subnet
    - Allow NAT access to private subnet via Security Group in the private subnet
  - IGW | Public Subnet: NAT | -> | Private Subnet: Security Group, Private RDS |
  - To access DynamoDB outside of the VPC
    - IGW can access DynamoDB via the internet
    - Or add a VPC Endpoint to private access AWS Services

### Demo

1. Create Lambda function
2. In default VPC create a security group
3. In Lambda function, VPC menu option - Choose VPC - choose default VPC
   - Choose VPC Subnets (all AZs)
   - Security groups - choose the created SG (step 2)
     - Saving will Fail, cannot create NAT in VPC
     - Provide Lambda function with permission to access the VPC, attach Policy to role: `AWSLambdaENIManagementAccess`
     - Save - will save
4. Test
5. Look in EC2 VPC, you will see new Network Interfaces for each Subnet

## Lambda Performance and Configuration

- RAM
  - 128MB to 10GB, scales in 1MB increments
  - More RAM => more vCPU credits
  - Cannot set vCPU directly, must increase RAM to get more vCPU.
  - 1792 MB == 1 vCPU
  - > 1792 MB can make use of multithreading (parallelism)
- Timeout
  - Default: 3 seconds
  - Max: 900 seconds (15 minutes)

## Execution Context

Temporary runtime environment

- E.g. setup DB Connections, HTTP Client, etc, long running tasks.
- Next invocation can reuse a context as it hang around between executions, AWS don't guarantee how long, mins to hours.
- Includes a /tmp directory, 10GB of disk.

### Pattern

- Initialise DB function outside of the Lambda function handler code.
- Use /tmp directory to share files for reuse - /tmp is ephemeral.
- To encrypt /tmp, use KMS Data Keys, and handle encrypting/decrypting in Lambda function code, there is nothing built-in to Lambda for this.

```
import psycopg2
import os

# Global variable: persists across warm invocations
conn = None

def lambda_handler(event, context):
    global conn

    if conn is None:
        # Cold start: create DB connection
        print("Creating new DB connection...")
        conn = psycopg2.connect(
            host=os.environ['DB_HOST'],
            database=os.environ['DB_NAME'],
            user=os.environ['DB_USER'],
            password=os.environ['DB_PASSWORD']
        )
    else:
        # Warm start: reuse existing connection
        print("Reusing existing DB connection...")

    # Use the connection
    with conn.cursor() as cur:
        cur.execute("SELECT NOW();")
        result = cur.fetchone()

    return {
        'statusCode': 200,
        'body': f"Current time from DB: {result[0]}"
    }
```

Use - for reuse between executions:

- /tmp for any large files
- context for long running processes, e.g. creating DB connection
- In .NET, create the connection in the ctor and assigned to a private static class variable. This must be done outside of the Handler method.

## Lambda Layers

- Custom runtimes: You can use Layers to run code in languages not natively supported by Lambda, like C++ or Rust.
- Dependencies: Layers let you package external libraries separately from your function code. This avoids repackaging unchanged dependencies with every deployment.
  Note: Layers simplify deployment and reuse, but they don’t improve execution speed or reduce cold start time.

### Demo

- In Lambda console, Layers menu, choose a layer:
- AWS Layers - adds a library prepared by AWS already, e.g. Python Panda used in the demo.
- Custom Layers
- Specify ARN

## Lambda File System Mounting

- Can mount an EFS if in the same VPC as the Lambda
- Note: each Lambda creates a new connection to the EFS so be aware of potentially hitting total connection limits on the EFS, or burst limits if lots of Lambdas spin up quickly.

## Storage Options

## AWS Lambda Storage Options Comparison

| Storage Type  | Max Size                        | Persistence         | Access Speed                    | Cost            | Use Case Examples                                     | Notes                                                                 |
| ------------- | ------------------------------- | ------------------- | ------------------------------- | --------------- | ----------------------------------------------------- | --------------------------------------------------------------------- |
| `/tmp`        | 10 GB (default)                 | Ephemeral           | Very fast (local)               | Included        | Temporary file storage, caching, intermediate results | Can request >10 GB via AWS support; cleared after each container ends |
| Lambda Layers | 250 MB per layer (max 5 layers) | Durable (read-only) | Fast (mounted)                  | Included        | Shared libraries, custom runtimes, static assets      | Mounted at `/opt`; total combined size limit: 1.25 GB                 |
| Amazon S3     | Virtually unlimited             | Durable             | Fast (network)                  | Additional cost | Large file storage, input/output data, backups        | Requires SDK/API access; eventual consistency                         |
| Amazon EFS    | Up to petabytes                 | Durable             | Very fast (network file system) | Additional cost | Shared file systems, ML models, persistent state      | Mounted at `/mnt/efs`; supports concurrent access across functions    |

## Lambda Concurrency and Throttling

- Max 1,000 concurrent functions - can request more from AWS with support ticket
- Reserved concurrency: limit of concurrent invocations
  - Triggers a Throttling error when the limit is reached
  - Synchronous function errors
  - Async function - retries after 1 sec, with exponential backoff, up to max of 5 mins between retries.
- 1,000 limit applies to ALL functions in the account, so another function reaching the limit will impact other functions trying to run.
- A limit can be configured on the function itself.

## Cold starts

New instance of a function takes time to start up, e.g. seconds.

## Provisioned Concurrency

- Concurrency is allocated before function is invoked, so a cold start does not happen.
- Application auto-scaling can manage concurrency (scheduled or target utilisation).

### Demo Config

- Qualifier Types: Alias or Version
- Provisioned Concurrency: this is the number of concurrency functions to have ready.

## Reserved vs Provisioned Concurrency

## Reserved vs Provisioned Concurrency in AWS Lambda

| Feature               | Reserved Concurrency                             | Provisioned Concurrency                    |
| --------------------- | ------------------------------------------------ | ------------------------------------------ |
| Purpose               | Limit and guarantee max concurrent executions    | Pre-warm environments to avoid cold starts |
| Cold Start Prevention | ❌ No                                            | ✅ Yes                                     |
| Guarantees Capacity   | ✅ Yes                                           | ✅ Yes                                     |
| Acts as Throttle      | ✅ Yes                                           | ❌ No                                      |
| Cost                  | No additional cost                               | Additional charges apply                   |
| Use Case              | Control concurrency, protect downstream services | Low-latency, predictable workloads         |

## Lambda Function Dependencies

- Dependencies need to be packaged and zipped and uploaded to the Lambda
- Example: Node.js node_modules is required to be uploaded
- <= 50MB goes to Lambda
- > 50MB goes to S3 bucket
- Native libraries (e.g. C, C++ Binaries) work, they need to be compiled on Amazon Linux
- AWS SDK already included in Lambda functions

## Lambda and CloudFormation

### Inline

For simple use cases without dependencies

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple Lambda Function with Inline Code

Resources:
  MyLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: SimpleInlineLambda
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: arn:aws:iam::123456789012:role/lambda-execution-role  # Replace with your IAM role ARN
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return {
                  'statusCode': 200,
                  'body': 'Hello from Lambda!'
              }
      MemorySize: 128
      Timeout: 3
```

### CF in S3

- Upload the zip to S3 (enable versioning in S3)
- In CF template reference the S3 bucket

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Lambda Function using S3 with Object Version

Resources:
  MyLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: S3VersionedLambda
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: arn:aws:iam::123456789012:role/lambda-execution-role  # Replace with your IAM role ARN
      Code:
        S3Bucket: my-lambda-code-bucket                         # Replace with your bucket name
        S3Key: lambda-code/my-function.zip                      # Replace with your object key
        S3ObjectVersion: 3H4k9xZlT8a1b2c3d4e5f6g7h8i9j0k1        # Replace with your actual object version
      MemorySize: 128
      Timeout: 3
```

## Lambda Container Images

- Deploy image up to 10GB for ECR
- The Base image must implement the **Lambda Runtime API**
  - Base images exist already for multiple languages
  - Can implement your own image
- Test the container locally using the **Lambda Runtime Interface Emulator**
- Deploy your image to ECR

```
FROM public.ecr.aws/lambda/provided:al2

# Install Python and dependencies
RUN yum install -y python3 && \
    pip3 install --upgrade pip

# Copy your function code and bootstrap script
COPY app.py bootstrap /var/runtime/

# Make bootstrap executable
RUN chmod +x /var/runtime/bootstrap

# Set the entrypoint
CMD ["/var/runtime/bootstrap"]
```

### Best Practice

- Use multi-stage build, so final image is small/simple.
- Use AWS provided built images
- Build stages from most frequently changing to least (caching)
- Use a single repository for functions with large Layers

## Lambda Versions and Aliases

### Versions

- $LATEST version for development
- Create a version when ready to publish - UI Publish - set Versions
- Immutable - cannot change assigned version for a function
- Code version is fixed, gets a unique ARN
- Each version can be accessed and used

### Alias

- Pointers to different Lambda **versions**
- Mutable, can change which version is pointed to.
- Cannot reference other Aliases
- Example
  - dev alias to alpha version
  - test alias to beta version
  - prod alias to v1
- Aliases can enable Canary deployment
  - Split weighting of Alias to 2 different **versions** using a percentage.

## Lambda and CodeDeploy

- CodeDeploy can automate traffic shift for **Aliases**
- E.g. shifting the weighting for the target version in the Alias during a Canary deployment

### Strategies

- **Linear**: grow traffic every N minutes until 100%
- **Canary**: try X percentage, then after time, switch to 100%
- **All At Once**: immediate 100%

#### Rollback

- Can create pre-post hooks to rollback on failure
- Uses **AppSpec.yml**

```
version: 0.0
Resources:
  - MyLambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: my-lambda-function-name
        Alias: live
        CurrentVersion: 1
        TargetVersion: 2
```

## Lambda Function URL

- Expose a function without using other services, use a function URL.
- Creates a fixed URL, any HTTP client can access it.
- Only works with Public internet, never private.
- Supports CORS
- Target an Alias, or ONLY the $LATEST version.
- Throttle using the Reserved Concurrency.

### Function URL Security

- Resource based policy
  - Authorised accounts
  - CIDR Ranges
  - IAM Principal
- CORS
  - When domains are different, to allow browser to use the Lambda response.

### AuthType: None

- Allows Public unauthorised access
- Uses Resource based policy granting public access

### AuthType: IAM

- IAM is used to authorise and authenticate requests
- Both principals Identity based policy & resource based policy are evaluated
- Same account: IBP OR RBP as Allow
- Cross account: IBP AND RBP as Allow

## CodeGuru

- Provides insights into your functions
- Creates a Profile group in the Lambda functions, needs permission to do it `AmazonCodeGuruProfilerAgentAccess`.
- Supports runtimes: Java, Python.
- Activated from within the AWS Console.

## Lambda Limits - Review

- Per region, per account

### Execution Limits

- Memory 128MB to 10GB (increases in 1MB increments).
- Max time: 900 seconds (15 minutes)
- Environment Variables: Max 4KB total for all vars.
- /tmp: 512MB to 10GB
- Concurrency: 1000 max

### Deployment Limits

- Lambda function deployment size
  - Compressed zip: 50MB max
  - On un-compress: 250MB max
- Environment variables: Max 4KB
