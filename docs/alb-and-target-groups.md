# Application Load Balancer and Target Groups

## 1. Overview

An Application Load Balancer (ALB) distributes incoming HTTP and HTTPS requests across registered targets, such as EC2 instances.

In this project, the ALB receives requests from CloudFront and distributes them to EC2 instances running Nginx.

| Component | Project configuration |
|---|---|
| AWS Region | `ap-south-1` — Mumbai |
| Load balancer | `high-traffic-alb` |
| Target group | `high-traffic-tg` |
| Target protocol | HTTP |
| Target port | 80 |
| Health-check path | `/` |
| Expected status code | `200` |

## 2. Architecture

```text
           User / Browser
                 |
                 v
             CloudFront
                 |
                 v
       Application Load Balancer
                 |
          Listener: HTTP : 80
                 |
                 v
            Target Group
             /         \
            v           v
        EC2 - A       EC2 - B
         Nginx         Nginx
            |           |
            +-----+-----+
                  |
             HTTP Response
```

CloudFront forwards requests to the ALB according to the configured behavior. The ALB then routes requests to eligible, healthy targets.

## 3. Key Components

### Application Load Balancer

The ALB operates at Layer 7 of the OSI model. It understands HTTP and HTTPS and can route requests using listener rules.

Typical responsibilities include:

- Distributing requests across healthy targets.
- Supporting listener rules and path-based routing.
- Performing health checks through target groups.
- Integrating with AWS monitoring and security controls.

### Listener

A listener checks incoming connections on a configured protocol and port and applies its default action or matching rules.

In this project, the ALB receives HTTP traffic on port 80 from the configured CloudFront origin.

### Target group

A target group defines the destinations to which the ALB can route traffic.

In this project, EC2 instances are registered as targets. The target group also defines the health-check configuration used to evaluate them.

## 4. Health-Check Configuration

The target group uses these settings:

| Setting | Value |
|---|---|
| Protocol | HTTP |
| Port | Traffic port |
| Path | `/` |
| Healthy threshold | 5 |
| Unhealthy threshold | 2 |
| Timeout | 5 seconds |
| Interval | 30 seconds |
| Success codes | `200` |

The ALB periodically sends health-check requests to registered targets. It considers the response status and configured thresholds when determining target health.

A running EC2 instance is not necessarily a healthy target. The web server must also respond successfully to the health check.

## 5. Troubleshooting Unhealthy Targets

### Observed issue

An EC2 instance was reported as unhealthy in the target group.

During investigation, this command returned:

```text
Unit nginx.service could not be found
```

This indicated that the Nginx service unit was missing on the affected instance.

### Step 1 — Check the target group's health reason

In the AWS Console:

1. Open **EC2 → Target Groups**.
2. Select `high-traffic-tg`.
3. Open **Targets**.
4. Select the unhealthy target and inspect its health status and reason.

### Step 2 — Check Nginx on the instance

For Amazon Linux 2023:

```bash
sudo systemctl status nginx
```

Check whether Nginx is installed:

```bash
nginx -v
```

### Step 3 — Install and start Nginx if missing

```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

Verify the service:

```bash
sudo systemctl status nginx
```

Test the local HTTP response:

```bash
curl -i http://localhost/
```

The response should match the target group's configured success-code expectation.

### Step 4 — Check network access if the target remains unhealthy

If local curl succeeds but the ALB health check still fails, investigate:

- Whether the EC2 security group allows inbound TCP port 80 from the ALB security group.
- Whether the ALB security group allows the required outbound traffic.
- Whether network ACLs or routing are blocking connectivity.
- Whether Nginx listens on the expected interface and port.
- Whether the configured health-check path returns an accepted status code.

### Step 5 — Confirm recovery

Return to **EC2 → Target Groups → `high-traffic-tg` → Targets** and wait for health checks to complete.

Confirm that the target becomes healthy before considering the issue resolved.

## 6. Security Group Traffic Flow

A typical configuration for this architecture is:

| Resource | Inbound rule |
|---|---|
| ALB security group | Allow the required listener traffic from the intended source |
| EC2 security group | Allow TCP port 80 from the ALB security group |

Avoid opening the EC2 web-server port to every IP address when the instances only need to receive application traffic from the ALB.

The exact source rules should match the deployed network design and any additional administrative access requirements.

## 7. Common Failure Scenarios

| Symptom | Investigation |
|---|---|
| Target is unhealthy | Review target health reason and health-check settings |
| Nginx service is missing | Install Nginx and configure it to start |
| Nginx is stopped | Check service status and logs |
| Local curl fails | Investigate Nginx configuration and application content |
| Local curl works but target is unhealthy | Check security groups, network paths, ports, and health-check response |
| All targets are unhealthy | Investigate the shared web-server configuration, ALB connectivity, and health-check settings |
| Target is healthy but users see an old page | Compare ALB and CloudFront responses |

## 8. Operational Best Practices

- Configure web servers automatically through launch-template user data, a prepared AMI, or configuration management.
- Keep the health-check endpoint lightweight and reliable.
- Monitor healthy-host count and target-health changes.
- Review ALB access logs when detailed request-level investigation is required and logging is configured.
- Validate target health after deployments and instance replacement.
- Ensure all targets serve compatible application versions.
- Use HTTPS where required by the architecture and security policy.

## 9. Interview Questions

### What is the difference between an ALB and a target group?

The ALB receives and routes application traffic. The target group defines the destinations for routing and the health checks used to evaluate those targets.

### Why can an EC2 instance be running while its target is unhealthy?

EC2 instance status reflects the virtual machine's state. Target health evaluates whether the application responds according to the configured health check.

### How did you fix an unhealthy target?

I inspected the target-group health status, connected to the affected EC2 instance, and found that the Nginx service was missing. I installed and started Nginx, verified the local HTTP response, and then confirmed target health in the AWS Console.

### Why restrict EC2 inbound traffic to the ALB security group?

It limits application access to the intended load-balancer path instead of exposing the web-server port unnecessarily.

## 10. Key Takeaway

An ALB distributes traffic, but reliable routing depends on correctly configured listeners, healthy targets, working applications, and valid network access. Troubleshoot these layers separately to identify the actual failure.
