# Monitoring

- CloudWatch
  - Metrics
  - Logs
  - Events - send notifications when certain events happen in AWS
  - Alarms - react to real-time metrics/events
- X-Ray
  - Troubleshooting app performance and errors
  - Distributed tracing of microservices
- CloudTrail
  - Monitoring of API calls internally
  - Audit changes made to AWS resources by users

## CloudWatch

- Provides metrics for every service in AWS, name of metric is descriptive.
- Metrics belong to namespaces (e.g., the service type, EC2, S3,etc. Or a custom name)
- Dimension - up to 30 attributes per metrics - additional data with the metric
- Metrics have timestamps
- Dashboards of metrics can be created in CloudWatch

### Example EC2 Detail monitoring

- EC2 instance metrics default every 5 mins
- Can upgrade (cost) for every 1 min
- AWS Free Tier: 10 detail monitoring metrics
- EC2 Memory metrics must be added to be pushed as a custom (not default)

### Custom Metrics

- E.g. RAM, Disc space, logged in users
- Use API call `PutMetricData`
- Use dimensions/attributes, e.g. Instance.id, Environment.name
- Metric resolution (`StorageResolution` API) - Standard 60 seconds, high (costs more) 1/5/10/30 seconds (**High Resolution**).
- A Custom metric is created from a new **Metric Filter**, created from Log Groups. You set the Metric Value for each instance of a match of the filter, e.g. 1 match == value of 1.
- Important: the metric timestamp you push with the metric can be from -2 weeks (in the past) to +2 hours. You need to make sure the time is syncrhonised from the service you are pushing from. CloudWatch will reject metrics pushed with a timestamp outside the allowed range.

#### In practice

- You can push a metric as a json file, or CLI command:
  Example using a json file:

```
https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html#examples

aws cloudwatch put-metric-data --namespace "Usage Metrics" --metric-data file://metric.json

# File content:

[
  {
    "MetricName": "New Posts",
    "Timestamp": "Wednesday, June 12, 2013 8:28:20 PM",
    "Value": 0.50,
    "Unit": "Count",
    "Dimensions": [{
      "Name": "InstanceID",
      "Value": "1-23456789"
      }]
  }
]
```

Example using CLI API command only:

```
aws cloudwatch put-metric-data --metric-name Buffers --namespace MyNameSpace --unit Bytes --value 231434333 --dimensions InstanceID=1-23456789,InstanceType=m1.small
```

### CloudWatch Logs

- Log groups: name, e.g of application
- Log stream: logs files, containers, application
- Expiration policies, e.g. 1 day to years or never expire (default).
  - Log Retention Policy is defined at Log Group level.
- CloudWatch can forward logs to:
  - S3 (Export)
  - Kinesis Data Streams
  - Kinesis Data Firehose
  - AWS Lambda
  - OpenSearch
- Encryption:
  - Logs are encrypted by default
  - Can use KMS-based encryption with your own keys

#### Sources

- SDK, CloudWatch logs agent, Unified Agent
- Elastic Beanstalk logs
- ECS logs from containers
- Lambda function logs
- VPC flow logs
- API Gateway - logs all requests
- CloudTrail based on filter
- Route53: logs DNS queries

### CloudWatch Logs Insights

- A tool to query logs in CloudWatch - historical logs, not a real-time tool
- Query language - lots of example queries available
  - Filter, sorting, limiting
- Visualises logs outputs/trends in graphs
- Can query multiple logs

### S3 Export

- Can export your logs into S3 using API call `CreateExportTask`
- Log data can take up to 12 hours to become available for export
- Batch export, not for real-time.

### CloudWatch Logs Subscriptions

- **Purpose:** Used for real-time streaming of log data from CloudWatch Logs to other AWS services like:
  - Amazon Kinesis Data Streams
  - Amazon Kinesis Data Firehose
  - AWS Lambda
- **Subscription Filter:**
  - Applied to a log group in the sender account.
  - Filters and forwards matching log events to a destination (e.g., Kinesis stream).
  - Up to 2 subscription filters per log group.
- **Cross-Account Aggregation:**
  - Enables centralized log collection from multiple AWS accounts and regions.
  - **Sender Account**:
    - Has CloudWatch log groups with subscription filters.
    - Each filter references a CloudWatch Logs destination ARN in the receiver account.
  - **Receiver Account:**
    - Hosts the **CloudWatch Logs Destination** resource, which points to the Kinesis Data Stream.
    - The destination has a resource policy (not an IAM role) that:
      - Grants logs:`PutSubscriptionFilter` to the sender account(s).
      - Allows the sender to attach subscription filters to the destination.
    - The Kinesis Data Stream itself must allow `PutRecord` or `PutRecords` from the CloudWatch Logs service.
