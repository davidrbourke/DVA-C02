# AWS Step Functions

- Step functions are a state machine for workflow. You model the workflow in JSON. Each step in the workflow uses a state machine.
- Can visualise in a diagram in AWS
- Flow is made up of Tasks (Task State).

## Examples

### Execute AWS Services

- Invoke a Lambda function
- Run AWS Batch job
- Publish a message to SQS, SNS

### Run on Activity

- Run EC2 action, ECS, On-prem action
- Poll step function for work

## States

- Choice state: test for a condition to send to a batch
- Fail or succeed state: stop execution with a failure or success.
- Pass state: passes input to an output, or inject data, does not execute an action.
- Wait state: delay for a time or until a time/date
- Map state: Dynamically iterate states
- Parallel state: begin parallel branches of execution

```
{
  "Comment": "A simple Step Functions example that invokes a Lambda function",
  "StartAt": "InvokeLambda",
  "States": {
    "InvokeLambda": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyLambdaFunction",
      "InputPath": "$",
      "ResultPath": "$.lambdaResult",
      "TimeoutSeconds": 10,
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "HandleFailure"
        }
      ],
      "Next": "SuccessState"
    },
    "SuccessState": {
      "Type": "Succeed"
    },
    "HandleFailure": {
      "Type": "Fail",
      "Cause": "Lambda invocation failed"
    }
  }
}

```

[ get diagram of flows ]

## State Machine - function demo

Configuration here: https://github.com/hovmikayelyan/AWS-developer/blob/main/Marek%20Course%20Developer/step-functions/0-hello-world/state-machine.json

Lambda -> Choice state -> Pass state
(ARN of Lambda) (Reviews output -> Fail state
of Lambda)

- Use Node JS runtime, Lambda function returns string
- State machine needs permission to execute Lambda - this permission is automatically created when Step Function is created.
- Events for the state function are logged in AWS.

## Error handling in Step functions

Error handling should be handled outside of the tasks, e.g. don't catch errors in the Lambda, let the state machine handle catch and retry, etc. All errors and retries will be in the event history, and the Lambda (or other service) code will be simpler.

### Retry

Example: https://github.com/hovmikayelyan/AWS-developer/blob/main/Marek%20Course%20Developer/step-functions/1-error-handling/state-machine.json

- Define how many times to retry based on errors (errorType field from response).
- Defined in the JSON
- Configuration fields:
  - InternalSeconds: time to wait before retrying
  - MaxAttempts: no of times to retry
  - BackOffRate: multiplier of time to wait for retry, exponential backoff.
  - ErrorEquals: condition of errorType to handle for.

### Catch

- ErrorEquals
- Next: action to perform next on error
- ResultPath:
  - a path that determines what input is sent to the next state.
  - $.error includes error in input of the next step (along with the input of the previous step)

## Wait For Task token

- Waiting for external response, e.g. human approval, AWS asynchronous service response.
- Append `.waitForTaskToken` to the `Resource` field to tell the step function to wait.
- APIs that service calls back to the Step Function to return the Task Token when the step is complete:
  - `SendTaskSuccess`
  - `SendTaskFailure`
- Example of flow:
  - Step Function -> passes message with Task Token to SQS -> SQS -> EC2 pulling from SQS -> When complete, EC2 calls API to send the token back to the Step Function ->
    -> SendTaskSuccess -> Step function continues.

```
{
  "Comment": "Callback pattern using waitForTaskToken and SQS",
  "StartAt": "SendJobToSQS",
  "States": {
    "SendJobToSQS": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/myQueue",
        "MessageBody": {
          "taskToken.$": "$$.Task.Token",
          "input.$": "$"
        }
      },
      "TimeoutSeconds": 900,
      "Next": "JobCompleted"
    },
    "JobCompleted": {
      "Type": "Pass",
      "Result": "Worker returned success",
      "End": true
    }
  }
}
```

## Activity Tasks

