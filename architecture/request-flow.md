# AWS Web Application — Request Flow

> **Project:** AWS High-Traffic Web Application  
> **Region:** Asia Pacific (Mumbai) — `ap-south-1`  
> **Live website:** [https://techwithshiva.in](https://techwithshiva.in)

## 1. Overview

This document explains how a request travels from a user's browser through AWS networking and content delivery services to the EC2 instances serving the application.

The architecture combines DNS resolution, HTTPS delivery, load balancing, health checks, auto scaling configuration, and monitoring.

## 2. Request Flow Architecture

```text
                 USER / BROWSER
                       |
                  HTTPS : 443
                       |
                       v
                AMAZON ROUTE 53
                 DNS resolution
                       |
                       v
                AMAZON CLOUDFRONT
                 CDN / Edge Cache
                       |
                Configured origin
                       |
                       v
             APPLICATION LOAD BALANCER
                    HTTP : 80
                       |
                       v
                 TARGET GROUP
                  Health checks
                    Path: /
                       |
              +--------+--------+
              |                 |
              v                 v
        EC2 INSTANCE A    EC2 INSTANCE B
             Nginx             Nginx
              |                 |
              +--------+--------+
                       |
                 HTML Response

        Supporting Services:
        Auto Scaling Group + Amazon CloudWatch
```

**Important:** Route 53 resolves the domain; it does not proxy application traffic. CloudFront can serve a cached response or forward the request to its configured origin. The default behavior in this project routes requests to the Application Load Balancer.

## 3. How the Request Works

### Step 1 — User accesses the website

The user opens [https://techwithshiva.in](https://techwithshiva.in).

The browser initiates an HTTPS request to the website.

### Step 2 — Route 53 resolves the domain

Amazon Route 53 hosts the public DNS records for `techwithshiva.in`.

The domain's alias record directs the user to the CloudFront distribution.

### Step 3 — CloudFront processes the request

Amazon CloudFront provides content delivery and caching.

- **Cache hit:** CloudFront can return a cached response.
- **Cache miss:** CloudFront forwards the request to the origin selected by the matching cache behavior.

For this project's default behavior, the configured origin is the Application Load Balancer.

The ACM certificate associated with CloudFront supports HTTPS for the custom domain.

### Step 4 — Application Load Balancer distributes traffic

The ALB receives the forwarded HTTP request on port 80.

It evaluates its listener configuration and sends the request to an eligible, healthy target in the target group.

### Step 5 — Target group checks instance health

The target group uses HTTP health checks to determine whether registered instances are healthy.

| Health-check setting | Configuration |
|---|---|
| Protocol | HTTP |
| Port | Traffic port |
| Path | `/` |
| Healthy threshold | 5 |
| Unhealthy threshold | 2 |
| Timeout | 5 seconds |
| Interval | 30 seconds |
| Success code | `200` |

An instance that fails health checks can be marked unhealthy and excluded from normal load-balancer routing.

### Step 6 — EC2 and Nginx serve the response

The selected EC2 instance runs Nginx and serves the webpage.

Both instances should have the same application version so that users receive consistent content regardless of which target handles the request.

### Step 7 — Auto Scaling manages capacity

The Auto Scaling Group is configured as follows:

| Setting | Value |
|---|---:|
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| CPU target tracking | 50% |
| Instance warm-up | 180 seconds |

The group maintains desired capacity and can add or remove instances according to its scaling policy, workload, and configured limits.

These settings describe the configuration; successful scale-out must be confirmed separately using scaling activities, instance changes, and CloudWatch metrics.

### Step 8 — CloudWatch supports monitoring

Amazon CloudWatch monitors configured metrics and evaluates alarm conditions.

The project includes an alarm for low healthy-host count. During an incident, check the alarm state alongside target health, instance status, and Auto Scaling activity.

## 4. Troubleshooting: Old Webpage Appearing

### Incident symptoms

The Application Load Balancer returned the updated webpage, but the custom domain continued to show an older version.

### Investigation

Check the ALB response:

```bash
curl -sS http://high-traffic-alb-1560459725.ap-south-1.elb.amazonaws.com/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

Check the public domain:

```bash
curl -sS https://techwithshiva.in/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

Inspect the response headers:

```bash
curl -sSI https://techwithshiva.in/
```

Useful headers include:

| Header | What it helps investigate |
|---|---|
| `X-Cache` | Whether CloudFront reports a cache hit or miss |
| `Age` | How long a cached response has been held |
| `ETag` | Response version identifier |
| `Last-Modified` | Origin content modification time |
| `Via` | Proxy and cache path information |

### Resolution

1. Confirm that the ALB returns the intended webpage.
2. Check the CloudFront distribution's origin and cache behavior.
3. Review the invalidation status and timing.
4. If stale content remains, invalidate the relevant CloudFront object path.
5. Wait for invalidation completion and retest the custom domain.

### Root-cause lesson

The observed issue was stale content being returned through CloudFront while the ALB already served the updated page. Compare the origin response with the CDN response before changing the application or EC2 configuration.

## 5. Key Takeaways

- **Route 53:** Resolves the custom domain to CloudFront.
- **CloudFront:** Provides content delivery and caching.
- **ALB:** Distributes requests among eligible healthy targets.
- **Target group:** Tracks registered target health.
- **EC2 and Nginx:** Serve the webpage.
- **Auto Scaling Group:** Manages EC2 capacity.
- **CloudWatch:** Supports monitoring and alarm evaluation.

## 6. Interview Explanation

**Question: Explain the request flow in your AWS project.**

A user accesses the application through its HTTPS custom domain. Route 53 resolves the domain to CloudFront. CloudFront either returns an eligible cached response or forwards the request to the configured origin. In this project's default behavior, that origin is the Application Load Balancer. The ALB routes requests to healthy EC2 instances registered in its target group, where Nginx serves the webpage. The Auto Scaling Group manages instance capacity, while CloudWatch monitors configured metrics and alarms.
