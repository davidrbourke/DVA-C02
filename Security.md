# AWS Security

- Encryption in flight

  - TLS (newer), SSL, encryption before sending, decryption after receiving.
  - Ensure no man-in-the-middle attack
  - Uses certificates

- Encryption at rest

  - stored encrypted on the server
  - encrypted using data keys
  - server must have access to the keys

- Client side encryption
  - server can never decrypt the data
  - client encryptions and decrypts the data

## AWS Key Management Service (KMS)

- AWS manages the encryption keys
- integrated with IAM, and many AWS services
- can audit calls to use the key using CloudWatch
- Can use KMS yourself though API calls, so not in code

### KMS Types

- Symmetric
  - AES-256, same key to encrypt and decrypt
  - when using integrated with AWS services, you never see this key, AWS manages the API calls to use the key
- Asymmetric
  - RSA & ECC key pairs
  - Public key encrypts, private key decrypts
  - Public key downloaded, only API calls against the private key
  - Uses:
    - encrypt/decrypt scenarios, encrypt data outside of AWS with public key, decrypt inside of AWS using private key.
    - sign/verify signature.

### KMS Key Types

- AWS Owned Keys (free): SSE-S3, SSE-DDB, etc (second part is the service, e.g. S3, DynamoDB).
- AWS Managed Keys (free); aws/service-name, only used with assigned service
- Customer Managed Keys (CMK)
  - Key created in AWS, costs $1 per month
  - Key imported into AWS, costs $1 per month + cost to call API $0.03 per 1000 calls

### Automatic Key Rotation

- AWS Managed Keys: automatically rotated every 1 year
- Customer Managed Keys
  - rotation must be enabled
  - rotation is automatic once enabled, can also be rotated on-demand
  - rotation time is 90 days to 2560 days
  - imported keys can only be rotated manually using the key alias (version)

### KMS and regions

- KMS Keys are scoped per region, the same key cannot be copied to another KMS in another region.
- Example: If Elastic Block Storage (EBS) is encrypted with a KMS key, and you want to copy the EBS to another region, you cannot use the same key in the new region. Steps to move it:
  - Snapshot the EBS - the snapshot will be decrypted with the KMS Key
  - Copy the snapshot to the new region
  - Re-encrypt the snapshot with a new key in the new region
  - Restore the snapshot to an EBS in the new region

### KMS Key Policies

- A **KMS Key Policy** is used to control access to the KMS key
- A default key policy is created that allows all users in an account access to they key
- A custom key policy can be created instead, can define users, roles, and cross-account users that can access the key.

This example KMS Key Policy allows the root user from another IAM account to access the KMS APIs:

```
{
    "Sid": "Allow cross-account access for EBS snapshot copy",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
    },
    "Action": [
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:CreateGrant",
        "kms:GenerateDataKey*",
        "kms:ReEncrypt*"
    ],
    "Resource": "*"
}
```

### Copying a snapshot across an account

1. Create a snapshot encrypted with own key
2. Attach a KMS Key Policy to allow cross account access
3. Share encrypted snapshot
4. In target account, create a copy of the snapshot encrypted with a new key in the second account
   - In this step, the target account must have access to the original source KMS CMK to decrypt the snapshot, and create the copy, to re-encrypt with a new KMS CMK in the target account.
5. Create a volume from the snapshot

### Encrypt/Decrypt API using CLI steps

- This example using the CLI Encrypt and Decrypt command to use KMS for encryption and decryption
- The actual encryption/decryption is performed in KMS, not locally.
- <= 4KB size limit of string to encrypt/decrypt.

#### Encrypt

1. Send secret to Encrypt API with specified CMK
2. KMS checks IAM permission (that user can access KMS CMK)
3. KMS performs the encryption using the specified CMK
4. KMS returns the encrypted string
5. The CLI returns base64-encoded text; you must decode it to binary to save it as a valid ciphertext blob

```
aws kms encrypt \
    --key-id alias/MyKmsKey \
    --plaintext "MySecretData" \
    --query CiphertextBlob \
    --output text | base64 --decode > encrypted_data.bin
```

--key-id is KMS key to use
--query is the part of the JSON response to return, CiphertextBlob or Plantext
--output text returns the result as a simple string instead of part of a json object

