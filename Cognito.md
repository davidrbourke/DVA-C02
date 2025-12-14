# Cognito

A way for **external** to AWS users to have an identity.

- Cognito User Pools:
  - Sign in functionality for app users
  - Integrate with API Gateway & ALB
- Cognito Identity Pools:
  - Provide temporary AWS credentials so users can access AWS resource directly.

## Cognito User Pools (CUP)

- A serverless DB for web & mobile app users
- Simple login: username & password
- Email and phone number
- MFA
- Or optionally, federated ID with Google, Facebook, SAML, OpenID Connect, etc.
- Block users for compromised credentials
- When user logs in, they get back an JWT

### Integration

- API Gateway
  - User authenticates with CUP, gets a JWT, user app passes JWT to API Gateway, it can evaluate the JWT against Cognito, to allow request on to backend.
- ALB
  - Using ALB Listeners and Rules, you can authenticate user against CUP, and then allow request onto the backend.

### Demo CUP

- Amazon Cognito, create User pools
- Options
  - Traditional webapp application (\*)
  - SPA
  - Mobile app
  - Machine to machine
- App name
- Sign in options
  - Email (\*)
  - phone number
  - Username
  - Social sign-in: setup federated users
- Return URL - where users go after successful signin
- Create
- In Created user pool, configure:

  - App clients - the app setup for CUP
  - Users
  - Groups to group users
  - Authentication methods
    - Send email with Amazon SES
    - Send email with Cognito (for testing)
    - SMS
    - Password policy
    - Passkey (for biometrics)
  - Sign-in
    - Can't edit original sign-in option (e.g. email)
    - MFA (setup)
    - User account recovery
  - Sign-up
    - Required attributes for user to have to register with.
  - Social and external providers
    - Where to setup using federated users, e.g. google
  - Extensions
    - Setup various Lambda triggers or events, Sign-up, authentication, messaging (customise SMS, etc)
  - Security
    - WAF
    - Threat protection
    - Log streaming (for suspicious requests)
  - Branding

    - Domain - can override automatically created domain with your custom domain.
    - Custom Domain
      - If using a custom domain, **you must create an ACM certificate in us-east-1 - no other region option**.
      - Custom domain must be defined in the app integration section of CUP, this is configuration for all app clients.
    - What you can customise:
      - Css, Logo, URL
      - You cannot override the javascript

  - Managed login
    - Can customise the login layout/design
    - Test with Overview -> view login page
    - AKA Hosted UI, CUP provides a nice login UI, that you can customise.
    - Custom Domains
  - Can edit messages, e.g. email and SMS

### CUP - Lambda Triggers

- Can invoke lambdas on events

# User Pool Flow Operations

| User Pool Flow                 | Operation                           | Description                                                           |
| ------------------------------ | ----------------------------------- | --------------------------------------------------------------------- |
| **Custom Authentication Flow** | Define Auth Challenge               | Determines the next challenge in a custom auth flow                   |
|                                | Create Auth Challenge               | Creates a challenge in a custom auth flow                             |
|                                | Verify Auth Challenge Response      | Determines if a response is correct in a custom auth flow             |
| **Authentication Events**      | Pre authentication Lambda trigger   | Custom validation to accept or deny the sign-in request               |
|                                | Post authentication Lambda trigger  | Logs events for custom analytics                                      |
|                                | Pre token generation Lambda trigger | Augments or suppresses token claims                                   |
| **Sign-Up**                    | Pre sign-up Lambda trigger          | Performs custom validation that accepts or denies the sign-up request |
|                                | Post confirmation Lambda trigger    | Adds custom welcome messages or event logging for custom analytics    |
|                                | Migrate user Lambda trigger         | Migrates a user from an existing user directory to user pools         |
| **Messages**                   | Custom message Lambda trigger       | Performs advanced customization and localization of messages          |
| **Token Creation**             | Pre token generation Lambda trigger | Adds or removes attributes in ID and access tokens                    |
| **Email and SMS Providers**    | Custom sender Lambda triggers       | Uses a third-party provider to send SMS and email messages            |

### Adaptive Authentication

