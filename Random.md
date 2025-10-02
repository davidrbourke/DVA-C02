# Random

These notes are from section 12 which seems to be some general miscellaneous things that might have been missed.

## AWS EC2 Instance Metadata

EC2 Instance Metadata is metadata about the EC2 Instance that the EC2 instance can query about itself, an example would be knowing which IAM Role is attached to the instance and the (temporary) credentials secret to use for the role to access AWS services.

### IMDSv1

- Version 1 of the metadata that is less secure, it is accessible at: http://169.254.169.254/latest/meta-data/ .
- You can execute this from the CLI to curl to the url to get the metadata, it is a local link only accessible from the EC2 Instance, but the metadata is hosted outside of the instance.
- When setting up the EC2 Instance you can choose whether to make IMDSv1 or IMDSv2 available (or both on older instance types).

### IMDSv2

- Version 2 is the only version available on newer instance types (after mid-2024), it requires getting a token to access the metadata.
- Commands to get the token and access the data:
  - TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  - Example to get the instance-id: `curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id`
  - Example to get the security-credentials: `curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/`
  - The top level request for metadata returns a list of directories or files, you can build the request to drill down into the metadata.

## AWS CLI with MFA

1. In the Console: IAM > Users > _user_ > Security credentials > Assign an MFA device
   - E.g. Virtual MFA Device (Authy), and go through the setup steps.
   - For the assigned device, an arn with be generated, e.g., arn:aws:iam:_id_:mfa/user
2. In Console: `aws sts get-session-token --serial-number [arn for mfa device] --token-code [code from mfa device]`
   - This will return temporary Credentials including; AccessKeyId, SecretAccessKey, SessionToken, Expiration.
3. Configure an AWS Profile locally:
   - `aws configure --profile mfa`
   - Edit the credentials file to set the credentials returned from the get-session-token command to the profile:
     - [mfa]
     - aws_access_key_id:
     - aws_secret_access_key:
     - aws_session_token:
4. When using the [mfa] profile locally now for the CLI, it will use these credentials generated using MFA.
   - E.g. Listing S3 buckets in the account region: `aws s3 ls --profile mfa`

## AWS SDK

Software Development Kit (SDK), available in various languages. When using the SDK, if you don't a region or configure a default region when using the SDK, then it will use use-east-1 by default.

## API Rate Limits

Limits when calling the API:

- DescribeInstances for EC2: max 100 calls per second
- GetObject for S3: max 5,500 GET per second per prefix
- For intermittent errors with hitting the limits (throttling exception), use an exponential backoff strategy.
- For consistent errors with hitting limits, request an API throttling limit increase from AWS.

### Exponential backoff

- SDK has a retry mechanism included.
- If calling API with custom code, you must implement your own retry mechanism.
  - Only implement retries on 5xx http response codes (server errors)
  - Don't implement retries on 4xx errors
- Example - double wait on retry:
  - Request 1
  - Request 2 - after 1 second
  - Request 3 - after 3 seconds
  - Request 4 - after 4 seconds, etc, 8, 16...

Service Quotas/Limits
Limits how many resources can be run in an account:

- Running On Demand Standard Instances: max 1152 vCPU
- Use Service Quota API, or open a support ticket to increate the limits.

## AWS CLI Credentials Provider Chain

This the order of where credentials are looked for when using the CLI, if they are not provided at the level, then the level below is used until a credential is found:

1. Command line inline credentials, e.g. --region, --output, and --profile (aws s3 ls --profile mfa)
2. Environment variables: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and AWS_SESSION_TOKEN
3. CLI Credentials file: ~/.aws/credentials
4. CLI Config file: ~/.aws/config
5. For ECS: Container credentials
6. For EC2 Instance Profiles: Instance profile credentials

If using an SDK, e.g. Java, the Java system properties would replace priority 1 for Command line credentials, but the rest would be the same.

This is important because if you are using an EC2 Instance Profile credentials with an IAM role, you want to avoid having some credentials higher up the chain, e.g. in application settings, or environment variables that would use a different credential that provides a level of access that you don't want, and can also be confusing.

### App Credentials Best Practice

- Never store credentials in your code
- Use IAM roles for the service if within AWS
- If from outside AWS, use environment variables or named profiles.

## API Signing with SigV4

When using the CLI or SDK to interact with AWS services, the request is automatically signed to ensure you are who you say you are and ensure the request has not been tampered with.
When using an HTTP API Request instead, e.g. to get an S3 bucket object, you need to include a SigV4 signed signature of your request signed with your secret key, etc. There are two ways to include the signature in the request:

1. HTTP Authorization Header

```
GET /my-bucket/my-object.txt HTTP/1.1
Host: s3.amazonaws.com
X-Amz-Date: 20251002T213200Z
Authorization: AWS4-HMAC-SHA256 Credential=AKIAEXAMPLE/20251002/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-date, Signature=abcdef1234567890...
```

2. HTTP Query String (X-Amz-Signature)

```
GET https://s3.amazonaws.com/my-bucket/my-object.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAEXAMPLE%2F20251002%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20251002T213200Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=abcdef1234567890...
```
