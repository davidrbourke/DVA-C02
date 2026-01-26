# API Gateway

Options to expose Lambda:

- Expose Lambda directly
- ALB
- API Gateway
  - Client talks to API Gateway, proxies request to Lambda function.

Expose and HTTP endpoints, e.g.

- On Prem HTTP API
- ALB
- Expose AWS API/Service through the Gateway

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
  - Custom Authorizer (implement your own Auth logic)
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

```
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "apigateway.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:eu-west-1:00000000000:function:demoFunction",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:execute-api:eu-west-1:00000000000:ls6mm5hwxb/*/GET/"
        }
      }
    }
  ]
}
```

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
  - Configure HTTP endpoints automatically, e.g. dev/test/v1, etc
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

## Integration Types

### Mock

- Return a response from the API without calling a backend

### HTTP/AWS Lambda

- Forwards a request, can modify it
- Setup in integration request and response
  - Can change the request and response between API and the backend
- Mapping Templates
  - Modify body, headers, query strings
  - Uses Velocity Template Language (VTL)
  - Content-Type must be application/json, or application/xml
  - Example 1: SOAP: transform an JSON request from client in SOAP format for backend, and reverse mapping on response, SOAP to JSON.
  - Example: Query param to Json body - map query parameters from a request into json body properties to forward to backend/Lambda.

- Demo
  - In AWS Console, when creating the API method to the Lambda, DO NOT enable **Lambda proxy integration** switch.
  - In the API method, there are options for **Integration Request** and **Integration Response**.
  - Integration Response - create new mapping template
    - Create the new JSON response, map from `example` field in backend response to `renamed-key`:
    - ```
      {
        "my-key": "my-value",
        "renamed-key": $input.json('$.example`)
      }
      ```

### AWS_PROXY

- A proxy from request to Lambda, cannot use any mapping or change any request/response. Request is forwarded as it is.
- Backend function must be able to understand the request.

### HTTP_PROXY

- HTTP passed directly to the backend, no manipulation by the API. But can add optional HTTP Headers, e.g. API Key to pass to the backend.
- So client would not need to know the key.

## Open API spec

- A common way to define REST APIs
- Import OpenAPI 3.0 spec into Gateway
  - Method
  - Method Request
  - Integration Request
  - +AWS extensions for API gateway
- Can export current API as OpenAPI spec
- Spec can be YAML or JSON
- Using OpenAPI you can generate SDKs for an application
- **Request Validation**
  - Can use OpenAPI spec to validate the request according to the schema
  - Caller gets 400 error if request is not valid, avoids call to backend.
  - In AWS setup an OpenAPI definition, and use `x-amazon-apigateway-request-validators` property with definition of what to validate.
- Demo:
  - When creating the API you can import the OpenAPI definition file, it will build out API methods.
  - You can export the definition from the 'Stage' section.
  - You can also generate the SDK with various languages from the definition, your application can use the SDK to interact with the API Gateway.

```
openapi: 3.0.1
info:
  title: Sample API with Validation
  version: 1.0.0
paths:
  /items:
    post:
      summary: Create an item
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - name
                - price
              properties:
                name:
                  type: string
                price:
                  type: number
      responses:
        '200':
          description: Item created
      x-amazon-apigateway-request-validator: all

# You don't need all of these, just the ones you want to apply
x-amazon-apigateway-request-validators:
  all:
    validateRequestBody: true
    validateRequestParameters: true
  body-only:
    validateRequestBody: true
    validateRequestParameters: false
  params-only:
    validateRequestBody: false
    validateRequestParameters: true