- Block sign-ins for suspicious logins, Cognito provides a risk score on each sign-in. You can require prompting for second MFA only when there is a risk. Example, a login from a new location/IP.
- Phone and email authentication can be u8sed for any account takeover recovery.
- All activity is logged to CloudWatch.

### CUP JWT

- JWT returned from login to CUP.
- Base64 encoded: Header, Payload, Signature
- Decoded when validated.
- Payload can have additional information.
  - `sub` UUID, the ID of the user, can use this to get info from the DB about the user.
- The signature is verified to trust the payload.
- Libraries to verify the JWT.

## ALB - User Authentication

- ALB can securely authenticate the user, to take this away from the application.
- Authenticate in a few ways:
  - Identity Provider (IdP): OpenID connect compliant (OIDC)
  - Cognito User Pools
    - For social IDs, Google, etc
    - Corporate using Microsoft AD, SAML, LDAP
- Must use an HTTPS listener to set authenticate-oidc OR authenticate-cognito rules.

### ALB - Cognito User Pools

- Flow:
  - Request comes into ALB
  - ALB checks with Cognito
  - on valid authentication allow traffic to forward to backend
  - invalid authentication - reject request
- ALB setup for CUP
  - Create the CUP
  - In ALB set Amazon Cognito as the Identity Provider, the app client ID, and the Cognito User Pool.

### ALB with OIDC

- Flow:
  - ALB redirects user to Authentication Endpoint (for OIDC provider)
  - Returns a Grant code
  - User request ALB
  - ALB sends code/token back to identity provider to validate the token
  - Identity provider returns an access token/ID token
  - ALB request back to ID provider to get claims
  - ALB forwards traffic onto backend

## Cognito Identity Pools

- User is outside of AWS, they want to access into AWS environment, e.g. S3, DynamoDB table.
- Give user temporary access though CIP.
- Identity sources:
  - Allows user to login via a trusted 3d party, e.g. Google, etc.
  - SAML, or OpenID
  - A user in Cognito User Pools
  - Developer Authenticated Identities
  - Can even use unauthenticated guest access
- Users can access AWS service directly or via API Gateway
  - IAM policies can be applied to the credentials defined in Cognito
    - Can be fine grained based on user ID

### Flow

- Web/mobile user connects to identity provider to get a token
- Requests to CIP, to exchange token for temporary AWS credentials
  - CIP gets this from STS
- User can use credentials to access AWS service

### CIP with CUP

- User logs into CUP to get token - using Identity provider setup, but all user end up in the CUP DB.
- User requests to CIP to exchange token for temporary credentials
  -CIP gets this from STS
- User can use credentials to access AWS service

### IAM Roles

- Can define roles for authenticated and guest users
- Can define rules to choose which IAM role to apply to the user based on user ID
- Can partition your user's access using policy variables
- Roles must have a trust policy of Cognito Identity Pools

Example of a IAM policy where users can only access a bucket matching their `sub` (user ID).

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME",
      "Condition": {
        "StringLike": {
          "cognito-identity.amazonaws.com:sub": "PREFIX*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/${cognito-identity.amazonaws.com:sub}/*",
      "Condition": {
        "StringLike": {
          "cognito-identity.amazonaws.com:sub": "PREFIX*"
        }
      }
    }
  ]
}
```

### CIP Demo

- Choose Authenticated or unauthenticated access (guest)
  - Authenticated
    - Choose source
      - CUP, Facebook, google, SAML, Open ID, etc
- Choose CUP
- Create new IAM Role
  - Name
  - Setup with the permissions of what resource they need to access
- Guest role
  - Also need to setup permission of IAM role
- Source configuration
  - CUP ID, and App ID
  - Role settings, choose default role created
  - Or setup role with rules, etc, or can update these roles later via the IAM Console.
  - Can setup Attribute mapping here
    - Attributes that can be used in the IAM Role
- Create Identity Pool - gets created
- Next need to setup you app now to use the Amazon Cognito Identity pool

## CUP vs CIP

- CUP is for **authentication**, DB of Users, authenticated with different providers.
- CIP is for **authorisation** of what Authenticated user can do with accessing AWS environment resources.
