# AWS High-Traffic Web Application — Project Explanation

## 1. Project Overview

The AWS High-Traffic Web Application is a hands-on cloud infrastructure project designed to demonstrate website delivery, traffic distribution, high availability, auto scaling configuration, and monitoring using AWS services.

The project uses Amazon Route 53, CloudFront, an Application Load Balancer, EC2 instances running Nginx, an Auto Scaling Group, and Amazon CloudWatch.

**Live application:** https://techwithshiva.in

**AWS Region:** Asia Pacific (Mumbai) — `ap-south-1`

## 2. Business Problem

A website hosted on a single virtual machine can become unavailable if the server fails or struggles to handle incoming traffic.

A scalable web architecture needs to address several concerns:

- How users access the application securely.
- How incoming requests are distributed across web servers.
- How unhealthy targets are detected.
- How instance capacity can adjust to workload.
- How operators monitor application infrastructure.
- How stale content and infrastructure failures are diagnosed.

This project demonstrates AWS services and operational practices that address these concerns.

## 3. Architecture

```text id="p7vr2x"
             Users / Browsers
                    |
                    v
               Route 53
              DNS resolution
                    |
                    v
                CloudFront
             CDN / HTTPS / Cache
                    |
                    v
       Application Load Balancer
                    |
                    v
               Target Group
                /         \
               v           v
           EC2 - A       EC2 - B
            Nginx         Nginx

       Auto Scaling Group
       CloudWatch Monitoring
```

Route 53 resolves the custom domain to CloudFront. CloudFront processes requests according to its cache behaviors. The default behavior in this project forwards requests to the Application Load Balancer when it does not serve an eligible cached response.

The ALB routes requests to healthy registered EC2 targets. Nginx serves the webpage from the selected instance.

The Auto Scaling Group manages instance capacity, while CloudWatch provides monitoring and alarm evaluation.

## 4. AWS Services Used

| AWS service | Purpose |
|---|---|
| Amazon Route 53 | Public DNS and custom-domain resolution |
| AWS Certificate Manager (ACM) | TLS certificate for the CloudFront custom domain |
| Amazon CloudFront | Content delivery and caching |
| Application Load Balancer | Distributes HTTP requests across eligible targets |
| Amazon EC2 | Hosts the Nginx web servers |
| EC2 Auto Scaling | Maintains and adjusts instance capacity |
| Elastic Load Balancing target group | Registers instances and evaluates target health |
| Amazon CloudWatch | Provides metrics and alarm-based monitoring |
| Amazon S3 | Configured as an additional CloudFront origin in the project |

## 5. Important Configuration

### Auto Scaling Group

The group was configured with:

| Setting | Value |
|---|---:|
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| CPU target tracking | 50% |
| Instance warm-up | 180 seconds |

These are the configured settings, not proof that the group successfully scaled from two to four instances. Scale-out behavior should be confirmed using activity history, instance changes, and metrics.

### Target Group Health Checks

| Setting | Value |
|---|---|
| Protocol | HTTP |
| Port | Traffic port |
| Path | `/` |
| Expected success code | `200` |
| Interval | 30 seconds |
| Timeout | 5 seconds |

The health check verifies that the target responds successfully on the configured path. A running EC2 instance is not necessarily a healthy application target.

## 6. End-to-End Request Flow

1. The user opens `https://techwithshiva.in`.
2. Route 53 resolves the custom domain.
3. CloudFront handles the request and checks its applicable cache behavior.
4. If needed, CloudFront forwards the request to the configured ALB origin.
5. The ALB selects an eligible healthy target.
6. Nginx serves the webpage from the selected EC2 instance.
7. The response returns to the user through the delivery path.

## 7. Practical Issues Investigated

### Issue 1 — Unhealthy ALB targets

**Symptom:** A target failed its HTTP health check.

**Finding:** Nginx was missing or not running on an affected EC2 instance.

**Resolution:** Installed or started Nginx, checked the local HTTP response, and verified target health.

**Learning:** Instance health and application health are different.

### Issue 2 — Old webpage displayed through CloudFront

**Symptom:** The ALB served updated content, but the custom domain showed an older page.

**Finding:** The investigation pointed to stale content in the CloudFront delivery path.

**Resolution:** Reviewed the origin response and cache headers, completed an invalidation, and verified the updated page.

**Learning:** Validate the origin and CDN responses separately.

### Issue 3 — EC2 vCPU quota failure

**Symptom:** A replacement-instance launch failed because the applicable regional vCPU quota was exceeded.

**Resolution path:** Reviewed the Auto Scaling activity and submitted a quota-increase request.

**Learning:** Capacity planning must include account-level service quotas. A submitted request must not be treated as approved until verified.

### Issue 4 — Auto Scaling validation

**Symptom:** The initial CPU test did not trigger the expected scaling behavior.

**Finding:** The initial stress test was run on a manually created EC2 instance rather than an instance managed by the Auto Scaling Group.

**Learning:** Generate controlled load on the intended group, inspect the relevant CloudWatch metrics, and validate scaling activity.

## 8. Security and Operational Considerations

For a production implementation, I would additionally validate:

- Least-privilege IAM roles and permissions.
- Restrictive security-group rules, including allowing application traffic from the ALB rather than unrestricted sources where possible.
- Private subnets for application instances where the network design permits.
- Controlled instance configuration through launch-template user data, an AMI, or configuration management.
- HTTPS configuration and appropriate cache-control policies.
- CloudWatch alarms for relevant availability and capacity signals.
- Logging, alerting, recovery procedures, and regular operational testing.
- Cost controls for CloudFront, EC2, load balancing, and other enabled resources.

These are production considerations and should not be interpreted as proof that every control has already been implemented in this lab.

## 9. Project Outcome

The project provided practical experience with:

- Configuring custom-domain delivery through Route 53 and CloudFront.
- Distributing requests through an Application Load Balancer.
- Configuring EC2 targets and HTTP health checks.
- Setting up Auto Scaling capacity and CPU target tracking.
- Investigating EC2 launch quota failures.
- Troubleshooting Nginx and target-health problems.
- Diagnosing stale content through a CDN.
- Using CloudWatch metrics and alarms for operational monitoring.

## 10. Interview Summary

**Question: Explain your AWS high-traffic web application project.**

I built a hands-on AWS web delivery architecture using Route 53, CloudFront, an Application Load Balancer, EC2 instances running Nginx, an Auto Scaling Group, and CloudWatch. The custom domain resolves to CloudFront, which forwards requests to the ALB according to its configured cache behavior. The ALB distributes requests across healthy EC2 targets. I also configured scaling policies and monitoring, and troubleshot practical issues involving unhealthy targets, stale CloudFront content, EC2 vCPU quotas, and scaling validation.

The project helped me understand the complete request path and how to investigate infrastructure issues layer by layer.