```

## API Gateway Caching

- Default cache TTL: 300 seconds, min 0 (no cache) to max 1 hour.
- Defined at the **Stage** level
- Can override cacher per method
- Can be encrypted
- Size: 0.5GB to 237GB
- Expensive, only using in Production
- Invalidate the cache, 2 ways:
  - From AWS Console
  - From Client:
    - Using header: `Cache-Control: max-age=0`
    - Client must have IAM permission, action: `execute-api:InvalidateCache` and resource.
- If you don't impose an InvalidateCache policy or enable **Require authorisation**, then any client can invalidate the cache (once you Enable the caching).
  - Assign what happens if unauthorised client attempts to invalidate cache, e.g. ignore, return 400, show warning.

## Usage Plan & API Keys

To make API available and control access to different clients, setup a Usage Plan:

- Key used to control access to Stages
- Limit how fast and much they can access
- Configure throttling limits and quota limits - applied per API Key level
- API Keys
  - Each client has an API key
  - Controls access for that client
  - Quota limits is the overall number of maximum requests

### Steps to setup Usage Plan

1. Create API, configure methods to require an API key, and deploy API to the Stages.
2. Generate or import API keys to distribute to clients
3. Create a usage plan
4. Associate API stage with API keys and usage plans
5. Callers must include their `x-api-key` header in requests to the API methods.

## Logging and Tracing

- CloudWatch logs
  - Request/response body logged
  - Enable at Stage level with log level (e.g. debug, error, info)
  - Can override on API basis
  - Be careful with sensitive information
- X-RAY
  - If Enabled for Gateway and Lambda, can get full picture
- CloudWatch Metrics can be enabled:
  - CacheHitCount
  - CacheMissCount
  - Count: no. of API requests per time period
  - IntegrationLatency: how long backend takes from request to response from backend.
  - Latency: time API GW receives request till time it sends the response, includes anything the Gateway is doing on the request.
- Any request over 29 seconds will result in API Gateway timeout
- Error metrics:
  - 4xx client side errors
    - 403 Bad request
    - 403 access denied
    - 429 throttling
  - 5xx server side errors
    - 502 bad gateway - to Lambda
    - 503 backend unavailable
    - 504 API gateway timeout

- Throttling
  - Default Account limit: 10,000 requests per minute
  - Soft limit that can be increased by requesting to AWS
- 429 too many requests response in case of throttling
- Can define Usage plans per customer (Key) to limit requests per client

## API Gateway CORS

- Enable to allow requests from other domains - in the AWS Console.
- OPTIONS pre-flight request must contain
  - Allow-Control-Allow-Methods
  - Allow-Control-Allow-Header
  - Allow-Control-Allow-Origin
- This for the Browser, the Browser will block response from a non-origin destination, unless CORS is enabled allowing the origin.

## API Gateway Security

(Need to know this well)

- IAM Permissions
  - Using IAM Policy to allow User/Role to access API Gateway.
  - To provide access from other AWS Services
  - Leverages Signature V4 capability where IAM credentials are in headers
  - Flow:
    - Client requests to gateway with with Sig v4
    - API gateway checks against IAM policy
    - If successful auth, request continues to Backend
- Resource Policies
  - Allow a JSON policy to be set on Gateway to define who and what they can do.
  - Allows for Cross Account Access (combined with IAM Security)
  - Allow for specific source IP address or allow for a VPC Endpoint
- Cognito User Pools
  - A Database of users, managed by Cognito
  - No custom implementation
  - Flow:
    - User authenticates with Cognito user pools to get a token and send token with request to API.
      - Auth can be with a number of providers, e.g. Google, etc.
    - API Gateway is integrated with Cognito and evaluates the token
    - Allows access if token is correct
    - Cognito manages token lifecycle
- Lambda Authorizer (Custom Authorizer)
  - Token based, JWT bearer token
  - Request includes token
  - Lambda returns an IAM Policy for the user, result policy is cached
  - Users are external
  - Flow:
    - Client authentication with 3rd party authentication system to get a token
    - Client passes token (e.g. via header)
    - API gateway calls Lambda Authorizer function to verify the token
      - **You have to implement this logic in the Lambda function**
    - If token is valid, Lambda returns IAM Principle and Policy and gets cached
    - API Gateway can then talk to backend.

## HTTP API vs REST API

- HTTP API
  - Low latency
  - All Proxy only - no data mapping
  - Only supports OIDC and OAuth 2.0
  - IAM and Cognito, no support for resource policy
- REST API
  - Supports all auth features except OpenID Connect/OAuth 2.0
  - More expensive than HTTP Policy

## API Gateway WebSocket API

- A two way interaction communication between user client browser and server
- Server can push back the client
- For stateful application use cases (persistent connection)
- Example: a chat application, other **real-time** type applications.
- Works with AWS Services (Lambda, DynamoDB, HTTP)
- Websocket URL:
  - `wss://{uniqueid}.execute-api/{region}.amazonaws.com/{stage name}`
- Flow:
  - Client connects to API Gateway
  - A connection is created and sent on to the next service, e.g. Lambda Function
    - Connection ID is going ot be persistent
  - Connection ID can be passed onto other service, e.g. DynamoDB
  - Client reuses Connection ID on following requests (frames)
  - Connection ID remains available while client is connected
  - Server calls client on connection URL callback that gets generated to allow Server to push data back to the client, example Connection call back URL:
    - `wss://{uniqueid}.execute-api/{region}.amazonaws.com/{stage name}/@connections/{connection-id}`
    - Connection URL operations
      - POST - send message to client
      - GET - gets latest connection status of the client
      - DELETE - disconnect the client
- **Routing:**
  - This is to send requests to different backends
  - Route is $default if not specified
  - Setup a route selection expression on incoming data, e.g. a field in the request JSON body: `$request.body.action` if field called action.
  - Setup route keys in your Gateway to route to specific backends based on the value of the field.

## Example Microservice Architecture

- A single interface for all your microservices
- Use different API Endpoints to different microservices
- Use Route53 to setup a domain for the API Gateway and apply SSL certificates.

## Signature V4

A method for authenticating API calls, it doesn't use an API Key. Instead you use the SigV4 signing process:

1. Create a canonical request based on the request details
2. Calculating a signature using your AWS Credentials
3. Adding the signature into a request header.

AWS replicates this process and compares the signatures. Key is scoped to the service, region, and day of the request.
