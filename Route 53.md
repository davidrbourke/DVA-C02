# Route 53

## Domain Name System (DNS)

- Translates human readable hostnames into the machine IP
- www.someapp.com => 111.111.111.11
- Hierarchical naming structure:
  .com
  example.com
  www.example.com
  api.example.com
- Domain Registrar: a place to register the domain names, e.g Amazong Route 53, GoDaddy, etc
- DNS Records: A, AAAA, CNAME, etc
- Zone File: contains DNS records to match hostnames to IP addresses
- Name Server: resolves DNS queries
- Root Level Domain (RLD): .
- Top Level Domain (TLD): .com, .co.uk
- Second Level Domain (SLD): **example** in example.com
- Sub Domain: **www** in www.example.com
- FQDN Fully Qualified Domain Name: **api** in api.www.example.com
- Protocol: **http** in http://api.wwww.example.com
- URL: The entire thing.

### How it works

- Web Browser requests a url, e.g. example.com
- Local DNS server (e.g. ISPs) if it knows, returns IP address, otherwise:
- Local DNS Server requests from Root DNS Server (RLD), if it knows, returns IP, otherwise returns TLD server (e.g. for .com) to lookup next.
- Local DNS Server requests from TLD server, if it knows, returns IP, otherwise returns Sub-Domain Register, e.g. for example.com - Amazon, GoDaddy, etc
- Local DNS Server requests from SLD DNS Server, returns IP
- Local DNS Server returns IP to browser and caches it with TTL for future requests
- Browsers requests to IP address

## Route 53 Overview

- A managed, scalable, authoritative DNS (authoritative means you can update DNS records)
- A domain name registrar, you can regiser domain names (e.g. myapp.com)
- Health checks on your resources
- 100% Availability (only service in AWS to be 100%)

## Route 53 Records

- Methods for routing traffic
- Each record:
  - Domain/subdomain name, e.g., myapp.com
  - Record type, e.g., A, AAAA, CNAME, NS, etc
  - Value, e.g., IP 1.0.1.1
  - Routing Policy: how Route 53 responds to queries
  - TTL: length of time to cache record with DNS resolvers

## Records Types

- A: hostname mapping to IPv4
- AAAA: hostname mapping to IPv6
- CNAME: hostname mapping to another hostname
  - Target CNAME must be mapped to an A or AAAA
  - Can only create for a top level domain, e.g., www.myapp.com is ok, myapp.com is not
- NS: defines which Name Servers are authoritative for the domain, so Root and TLD server know where to look for the domain.

## Hosted Zones

- A container for DNS records, defines how traffic should be routed to domains/subdomains, and whether public or private.
- Public Hosted Zones: DNS records for how to route traffic on the internet
- Private Hosted Zones: DNS record for routing private traffic, e.g. in a VPC, internally in a company, for private domain names.
  - So you can identify private services (e.g., EC2 Instances) within you VPC with a domain name instead of an IP address.
- Cost: $0.50 per month per hosted zone

## Register a Domain Name

You have to pay to register a domain in AWS Route 53, e.g. example.com, but you can buy one here.

- Turn on privary protection so your person details are not made public for the DNS record lookup - can lead to spam.
- DNS Records get automatically created:
  - A, NS, CNAME
  - NS Namespace record will point to AWS Route 53 servers as the source.

## Create a DNS Record

Create a record for the domain name:

- Record name, e.g. test for test.example.com
- Record type, e.g. A, CNAME, etc
- Value info - the target details, e.g. IP address if using A record.
- TTL - time to live, how long before the DNS expires, e.g. default 300 seconds
- Routing policy, e.g Simple
- Query the DNS record from console - In AWS Cloud Shell

```
sudo yum install -y bind-utils
nslookup test.example.com # returns non-authoritate answer of target IP
dig test.example.com # returns more details about the record type and TTL
```

## Route 53 Setup

Pre-req - have EC2 instances created with an ALB.

## CNAME vs Alias

- CNAME points a hostname to any other hostname, e.g. app.myapp.com to test.mytest.co.uk, only for non-root domain
  - Clarification: the . is the Root Level Domain for DNS resolution. In AWS Route 53, the root-domain refers to e.g., example.com. You can't change at this level the routing, only the non-root, e.g. app.example.com or test.example or www.example, the app/tests/www is non-root.
- Alias: In AWS an Alias points a hostname to an AWS Resource, allowable for non-root and root domain (root domain is also called the 'zone apex').
  - E.g., you could point explain.com to an AWS resource.
  - Free.
  - Can ONLY point to an AWS resource, is an extension of DNS within AWS.
  - Of type A or AAAA.
  - TTL is set automatically
  - Targets: ELB, CloudFront distributions, API Gateway, Elastic Beanstalk environments, S3 websites, VPC interface endpoints, Global accelerator, Route 53 record in same hosted zone.
  - Cannot set an Alias to an EC2 DNS Name.
  - When creating an Alias in Route 53, you would do it like how you created an A or AAAA name record, but check the 'Alias' toggle, and the list of AWS services you can route to will be displayed.

## Routing Policies

This is how the DNS responds to requests for information, it is not concerned with actually routing traffic.

- Simple
  - Route traffic to a single resource
  - Can specify multiple values, e.g. IPs, a random one is chosen by the client. For Alias, only one AWS resource can be specified.
  - Can't be assocaited with health checks.
