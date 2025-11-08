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
- Can provision up to 10GM of RAM per function
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
    - Batch window - amount of time in seconds to gather messages be invoking function
    - Enable trigger
  - Need to configure IAM Role so Lambda has permission to read form the SQS by attaching `AWSLambdaSQSQueueExecutionRole` permission to the IAM Policy for the Lambda.
  - Update the code in the function - print event and return "success"
- To Test
  - Send a message to SQS via the SQS test send message console page
  - Lambda is continuously polling the SQS, so should pick up the message
  - CloudWatch log events will show the data returned and the invocation working
  - Message in Queue will be removed as now processed by the Lambda function
- **Disable the trigger** in the Console - to prevent cost and polling

- Kinesis - if using Kinesis you would chose the trigger as Kinesis
  - Select the Data stream
  - Consumer - if using a fan out consumer mode in kinesis you can choose the consumer to listen for updates on.
  - Batch size: 100 default, number of records to read at once
  - Batch window: wait time to build bigger batch (5 mins max)
  - Starting position - where to start reading the data stream from:
    - Latest
    - Trim horizon
    - At timestamp
  - On-failure destination - destination to send discard records that could not be read
  - Retry attempts: -1 default, times to retry the function on error.
  - Max age of record: -1 default no limit, Max up to 7 days, max age of record Lambda sends to a function for processing
  - Split batch on error (checkbox) - if function returns an error, split the batch into two and retry.
  - Concurrent batches per shard: default 1 - process batches from the same shard concurrently.
  - Tumbling window duration: 0-900 (15 mins max), time window for an aggregation
  - Report batch item failures (checkbox) - allow function to return a partial successful response for a batch of records.
  - Enable trigger option