#### Decrypt

1. Request decryption API - send the object to decrypt
2. KMS detects the correct CMK to use from metadata in the encrypted blob (don't need to send the CMK), checks IAM permissions (that user can access CMK and decrypt API)
3. KMS performs decryption
4. KMS returns the decrypted string

```
aws kms decrypt \
    --ciphertext-blob fileb://encrypted_data.bin \
    --query Plaintext \
    --output text | base64 --decode
```

### Envelope Encryption

- For secrets > 4KB
- Must use the `GenerateDatKey` API

#### To Encrypt with Envelope Encryption

1. Request to `GenerateDataKey` API check IAM Policy, returns a plaintext Data Encryption Key (DEK)
2. Locally use plantext DEK to encrypt the file
3. New **envelope** file is a file wrapper around the encrypted file + encrypted DEK

#### To Decrypt with Envelope Encryption

1. Call `Decrypt` API, pass encrypted DEK, checks IAM, returns plaintext DEK
2. Use plaintext DEK to decrupt the file locally

#### Encryption SDK

- Use the Encryption SDK (Java, Python, C, JavaScript), or CLI for Envelope Encryption
- SDK has data key caching, to use same key locally (avoid extra API calls), but key is re-used.
- Use `LocalCryptoMaterialsCache` to limit how much the key can be used in the cache, before is it ejected.

Example of Envelope Encryption

```
aws kms generate-data-key \
    --key-id alias/MyKmsKey \
    --key-spec AES_256 \
    --output json > keys.json

# Extract the Plaintext key for immediate encryption
jq -r .Plaintext keys.json | base64 --decode > datakey.bin

# Extract the Encrypted version to save with your data
jq -r .CiphertextBlob keys.json | base64 --decode > datakey_encrypted.bin

# Encrypt data locally
openssl enc -aes-256-cbc -salt -pbkdf2 \
    -in large_file.zip \
    -out large_file.zip.enc \
    -pass file:./datakey.bin

# Remove key locally/clean up
rm datakey.bin keys.json
```

Example of Envelope Decryption

```
aws kms decrypt --ciphertext-blob fileb://datakey_encrypted.bin --query Plaintext --output text | base64 --decode > datakey_plaintext.bin

openssl enc -d -aes-256-cbc -pbkdf2 -in large_file.zip.enc -out large_file.zip -pass file:./datakey_plaintext.bin

```

### API Summary

- `Encrypt`: encrypt up to 4KB of data
- `GenerateDataKey`: return plaintext AND encrypted DEK (for using now)
- `GenerateDataKeyWithoutPlaintext`: return only the encrypted DEK (for using later - you must call the `Decrypt` API to decrypt the encrypted DEK).
- `Decrypt`: decrypt up to 4KB of data, including a DEK.
- `GenerateRandom`: returns a random byte string

### KMS Limits

- Request quotas - you can increate the quota by requesting directly from AWS support
- A **Throttling Exception** is thrown (400 error), if exceeding the request quota.
- Quote is **shared** across all services for a key and all API calls.
- Use Data Encryption Key caching if using the SDK, to reduce chances of hitting the quota limit.
- Symmetric quotas: 5,000 - 30,000 depending on region (per second)
- Asymmetric quotas: 500 for RSA CMK, 300 for Elliptic Curve CMK (per second)

### Lambda with KMS Demo

- Example: Lambda needs a DB password, comes from environment variables that can be encrypted with KMS
- To encrypt the environment variable, can choose a Customer Managed Key (CMK).
- Lambda needs codes to decrypt the env var using the CMK.
- Permissions: Lambda needs permission to access KMS, e.g. using an inline policy.

Example of Python Lambda decrypting the env var.

```
import boto3
import os
from base64 import b64decode

# Initialize the KMS client
kms_client = boto3.client('kms')

# Decrypt outside the handler to cache the value for subsequent warm starts
def get_decrypted_value():
    encrypted = os.environ['MY_SECRET']
    # Decrypt returns a dict; extract 'Plaintext' and decode from binary to string
    decrypted = kms_client.decrypt(
        CiphertextBlob=b64decode(encrypted),
        # If an EncryptionContext was used during encryption, include it here
        EncryptionContext={'LambdaFunctionName': os.environ['AWS_LAMBDA_FUNCTION_NAME']}
    )['Plaintext'].decode('utf-8')
    return decrypted

# Cache the secret value
SECRET_VALUE = get_decrypted_value()

def lambda_handler(event, context):
    print(f"Decrypted secret is: {SECRET_VALUE}")
    return {
        'statusCode': 200,
        'body': 'Successfully used decrypted secret'
    }
```

Note: For symmetric KMS keys, AWS automatically embeds the Key ID into the metadata of the encrypted ciphertext. When your Lambda code calls kms.decrypt(CiphertextBlob=...), the KMS service reads this metadata from the binary blob to identify which Customer Managed Key (CMK) is required to unlock it.

Inline policy attached to the Lambda's execution role:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowLambdaToDecryptSecret",
            "Effect": "Allow",
            "Action": "kms:Decrypt",
            "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/YOUR_KEY_ID"
        }
    ]
}
```

### S3 Bucket Key

- SSE-KMS: this is a setting to decrease the no. of API calls to KMS by 99%.
- A Customer Manager Key is used to generate an S3 Bucket Key
- The S3 Bucket Key is then used to encrypt data in the bucket, now encrypting/decrypting uses the S3 Bucket Key, and reduces the KMS API calls, and cost, and avoids hitting request quota limits.
- Setting in S3 Console:
  1. Enable encryption - AWS Managed Key
  2. Enable bucket key
- FYI, you won't see API calls in KMS in logs (CloudTrail/CloudWatch)

### Key Policy Examples

- Default - anyone in the account can access KMS if their IAM Role allows it.
- Allow federated user - Principal of federated user is set in the policy.
- Principals:
  - AWS Account
  - Root user
  - IAM Roles
  - IAM Role sessions: used for 'Assumed roles', Cognito or SAML.
  - Federated User sessions
  - AWS Service
  - All Principals

Example - root account full access (default):

```
{
  "Version": "2012-10-17",
  "Id": "key-default-1",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

More restricted example:

```
{
  "Sid": "Allow use of the key",
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::111122223333:role/EncryptionApp"},
  "Action": [
    "kms:DescribeKey",
    "kms:GenerateDataKey*",
    "kms:Encrypt",
    "kms:ReEncrypt*",
    "kms:Decrypt"
  ],
  "Resource": "*"
}
```

## Cloud HSM

- KMS is software encryption in AWS
- Hardware Security Module (HSM) is a dedicated device for encryption
- Can manage your own keys (CMK ONLY, not for AWS Managed Keys)
- Device tamper resistant (FIPS 140-2 Level 3)
- Supports symmetric and asymmetric, SSL, TLS
- Not free
- Must use client software to manage the keys in HSM, accessed using HTTPS.
- Integration with Red Shift
- High Availability, multi-AZ replicas.
- HSM is integrated with KMS, in KMS you define a KMS custom key store that connects to HSM.
- Your services, e.g. EBS, RDS, still go via KMS, but KMS connects to the HSM.
- API Calls via KMS are logged (CloudTrail, CloudWatch)
- Supports MFA
- Deployed and managed in a VPC, use VPC peering to share across regions.
- HSM has it's own tooling for you to managed users/permissions - does not use IAM.
- Has hardware cryptographic acceleration

## SSM Parameter Store

(Systems Manager Parameter store, Similar to KMaaS)

- Secure storage for secrets and parameters
- Optional integration with KMS
- Serverless, scalable, has SDK.
- Version tracking of parameters/secrets
- Security is via IAM, can control access to 'paths'.
- Can use Event Bridge for Notifications
- Integrated with Cloud Formation
- Store parameters within a hierarchy path, e.g. /my-app/area/item
- Can access secrets using URL `/aws/reference/secretsmanager/secret_ID_in_secrets_manager`
- There are public parameters, e.g. for latest AMI versions, `/aws/service/ami-amazon-linux-latest/ami-version` (shows latest version if AMI image).

### SSM Parameter Store Tiers

|                                           | Standard Tier | Advanced Tier                      |
| ----------------------------------------- | ------------- | ---------------------------------- |
| Total Parameters (per account per region) | 10,000        | 100,000                            |
| Max parameter size                        | 4KB           | 8KB                                |
| Parameter policies                        | no            | yes                                |
| Cost                                      | None          | Yes                                |
| Storage price                             | None          | $0.05 per advanced param per month |

### Parameter Policies

- Allow TTL (expiration) to delete a parameter
- Event Bridge notification to notify of upcoming expiration

### SSM Parameter Store Demo CLI

- Create Store
  - Create Paramater: name, use slashes for path, e.g. /my-app/dept/etc, use hierarchy.
    - Type:
      - string
      - stringList
      - secureString (secureString get encrypted, need to specify the KMS key, either default of an existing one in your KMS)
    - Data Type:
      - text (value)
      - aws:ec2:image - an AMI ID - gets validated.
      - aws:ssm:integration - referencing AWS secrets manager within the parameter store

### CLI using SSM Parameters

Examples of getting one parameter vs getting many, with decryption for secure strings, and recursive flag to get all parameters in a path.

```
aws ssm get-parameter --name "MyStringParameter"
aws ssm get-parameter --name "MySecureStringParameter" --with-decryption
aws ssm get-parameters-by-path --path "/Prod/Config/" --recursive
aws ssm get-parameters-by-path --path "/Prod/" --recursive --with-decryption
aws ssm get-parameter --name "MyKey" --with-decryption --query "Parameter.Value" --output text
```

## AWS Secrets Manager

- Service for storing secrets
- Can force secret rotation every X number of days
- Use Lambda to automate secret rotation
- Amazon RDS integration - mostly meant for RDS
- Secrets are encrypted using KMS

### Multi-region Secrets

- Can replicate secrets across region, main region has secret, and it is replicated to a secondary region.
- Access is managed by IAM

### Demo

- If using RDS DB Secret; username and password is required when entered, can also chose the RDS DB and it will set the secret as the DB credentials.
- Other type is key-value pair
- Choose encryption key - default or pre-created in KMS
- Configure rotation - automated or disabled, up to 1 year rotation.
- Choose Lambda function - this is called on rotation to update services using the secret.
  - Examples of functions: https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html
- On secret deletion, it can be scheduled for deletion up to 30 days.

## SSM Parameter Store vs Secrets Manager

| Cost                        | SSM Parameter Store                                                                   | Secrets Manager |
| --------------------------- | ------------------------------------------------------------------------------------- | --------------- |
| Cost                        | $                                                                                     | $$$             |
| Automatic rotation          | No - but can use other services to rotate the key, e.g. Event Bridge -> Custom Lambda | Yes + Lambda    |
| KMS Encryption              | Optional                                                                              | Mandatory       |
| Cloud Formation integration | yes                                                                                   | yes             |
| API                         | Simple API                                                                            | None            |

## Cloud Formation integration - Dynamic references

- External values reference in CF templates to SSM Parameter Store and Secrets Manager
- CF retrieves values during create/update/delete operations
  - e.g. get a DB password when creating an RDS
- Supports:
  - SSM: plaintext, SSM-secure, secure strings
  - Secrets Manager: secrets values
- Dynamic references use a specific curly-bracket syntax:
  - Plaintext (SSM): `{{resolve:ssm:parameter-name:version}}`
  - Secure String (SSM-Secure): `{{resolve:ssm-secure:parameter-name:version}}`
  - Secrets Manager: `({{resolve:secretsmanager:secret-id:SecretString:json-key}})`
- Example:

```
AWSTemplateFormatVersion: '2010-09-09'
Description: RDS Instance using SSM dynamic references for credentials.

Resources:
  MyDatabase:
    Type: 'AWS::RDS::DBInstance'
    Properties:
      DBInstanceClass: db.t3.micro
      Engine: mysql
      AllocatedStorage: '20'
      # Dynamic reference for plaintext SSM parameter (Username)
      MasterUsername: '{{resolve:ssm:/my-app/db/username:1}}'
      # Dynamic reference for SecureString SSM parameter (Password)
      MasterUserPassword: '{{resolve:ssm-secure:/my-app/db/password:1}}'
      PubliclyAccessible: false
```

### Options for using Dynamic references

1. In CF template to create RDS, e.g. Aurora

- Must set property `ManageMasterUserPassword: true` for the RDS YAML
- CF RDS creates password and sends it to Secrets Manager
- RDS also manages secret rotation
- Use SF Output template `!GetAtt: MyCluster.MasterUserSecret.SecretArn` to output the password arn

2. Dynamic reference

- Create a secret in CF template using `GenerateSecretString`, it gets put into Secrets Manager at this point also (see the Type).

```
MyRDSSecret:
  Type: 'AWS::SecretsManager::Secret'
  Properties:
    Name: MyDatabaseSecret
    Description: Secret for RDS instance with auto-generated password
    GenerateSecretString:
      SecretStringTemplate: '{"username": "admin"}'
      GenerateStringKey: password
      PasswordLength: 32
      ExcludeCharacters: '"@/\'

```

- CF DB Instance using reference to secret username & password

```
MyRDSInstance:
  Type: 'AWS::RDS::DBInstance'
  Properties:
    DBInstanceClass: db.t3.micro
    Engine: mysql
    AllocatedStorage: '20'
    # Use !Sub or !Join to inject the secret's name/ARN into the dynamic reference
    MasterUsername: !Sub '{{resolve:secretsmanager:${MyRDSSecret}:SecretString:username}}'
    MasterUserPassword: !Sub '{{resolve:secretsmanager:${MyRDSSecret}:SecretString:password}}'
    DBInstanceIdentifier: my-rds-instance

```

- Also create a SecretRDSAttachment block YAML to link the secret to RDS for rotation. Once attached, you can add an AWS::SecretsManager::RotationSchedule to enable automatic rotation.

```
SecretRDSInstanceAttachment:
  Type: 'AWS::SecretsManager::SecretTargetAttachment'
  Properties:
    SecretId: !Ref MyRDSSecret
    TargetId: !Ref MyRDSInstance
    TargetType: 'AWS::RDS::DBInstance'
```

### CloudWatch Logs Encryption

- You can encrypt logs with KMS CMK
- Encryption enabled at log group level, on creation of log group or after by associating it with a CMK.
- You can only do this using the API, not available to do it in the AWS Console.
- The actual content message and timestamp of every log entry ingested after the encryption is enabled are encrypted.
- APIs:
  - `associate-kms-key`: if log group is already created

# Encrypt an existing log group

```
aws logs associate-kms-key \
 --log-group-name "my-existing-log-group" \
 --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/your-key-id"

```

- `create-log-group`: if creating a new log group with encryption

# Encrypt a new log group upon creation

```
aws logs create-log-group \
 --log-group-name "my-secure-log-group" \
 --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/your-key-id"

```

## CodeBuild Security

- To access resources in your VPC, make sure you specify the VPC (and Subnets) for your CodeBuild when creating it.
- Environment variables can reference SSM Parameter Store or Secrets Manager, so you don't have to store secrets in plaintext in CodeBuild environment variables.
- Environment variable types:
- Plaintext
- Parameter - SSM Parameter Store
- Secrets Manager

## AWS Nitro Enclaves

- Process highly sensitive data in an isolated VM, very secure.
- You can do processing in the VM.
- Uses Cryptographic Attestation: only signed code can execute; a signed 'document' of the code is provided to AWS KMS and used to compare the code passed in, to make sure it is the exact code.
- To use:
- Launch a Nitro based EC2 with `EnclaveOption: true`
- Use Nitro CLI to convert app to an Enclave image file (EIF)
- Use the EIF file as an input with the CLI, to create the Enclave instance.

## S3 HTTPS

To enforce SSL (HTTPS) for requests to objects in an S3 bucket, you must use a Bucket Policy that includes a Deny statement triggered when the aws:SecureTransport condition is false.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSSLRequestsOnly",
            "Action": "s3:_",
            "Effect": "Deny",
            "Principal": "_",
            "Resource": [
            "arn:aws:s3:::YOUR-BUCKET-NAME",
            "arn:aws:s3:::YOUR-BUCKET-NAME/*"
            ],
            "Condition": {
                "Bool": {
                "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```
