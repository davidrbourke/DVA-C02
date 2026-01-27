# Cloud Development Kit (CDK)

- Allows you to define your infrastructure in a Programming Language: JavaScript, Typescript, Python, Java, .NET,
- Type safe, won't compile if invalid. Advantage over YAML.
- CloudFormation template is generated from the CDK CLI.

## CDK vs SAM

- SAM is serverless focused YAML, e.g. Lambda, also uses CloudFormation in background.
- CDK supports every AWS service, in a programming language.

## CDK & SAM together

- Use SAM CLI to test app locally, e.g. Lambda `sam local invoke ..`
- Use CDK for code infrastructure, and run `sam synth` to generate a CloudFormation template, and deploy to AWS.

## Demo

1. Install CDK for your language, e.g. `npm install -g aws-cdk-lib` for JavaScript.
2. `cdk init app --language javascript` creates the CdkAppStack
3. In /lib directory - this is where the code goes.
4. Example code

- Creates S3 bucket
- Role for Lambda
- Add Policy statement to role
- Create a DynamoDB table
- Create Lambda function
- Lambda trigger from AWS S3
- Grant read/write access for Lambda function to S3 and DynamoDB table

5. Create a Lambda in code (python in example)

- Detects labels and images using Rekognition and writes target data into DynamoDB

6. Run `cdk bootstrap` - this sets up in your account for CDK to be able to deploy, it's actually using CloudFormation to create the CDK infra, only needs to be run once per region per account.
7. `cdk synth` - this creates the CloudFormation template for the infra. You can preview it.
8. `cdk deploy` to generate the template and deploy it into AWS.

Javascript example

```
const cdk = require('aws-cdk-lib');
const s3 = require('aws-cdk-lib/aws-s3');
const dynamodb = require('aws-cdk-lib/aws-dynamodb');
const lambda = require('aws-cdk-lib/aws-lambda');
const iam = require('aws-cdk-lib/aws-iam');
const s3n = require('aws-cdk-lib/aws-s3-notifications');

class MyStack extends cdk.Stack {
  constructor(scope, id, props) {
    super(scope, id, props);

    // 1. Create S3 bucket
    const bucket = new s3.Bucket(this, 'MyBucket', {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    // 2. Role for Lambda
    const lambdaRole = new iam.Role(this, 'MyLambdaRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
    });

    // 3. Add Policy statement to role
    lambdaRole.addToPolicy(new iam.PolicyStatement({
      actions: ['logs:CreateLogGroup', 'logs:CreateLogStream', 'logs:PutLogEvents'],
      resources: ['*'],
    }));

    // 4. Create DynamoDB table
    const table = new dynamodb.Table(this, 'MyTable', {
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    // 5. Create Lambda function
    const myLambda = new lambda.Function(this, 'MyLambda', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        exports.handler = async (event) => {
          console.log("Event: ", JSON.stringify(event, null, 2));
          return { statusCode: 200, body: "Success" };
        };
      `),
      role: lambdaRole,
    });

    // 6. Lambda trigger from AWS S3
    bucket.addEventNotification(
      s3.EventType.OBJECT_CREATED,
      new s3n.LambdaDestination(myLambda)
    );

    // 7. Grant read/write access for Lambda function to S3 and DynamoDB table
    bucket.grantReadWrite(myLambda);
    table.grantReadWriteData(myLambda);
  }
}

module.exports = { MyStack };

```

.NET example

```
using Amazon.CDK;
using Amazon.CDK.AWS.S3;
using Amazon.CDK.AWS.DynamoDB;
using Amazon.CDK.AWS.Lambda;
using Amazon.CDK.AWS.IAM;
using Amazon.CDK.AWS.S3.Notifications;

namespace MyCdkApp
{
    public class MyStack : Stack
    {
        internal MyStack(Construct scope, string id, IStackProps props = null) : base(scope, id, props)
        {
            // 1. Create S3 bucket
            var bucket = new Bucket(this, "MyBucket", new BucketProps
            {
                RemovalPolicy = RemovalPolicy.DESTROY,
                AutoDeleteObjects = true
            });

            // 2. Role for Lambda
            var lambdaRole = new Role(this, "MyLambdaRole", new RoleProps
            {
                AssumedBy = new ServicePrincipal("lambda.amazonaws.com")
            });

            // 3. Add Policy statement to role
            lambdaRole.AddToPolicy(new PolicyStatement(new PolicyStatementProps
            {
                Actions = new[] { "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents" },
                Resources = new[] { "*" }
            }));

            // 4. Create DynamoDB table
            var table = new Table(this, "MyTable", new TableProps
            {
                PartitionKey = new Attribute { Name = "id", Type = AttributeType.STRING },
                BillingMode = BillingMode.PAY_PER_REQUEST,
                RemovalPolicy = RemovalPolicy.DESTROY
            });

            // 5. Create Lambda function
            var myLambda = new Function(this, "MyLambda", new FunctionProps
            {
                Runtime = Runtime.NODEJS_18_X,
                Handler = "index.handler",
                Code = Code.FromInline(@"
                    exports.handler = async (event) => {
                        console.log('Event: ', JSON.stringify(event, null, 2));
                        return { statusCode: 200, body: 'Success' };
                    };
                "),
                Role = lambdaRole
            });

            // 6. Lambda trigger from AWS S3
            bucket.AddEventNotification(EventType.OBJECT_CREATED, new LambdaDestination(myLambda));

            // 7. Grant read/write access for Lambda function to S3 and DynamoDB table
            bucket.GrantReadWrite(myLambda);
            table.GrantReadWriteData(myLambda);
        }
    }

