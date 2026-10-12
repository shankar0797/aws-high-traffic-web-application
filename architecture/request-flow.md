AWS Web Application — Request Flow

1. Purpose

This document explains how a user request travels through the AWS High-Traffic Web Application infrastructure.

Live application: https://techwithshiva.in

2. High-Level Traffic Flow

User / Browser
      |
      | HTTPS request
      v
Amazon Route 53
      |
      | DNS resolution
      v
Amazon CloudFront
      |
      | Forward request according to cache behavior
      v
Application Load Balancer
      |
      v
EC2 Target Group
      |
      +-------------------+
      |                   |
      v                   v
EC2 Instance A       EC2 Instance B
Nginx                Nginx

The Auto Scaling Group manages EC2 capacity. CloudWatch monitors configured metrics and evaluates alarms.

3. Request Lifecycle

Step 1 — User accesses the application

The user opens:

https://techwithshiva.in

The browser initiates an HTTPS request.

Step 2 — Route 53 resolves the domain

Amazon Route 53 hosts the public DNS records for the custom domain.

The domain's alias record directs users to the CloudFront distribution.

Route 53 performs DNS resolution; it does not proxy the user's HTTP request.

Step 3 — CloudFront receives the request

Amazon CloudFront acts as the content delivery layer.

It can return a cached response when the configured cache behavior permits it. Otherwise, it forwards the request to the origin selected by that behavior.

For the default behavior in this project, the configured origin is the Application Load Balancer.

Step 4 — The Application Load Balancer receives the request

The ALB listener receives the forwarded HTTP request on port 80 and evaluates its listener rules.

The ALB distributes requests to eligible, healthy registered targets in the target group.

Step 5 — The target group evaluates instance health

The target group uses its configured HTTP health check:

Protocol: HTTP

Port: Traffic port

Path: /

Healthy threshold: 5

Unhealthy threshold: 2

Timeout: 5 seconds

Interval: 30 seconds

Success code: 200

These settings determine how the load balancer evaluates target health. A failed health check can cause a target to be marked unhealthy and excluded from normal routing.

Step 6 — Nginx serves the webpage

The selected EC2 instance runs Nginx and serves the application HTML.

Both registered instances should serve the same application version to provide consistent responses regardless of which instance receives the request.

Step 7 — Auto Scaling manages capacity

The Auto Scaling Group is configured with:

Setting

Value

Minimum capacity

2

Desired capacity

2

Maximum capacity

4

CPU target tracking policy

50%

Instance warm-up

180 seconds

The group maintains its desired capacity and can adjust capacity according to its scaling policies and limits.

The configuration does not by itself prove that a scaling event has occurred. A successful scale-out test should be validated using scaling activities, instance changes, and metrics.

Step 8 — CloudWatch supports monitoring

Amazon CloudWatch collects and evaluates configured metrics and alarm thresholds.

The project includes an alarm for low healthy-host count. Alarm state should be interpreted alongside target health, metrics, and Auto Scaling activity.

4. Troubleshooting Example — Stale CloudFront Content

Observed symptoms

The ALB returned the updated webpage, but the custom domain continued to display an older version.

Diagnostic commands

Check the ALB response:

curl -sS http://high-traffic-alb-1560459725.ap-south-1.elb.amazonaws.com/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'

Check the public domain:

curl -sS https://techwithshiva.in/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'

Inspect CloudFront response headers:

curl -sSI https://techwithshiva.in/

Useful headers include X-Cache, Age, Via, ETag, and Last-Modified.

Resolution approach

Confirm that the ALB serves the intended version.

Verify the CloudFront distribution's origin and default behavior.

Review the invalidation status and its creation time.

If required, invalidate the relevant CloudFront object paths.

Retest the custom domain after the invalidation completes.

Key lesson

A successful response from the origin does not guarantee that a CDN immediately returns the same content. Compare the origin response, CDN headers, cache configuration, and invalidation timing before deciding on a fix.

5. Interview Summary

Question: Explain the request flow in your AWS project.

A user accesses the application using the custom HTTPS domain. Route 53 resolves the domain to CloudFront. CloudFront applies the configured cache behavior and forwards uncached requests through its default behavior to the Application Load Balancer. The ALB routes requests to healthy EC2 targets registered in its target group. Nginx serves the webpage. The Auto Scaling Group manages instance capacity, while CloudWatch monitors configured metrics and alarms.
