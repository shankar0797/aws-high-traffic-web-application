# AWS High-Traffic Web Application

A hands-on AWS infrastructure project demonstrating a highly available web application using Amazon CloudFront, Application Load Balancer (ALB), EC2 Auto Scaling, Amazon Route 53, and Amazon CloudWatch.

The application is accessible at **https://techwithshiva.in**.

## Project Overview

This project demonstrates how to deploy and serve a web application through multiple AWS infrastructure layers, with load balancing, instance health checks, scaling configuration, and monitoring.

### Architecture

```text
User / Browser
      |
      v
Amazon Route 53
(DNS resolution)
      |
      v
Amazon CloudFront
(HTTPS delivery and caching)
      |
      v
Application Load Balancer
      |
      v
EC2 Target Group
      |
      v
Amazon EC2 Instances
(Nginx Web Server)

Auto Scaling Group manages EC2 capacity.
Amazon CloudWatch monitors infrastructure metrics and alarms.
```

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon Route 53 | DNS resolution for the custom domain |
| Amazon CloudFront | Content delivery, HTTPS access, and caching |
| Application Load Balancer | Distributes incoming HTTP traffic across targets |
| Amazon EC2 | Runs the Nginx web application |
| EC2 Auto Scaling | Maintains desired capacity and supports scaling |
| EC2 Target Group | Registers instances and performs health checks |
| Amazon CloudWatch | Monitors metrics and evaluates alarms |
| AWS Certificate Manager | Provides the TLS certificate used for HTTPS |

## Request Flow

1. A user opens `https://techwithshiva.in`.
2. Route 53 resolves the domain.
3. CloudFront receives the HTTPS request.
4. CloudFront forwards requests according to its configured behavior.
5. The ALB distributes requests to healthy registered EC2 targets.
6. Nginx serves the web application.
7. Auto Scaling manages the EC2 fleet, while CloudWatch monitors configured metrics and alarms.

## Validation

The following checks have been performed during the project:

- The custom-domain homepage loads over HTTPS.
- The ALB serves the updated application page.
- The target group has two healthy EC2 targets.
- The Auto Scaling Group is configured with a minimum of 2, desired capacity of 2, and maximum capacity of 4.
- CloudWatch alarms are configured for infrastructure monitoring.

> Note: The configured scaling capacity does not, by itself, prove that a scale-out event has been successfully triggered. Scaling behavior will be documented separately if validated.

## Evidence

Screenshots will be added to the `evidence/` directory as the project documentation is completed.

## Project Documentation

Detailed architecture, configuration, troubleshooting, root-cause analysis, and interview notes will be added progressively.

## Key Learning Outcomes

- DNS configuration and custom-domain integration
- CloudFront distribution and cache troubleshooting
- ALB target health checks and traffic routing
- EC2 Auto Scaling configuration
- CloudWatch alarms and infrastructure monitoring
- Troubleshooting using command-line tests and AWS Console evidence

## Author

**Cloud & Infrastructure Engineering Portfolio**

Project domain: [techwithshiva.in](https://techwithshiva.in)
