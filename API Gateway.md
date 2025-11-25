# API Gateway

Options to expose Lambda:

- Expose Lambda directly
- ALB
- API Gateway
  - Client talks to API Gateway, proxies request to Lambda function.

Expose and HTTP endpoints, e.g.

- On Prem HTTP API
- ALB
- Expose and AWS API/Service through the Gateway

- Example: Need public clients to send data to Kinesis Data Streams. Use the API Gateway:
  Clients -> API Gateway -> Kinesis Data Streams

## Endpoint Types

- Edge-optimized: for global clients. API Gateway still in one region, but accessible efficiently globally though the Edge locations.
- Regional:
  - For clients all in same region
  - Can combine with CloudFront to have CloudFront features, e.g. control over caching.
- Private API Gateway: accessed only from your VPC using a VPC endpoint (ENI).
  - Control access with a policy.

## Security

- User authentication through:
  - IAM Roles (for internal applications)
  - Cognito for external users (e.g. mobile users)
  - Custom Authorizer (implement your onw Auth logic)
- HTTPS Security with your own Custom Domain Name with integration through AWS certificate manager.
  - If using **Edge-Optimized** endpoint, then certificate must be in **us-east-1**.
  - If using **Regional endpoint**, the certificate must be in the API Gateway region
  - Must setup CNAME or A-alias record in Route 53.

## Demo

- Types
  - Http API
  - WebSocket API
  - Rest API
    - Public
    - Private
- Rest API
  - New, Import, Clone, Example
  - Name
  - API Endpoint type
    - Regional (\*)
    - Edge-optimized
    - Private
- Create Method
  - Method/verbs
  - Integration type
    - Lambda function (\*)
    - HTTP
    - Mock
    - AWS Service - any service in any region
    - VPC Link
- Lambda function (need to create function separately)
  - The Lambda returns 200 with json body.
  - Select region of function and function ARN.
  - Select Lambda proxy integration
  - Default timeout 29 second (500ms to 29000ms)
  - Creating the method grants API Gateway the right to execute the Lambda function.
    - API Gateway calls the Lambda API add-permission to add a statement to the Lambda's resource policy.
- Deploy API, including a name for the Stage, e.g. dev
  - This will generate a public URL.
  - Changes to the API need to be **Deployed**, or they will not be live.

## Deployment Stages

- Can create multiple, e.g. /v1, /v2
- Get their own URL for each stage
- Can rollback deployments of stages

### Stage variables

- Like env vars for each stage, to change config values without having to redeploy
- Can be used for:
  - Lambda Function ARN
  - HTTP Endpoint
  - Parameter mapping templates
- Uses:
  - Configure HTTP endpoints automatically, e.g. de/test/v1, etc
  - Pass config parameters to AWS Lambda through mapping templates
- Stage variables are passed to the "context" object in AWS Lambda
- To access variable: `${stageVariables.variableName}`

### Stage Variables Example

- Lambda Alias dev -> points to Lambda version $LATEST
- Lambda Alias test -> points to Lambda version v2
- Lambda Alias prod -> points to Lambda version v1

In the API Gateway, when setting up the method to the Lambda ARN, include a variable in the Lambda ARN: arn:aws:lambda:eu-west-1:1212121212:function:my-lambda:$
{stageVariables.lambdaAlias}

AWS generates a statement to give IAM Permission for the Lambda, run it in the console to allow the API Gateway to invoke the specific function, this is different than normal because a variable has been used for the Lambda, so AWS doesn't know which Lambda to apply the permission to automatically.

Testing the invocation, you need to provide the variable value (for the Alias), e.g. PROD/TEST/DEV.

Deploy API into Stages, Dev, Test, and Prod. On each stage, add a Stage Variable called **lambdaAlias** and set the values, DEV, TEST, PROD.

When running each stage, the correct parameter will be passed into the Lambda version to invoke.

- API Gateway /dev -> Lambda Alias dev -> points to Lambda version $LATEST
- API Gateway /test -> Lambda Alias test -> points to Lambda version v2
- API Gateway /prod -> Lambda Alias prod -> points to Lambda version v1
