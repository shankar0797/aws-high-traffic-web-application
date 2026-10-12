# AWS High-Traffic Web Application — Interview Questions and Answers

## 1. Project Overview

### Q1. Explain your AWS project.

**Answer:**

I built an AWS high-traffic web application infrastructure project to learn and demonstrate highly available web hosting, load balancing, automatic scaling, content delivery, DNS configuration, and monitoring.

The main services used were:

- Amazon EC2 for hosting the web servers.
- EC2 Auto Scaling for managing instance capacity.
- Application Load Balancer (ALB) for distributing application traffic.
- Amazon CloudWatch for monitoring metrics and alarms.
- Amazon CloudFront for content delivery and caching.
- Amazon Route 53 for DNS management.
- AWS Certificate Manager (ACM) for the HTTPS certificate.
- GoDaddy for domain registration.

The website is accessible through my custom domain, `techwithshiva.in`.

### Q2. What was the main objective?

The objective was to build a web hosting architecture that supports traffic distribution, instance replacement and scaling, health monitoring, HTTPS, DNS management, and content caching.

The project also helped me understand how to troubleshoot failures across multiple AWS services.

## 2. Architecture and Traffic Flow

### Q3. Explain the request flow.

**Answer:**

A user accesses the website through the custom domain. Route 53 resolves the domain to the CloudFront distribution. CloudFront evaluates the request against its cache behaviors and either serves a cached response or forwards the request to the configured origin.

When the ALB is selected as the origin, it routes traffic to healthy EC2 instances running Nginx. CloudFront can return the origin response to the user and cache eligible content.

### Q4. Why did you use CloudFront?

CloudFront is a content delivery network. It can cache eligible responses closer to users, reduce repeated requests to the origin, and provide an HTTPS entry point.

It is particularly useful for static assets and cacheable content.

### Q5. Why did you use an ALB?

The Application Load Balancer distributes incoming HTTP or HTTPS requests across registered targets according to its listener rules and target-group configuration.

It also performs health checks so unhealthy targets can be excluded from normal traffic routing.

### Q6. What is the difference between CloudFront and an ALB?

CloudFront is primarily a content delivery network and caching layer. An ALB is a regional load balancer that distributes application traffic to configured targets.

They perform different roles and can be used together.

## 3. EC2 and Auto Scaling

### Q7. What is an Auto Scaling Group?

An Auto Scaling Group manages a group of EC2 instances and attempts to maintain the configured capacity. It can launch or terminate instances according to capacity requirements and scaling policies.

### Q8. What were your Auto Scaling settings?

**Answer:**

The project was configured with the following capacity settings:

| Setting | Value |
|---|---:|
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| CPU target tracking target | 50% |
| Instance warm-up | 180 seconds |

These are the configured settings. Actual scaling behavior must be confirmed using the Auto Scaling activity history, instance count, and CloudWatch metrics.

### Q9. What is a launch template?

A launch template defines the configuration used to launch EC2 instances. It can specify the AMI, instance type, security groups, IAM instance profile, storage, and user data.

In this project, the launch template was used by the Auto Scaling Group to create web-server instances.

### Q10. Why might an Auto Scaling Group not scale out?

Possible causes include:

- The workload is not running on instances managed by the group.
- The relevant metric does not reach the policy's scaling requirements.
- The scaling policy is misconfigured.
- The maximum capacity has been reached.
- EC2 capacity or regional quota limits prevent a launch.
- The instance is still initializing or warming up.

I would inspect CloudWatch metrics, scaling policies, and the Auto Scaling Activity tab before deciding the root cause.

### Q11. How did you validate Auto Scaling?

**Answer:**

During an initial test, I generated CPU load on a manually created EC2 instance instead of an instance managed by the Auto Scaling Group. That test did not reliably validate the group's scaling policy.

The correct validation is to generate controlled load on an ASG-managed instance, monitor CPU metrics and scaling activities, and confirm whether a new instance launches and becomes healthy in the target group.

## 4. ALB and Target Health

### Q12. How does an ALB determine whether a target is healthy?

The ALB target group performs configured health checks against registered targets. The result depends on the configured protocol, port, path, timeout, thresholds, and expected response codes.

### Q13. What health-check configuration did you use?

| Setting | Value |
|---|---|
| Protocol | HTTP |
| Port | Traffic port |
| Path | `/` |
| Expected success code | `200` |
| Healthy threshold | 5 |
| Unhealthy threshold | 2 |
| Timeout | 5 seconds |
| Interval | 30 seconds |

### Q14. Describe a critical issue you resolved.

**Answer:**

During troubleshooting, an EC2 instance was failing ALB health checks because Nginx was not installed or running.

I checked the service status and received:

```text
Unit nginx.service could not be found
```

This indicated that the Nginx service was missing on that instance.

I installed Nginx, enabled and started the service, and tested the local web server using `curl`. I then rechecked the target group and confirmed that the affected targets became healthy.

The key lesson was to verify the application service, port, and health-check response instead of assuming every unhealthy target is a load-balancer problem.

### Q15. What would you check if a target is unhealthy?

I would:

