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

Prerequesites: Create an S3 bucket with index.html and images. Do not make it a static website or make the file public.

1. CloudFront -> Create Distribution
2. Name and description
3. Distribution type: **Single website or app**, Multi-tenant architecture
4. Origin type: Use Amazon S3
5. Select bucket/origin
6. Origin path is where content is stored
7. Settings
   - Allow privaate S3 bucket access to CloudFront - checked - CloudFront will create the Access Policies on the S3 bucket to be able to access it.
   - Use recommended origin settings
8. WAF rules - don't need to enable

## Caching

The cache key by default is made up of the hostname and the resource in the URL (not including query parameters), e.g. **d3kw2ebt0wpdsy.cloudfront.net/car.jpg**?some=param

### CloudFront Cache Policy

Enhance the cache key to be more specific with a Cache Policy:

- HTTP Headers:
  - None - fastest caching - no headers are used or forwarded to the Origin
  - Whitelist - Headers included in the whitelst (so not all headers) are forwarded to the Origin.
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
