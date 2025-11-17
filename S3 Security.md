# AWS S3 Security

## S3 Encryption

S3 Encryption is the encryption of objects 'at-rest'. There are 5 methods to encrypt objects inside S3 Buckets (4 Server side, 1 client side).

### Server-Side Encryption

- You specify a default encryption type for the bucket when creating the S3 bucket.
- You can edit the encryption type for an existing object in the bucket, a new copy is created with the updated encryption type.
- You can override the default encryption type with another type when writing the file to S3, either by selecting the properties for uploading the object in the AWS Console, or using the required HTTP headers.
- You can use a bucket policy to enforce an encryption type by enforcing the required headers.

Example of a bucket policy to enforce an encryption type of SSE-KMS:

```
{
  "Version": "2012-10-17",
   "Statement": [
    {
       "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### Server-side encryption types

#### Amazon S3 Managed Keys (SSE-S3)

- **Enabled by default for new buckets and new objects**
- Keys managed, and owned by AWS
- AES 256 Encryption
- To use SSE-S3, you must set header to **x-amz-server-side-encryption: "AES-256"** when saving the object (if overridding a different default encryption).
- You have no control over this key and no access to it, you don't pay to access it.

#### Key Management Service Keys (SSE-KMS)

- Using AWS KMS to manage encryption keys
- You control the key and can audit the key's usage in Cloud Trail
- You can recycle the key
- Must set header to **x-amz-server-side-encryption: "aws:kms"** (if overridding a different default encryption).
- You may be impacted by KMS limits and you pay for accesses to the key when the object is encrypted/decrypted.
- Each call to encrypt or decrypt will call the KMS API, so may going into a throttling limit (5500, 10000, 30000 req/s based on region)
- You can request a quota to increase the limit using Service Quota Console.

#### Dual-layer server-side encryption with AWS KMS (DSSE-KMS)

- Double encryption

#### Customer-Provided Keys (SSE-C)

- Managing encryption with your own keys
- The request to upload the object must be HTTPS
- The key is included in a header in the request, in both requests to save the object (to encrypt) and requests to read it (to decrypt)
- This cannot be done when uploading files in the AWS Console, must be done from the CLI, or app.

### Client-Side Encryption

- The encryption and decryption happens in the client application outside of AWS
- The object is sent already encrypted, and downloaded encrypted.
- The client app is responsible for the encryption and decryption.

## Encryption in Transit

- S3 HTTP endpoint is not encrypted in-flight
- S3 HTTPS endpoints is encrypted in-flight
- Use a Bucket Policy to enforce HTTPS, by denying in-secure transport:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:PutObject",
        "s3:PostObject",
        "s3:DeleteObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

## CORS (Cross Origin Resource Sharing)

- **CORS** is a browser-enforced security mechanism that prevents JavaScript running on one origin (e.g., `https://example.com`) from accessing responses from a different origin (e.g., `https://api.other.com`) unless explicitly permitted by the server.

- **Example**:  
  A user visits `https://example.com`. The page includes JavaScript that sends a request to `https://api.other.com`.

  - The browser _does_ send the request to `api.other.com`.
  - However, the browser will **block access to the response** unless `api.other.com` includes the header:  
    `Access-Control-Allow-Origin: https://example.com`

  - This header tells the browser:

    > “It’s okay for scripts from `https://example.com` to read this response.”

  - The browser doesn’t “execute” the response — it simply **exposes it to JavaScript** if the origin is allowed.  
    Without that header, the response is received but **inaccessible to the script**.

### S3 CORS

CORS needs to be enabled in Amazon S3 when you want web applications hosted on one domain to access resources (like images, fonts, or data files) stored in an S3 bucket that’s considered a different origin by the browser. Options:

1.  Allow all origins: Access-Control-Allow-Origin: \*
2.  Allow the origin website only: Access-Control-Allow-Origin: https://example.com

### Setup CORS in S3

- In AWS S3 Console > Permission
- CORS Setting - add a JSON block as below (ensure no slash '/' at the end of the AllowedOrigins URLs):