1. Check the target health reason in the EC2 console.
2. Confirm that Nginx or the application is running.
3. Test the local endpoint using `curl`.
4. Verify that the target listens on the expected port.
5. Check the ALB and instance security-group rules.
6. Validate the health-check path and expected status code.
7. Recheck target health after resolving the issue.

## 5. CloudWatch Monitoring

### Q16. Why did you use CloudWatch?

CloudWatch provides metrics, alarms, dashboards, and monitoring capabilities for AWS resources. It helps identify conditions requiring investigation.

### Q17. What does HealthyHostCount mean?

`HealthyHostCount` indicates the number of healthy targets for the relevant load-balancer dimensions. A value below one can indicate that the target group has no healthy targets available.

### Q18. Explain your healthy-host alarm.

**Answer:**

I configured an alarm named `high-traffic-low-healthy-hosts-1a` using the `AWS/ApplicationELB` namespace and `HealthyHostCount` metric.

The documented settings were:

- Statistic: Minimum
- Period: 5 minutes
- Condition: Less than 1
- Evaluation: 1 datapoint within 5 minutes

The purpose is to detect when healthy target capacity falls to zero. I would check the alarm history and target health to investigate the underlying cause.

### Q19. Does a CloudWatch alarm automatically fix a problem?

No. An alarm evaluates a metric condition and changes state. Remediation requires a configured action or an operational response, such as investigating an unhealthy target.

## 6. CloudFront, DNS, and HTTPS

### Q20. Why did you use Route 53 with GoDaddy?

GoDaddy was used for domain registration, while Route 53 hosts the DNS zone. The domain's nameservers were configured to delegate DNS management to the Route 53 hosted zone.

### Q21. Why is ACM configured in `us-east-1`?

CloudFront requires its ACM certificate to be in the US East (N. Virginia) region, even when the application's ALB is in another region.

### Q22. Describe the stale-content issue.

**Answer:**

The ALB was serving an updated webpage, but the custom domain continued displaying older content.

I inspected the HTTP response headers, including CloudFront cache indicators such as `X-Cache` and `Age`. A CloudFront invalidation was performed, after which the domain displayed the updated webpage.

The practical lesson was to compare the origin response with the CloudFront response and inspect caching behavior before concluding that the application deployment failed.

### Q23. What does `X-Cache: Hit from cloudfront` indicate?

It indicates that CloudFront served the response from its cache. The `Age` header, when present, can help indicate how long a response has been cached.

These headers are useful diagnostic clues, but they should be interpreted alongside the cache behavior, origin response, and other response headers.

## 7. Quotas and Scaling Failures

### Q24. Why can an EC2 instance launch fail even when the Auto Scaling Group is configured correctly?

An instance launch may fail because of service quotas, unavailable instance capacity, invalid launch-template settings, subnet limitations, or other EC2 launch constraints.

The Auto Scaling Activity tab is a useful starting point for identifying the reported failure reason.

### Q25. What is an EC2 vCPU quota?

An EC2 vCPU quota limits the number of vCPUs that can be used by applicable EC2 instance families or quota categories in a region. The exact quota and usage depend on the account and instance category.

When a launch fails due to a quota, I would inspect the error message, review the applicable quota in Service Quotas, and request an increase if needed.

## 8. Scenario-Based Questions

### Q26. The website is down. How would you troubleshoot it?

**Answer:**

I would investigate layer by layer:

1. Check whether the domain resolves correctly.
2. Test the CloudFront endpoint and inspect HTTP response headers.
3. Check the CloudFront distribution and selected origin.
4. Verify the ALB listener and target group.
5. Check target health and health-check failure reasons.
6. Verify Nginx and the application on the affected EC2 instances.
7. Review CloudWatch metrics, alarms, and relevant logs.

This approach helps narrow down whether the issue is related to DNS, caching, load balancing, instance health, or the application.

### Q27. The ALB works, but the custom domain shows an old page. What would you do?

I would compare responses from the ALB and the custom domain, inspect CloudFront response headers and cache behavior, verify the configured origin, and use a targeted invalidation if the stale response is confirmed to be cached.

### Q28. CPU usage is high, but no additional EC2 instance launches. What would you check?

I would verify that CPU load is on an ASG-managed instance, review the target tracking policy and CloudWatch metrics, inspect scaling activities, check maximum capacity, and look for quota or launch failures.

### Q29. New instances launch but do not become healthy. What could be wrong?

The web server might not be installed or running, the application may not be listening on the expected port, security groups may block traffic, or the health-check path may return an unexpected status code.

I would inspect the target health reason and validate the application locally.

## 9. Interview Preparation Notes

When discussing this project:

- Clearly distinguish configured features from successfully tested behavior.
- Explain the request flow from DNS through CloudFront and the ALB to EC2.
- Use actual troubleshooting evidence, symptoms, root causes, and resolutions.
- Do not claim successful Auto Scaling unless the scaling activity and new healthy instance are verified.
- Be prepared to explain why each AWS service was used and how its failure would be investigated.

## 10. Key Takeaway

This project demonstrates practical understanding of AWS web hosting, load balancing, Auto Scaling configuration, CloudWatch alarms, DNS, HTTPS, caching, and layered troubleshooting. The strongest interview answers explain not only what was configured, but also how the behavior was verified and what was learned from failures.
