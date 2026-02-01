# AWS STS

- Security Token Service
- Manages obtaining a temporary token to access AWS Services, for up to 1 hour.
- APIS:
  - `AssumeRole`: assume an IAM Role in your account or cross account
  - `AssumeRoleWithSAML`: returns credentials of user logged in with SAML
  - `AssumeRoleWithWebIdentity`
    - Returns credentials for user logged in with IdP, e.g. google, Facebook, OIDC.
    - AWS recommended using Cognito User Pools instead
  - `GetSessionToken`: for MFA, from a user or AWS account root user.
  - `GetFederationToken`: obtain temporary credentials for federated user.
  - `DecodeAuthorizationMessage`: decode error message when AWS API access denied.

## AssumeRole

- Define an IAM Role within account or cross-account
- Define the principals that can access the IAM Role
- Use AWS STS to retrieve the credentials and impersonate the IAM Role
- Credentials are temporary; 15 minutes to 1 hour.

## STS With MFA

This is where the IAM Policy requires the user to authenticate with MFA before returning credentials to impersonate the IAM Role.

- Use `GetSessionToken` from STS
- IAM Policy must use Condition: `aws:MultiFactorAuthPresent: true`
- `GetSessionToken` returns the following to be used in the request:
  - AccessID
  - SecretKey
  - SessionToken
  - ExpirationDate

### Example STS with MFA Policy

This is a **trust policy**. A Trust policy is assigned to an IAM role, and it include the action sts:AssumeRole. It defines who is allow to assume the role. The User also needs permission on their own role to use sts:AssumeRole.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Alice"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

The Role needs to have separate Policy setup to define what is can do, e.g. which AWS Resources is can access with what permissions.

# Advanced IAM Concepts

## Authorisation Model Evaluation

Flow:

1. User requests access
2. Evaluate all policies
   1. Is there a DENY policy? Yes - DENY access (final), No - continue to next policies
   2. Is there an ALLOW policy? Yes - Allow access (final), No - Deny access (final)

DENY policies are evaluated before any ALLOW policies, so a DENY policy will prevent continuing the evaluation and return DENY.

## IAM Policies & S3 Bucket Policies

- IAM Policy is on User, Roles, Groups
- S3 Policy is on the S3 bucket
- Both policies are combined to determine what action can be performed on the S3.

Examples - IAM Policy on EC2

| IAM on EC2  | S3 Policy            | Outcome      |
| ----------- | -------------------- | ------------ |
| Read (R)    | None                 | EC2 can Read |
| R Write (W) | DENY to IAM Role     | Denied       |
| Empty       | RW to IAM Role       | EC2 can RW   |
| DENY to S3  | RW Allow to IAM Role | Denied       |

## Dynamic IAM Policy

This is to avoid having lots of policies for e.g., each user, you can use a variable of the `${aws.username}` Dynamically in a single policy.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-company-bucket",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "${aws:username}/"
          ]
        }
      }
    },
    {
      "Sid": "AllowUserFolderAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-company-bucket/${aws:username}/*"
    }
  ]
}
```

## Inline Vs Managed Policies

- AWS Managed Policy
  - Defined and updated by AWS, usually prefixed with `AWS`
  - Good for power/admin users
- Customer Managed Policy
  - More granular, and reusable
  - Version controlled and can rollback changes
  - Best practice, these are the policies you create.
- Inline
  - A policy embedded directly inside a single IAM Identity (User/role/group)
  - Not reusable by another identity
  - When the identity is deleted, the inline policy is also deleted
  - Use for one-off, specific permissions
  - Max size is 2048 bytes, limits how many permissions can be put in it.

## iam:PassRole and iam:GetRole

- When creating some services, you can attach an IAM role to the service so the service can access AWS resources. You are not passing a policy — you are passing a role.
- Your IAM user must have iam:PassRole permission to attach an IAM role to a service (e.g., EC2, Lambda).
- iam:PassRole does not let you pass a role to any service. The role's trust policy must allow the service to assume the role.

User can only pass the roles listed - on this IAM User Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPassingSpecificRoles",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::123456789012:role/EC2AppRole",
        "arn:aws:iam::123456789012:role/LambdaExecutionRole"
      ]
    }
  ]
}
```

# Microsoft AD and AWS

AWS Directory Services is a way to create an Active Directory on AWS. Purpose is to create EC2 instance running Windows and they need Active Directory. 3 types:

## AWS Managed Microsoft AD

- Create an AD in AWS, manage users locally, +MFA
- Can establish a 'trust' connection with on-prem AD, to lookup users not found in AWS (up to 500,000 users)

## AD Connector

- A gateway proxy to connect to on-prem AD
- Users are managed entirely in Microsoft AD on-prem, supports MFA.

## Simple AD

- AD Compatible (not Microsoft) Linux-samba
- Cannot be joined with on-prem AD (up to 5,000 users)