```
[
  {
    "AllowedHeaders": [
      "Authorization"
    ],
    "AllowedMethods": [
      "GET",
    ],
    "AllowedOrigins": [
      "https://example.com"
    ],
    "ExposeHeaders": [
      "ETag",
      "x-amz-request-id"
    ],
    "MaxAgeSeconds": 3000
  }
]
```

## S3 MFA Delete

MFA Delete is an option that can be enabled for S3 buckets so a Multifactor Authentication generated code must be supplied to:

- Permanently delete an object
- Suspend versioning on the bucket

### Other notes for MFA Delete

- The MFA code is not required for enabling versioning, or listing deleted versions.
- Versioning must be enabled on the bucket to enable MFA delete.
- Only the bucket owner (root account) can enable/disable MFA Delete.

### Demo

1. On the S3 Bucket > Properties > Edit bucket versioning
2. Mutlifactor authentication delete will be disabled - it cannot be enabled via the Console. Use the CLI.
3. IAM root account must have an MFA device setup already
   - From Security Credential - Access Keys: Create an Access Key ID and Secret Access Key for the root user - to configure the CLI.
4. Configure AWS CLI to use the root account

```
aws configure --profile root-mfa-delete-demo
AWS Access Key ID:
AWS Secret Access Key:
Default region name:

# enable MFA delete
# arn-of-mfa-device: found in the Console for the MFA setup
# mfa-code: generated by the MFA device
aws s3api put-bucket-versioning --bucket {bucketname} --versioning-configuration Status=Enabled,MFADelete=Enabled --mfa "arn-of-mfa-device mfa-code" --profile root-mfa-delete=demo

# delete MFA delete
aws s3api put-bucket-versioning --bucket {bucketname} --versioning-configuration Status=Enabled,MFADelete=Disabled --mfa "arn-of-mfa-device mfa-code" --profile root-mfa-delete=demo

```

5. Best practice

- Only use the root account to enable/disable MFA Delete
- Delete the root account local profile after using it

## S3 Access Logs

- Logging all requests into the S3 buckets - logged to another bucket.
- Target bucket must be in the same region.
- Enable Access logging
- Log format: https://docs.aws.amazon.com/AmazonS3/latest/userguide/LogFormat.html
- Do not set the log bucket to be the same as the monitored bucket, creates a logging loop.

### Demo

1. Create 2 S3 buckets, a source and a logging target.
2. Source bucket > Properties > Server access logging

- Enable
- Destination: set to logging target bucket
  - The target bucket Access Policy will be updated to allow the source to access the target.
- Log object key format

## S3 Presigned URL

For a scenario where the Users are not part of AWS, and you don't want to make the bucket public. You can generate a pre-signed URL.

- URL Expiration
  - S3 Console generated: 1 min to 12 hours
  - AWS CLI generated: --expire-in seconds to 168 hours
- The URL inherits the permissions GET/PUT, etc of the AWS User the pre-signed URL was generated under.

## S3 Access points

For a scenario where you have separate directories in the S3 bucket, and you want specific user groups to only access their relevant directories. To avoid creating a complex Access Policy on the S3 bucket, Access points can be used. Access points move the complexity of the access policy away from the bucket and onto the specific access points.

- Each Access point has its own DNS name
- Each Access point has its own Access policy, e.g., the bucket directory it has read and/or write access to.
- Users will need the correct IAM policy to access the Access point, they go via the access point and not directly to the S3 bucket.
- For and EC2 instance in a VPC:
  - A VPC Endpoints must be created in the VPC with a policy allowing it to access the the Access point
  - | VPC: EC2 Instance -> VPC Endpoint | Access point (VPC Origin) -> S3 Bucket |

## S3 Object Lamda

For a scenario where some operation needs to be run on the object extracted from the S3 bucket before it is returned, e.g. an additional analytics application needs to get the object with some data redacted. An AWS Lambda can be used, the analytics application requests to the AWS Lambda, the AWS Lamdba extracts the object and redacts the data, and returns the redacted object. There are access points between each resource:

- User application -> S3 Object Lambda Access Point -> AWS Lambda -> S3 Access Point -> S3 Bucket.
- In this scenario, multiple different lambdas can be setup to perform operations on the S3 Bucket object, without having to maintain separate modified copies of the Object.
