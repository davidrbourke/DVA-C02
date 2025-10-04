# AWS S3 Security

## S3 Encryption

S3 Encryption is the encryption of objects 'at-rest'. There are 5 methods to encrypt objects inside S3 Buckets (4 Server side, 1 client side).

### Server-Side Encryption

- You specify a default encryption type for the bucket when creating the S3 bucket.
- You can edit the encryption type for an existing object in the bucket, a new copy is created with the updated encryption type.
- You can override the default encryption type with another type when writing the file to S3, either by selecting the properties for uploading the object in the AWS Console, or using the required HTTP headers.
- You can use a bucket policy to enforce an encryption type by enforcing the required headers.

### Server-side encryption types

#### Amazon S3 Managed Keys (SSE-S3)

- Enabled by default for new buckets and new objects
- Keys managed, and owned by AWS
- AES 256 Encryption
- To use SSE-S3, you must set header to **x-amz-server-side-encryption: "AES-256"** when saving the object
- You have no control over this key and no access to it, you don't pay to access it.

#### Key Management Service Keys (SSE-KMS)

- Using AWS KMS to manage encryption keys
- You control the key and can audit the key's usage in Cloud Trail
- Must set header to **x-amz-server-side-encryption: "aws:kms"**
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

1.  Allow all origins: Access-Controll-Allow-Origin: \*
2.  Allow the origin website only: Access-Controll-Allow-Origin: https://example.com

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