    sealed class Program
    {
        public static void Main(string[] args)
        {
            var app = new App();
            new MyStack(app, "MyStack");
            app.Synth();
        }
    }
}
```

## CDK Constructs

- These are the constructors used to create one of multiple resources in AWS.
- Full list available in the AWS Construct Library.
- Construct Hub - hub for AWS and 3rd party created constructs.

### Construct Levels

Each construct has 3 levels.

#### Level 1

- CFN (CloudFormation) Resources - represents all resource directly available in CloudFormation.
- Start with `Cfn`
- Must explicitly configure all resources.

```
const bucket = new s3.CfnBucket(this, 'MyBucket', {
bucketName: "MyBucket"
});
```

#### Level 2

- AWS Resources at a higher level, specify **intent**.
- Has convenient defaults and boilerplate.

```
const bucket = new s3.Bucket(this, 'MyBucket', {
versioned: true,
encryption: s3.BucketEncryption.KMS
});

const objectUrl = bucket.urlForObject('MyBucket/MyObject);
```

#### Level 3

- Patterns - represents multiple related resources.
- Does a lot of heavy configuration in the background.

```
const cdk = require('aws-cdk-lib');
const { Construct } = require('constructs');
const lambda = require('aws-cdk-lib/aws-lambda');
const apigateway = require('aws-cdk-lib/aws-apigateway');

class LambdaRestApi extends Construct {
  constructor(scope, id, props) {
    super(scope, id);

    // Create Lambda function
    this.lambdaFunction = new lambda.Function(this, 'LambdaHandler', {
      runtime: props.runtime ?? lambda.Runtime.NODEJS_18_X,
      handler: props.handler ?? 'index.handler',
      code: props.functionCode,
    });

    // Create API Gateway REST API
    this.api = new apigateway.RestApi(this, 'RestApi', {
      restApiName: props.apiName ?? 'LambdaRestApi',
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS,
      },
    });

    // Add resource and methods
    const resource = this.api.root.addResource(props.resourcePath ?? 'items');
    resource.addMethod('GET', new apigateway.LambdaIntegration(this.lambdaFunction));
    resource.addMethod('POST', new apigateway.LambdaIntegration(this.lambdaFunction));
  }
}

module.exports = { LambdaRestApi };
```

## CDK Commands

- `npm install -g aws-cdk-lib`
- `cdk init app`
- `cdk synth` - synthesizes and prints CloudFormation template
- `cdk bootstrap` - Deploys the CDK Toolkit staging stack
- `cdk deploy` - Deploys the stack
- `cdk diff` - Diffs between local and deployed stack
- `cdk destroy`

## CDK Bootstrapping

- The process of provisioning resources for CDK before you can use CDK to deploy an app.
- AWS Environments - region and & account - only needs to be done once.
- Contains an S3 bucket and an IAM role - required.
- `cdk bootstrap aws://{account}/{region}`
- If you try to redeploy the bootstrap, it will just work, but any other errors will show an error.

## CDK testing

- Use the CDK Assertions Module
- Run with normal language test runner, e.g. XUnit, Jest.
- Assertions
  - Fine graining
  - Snapshots
- IMPORTANT:
  - You import the template from the Stack using `Template.fromStack(MyStack {object});`
  - Or using a local template if not yet deployed `Template.fromString(mystring {local});`
- You can then assert against the template returned.

```
const { App } = require('aws-cdk-lib');
const { Template } = require('aws-cdk-lib/assertions');
const { MyStack } = require('../lib/my-stack');

test('S3 Bucket Created', () => {
  const app = new App();
  const stack = new MyStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  template.hasResourceProperties('AWS::S3::Bucket', {
    // Example property check
    BucketEncryption: {
      ServerSideEncryptionConfiguration: [{
        ServerSideEncryptionByDefault: { SSEAlgorithm: 'AES256' }
      }]
    }
  });
});

test('Lambda Function Created', () => {
  const app = new App();
  const stack = new MyStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  template.hasResourceProperties('AWS::Lambda::Function', {
    Runtime: 'nodejs18.x',
    Handler: 'index.handler'
  });
});
```
