# Amazon CloudFront — Configuration and Troubleshooting

## 1. Overview

Amazon CloudFront is AWS's content delivery network (CDN). It delivers content to users through a distributed network of edge locations and supports caching to reduce repeated requests to an origin.

In this project, CloudFront provides the delivery layer for the custom domain:

**Website:** https://techwithshiva.in

**Distribution:** `aws-CF-lab`

**Region of the application infrastructure:** `ap-south-1` — Mumbai

CloudFront is a global service. The ACM certificate used for a CloudFront custom domain must be provisioned in `us-east-1` (US East — N. Virginia).

## 2. CloudFront's Role in the Architecture

```text
User / Browser
      |
      | HTTPS
      v
Amazon CloudFront
      |
      | Cache hit: return eligible cached response
      |
      | Cache miss: forward request according to behavior
      v
Application Load Balancer
      |
      v
EC2 Target Group
      |
      v
EC2 + Nginx
```

CloudFront can reduce repeated origin requests when content is cacheable. The exact result depends on the configured cache behavior, cache policy, origin, and response headers.

## 3. Important Configuration Components

### Distribution

A distribution defines how CloudFront delivers content and connects to its origins.

In this project, the distribution is named `aws-CF-lab`.

### Origin

An origin is the backend from which CloudFront obtains content when it cannot serve an eligible cached response.

The project has these configured origins:

- Application Load Balancer
- Amazon S3 bucket: `shiva-cloudfront-lab-20261006`

The default behavior was configured to route requests to the ALB origin. The existence of an S3 origin does not mean the default website requests are served from S3.

### Cache behavior

A cache behavior determines how CloudFront handles matching URL paths. Its configuration can specify:

- Origin
- Viewer protocol policy
- Allowed HTTP methods
- Cache policy
- Origin request policy
- Response headers policy, if configured

The default behavior in this project uses the Managed-CachingOptimized cache policy.

### Viewer protocol policy

The viewer protocol policy was configured to redirect HTTP requests to HTTPS.

This helps ensure that users accessing the website over HTTP are redirected to HTTPS.

### TLS certificate

AWS Certificate Manager provides the certificate for the custom domain.

For CloudFront, the ACM certificate must be in `us-east-1`, even when the ALB and EC2 resources are in another AWS Region.

## 4. How CloudFront Caching Works

### Cache hit

CloudFront has a usable cached object for the request and can return it without retrieving the object again from the origin.

### Cache miss

CloudFront does not have a usable cached object for the request. It forwards the request to the origin according to the applicable behavior and caching configuration.

### Cache expiration

Cached objects are governed by their TTL and cache-policy settings. A response can expire and require CloudFront to retrieve or revalidate content with the origin.

A cache hit is not inherently a problem. It becomes a problem when the cached object is older than the version the website owner intends users to see.

## 5. Real Project Incident: Old Webpage Displayed

### Symptoms

The ALB returned the updated professional webpage, but the public domain continued to display an older version.

During the investigation, CloudFront response headers showed a cache hit.

### Step 1 — Check the public response

```bash
curl -sSI https://techwithshiva.in/
```

Inspect the response headers, especially:

- `X-Cache`
- `Age`
- `ETag`
- `Last-Modified`

### Step 2 — Compare the response body

Check the ALB origin:

```bash
curl -sS http://high-traffic-alb-1560459725.ap-south-1.elb.amazonaws.com/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

Check the public domain:

```bash
curl -sS https://techwithshiva.in/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

If the origin returns the updated content while CloudFront returns the old content, investigate the CDN configuration and cache state.

### Step 3 — Review the distribution

In the AWS Console:

1. Open **CloudFront → Distributions**.
2. Select `aws-CF-lab`.
3. Review the origin associated with the default behavior.
4. Check the cache policy and viewer protocol policy.
5. Confirm the distribution is deployed before relying on configuration changes.

### Step 4 — Invalidate the affected content

In the CloudFront distribution:

1. Open the **Invalidations** tab.
2. Create an invalidation for the affected object path.
3. For a small root-page issue, consider `/index.html` or `/` depending on the URL and origin mapping.
4. For a broad refresh, `/*` invalidates all objects and may have a larger operational or cost impact depending on usage and current AWS pricing.
5. Wait until the invalidation status is **Completed**.
6. Retest the public domain.

In this project, an invalidation was completed and the updated webpage was subsequently displayed through the public domain.

## 6. Common CloudFront Troubleshooting Scenarios

| Symptom | What to investigate |
|---|---|
| Old page appears | Cache state, TTL, origin content, invalidation |
| HTTP does not redirect | Viewer protocol policy and distribution deployment |
| HTTPS certificate error | Custom-domain alternate names and ACM certificate |
| CloudFront returns an origin error | Origin reachability, ALB health, security groups, and origin response |
| ALB page differs from public page | Cache behavior, selected origin, and cached response |
| S3 content appears unexpectedly | Matching path behavior and selected origin |

## 7. Best Practices

- Use versioned asset filenames where practical.
- Set cache-control headers according to content update frequency.
- Use narrow invalidations when only a few objects need refreshing.
- Validate the origin before investigating CDN behavior.
- Check response bodies as well as headers.
- Keep origin and cache-behavior configurations documented.
- Restrict origin access according to the intended architecture.
- Monitor delivery errors and validate changes after deployment.

## 8. Interview Questions

### What is Amazon CloudFront?

CloudFront is an AWS CDN that delivers content through edge locations and can cache eligible responses closer to users.

### What is the difference between an origin and a cache behavior?

An origin identifies the backend serving content. A cache behavior defines how CloudFront handles matching requests and selects the origin and policies to apply.

### Why did you invalidate CloudFront?

The ALB returned the updated webpage while the public domain returned older content. After comparing the origin and CDN responses, I completed an invalidation and verified the updated page.

### Where must the ACM certificate for CloudFront be created?

In `us-east-1` (US East — N. Virginia).

### Does every CloudFront request reach the ALB?

No. CloudFront can serve a valid cached response. Requests that require origin retrieval are forwarded according to the applicable cache behavior.

## 9. Key Takeaway

CloudFront is more than an HTTPS endpoint. Its origins, cache behaviors, TTLs, and response headers determine how content reaches users. When users report outdated content, compare the origin and CDN responses before making changes to healthy application infrastructure.
