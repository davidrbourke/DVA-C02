# CloudFormation

CloudFormation (CF) allows you to define you AWS infrastructure in code; Infrastructure as Code.

## Features

- Code is declarative - CF deploys it in the correct order - also delete it in the correct order.
- Can be version controlled
- Code review infra changes
- Tag resources with ID, so can see exact cost by Tag.
- Auto-timing to schedule tear up/down environments - to save cost when not in use.
- Separation of stacks, e.g., app infra, network infra, etc can be managed separately.

## Templates

The CF templates are uploaded to an S3 bucket, in CF you reference the template in the S3.

- A **stack** is a collection of service
- the **stack** infra is created based on the template content
- Upload a new template to S3 to update the **stack**

## Deployment

- Manual - using a code editor or AWS Console parameters
- Automated - YAML + CLI, or a continuous deployment pipeline.

## CloudFormation Blocks

These the Components:

- AWS Template Format Version
- Description
- Resources - only **resources** are mandatory in the template.
- Parameters
- Mappings
- Outputs
- Conditionals

### Template helpers

- References
- Functions

## Demo 1 - Create

- Select Region - us-east-1 if the most functional region, so all templates work with us-east-1.
- CloudFormation > Stack > Create
  - Sample template (\*)
  - Existing template
  - Build from Application Composer
- Select 'Sample template' - displays the Application composer UI

  - will show a diagram of resources that will be made
  - shows the code (YAML or JSON)
  - the demo uses a sample app from github
  - Note: for EC2 instance, the AMI template must be in the same region.

- **Events** will show the various resources being created.
- Example YAML that creates an EC2 instance:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Create an EC2 instance with a security group

Parameters:
  KeyName:
    Description: Name of an existing EC2 KeyPair to enable SSH access
    Type: AWS::EC2::KeyPair::KeyName

Resources:
  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      KeyName: !Ref KeyName
      ImageId: ami-0c02fb55956c7d316 # Amazon Linux 2 AMI (us-east-1)
      SecurityGroups:
        - !Ref EC2SecurityGroup

Outputs:
  InstanceId:
    Description: The Instance ID
    Value: !Ref EC2Instance
  PublicIP:
    Description: Public IP of the instance
    Value: !GetAtt EC2Instance.PublicIp
```

## Demo 2 - Update

- Need a new YAML template with updates, e.g. additional Security Group, Elastic IP, etc.
- Upload the new template file - might be prompted for parameters if they are defined in the template, e.g. names of things.
- **Change set preview**
  - A list of CF updates from the YAML
  - Shows what will be added/modified or deleted
  - Submit: **events** will show the update being applied to the resources.

## Demo 3 - Delete

- CloudFormation > Stacks > Delete
  - Will clean up everything created by CF.
  - CF determines the order to correctly delete resources.

## YAML

Exmaple of YAML and it's structure

```
name: "Joe Blogs"                  # key-value pair
age: 99

address:                        # nested object
  street: "123 Logic Lane"
  city: "London"

hobbies:                        # array
  - films
  - reading

bio: |                          # multiline string
  Joe is a systems thinker.
  Joe enjoys exploring technical and social frameworks.

```

## Resources

- The only mandatory section in the template
- This is the AWS component - 700+ types of AWS resources
- Resource type identifiers
  - service-provider::service-name::data-type-name
  - [AWS docs for all CF resource type identifiers](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html)
    - This doc contains all the YAML/JSON for a resource and all key-value pairs, required (y/n), type, and whether they are replaced or updated in-place.
  - e.g. AWS::EC2::Instance
- Dynamic resources - using Macros and Transform
- Almost every AWS resource is supportedm but use **Custom Resources** if it is not supported.

## Parameters

- A way to get inputs from the user when creating from the templates.
- Use if:
  - resource config value is likely to change
  - config value cannot be determined ahead of time
- Types:
  - String
  - Number
  - Comma delimited list
  - List<Number>
  - AWS-Specific-Parameter - to prevent invalid entries
  - List<AWS-Specific-Parameter>
  - SSM Parameter - get value from SSm Parameter Store
- Example:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Launch an EC2 instance with a user-defined instance type

Parameters:
  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t3.micro
    ConstraintDescription: Must be a valid EC2 instance type

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0c02fb55956c7d316 # Amazon Linux 2 AMI (us-east-1)

Outputs:
  SelectedInstanceType:
    Description: The EC2 instance type used
    Value: !Ref InstanceType
```

