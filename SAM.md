# Serverless Application Model (SAM)

(Basic questions in exam)

- A configuration in YAML to generate CloudFormation files, can use Outputs, Mappings, Parameters, Resources,etc.
- SAM can use CodeDeploy to deploy Lambda functions
- SAM Can help run Lambda, API Gateway, DynamoDB locally, to debug locally, and deploy into AWS cloud.

## SAM Recipe

SAM is made of recipes.

- Transform Header (at top of template) - to indicate it is a SAM template, e.g.

  - `Transform: AWS::Serverless-2016-10-31`

- Write code, use SAM constructs inteads of CloudFormation constructs, e.g.
  - `AWS::Serverless::Function`, etc.
- Package and deploy:
  - sam package (optional)
  - sam deploy (just using deploy will upload the changes, and also package them if the package cmd was not run).
- SAM Accelerate - to speed up deployment

## Structure locally

1. You have your application code
2. You have your SAM template YAML -> gets transformed into CloudFormation template
3. SAM deploy - upload package into S3 bucket .zip for deployment

## SAM Accelerate

These are commands to speed up deployment.

- Cmd: `sam sync`: Synchronise code and infra.
- Cmd: `sam sync --code`: Bypasses CloudFormation changes if you are only doing code changes and no infra changes.
- Cmd: `sam sync --code --resource-id {resource}`: only update a specific service
- Cmd: `sam sync --watch`: monitors code base for changes and automatically deploys just the changes.

## Demo

- `sam` tool is installed in AWS CLI in Console - otherwise install it locally.
- `sam init` to create a new application, there are sample applications, or custom templates.
- Config file: `samconfig.yaml`
- SAM template: `template.yaml` - definitions for SAM
- Example template.yaml for Lambda:

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Example SAM template for a Lambda function with API Gateway

Globals:
  Function:
    Timeout: 5
    Runtime: python3.9

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: HelloWorldFunction
      Handler: app.lambda_handler
      Runtime: python3.9
      CodeUri: src/
      Description: A simple Lambda function triggered by API Gateway
      MemorySize: 128
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello"
  HelloWorldFunctionArn:
    Description: "ARN of the HelloWorld Lambda function"
    Value: !GetAtt HelloWorldFunction.Arn
```

- Cmd: `sam build`, outputs build to `.aws-sam/build`.
- Can test locally with `sam invoke`
- Deploy with `sam deploy -- guided`, this will deploy the Lambda and application in the Cloud.

- A DynamoDb template example:

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Example SAM template with DynamoDB integration

Globals:
  Function:
    Timeout: 10
    Runtime: python3.9

Resources:
  MyDynamoDBTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: MyItemsTable
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

  MyLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: MyLambdaWithDynamoDB
      Handler: app.lambda_handler
      Runtime: python3.9
      CodeUri: src/
      Description: Lambda function that interacts with DynamoDB
      MemorySize: 256
      Environment:
        Variables:
          TABLE_NAME: !Ref MyDynamoDBTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref MyDynamoDBTable
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /items
            Method: post

Outputs:
  ApiEndpoint:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/items"
  DynamoDBTableName:
    Description: "Name of the DynamoDB table"
    Value: !Ref MyDynamoDBTable
```

# #SAM Policy Templates

For AWS Lambda created from SAM, you can assign the Lambda's permissions using pre-defined SAM policy names, full list: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-template-list.html. Examples:

- `S3ReadPolicy`: Lambda can read from S3
- `SQSPollerPolicy`: Lambda can poll an SQS queue
- `DynamoDBCrudPolicy`: Lambda can perform CRUD operations on a DynamoDB

Example:

```
Policies:
    - DynamoDBCrudPolicy:
        TableName: !Ref MyDynamoDBTable
```

## SAM with CodeDeploy

- SAM Framework natively uses CodeDeploy to update Lambda functions
  - Update is deployed to a new version in Lambda, and a new Alias (e.g. V2 is created)
- Traffic shifting feature
  - On successful validation, traffic shifted from V1 to V2 Alias
- Pre and Post traffic hooks to validate deployment
  - Hook Lambda functions can validate the deployment/run tests, etc