- **Activity Workers** perform tasks from the step functions
- They run on your service, e.g. EC2
- The Step function reaches a point where it reaches an Activity Task, it waits.
- The Activity Worker polls the step function using the activity ARN to see if it has any tasks waiting.
- When complete, the service calls the `SendTaskSuccess` (or Failure) API
- The Step Function continues on its Flow
- Different from 'wait' state, as a **wait** pushes out, where as an **Activity Task** pulls from the step function.
- Services calls the `GetActivityTask` API to get tasks.

### Parameters - Activity Task

- **TimeoutSeconds**: how long before task must complete
- **HeartBeatSeconds**: periodically the activity task sends a heart beat back to the step function to keep the task alive from the step functions perspective.
- Using both above in combination, a step function can wait up to 1 year for task completion.

### Standard vs Express - Activity Task

|                   | Standard (default)                             | Express                                         |
| ----------------- | ---------------------------------------------- | ----------------------------------------------- |
| Max Duration      | 1 year                                         | 5 minutes                                       |
| Execution model   | Once                                           | At-least once (Async), At most once (Sync)      |
| Execution rate    | 2000+ per second                               | 100,000+ per second                             |
| Execution history | 90 days or CloudWatch logs                     | CloudWatch logs                                 |
| Pricing           | No. of state transitions                       | No. of executions, duration, memory consumption |
| Use case          | non-idempotent action, e.g. payment processing | IoT data ingestion, mobile backend              |

### Asynchronous - At-least once

- You don't need a response, have to check CloudWatch logs to see status
- Action re-run on failure, must be safe to re-run

### Synchronous - At-most once

- You need to wait for a response
- On failure, work will not automatically retry, you need to handle it.

# AWS AppSync

- A GraphQL managed service.
- Define only the data you want
- Uses different data sources that can be combined, e.g. NoSQL, RDMS, Custom sources with Lambda, etc.
- Uses:
  - Realtime integration with WebSockets
  - Mobile applications - local data access & data synchronisation
- AppSync is a replacement for CognitoSync.
- To use it, upload or create a GraphQL Schema
- GraphQL resolvers setup to know how to return the various parts of the data that the client has requested.

## Example flow

Webapp/Mobile app, Realtime dashboard, etc => AppSync -> Schema/resolvers -> DynamoDB, Aurora, opensearch, Lambda, HTTP, etc

## AppSync Security

Four ways to authenticate:

- API_KEY
- AWS_IAM (IAM Users/role/cross-account)
- OPENID_CONNECT (Integrate with 3rd part provider, returns JWT)
- AMAZON_COGNITO_USER_POOLS

For HTTPS, you can use CloudFront in front of AppSync.

## AppSync Demo

- Design: schema model
- Resolver data source: e.g. dynamo DB, others
- Functions
- Queries: can run GraphQL queries in UI to Get or Create data.
- Settings: include authentication mode, and providers
- Monitoring
- Custom domain names

# AWS Amplify

- Service to create mobile and web applications.
- Tools:
  - Studio - build app
  - CLI - configure backend
  - Libraries - connect to AWS Service
  - Hosting
- Similar to ElasticBeanstalk but for mobile and web applications
- Relies on:
  - DynamoDB
  - AWS AppSync for GraphQL
  - Amazon Cognito
  - Amazon S3
- Has frontend libraries for different languages, e.g., React IOS, etc.

## Amplify - Important Features

- Authentication out of the box
  - using Cognito, CLI: `amplify add auth`
  - Manages user registration, account recovery, etc.
  - Supports MFA, Social sign-in, e.g. Google, Facebook.
  - Prebuilt UI components for Auth
- Datastore
  - Uses DynamoDB using CLI: `amplify add api`
  - Syncs local data from app to the cloud
  - Offline and real-time capabilities
  - Visual data modelling with Amplify Studio
- Hosting
  - CLI: `amplify add hosting`
  - Build and host webapps
  - CICD built/test/deploy
  - Pull request previews - source code repos in GitHub, Bit bucket, etc.
  - Custom domains
  - Monitoring
  - Redirect and custom headers
  - Password protection
- Testing
  - Support for Unit testing and E2E
  - Unit tests and E2E can be run in Amplify Build step, configured in **amplify.yml**.
  - Integrated with the Cypress testing framework for E2E.

Underneath, Amplify uses CloudFormation to deploy the required infrastructure.
