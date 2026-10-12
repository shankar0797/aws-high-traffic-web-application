# AWS Services Cheat Sheet — High-Traffic Web Application

## 1. Project Overview

This project demonstrates a web application architecture using AWS services for web hosting, traffic distribution, scaling, monitoring, DNS, HTTPS, and content delivery.

**Domain:** `techwithshiva.in`

**AWS Region:** `ap-south-1` — Mumbai

## 2. Architecture at a Glance

```text
                    Website Users
                         |
                         v
                  Route 53 DNS
                         |
                         v
                    CloudFront
                    /         \
                   v           v
                  ALB          S3
                   |
                   v
            ALB Target Group
                   |
                   v
             EC2 Auto Scaling
              /           \
             v             v
          Nginx EC2     Nginx EC2

       CloudWatch monitors metrics
       ACM provides the HTTPS certificate
       GoDaddy handles domain registration
```

CloudFront selects an origin according to its cache behaviors. The diagram shows the project's main components, not every possible request path.

## 3. Service-by-Service Explanation

### Amazon EC2

**Purpose:** Provides virtual servers to run applications.

**How this project uses it:** EC2 instances run Nginx and serve website content.

**Troubleshooting:** Check instance state, system status checks, Nginx service status, application logs, and network access.

Useful command:

```bash
aws ec2 describe-instances --region ap-south-1
```

### EC2 Auto Scaling

**Purpose:** Manages EC2 instance capacity according to configured capacity settings and scaling policies.

**How this project uses it:** The `high-traffic-asg` group manages the web-server instances.

Configured capacity:

- Minimum: 2
- Desired: 2
- Maximum: 4

**Troubleshooting:** Inspect scaling activities, launch errors, capacity settings, and applicable EC2 quotas.

### Application Load Balancer (ALB)

**Purpose:** Distributes application traffic across registered targets.

**How this project uses it:** The ALB forwards requests to EC2 instances registered in `high-traffic-tg`.

**Troubleshooting:** Check listener rules, target health, security groups, and health-check configuration.

### Elastic Load Balancing Target Group

**Purpose:** Defines the registered targets and health checks used by a load balancer.

**How this project uses it:** The `high-traffic-tg` target group checks the web servers using HTTP on port 80 and path `/`.

**Troubleshooting:** Investigate the reported target health reason, application service, port, and expected HTTP response.

### Amazon CloudWatch

**Purpose:** Collects metrics, supports alarms, and provides monitoring capabilities.

**How this project uses it:** Monitors infrastructure metrics and includes a healthy-host alarm.

**Troubleshooting:** Inspect metric graphs, alarm state, alarm history, and the corresponding AWS resource.

### Amazon CloudFront

**Purpose:** Delivers content through a distributed network and can cache eligible HTTP responses.

**How this project uses it:** Provides the public entry point for the website and can serve cached content or forward requests to a configured origin.

**Troubleshooting:** Inspect cache behavior, origin configuration, response headers, distribution status, and invalidation status.

### Amazon Route 53

**Purpose:** Provides DNS hosting and routing capabilities.

**How this project uses it:** Hosts the public DNS zone for `techwithshiva.in` and directs the root domain to CloudFront through an alias record.

**Troubleshooting:** Check authoritative nameservers, hosted-zone records, and DNS resolution.

### AWS Certificate Manager (ACM)

**Purpose:** Provisions and manages supported TLS certificates.

**How this project uses it:** Provides the certificate configured for HTTPS on CloudFront.

**Important:** The ACM certificate attached to CloudFront must be in `us-east-1` (US East, N. Virginia).

**Troubleshooting:** Check certificate status, covered domain names, validation, and CloudFront certificate settings.

### Amazon S3

**Purpose:** Provides object storage.

**How this project uses it:** An S3 bucket is configured as an additional CloudFront origin.

**Troubleshooting:** Check the requested object key, bucket policy, origin access configuration, and CloudFront behavior that selects the S3 origin.

### GoDaddy

**Purpose:** Domain registration and domain-management services.

**How this project uses it:** Registered `techwithshiva.in`. The domain's nameservers were changed to those assigned to the Route 53 hosted zone.

**Troubleshooting:** Verify domain status and nameserver delegation.

## 4. How the Services Work Together

### Step 1 — Domain resolution

The visitor enters `techwithshiva.in`. DNS resolution directs the request toward CloudFront.

### Step 2 — Content delivery

CloudFront evaluates the requested path and cache behavior. It may return a cached response or request content from the selected origin.

### Step 3 — Load balancing

When CloudFront selects the ALB origin, the ALB routes the request to a healthy registered target.

### Step 4 — Application response

The EC2 instance runs Nginx and returns the website response.

### Step 5 — Monitoring

CloudWatch metrics and alarms help identify abnormal conditions. Troubleshooting then focuses on the affected layer.

## 5. Quick Comparison

| Service | Main responsibility | Example issue investigated |
|---|---|---|
| EC2 | Compute | Nginx is not running |
| Auto Scaling | Instance capacity | New instance fails to launch |
| ALB | Traffic distribution | Requests fail to reach targets |
| Target group | Target registration and health checks | Target is unhealthy |
| CloudWatch | Monitoring and alarms | Healthy-host count falls to zero |
| CloudFront | Content delivery and caching | Old content appears |
| Route 53 | DNS | Domain does not resolve correctly |
| ACM | TLS certificates | HTTPS certificate error |
| S3 | Object storage | Origin object cannot be retrieved |
| GoDaddy | Domain registration | Domain or nameserver configuration issue |

## 6. Interview Questions

### Why use multiple AWS services instead of only EC2?

EC2 provides compute, but additional services support traffic distribution, capacity management, monitoring, DNS, HTTPS, and content delivery. Each service addresses a different responsibility.

### What happens if one EC2 target becomes unhealthy?

The ALB health-check mechanism can stop routing normal traffic to that unhealthy target. Auto Scaling may replace instances depending on the group's configuration and health-check settings.

### Does CloudFront always contact the ALB?

No. CloudFront can serve a cached response or select another origin through a matching cache behavior.

### Does CloudWatch automatically fix all failures?

No. CloudWatch monitors and evaluates conditions. Remediation depends on configured actions or an operational response.

### What is the difference between Route 53 and CloudFront?

Route 53 handles DNS resolution and routing policies. CloudFront delivers HTTP content and can cache responses.

## 7. Key Takeaway

Understanding each service's responsibility makes troubleshooting easier. Start by identifying the failing layer—DNS, TLS, CDN, load balancer, target health, compute, or application—and then validate the relevant configuration and evidence.