- **IAM Permissions:**
  - Sender account IAM role/user needs permission to call logs:`PutSubscriptionFilter`.
  - Receiver account must ensure:
    - the CloudWatch logs destination policies allow access from the sender.
    - the Kinesis stream policies allow access from the subscription CloudWatch logs.

### CloudWatch Logs Live Tail

- Can filter on log group
- As events are posted onto CloudWatch, they can be posted onto Live Tail to see new log entries and filtered.
- Only a few hours a day of free Cloud Tail.

### CloudWatch Agents

- Agents are a small linux program running on the server
- Agents required for pushing EC2 logs to CloudWatch
- Agents can be setup on on-prem servers
- IAM permissions needed to send the logs
- Two agents
  - CloudWatch Logs Agents
    - Older service
    - Only sends CloudWatch logs
  - CloudWatch Unified Agent
    - Newer service
    - Can send both metrics and logs
    - Centralised configuration using SSM Parameter store - so all your Agents share the same configuration.

#### CloudWatch Unified Agent - Metrics

Unified Agent metrics allow getting more detailed metrics than normal EC2 monitoring, can get more info about these specific metrics:

- CPU
- Disk metrics (IOPS, reads, etc)
- RAM
- Netstat
- Processes data
- Swap Space (free, used, used %)

### CloudWatch Logs Metric Filter

- Can get more specific filter on logs
- Only works on logs pushed after the metric is created
- Can create alerts/alarms based on the metric filter
  - E.g. 5 instances of an error in an hour, raise alarm
  - Select the created metric filter
  - Select threshold condition and value, e.g. >= 50
    - (Another option is Anomaly detection)
  - Set Alarm state trigger and notification target, e.g. SNS topic.
- When create the metric, you can test the filter to see the results work
  - There is a query syntax: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html
  - Metric name and value need to be set - value is for each match

### CloudWatch Alarms

- Alarm states: OK, INSUFFICIENT_DATA, ALARM
- Period: how long to evaluate the metric for, e.g. 10, 30 or 60 second intervals
- Targets:
  - Actions on EC2 Instance, e.g. stop, terminate, reboot, recover
  - Trigger Auto-scaling action
  - Trigger SNS notification - from here you can forward to many other services, e.g. Lambda.
- Composite alarms:
  - Monitor the state of **multiple** other CloudWatch Alarms using AND or OR conditions.
  - E.g. 2 Alarms on CPU and Memory, can trigger composite alarm if both CPU and Memory alarms have alerted.
- Test alarm using CLI:
  - `aws cloudwatch set-alarm-state --alarm-name "alarm" --state-value ALARM --state-reason "testing alarm"`

### EC2 Instance Recovery

Status check on EC2 is on: EC2 Instance, System (underlying hardware), or EBS status. If alarm is in alert, can take action, e.g. a recovery that moves EC2 instance to another host.

### CloudWatch Synthetics Canary

- A script to monitor APIs, URL, Websites, etc.
  - Written in Node.js or Python.
- This script replicates a customer or consumer behaviour
- You can run it to test your application's behaviour and status
  - Can store results, including screenshots.
- If this fails, you can trigger a CloudWatch alarm to, e.g., call Lambda that re-routes traffic elsewhere.
- It can use a headless chrome browser to replicate a users actions in a browser.
- Can run adhoc or on a schedule
- Blueprints/patterns:
  - Heartbeat monitoring - load URL, screenshot and save
  - API Canary - perform basic API operations
  - Broken link checker - checks the links on your site
  - Visual monitoring - compares a pre-existing screenshot against the live site
  - Canary recorder - record UI actions and convert to a script
  - GUI workflow builder - build actions to perform on the website

### Amazon EventBridge

- Event pattern: **Rules** to react to events, E.g. Someone signing in with the root account, send an email to an security account.
- **Rules** include:
  - **Event Source**, e.g.:
    - EC2 Instance
    - Codebuild
    - S3 Event
    - Trusted Advisor
    - Schedule/cron - Schedule: cron-jobs, e.g. to run a Lambda script
    - CloudTrail
  - Filters:
    - Can filter events
    - AWS **Service** and **Event type**, e.g. EC2 shutting-down
  - **Target**:
    - A JSON document of the event is generated, it can be sent to various destinations, e.g.:
    - Lambda, SQS, SNS, etc
- Event Bus - EventBridge reads events from these events buses:
  - **Default Event Bus:** AWS service events are sent into the Default Event Bus
    - You can also add additional Event buses sourcing AWS events, e.g. for separation of event buses for security reasons.
  - **Partner Event Bus:** SaaS partners (e.g. Datadog, Adobe, etc) can push events into the Partner Event Bus, so EventBridge can react to external partner events.
  - **Custom Event Bus:** applications can push into a Custom Event Bus
  - Event Buses can be accessed by other AWS Accounts using resource-based policies
  - Events sent to event bus can be filtered and archived for indefinite or fixed period, can replay these archived events during debugging.