## Constraints

- Includes a description about the constraint to input.
- Types:
  - Default
  - AllowedValues (array)
  - AllowedPattern (regex)
  - Min/Max length/value
  - NoEcho (bool) - this prevents the parameter being output in the Console or logs, e.g. if it is secure/private information.
- Examples:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Example showing all parameter constraints

Parameters:
  EnvironmentType:
    Description: Choose the deployment environment
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
    ConstraintDescription: Must be one of: dev, staging, prod

  AdminEmail:
    Description: Admin contact email
    Type: String
    AllowedPattern: ^[\\w.-]+@[\\w.-]+\\.\\w+$
    ConstraintDescription: Must be a valid email address

  AppName:
    Description: Application name
    Type: String
    MinLength: 3
    MaxLength: 15
    ConstraintDescription: Must be between 3 and 15 characters

  DBPassword:
    Description: Database password (hidden)
    Type: String
    NoEcho: true
    MinLength: 8
    MaxLength: 32
    AllowedPattern: ^[a-zA-Z0-9@#$%^&+=]+$
    ConstraintDescription: Must be 8–32 characters and contain only valid symbols

Resources:
  DummyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AppName}-${EnvironmentType}-bucket"

Outputs:
  Email:
    Description: Admin email used
    Value: !Ref AdminEmail
```

## Reference

- Reference other components, parameters, or elements in the template.
- E.g. `VpcId: !Ref MyVpc`
- `MyVpc` will be an element in the template.

## Pseudo Parameters

- These are predefined built-in parameters in AWS about the Stack that you can be referenced in the template.

  - Common pseudo parameters

  | Pseudo Parameter | Description                                                      |
  | ---------------- | ---------------------------------------------------------------- |
  | `AWS::AccountId` | The ID of the AWS account where the stack is deployed            |
  | `AWS::Region`    | The region where the stack is deployed (e.g., `us-east-1`)       |
  | `AWS::StackName` | The name of the stack                                            |
  | `AWS::StackId`   | The unique ID of the stack                                       |
  | `AWS::Partition` | The partition the resource is in (`aws`, `aws-cn`, `aws-us-gov`) |
  | `AWS::URLSuffix` | The domain suffix (e.g., `amazonaws.com`)                        |

- Example:

```
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "my-app-${AWS::Region}-${AWS::AccountId}"
```

## Mappings

- Fixed varibles in the template, e.g. hardcoded for various environments.
- To access the mapping value:
  - `!FindInMap [MapName, Ref! TopLevelKey, SecondLevelKey]`
- This example below is using a Ref! to the pseudo parameter AWS::Region to return the current region, e.g. us-east-1.
- Mappings vs Parameter
  - Mapping: you know the values in advance, e.g the region
  - Parameter: values depend on what the user wants at runtime

```
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2 instance using region-specific AMI via Mappings

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c02fb55956c7d316
    us-west-2:
      AMI: ami-0b2f6494ff0b07a0e
    eu-west-1:
      AMI: ami-047bb4163c506cd98

Parameters:
  InstanceType:
    Type: String
    Default: t2.micro

Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]

Outputs:
  InstanceAMI:
    Description: AMI used for this region
    Value: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
```

## Outputs and Exports

- Declare optional output values that can be used in another application stack, collaboration across stacks.
- View outputs in the AWS Console or CLI.
- Export must be globally unique across the **account** and **region**.
- To use exported value: `!ImportValue {unique name}`
- Once the exported value is used in another Stack, you cannot delete the source Stack until all the stacks consuming the exported value are also deleted.
- Example of an output
  - **Note**: using !Ref on the resource type returns different things depending on the type, e.g. for a VPC, !Ref will get the VPC ID, for an S3::Bucket, it would return the bucket name.

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Create a VPC and export its ID

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: MyExportedVPC

Outputs:
  VPCId:
    Description: The ID of the created VPC
    Value: !Ref MyVPC
    Export:
      Name: MySharedVPCID
```

- Use exported VpcId in another stack:

```
Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue MySharedVPCID
      CidrBlock: 10.0.1.0/24
```

## Conditions

- Control the creation of resources based on a condition. E.g. dev/prod env conditions.
- This example only creates the logging bucket if the parameter is ~true~.

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Create an S3 bucket only if logging is enabled

Parameters:
  EnableLogging:
    Description: Whether to enable logging
    Type: String
    AllowedValues:
      - true
      - false
    Default: false

