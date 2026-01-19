# CloudFront

- CloudFront is a Content Delivery Network (CDN). The content gets distributed globally, and is cached at the edge locations.
- DDoS protection, integration with Shield AWS Web Application Firewall.
- When a user requests content, they go via the closest Edge location, the Edge location requests it from the source region (Origin), and caches it, other users in that location use the cached content if they are using the same Edge location.

## Origins

Sources of content for CloudFront:

- S3 Bucket
  - For reading files
  - For uploading files through CloudFront
  - CloudFront -> S3 connection is secured using Origin Access Control (OAC)
- VPC Origin
  - An application hosted in a VPC, using an ALB, NLB, EC2 Instances
- Custom Origin (Http)
  - Anything that uses Http, e.g, S3 Website with static content
  - Any public HTTP backend in or outside of AWS

## CloudFront vs S3 Cross region replication

- CloudFront is for static content, exposed via the Global Edge Network, files are cached.
- S3 Cross Region Replication is for dynamic content, there is no caching, and you must setup the replication in the required regions.

## CloudFront Demo

Prerequisites: Create an S3 bucket with index.html and images. Do not make it a static website or make the file public.

1. CloudFront -> Create Distribution
2. Name and description
3. Distribution type: **Single website or app**, Multi-tenant architecture
4. Origin type: Use Amazon S3
5. Select bucket/origin
6. Origin path is where content is stored
7. Settings
   - Allow private S3 bucket access to CloudFront - checked - CloudFront will create the Access Policies on the S3 bucket to be able to access it.
   - Use recommended origin settings
8. WAF rules - don't need to enable

## Caching

The cache key by default is made up of the hostname and the resource in the URL (not including query parameters), e.g. **d3kw2ebt0wpdsy.cloudfront.net/car.jpg**?some=param

### CloudFront Cache Policy

Enhance the cache key to be more specific with a Cache Policy:

- HTTP Headers:
  - None - fastest caching - no headers are used or forwarded to the Origin
  - Whitelist - Headers included in the whitelist (so not all headers) are forwarded to the Origin.
- Query String:
  - None: No query strings in cache key and none forwarded to the Origin
  - Whitelist: only specified query strings used in cache key and forwarded to the Origin
  - Include All-Except: all expect the specified query string are used in cache key and forwarded to the Origin
  - All: all query string used in cache key and forwarded to the Origin (worst caching performance if there are a lot of query strings)
- Cookies:
  - None
  - Whitelist
  - Include All-Except
  - All

- Configure TTL 0 seconds to 1 year
- Supports Custom or Managed policies provided by AWS.

#### Origin Request Policy

This is where you want headers, query string, cookies forwarded to the Origin but not used in the cache key. You can also add Custom and CloudFront headers at the CloudFront level to forward to the Origin.

- HTTP Headers:
  - None
  - Whitelist
  - All viewer header options
- Query String:
  - None
  - Whitelist
  - All
- Cookies:
  - None
  - Whitelist
  - All

- Supports Custom or Managed policies provided by AWS.

## Cache Invalidations

Cached content at Edge locations expires with the TTL. If you update Origin content, the Edge location won't know. You can invalidate the cache in the AWS Console CloudFront Distribution. Options are to:

- Invalidate the entire cache (e.g. using \*)
- Invalidate specific files (e.g. index.html)
- Invalidate specified paths, (e.g. /images/\*)

When you create the invalidations, you set the object path to invalidate and run it, and wait for it to complete invalidating the cache.

## Cache Behaviours

For when you have multiple different Origins, or you want different cache behaviours for different URL patterns;

- /api/\* -> ALB
- /images/\* -> S3 bucket
- /\* -> EC2 Instance (/\* is the default behaviour - this is always the last to be processed after more specific patterns)

### Cache Behaviours - Use Cases