- Weighted
  - With multiple target values, you can weight the targets as a percentage of overall requests to go to a specific target, e.g. 10, 10, 30 = 50, traffic would be distributed 20%, 20%, 60%.
  - DNS records - all targets must be the same name and type
  - Example uses: testing a new service, to route only a small amount of traffic, or for basic load balancing.
  - A weight of 0 would send no traffic to a resource, but if all are 0, it means traffic is distributed evenly.
  - To create in AWS, you add multiple DNS records with the same 'name' (e.g. 'app' for app.example.com), and then configure each with the same DNS type (A, AAAA, etc), and assign the target and the weighting, and a unique ID for the DNS record.
- Failover
  - For handling unhealthy target resources, and directing traffic to a health instance.
  - Must specify the Failover record type (Primary/Secondary)
  - Primary: Must be associated with a **health check** (see Health Checks).
  - Secondary: Optionally associated with a health check.
  - Traffic is routed to the Primary, if the health check starts failing, traffic will be routed to the Secondary.
- Latency based
  - In the instance you have resources in mutiple regions, AWS Route 53 will calculate the target resource with the lowest latency for the users request, e.g. a user in the UK would be directed to a resource nearer to the UK, rather than a resource in the USA.
  - Can be used with health checks.
  - To create in AWS, you add multiple DNS records with the same 'name', but for each one, specify the target resource, **the region for it** (AWS does not workout where the target resource is based - you must tell it), and record ID.
- Geolocation
  - This is different from Latency, where AWS Route 53 determines the lowest latency target.
  - You specific the actual location of a client and which resource they are routed to.
  - Location options: Countries and Continents
  - You should also have a 'Default' record for clients outside of the ones you have Countries/Continents you have specified, otherwise they will not be able to reach your service.
- Multi-value answer
  - The DNS record returns contains multiple IPs
  - Returns up to 8 IPs, so for example you could have 10 EC2 Instances.
  - If you configure to use a Health check, it will return only IPs for healthy instances.
  - Example use: external load balancer. The client will determines which IP to use.
- Geoproximity
  - Route traffic to resources based on the geographic location of the clients and the region
  - Define a **bias** to increase/decrease the 'area' of the geographic location to route clients to a region
  - To expand the area, increase the bias (1 to 99)
  - To shirnk the area, decrease the bias (-1 to -99)
  - Resources can be AWS resources, or non-AWS (e.g. on-prem, you need to specific the long/lat for the non-AWS region)
  - Examples:
    1. Resources are in 2 regions, bias on both is 0, AWS chooses an 'equal' area in geographic proximity to each region.
    2. Resources in 2 regions, bias on one is 0, on the other is 50, it increases the geographic area of clients that will now be routed to the resource in the 2nd region.
  - Uses Route 53 **Traffic Flow** (advanced)
    - Traffic flow gives a UI for mapping complex flows - for other Routing Policies, not just Geoproximity.
    - Starting point is the DNS record type
    - Rules: all the rules above, Latency, Weighted, etc, New Endpoints, and including **Geoproximity**
      - If you choose Custom Endpoints, you have to enter the long/lat coordinates.
    - Once you create a Traffic Policy, the Routing Policy target is the Traffic policy.
- IP Routing
  - Routing based on CIDR blocks (an IP range)
  - Configured with multiple lists of CIDRs, to route to a specific target

### Routing Policy Health Checks

- HTTP Health Checks are only for public resources.
- It is automated DNS failover
- Health checks can monitor
  - Endpoints, e.g, an EC2 instance, etc
  - Other health checks (Calcualted Health Checks)
  - CloudWatch alarms, e.g. RDS metrics, custom metrics
- There are 15 global health checkers that check the endpoints
  - Threshold is 3 unhealthy checks (default - configurable)
  - > 18% of health checks report unhealthy
  - Interval is 30 seconds, can increase frequency to 10 seconds but costs more.
  - Protocols: HTTP, HTTPS, TCP
  - Can choose localtions you want Route 53 to use
- 2xx and 3xx HTTP response code are healthy
- Can read first 5120 bytes of response body to check for some text, etc.
- You must allow IP address of health checkers to access your resource, e.g, firewall/Security Group.
- For private resource not accessible from the internet, the way to allow health checks to be used, is to create Cloud Watch metrics/alarms for the resource, then base the health check off of the Cloud Watch metric. You will not be able get the health checker to make a request directly to your private resource.
- Health checks can be enabled on all Routing Policies except Simple. If enable (not Failover), Route 53 won't send traffic to an instance failing a health check.
- The Failover policy is different, in that is just for having a Primary and Seconday, but you can use Daisy Chaining underneath to combine it with other policies.

## Domain Registrar vs DNS Service

- A Domain Registrar is where you can purchase a Domain, it has a record that points to the Namespace service.
- A Domain registrar is not a DNS, but usually comes with one.
- You can configure the domain to use a different DNS, e.g. you buy a Domain with GoDaddy, but use Route 53 as the DNS.
- To configure:
  - Buy the domain with a registrar (if not buying through Route 53)
  - Create a Route 53 public Hosted Zone, Name Servers will be provided for the Hosted Zone
  - Update the Name Servers on the domain registrator with the Name Servers for the Route 53 DNS Hosted Zone.