- Easy rollback using CloudWatch Alarms

Example:

- `AutoPublishAlias: live`: this allow SAM to auto-route the Lambda traffic to the new version when a `sam deploy` happens.
- `Canary10Percent5Minutes`: it does a CodeDeploy Canary deployment option to route traffic. (Linear, Canary, AllAtOnce options, same as CodeDeploy).
- See DeploymentPreference block in example:

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: SAM template with Lambda and CodeDeploy DeploymentPreference

Globals:
  Function:
    Timeout: 10
    Runtime: python3.9

Resources:
  MyLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: MyLambdaWithCodeDeploy
      Handler: app.lambda_handler
      Runtime: python3.9
      CodeUri: src/
      Description: Lambda function with CodeDeploy deployment preferences
      MemorySize: 256
      AutoPublishAlias: live
      DeploymentPreference:
        Enabled: true
        Type: Canary10Percent5Minutes
        Alarms:
          - !Ref MyAlarm
        Hooks:
          PreTraffic: !Ref PreTrafficLambda
          PostTraffic: !Ref PostTrafficLambda
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /deploytest
            Method: get

  # Example CloudWatch Alarm for rollback
  MyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "Alarm if Lambda errors exceed threshold"
      Namespace: AWS/Lambda
      MetricName: Errors
      Dimensions:
        - Name: FunctionName
          Value: !Ref MyLambdaFunction
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold

  # PreTraffic hook Lambda
  PreTrafficLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: pretraffic.lambda_handler
      Runtime: python3.9
      CodeUri: hooks/
      Description: PreTraffic validation Lambda

  # PostTraffic hook Lambda
  PostTrafficLambda:
    Type: AWS::Serverless::Function
    Properties:
      Handler: posttraffic.lambda_handler
      Runtime: python3.9
      CodeUri: hooks/
      Description: PostTraffic validation Lambda

Outputs:
  ApiEndpoint:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/deploytest"
  LambdaAliasArn:
    Description: "Alias ARN for the deployed Lambda"
    Value: !Ref MyLambdaFunction

```

## Local SAM Capabilities

The sam tool can be used for local development and testing of Lambdas without having to deploy it to AWS to test it. Commands:

- `sam local start-lambda`: uses an emulation of the Lambda locally.
- `sam local invoke`: invoke the local Lambda with a payload for testing.
  - If Lambda interacts with AWS, must using correct AWS `--profile`, so the local Lambda can access AWS services, e.g. S3 bucket.
- `sam local start-api`: starts a local HTTP API server for all your functions, changes to functions are automatically reloaded.
- `sam local generate-event`: generate an event for event sources, e.g. S3 for them to react.

## SAM Multiple environments

- Allows managing multiple envs, e.g. dev/prod, etc.
- `samconfig.toml`: define parameters for different environments

```
version = 0.1

[default]
[default.deploy]
[default.deploy.parameters]
stack_name = "my-sam-app"
s3_bucket = "my-sam-artifacts"
s3_prefix = "my-sam-app"
region = "us-east-1"
capabilities = "CAPABILITY_IAM"

[dev]
[dev.deploy]
[dev.deploy.parameters]
stack_name = "my-sam-app-dev"
s3_bucket = "my-sam-artifacts-dev"
s3_prefix = "my-sam-app-dev"
region = "us-east-1"
capabilities = "CAPABILITY_IAM"
confirm_changeset = true
resolve_s3 = true
parameter_overrides = "Environment=dev LogLevel=DEBUG"

[prod]
[prod.deploy]
[prod.deploy.parameters]
stack_name = "my-sam-app-prod"
s3_bucket = "my-sam-artifacts-prod"
s3_prefix = "my-sam-app-prod"
region = "us-east-1"
capabilities = "CAPABILITY_IAM"
confirm_changeset = true
resolve_s3 = true
parameter_overrides = "Environment=prod LogLevel=ERROR"

```

## SAM CLI + AWS Toolkits

SAM CLI and AWS Tookits together provide a comprehensive environment for debugging Lambda functions locally, allowing you to inspect variables and execute code line-by-line. This combination enhances your development efficiency and ensures thorough testing before deploying your Lambda functions.