1. Sign-in Scenario
   - /login path routes to an EC2 Instance to get a sign-in Cookie
   - /\* paths route to Static content in an S3 that requires the Cookie
2. Separating static from dynamic
   - Requests to S3 static content have no specific caching rules - so all content is cached
   - Requests to Dynamic content (EC3 Instance) require more specific caching, so cache based on headers, etc.

## Caching - Demo

Prerequesites: CloudFront and S3 setup from **CloudFront Demo**

1. CloudFront > Distribution > Behaviors
2. Edit default behaviour - (\*):
3. Cache key and origin requests
   - Cache Key Policy - create: these cache keys are passed to the Origin
     - Name
     - TTL
     - Cache Key setting:
       - Headers
       - Query String
       - Cookies
   - Origin request policy - create: these are not cache keys, but passed to Origin
     - Name
     - Origin request setting:
       - Headers
       - Query String
       - Cookies
4. Create a new behaviour
   - Path pattern e.g. /images/\*
   - Origin - S3, EC2 Instance, etc
   - etc.
   - This one is more specific so would be processed before the default (\*)

## VPC Origins

Route traffic though CloudFront to VPCs without having to expose them directly to the internet:

- ALB
- NLB
- EC2 Instances

You don't need public IPs for the resources in the VPC.

## CloudFront Geo Restrictions

You can restrict access to the distributions, using either:

- Allow List - approved countries
- Block List - blocked countries

Uses a 3rd-party Geo-IP database.

### Demo

1. In Cloud Distribution > Security
2. Security - WAF
   - CloudFront geographic restrictions
   - Setup countries on blocked or allow list

## Signed URL and Cookies

Similar to signed url with S3 bucket files, you can allow access to content with a signed-url or cookie:

- Set the expiry: minutes to years
- Signed URLs: for when you want a unique signed url per file - the signed Url is only ever for a single file.
- Signed Cookies: for when you want to give access to multiple files.
- Different from S3 signed URL, in that S3 signed URL assumes access level of the IAM user who generated it, CloudFront signed URL allows access to the path no matter who signed it.

## Signed URL/Cookie flow

1. User authorised and authenticated though an application (we have to build).
2. The application requests the signed URL from CloudFront
3. The application returns the signed URL to the user
4. The user can use the signed URL to request the content through CloudFront.

### Creating Signed URLs

- Two key signers:
  - Trusted key group (recommended) - can use APIs to create and rotate keys
  - AWS account CloudFront key pair - manage keys using the root account and AWS Console (not recommended)
- In CloudFront distribution, create 1+ trusted key groups
- Generate a public/private key pair:
  - Private key in your application generate the signed URL
  - Public key is used by CloudFront to verify the signature - the public key is added to the Trusted key group

## CloudFront Pricing

- Cost of data out of an edge location varies by location
- Price classes vary by the Edge locations that are included:
  - All: most expensive, all Edge locations
  - 200: most regions but excludes the most expensive
  - 100: least expensive regions (includes USA and Europe)

## Origin Groups

This is for high-availability and failover. You set a Primary and Secondary origin, if the primary fails, CloudFront retries the secondary origin for the content.
Using this with S3 buckets, you can also setup the Primary and Secondary S3 buckets in different regions with replication enabled, to get region level DR for S3 buckets.

## Field level encryption

- Additional security on-top of HTTPs encryption
- When data is sent in a POST request from a user to the Edge location, a public key on the Edge location encrypts up to 10 fields in the request (e.g. customer account no.)
- The data sent onwards through to the AWS infrastructure to the Origin, e.g. via Amazon CloudFront > ALB > EC2 Instance has been encrypted (on top of HTTPS), and a private key in the application on the EC2 instance can decrypt the fields.

## CloudFront realtime logs

- Information about real-time requests to CloudFront can be logged to Kinesis Data Streams
- Can get all traffic, or filter traffic (sampling rate % of traffic)
- Can select specific fields and cache behaviours to filter into Kinesis