Conditions:
  CreateLoggingBucket: !Equals [!Ref EnableLogging, "true"]

Resources:
  LoggingBucket:
    Type: AWS::S3::Bucket
    Condition: CreateLoggingBucket
    Properties:
      BucketName: !Sub "my-logging-bucket-${AWS::Region}"

Outputs:
  LoggingBucketName:
    Description: Name of the logging bucket (if created)
    Value: !Sub "my-logging-bucket-${AWS::Region}"
    Condition: CreateLoggingBucket

```

| Logical Function (JSON Syntax) | YAML Syntax | Description                                                                                               |
| ------------------------------ | ----------- | --------------------------------------------------------------------------------------------------------- |
| `Fn::And`                      | `!And`      | Returns `true` if **all** conditions are true                                                             |
| `Fn::Or`                       | `!Or`       | Returns `true` if **any** condition is true                                                               |
| `Fn::Not`                      | `!Not`      | Returns the **inverse** of a condition                                                                    |
| `Fn::Equals`                   | `!Equals`   | Compares two values for equality                                                                          |
| `Fn::If`                       | `!If`       | Returns one value if condition is true, another if false (used in properties, not in `Conditions:` block) |

| Intrinsic Function | YAML Syntax    | Description                                                       |
| ------------------ | -------------- | ----------------------------------------------------------------- |
| `Ref`              | `!Ref`         | Returns the value of a parameter or the physical ID of a resource |
| `Fn::GetAtt`       | `!GetAtt`      | Retrieves an attribute (e.g., ARN, DNS name) from a resource      |
| `Fn::FindInMap`    | `!FindInMap`   | Looks up a value from a `Mappings` section using keys             |
| `Fn::ImportValue`  | `!ImportValue` | Imports a value exported from another stack                       |
| `Fn::Base64`       | `!Base64`      | Encodes a string to Base64, often used in `UserData` for EC2      |

## CF Rollbacks

- When deploying a stack update, you set the failure option. So the rollback happens automatically based on your selection.
- Stack creation failure options:
  - Default: rollback all to the last working state
  - Option: disable so you can investigate the deployment errors.
    - Use `Preserve successfully deployed resources` option.
- Rollback failures
  - Might be because of manual changes to the stack that you need to manually fix something.
  - once fixed, you can instruct the the rollback to continue (from the CLI, API or AWS Console)
- CF Events will show what failed
- If it is the initial deployment that failed, you can delete the entire stack, no previous state to rollback to.

## CF Service Role

User may not have permission to AWS resources to update them via the CF.

- Create **IAM Service Roles** for CF so it has permission on behalf of the user.
- User must have **IAM:PassRole**
- When creating the stack, use the optional IMA Role setting to choose the IAM Service Role.

## CF Capabilities

When CF is required by the template to create any IAM Resources, you must specify Capabilities:

- **CAPABILITY_NAME_IAM** - if resources or are named.
- **CAPABILITY_IAM**
- **CAPABILITY_AUTO_EXPAND** - when template has macros or nested (dynamic) stacks

If your CF stack doesn't have permission you will get an **InsuffientCapabilitiesException**.

## CF Deletion Policy

Control what happens to a resource if you delete the stack or update the stack with a template that removes a resource.

- Declare in the template YAMl, on the resource object:

  - These work for all AWS resources:
    - Default is: DeletionPolicy=Delete
    - Keep: DeletePolicy=Keep
  - This works for specific resources, e.g. RDS:
    - Snapshot: DeletionPolicy=Snapshot, this will take a final snapshot before deletion happens.

Note: Some resources will fail to delete, e.g. an S3 bucket that is not empty. You will need some automation to delete the bucket content first (see **Custom Resources**).

## CF Stack Policies

- Default: all stack resources can be updated.
- Policy: JSON file to control what can/cannot be updated.
- Goal is to protect from accidental updates.

- Example - this will prevent updating the database:

```
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "LogicalResourceId/ProdDBInstance"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

## CF Termination protection

- Enabled in the Console to prevent accidental deletion of the entire stack.
- CF > Stack > Actions > Enable termination protection
- This can be disabled if you are certain you want to delete the stack.

## Custom Resources

Use:

- When a resource is not supported by AWS CloudFormation
- When needing to run a custom script, e.g. call an AWS Lambda to delete objects in an S3 bucket before deletion in the stack.

## CF Stack Sets

Stack sets allow for Create, update, and delete operations across multiple accounts and regions simultaneously. Only Admin users can create these.
