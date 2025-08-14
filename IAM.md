# Identity and Access Management (IAM)

## Users and Groups

- User: a user is an account for a physical user.
- Groups: only contain users, a Group can have a set of permissions by assigning Policies to the group.
- Users don't have to be in a group, Policies can be applied directly to a user, but it is not best practice.
- Users can be in multiple groups
- The Policies can be configured with json documents or in a UI in the AWS Console.

### Example

1. A user is created called user-one, and setup with a password, to change on first login. There is also an option to allow access to the AWS Console.
2. A group can be created called basic-user-group.
3. The group is added to the user's groups.
4. The group can be given permissions by assigning existing **Managed policies** (up to 10 Managed Policies), e.g.
   - AdministratorAccess
   - AmazonEC2ReadOnlyAccess
   - AmazonS3FullAccess
5. A Custom Policy can be created called my-custom-policy-1 (see **IAM Policies**).

## MFA options

- Virtual MFA (Authenticator App)
  - Google authenticator (phone only)
  - Authy (phone only)
- U2F Universal 2nd factor (e.g. Yubi key – Security Key)
- Hardware key fob MFA device (Time-based one-time password TOTP) - by Gemalto.
- Hardware key fob MFA device for GovCloud (US) - by SurePassID.

## Password rules can be configured

Password expiration between 1 and 1095 days

## Ways to access AWS

- AWS Management Console – the web UI (password + MFA)
- AWS CLI (command line interface) – access keys
- AWS Software Developer Kit – for code – access keys

## Access Keys

These are passwords generated in AWS Management Console:

- Access Key ID
- Secret Access Key

## IAM Policies

JSON documents that define a set of permissions for making requests to AWS service, and can be used by IAM Users, Groups and IAM Roles.
Principal is available for resource-based policy, not used for identity-based policy (attached to user/group/role).

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::123456789012:user/SomeUser"
        },
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::my-bucket/*"
        }
    ]
}
```

## CLI Command

aws iam list-users

## IAM Roles

- Permissions you give to a service to perform actions, uses IAM Roles, e.g. and EC2 trying to access some information using the IAM Role.
- The Role is created for the Service, and a Policy is assigned to the Role.

## Reports

- IAM Credentials Report – lists all users and the status of their various credentials
- IAM Access Advisor – lists the service permissions granted to a user and when last accessed

## Best practice

- Do not use root account, setup your own admin account
- One physical user = one AWS user
- Use group level permissions assigned to a user and assign permissions to groups
- Create a strong password policy (+ MFA)
- Create and use Roles for giving permissions to AWS services
- Use access keys for programmatic access
- Audit your permissions with the reports

## Shared responsibility model for IAM

- AWS is responsible for:
  - infrastructure
  - configuration and vulnerability analysis
  - compliance validation.
- The user (you) is responsible for:
  - user, groups, roles management
  - monitoring
  - enabling MFA
  - key rotation
  - using IAM tools to apply permissions
  - analysing access patterns and reviewing permissions.