#### Amazon EventBridge - Schema Registry

- Events are passed as JSON documents. There are schema for JSON documents for all the AWS service's various events, so you can see what data is coming in each event.
- For your own application events, the Schema Registry will generate code/class bindings to generate events in the correct structure.
- Schema Registry discovery can detect the structure of events as they pass through and add them to the Schema Discovery Registry so you don't have to do it manually.
- Schemas can be versioned.

#### Amazon EventBridge - Resource-based policies

- Manage permissions for a specific event bus, e.g. to allow/deny events from other accounts or regions.
- This would be used when combining events into a central event bus. Example scenario:
  - You have multiple EC2 instances in different accounts
  - Define an event pattern (JSON) in one account to send to an Event Rule in the source account using the central account API, e.g `PutEvents` API.
  - In the central account Event Bus create a Resource Policy to allow the EC2 instances to access it.
  - Setup the same event rule in the source accounts
  - They can all send events to the central even bus.

## AWS X-Ray

- Visual tool for debugging AWS services.
- Can troubleshoot bottlenecks
- Can visualise dependencies
- Understand how each request is behaving
- Finding errors in exceptions
- Can see which services are throttling performance
- Can see which users are impacts
- Services supported: AWS Lambda, Elastic Beanstalk, ECS, ELB, API Gateway, EC2 Instances, even on-prem application servers.
- X-Ray is found in the CloudWatch Console in AWS.

### Tracing

- Follows a request through the infrastructure, each component adds it's own trace.
- Annotation can be added to provide extra-information.
- Security: IAM Authorisation and KMS for encryption at-rest.

### Enabling

1. Must import AWS X-Ray SDK into your code (Java, Python, Go, Node.js, .NET), and configure code to use it, SDK will capture calls to AWS services, HTTP(S) requests, DB calls, Queue calls.
2. Setup the X-Ray daemon:

- If running on on-prem servers (Linux, windows, Mac), install the X-Ray daemon, this is an small app that runs on the server that works as a low level UDP packet interceptor.
- If running on AWS services (e.g. Lambda, EC2, etc), enable X-Ray AWS Integration (already installed depending on the AMI image).
- Each application must have the IAM rights to write data to X-Ray.

#### Troubleshooting X-Ray not working

- On EC2:
  - Ensure the EC2 IAM Role has permissions to X-Ray
  - And the EC2 instance is running the X-Ray Daemon
- On Lambda
  - Ensure Lambda has IAM execution role policy `AWSX-RayWriteOnlyAccess`
  - Ensure X-Ray is imported in the code
  - Enable Lambda X-Ray **Active Tracing**

### Instrumentation in your code

- Measure performance, diagnose errors, write trace information.
- To instrument your code you use the X-Ray SDK
- Customize and annotate the data the SDK sends to X-Ray
- Using filters, handlers, middleware.
- Steps to setup in a .NET project:
  - dotnet add package AWSSDK.XRay
  - download and configure the X-Ray Daemon. This can run in a sidecar in K8s, or locally, the SDK pushes telemetry to the daemon, which forwards it to AWS.
  - Setup middleware in Startup.cs
    - app.UseXRay("app name");
    - services.AddXRay();
    - This automatically captures all service calls
    - You can add custom segments.
    - Note: The X-RAY Daemon is end-of-life in Feb 2027. Use OpenTelemetry instead.

### Concepts

- Segments: a unit of trace data, e.g. a single request
- Subsegments: nested sub-data within a segment
- Trace: collection of segments to form end-to-end trace
- Sampling: decrease the amount of requests sent to X-Ray to reduce cost
  - Default:
    - **Reservoir:** every first request each second
    - **Rate:** + 5% of any additional requests
  - You can define you own reservoir and rate e.g.:
    - 10 and 0.10 is first 10 requests per second + 10% of request above that
    - 1 and 1 is all requests, 1 reservoir + 100% rate - expensive - but good for debugging.
  - This change is done in the X-Ray console, not in the app of daemon.
    - You can also add filter criteria, e.g. service name, URL path, HTTP method, etc.
- Annotations: indexed key-value pairs - can be used in search filters
- Metadata: non-indexes key-value pairs - not available in search filters
- X-Ray Daemon has a config to send data to AWS, it must have correct IAM permissions.

### X-Ray APIs

Daemon needs correct API Policies to operate:

