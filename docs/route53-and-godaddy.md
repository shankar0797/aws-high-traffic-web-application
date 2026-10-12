# Route 53, GoDaddy, ACM, and CloudFront

## 1. Overview

This document explains how the custom domain `techwithshiva.in` connects to the AWS-hosted web application.

The domain was registered through GoDaddy, while Amazon Route 53 manages its DNS records. Amazon CloudFront provides the public HTTPS entry point for the website.

## 2. Architecture

```text
             Website Visitor
                    |
                    v
       https://techwithshiva.in
                    |
                    v
              Route 53 DNS
                    |
                    v
             CloudFront CDN
              /           \
             v             v
       Application       S3 Origin
       Load Balancer     (configured)
             |
             v
       EC2 Auto Scaling
        Nginx Web Servers
```

The diagram represents the main request path. CloudFront selects an origin according to its cache behaviors; not every request necessarily reaches the ALB.

## 3. GoDaddy — Domain Registration

GoDaddy was used to register `techwithshiva.in`.

Domain registration gives ownership and renewal control over the domain. DNS hosting can be provided by a different service.

For this project, the domain's nameservers were changed to the nameservers assigned to the public Route 53 hosted zone.

**Important:** Use the actual nameservers shown in your Route 53 hosted zone. Do not copy nameservers from another AWS account or hosted zone.

## 4. Route 53 — DNS Management

Amazon Route 53 hosts the public DNS zone for `techwithshiva.in`.

The root domain uses an alias record that points to the CloudFront distribution.

Conceptually, the record is:

| Setting | Configuration |
|---|---|
| Hosted zone | `techwithshiva.in` |
| Record name | Root domain |
| Record type | A |
| Alias | Enabled |
| Alias target | CloudFront distribution |

An IPv4 `A` alias record can point to a supported AWS resource such as a CloudFront distribution without requiring a fixed IP address.

If IPv6 access is enabled and required, an appropriate `AAAA` alias record may also be configured.

## 5. CloudFront — Content Delivery Network

CloudFront is the public entry point for the website.

The distribution is named `aws-CF-lab`.

Its configured behavior includes:

- Redirecting HTTP viewer requests to HTTPS.
- Using the Application Load Balancer as the default origin.
- Having an S3 bucket configured as another origin.
- Using the AWS managed `CachingOptimized` cache policy for the documented default behavior.

CloudFront uses its cache behaviors to select an origin. The default origin is not necessarily used for every URL path if another behavior matches first.

## 6. ACM — HTTPS Certificate

An AWS Certificate Manager (ACM) certificate was configured for use with CloudFront.

For a certificate attached to a CloudFront distribution, the ACM certificate must be in the **US East (N. Virginia), `us-east-1` region**.

This is different from the ALB's region, which is `ap-south-1` for this project.

HTTPS allows visitors to connect to the website using an encrypted connection.

## 7. Request Flow

1. A visitor opens `https://techwithshiva.in`.
2. DNS resolution identifies the CloudFront distribution.
3. The browser establishes an HTTPS connection with CloudFront.
4. CloudFront evaluates the requested path and matching cache behavior.
5. If a suitable cached response is available, CloudFront can return it directly.
6. If an origin request is required, CloudFront forwards the request to the selected origin.
7. For requests routed to the ALB, the ALB forwards traffic to healthy EC2 targets.
8. The Nginx web server responds to the request.
9. CloudFront returns the response to the visitor and may cache it according to the applicable policy and response headers.

## 8. Troubleshooting: Old Website Content

### Problem

The ALB was serving an updated webpage, but the custom domain continued showing older content.

### Investigation

Response headers included CloudFront cache indicators such as:

```bash
curl -sI https://techwithshiva.in
```

Useful headers to inspect include:

- `X-Cache`
- `Age`
- `ETag`
- `Last-Modified`
- `Cache-Control`
- `Expires`

A cache hit and a positive `Age` value can indicate that CloudFront is serving a cached response. These headers alone do not prove the exact root cause.

### Resolution

A CloudFront invalidation was performed, after which the custom domain displayed the updated page.

For example, to invalidate all paths through the AWS CLI:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

Replace `YOUR_DISTRIBUTION_ID` with the actual distribution ID.

Invalidations can incur charges after the applicable free allowance. Before invalidating everything, consider whether a targeted path invalidation is sufficient.

### Prevention

- Understand the cache policy and origin response headers.
- Use versioned asset filenames for frequently updated static files.
- Configure suitable cache-control headers.
- Verify the origin response before troubleshooting CloudFront.
- Use invalidations when required rather than treating them as the default deployment method.

## 9. Troubleshooting Checklist

| Symptom | What to check |
|---|---|
| Domain does not resolve | GoDaddy nameservers and Route 53 records |
| HTTPS certificate error | Certificate status, domain names, and CloudFront configuration |
| HTTP does not redirect | CloudFront viewer protocol policy |
| Old webpage appears | Cache behavior, response headers, origin content, and invalidation status |
| CloudFront returns an error | Selected origin, origin connectivity, ALB target health, and response status |
| Root domain works but another hostname fails | DNS record and certificate coverage for that hostname |

## 10. Production Best Practices

- Keep domain registration and renewal information current.
- Verify the authoritative nameservers after DNS changes.
- Use HTTPS and valid certificates.
- Avoid hardcoding AWS resource IP addresses in DNS records.
- Document CloudFront origins and path-based cache behaviors.
- Use cache-control headers appropriate to each content type.
- Validate DNS, TLS, origin health, and cached responses independently during incidents.

## 11. Interview Questions

### Why use Route 53 with GoDaddy?

GoDaddy can handle domain registration while Route 53 hosts the DNS zone. Updating the authoritative nameservers delegates DNS management to Route 53.

### Why use CloudFront?

CloudFront is a content delivery network that can cache content closer to viewers, reduce repeated origin requests, and provide an HTTPS entry point.

### Why must the CloudFront ACM certificate be in `us-east-1`?

CloudFront requires its ACM certificate to be provisioned in the US East (N. Virginia) region.

### Why might a website show old content after the origin is updated?

CloudFront may still have a cached response. Check cache headers, the matching behavior, origin content, and invalidation status before deciding on the cause.

### What is the difference between DNS and a CloudFront cache?

DNS resolves a hostname to the relevant destination. CloudFront caching stores eligible HTTP responses to reduce the need to fetch content from an origin on every request.

## 12. Key Takeaway

GoDaddy manages domain registration, Route 53 manages DNS, ACM supplies the TLS certificate, and CloudFront delivers content through configured origins and cache behaviors. Understanding each layer helps isolate DNS, certificate, origin, and caching issues.