- PutTraceSegments: write to X-Ray
- PutTelemetryRecords: write metrics to X-Ray
- GetSamplingRules: daemon can retrieve the sampling rules from X-Ray
- GetSamplingTargets: related to getting sampling rules
- GetSamplingStatisticsSummaries: related to getting sampling rules
- GetServiceGraph: get main graph in console
- BatchGetTraces: retrieve list of traces by ID.
- GetTraceSummaries: retrieve IDs and annotations for traces available using a filter, or trace IDs
- GetTraceGraph: retrieve a service graph for specified trace ID(s).

Example policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
```

### X-Ray with Beanstalk

- Elastic Beanstalk already includes the X-Ray Daemon. To enable it:
  - option_settings in the .ebextensions/xray-daemon.config:
    - ```
      option_settings:
        aws:elasticbeanstalk:xray:
          XRayEnabled: true
      ```
  - Or, if creating via the AWS Elastic Beanstalk Console, there is an option to enable it.
- EC2 instance profile must have correct permissions (see API policy)
- Application code instrumented with the SDK.

### ECS with X-Ray

Options:

1. Run one X-Ray Daemon on each EC2 instance in the cluster.
2. Side-car pattern: run one daemon instance per container as side-car.
   1. side-car **port mapping is containerPort 2000, protocol is udp**
   2. app container needs env var: **AWS_XRAY_DAEMON_ADDRESS: xray-daemon:2000**
   3. link containers using links: **links: xray-daemon**

3. Fargate cluster - no control over the EC2 instances, so have to use side-car pattern.

## AWS Distro for OpenTelemetry

A single set of API for collecting traces and metadata for applications. It's similar to X-Ray but Open Source. Common API to allow metrics/traces to be sent to AWS, or other services that use the OpenTelemetry standard API. E.g. targets could be:

- AWS X-Ray
- Amazon CloudWatch
- Amazon Managed Service for Prometheus
- Partner monitoring solutions

## AWS CloudTrail

- Records all events and API calls made within the AWS account, available in Console, SDK, CLI, other AWS Services.
- Can put CloudTrail records into CloudWatch Logs or S3
- Can apply to all regions or a single region
- Enabled by default
- Useful for governance/audit/compliance of AWS Account
- Useful for investigation, e.g. find out who deleted something in AWS
- Event types:
  - Management Events
    - Operations performed on AWS resources in your account
    - E.g. Configuring security, routing
    - Can separate read from write events
    - Logged by default
  - Data Events
    - Not logged by default - because might be high volume
    - E.g. S3 level object activity, AWS Lambda function invocations
  - CloudTrail Insights Events
    - Need to enable it and has a cost
    - It analyses your account for normal vs abnormal activity
- Event Retention
  - Default stored for 90 days
  - To keep events beyond 90 days, log them to S3 and use Athena to analyse them.
- What should you do to configure X-Ray Daemon to send traces from multiple AWS accounts to a central AWS account? **Create an IAM role in the central account, then create IAM roles in the other accounts to assume this IAM role**

### CloudTrail and EventBridge integration

Examples of integration:

- Dynamo DB event (e.g. delete table) -> logs to CloudTrail -> event raised to EventBridge -> Alert to SNS.
- User assumes IAM Roles -> logs to CloudTrail -> event raised to EventBridge -> Alert to SNS.
- User edits an EC2 Security group rules -> logs to CloudTrail -> event raised to EventBridge -> Alert to SNS.

## Service Comparison

| Feature / Aspect  | CloudTrail                                       | CloudWatch                                          | X-Ray                                                |
| ----------------- | ------------------------------------------------ | --------------------------------------------------- | ---------------------------------------------------- |
| **Purpose**       | Audit and log API activity across AWS services   | Monitor metrics, logs, and alarms for AWS resources | Trace requests and visualize application performance |
| **Data Type**     | API calls and events                             | Metrics, logs, custom events                        | Segments, subsegments, trace maps                    |
| **Scope**         | Account-level across AWS                         | Resource-level (EC2, Lambda, etc.)                  | Application-level (end-to-end request tracing)       |
| **Use Cases**     | Security auditing, compliance, forensic analysis | Performance monitoring, alerting, log aggregation   | Debugging latency, service maps, root cause analysis |
| **Granularity**   | Coarse (who did what, when)                      | Medium (resource metrics, log entries)              | Fine (request-level breakdown, timing, dependencies) |
| **Retention**     | Stored in S3 (configurable)                      | Metrics: 15 months; Logs: configurable              | Traces: configurable retention                       |
| **Integration**   | IAM, S3, EventBridge                             | Lambda, EC2, ECS, CloudTrail, X-Ray                 | CloudWatch, Lambda, ECS, API Gateway                 |
| **Visualization** | Console logs, Athena queries                     | Dashboards, log insights, alarms                    | Service map, trace timeline, annotations             |
| **Pricing Model** | Based on number of events and storage            | Based on metrics, logs, dashboards, alarms          | Based on trace volume and sampling rate              |
