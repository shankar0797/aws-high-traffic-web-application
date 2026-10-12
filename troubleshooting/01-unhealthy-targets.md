# Incident Report: ALB Targets Unhealthy

## 1. Incident Summary

| Field | Details |
|---|---|
| AWS service | Application Load Balancer |
| Region | `ap-south-1` — Mumbai |
| Target group | `high-traffic-tg` |
| Health-check protocol | HTTP |
| Health-check port | Traffic port (80) |
| Health-check path | `/` |
| Expected response | HTTP `200` |
| Root cause identified | Nginx was not installed or running on an affected EC2 instance |
| Resolution | Corrected the instance web-server setup and verified target health |

## 2. Problem Statement

The Application Load Balancer target group reported unhealthy EC2 instances.

The health check attempted to access the configured path `/` over HTTP. However, an affected instance did not have a working Nginx service.

During investigation, this command returned:

```text
Unit nginx.service could not be found
```

This indicated that systemd could not find the Nginx service unit on that instance.

## 3. Architecture and Failure Flow

```text
User
 |
 v
Application Load Balancer
 |
 v
Target Group
 |
 | HTTP health check: GET /
 v
EC2 Instance
 |
 v
Nginx
 |
 +-- Running and returning HTTP 200 --> Healthy
 |
 +-- Missing / stopped / failing ------> Unhealthy
```

When a target is unhealthy, the ALB normally excludes it from routing requests. If no healthy targets remain, users may experience application failures.

## 4. Investigation

### Step 1 — Check target health

In the AWS Console:

1. Open **EC2 → Target Groups**.
2. Select `high-traffic-tg`.
3. Open the **Targets** tab.
4. Review each target's health status and reason.

Record the instance ID and the health-check failure reason.

### Step 2 — Connect to the affected instance

Connect using your approved access method, such as AWS Systems Manager Session Manager or SSH when configured.

Check the service:

```bash
sudo systemctl status nginx
```

Check whether Nginx is installed:

```bash
nginx -v
```

If the command is unavailable and the service unit is missing, investigate the installation before troubleshooting the ALB itself.

### Step 3 — Check port 80

```bash
sudo ss -lntp | grep ':80'
```

If no process is listening on port 80, Nginx may be stopped, missing, or configured to use another port.

### Step 4 — Test the local HTTP response

```bash
curl -i http://localhost/
```

The expected response is HTTP `200 OK`.

If this local request fails, fix the web server or application before investigating network paths between the ALB and the instance.

### Step 5 — Review logs if needed

```bash
sudo journalctl -u nginx --no-pager -n 50
```

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

These commands help identify startup errors, configuration problems, and other service failures.

## 5. Root-Cause Analysis

**Root cause:** Nginx was missing or not running on the affected EC2 instance, so the instance could not reliably serve the configured HTTP health-check path.

The ALB health check depends on a successful response from the target. An EC2 instance can be running while its web application is unavailable.

This distinction is important: **EC2 instance status and ALB target health measure different things.**

## 6. Resolution

For an Amazon Linux 2023 instance, if Nginx is not installed, use the following commands.

Install Nginx:

```bash
sudo dnf install -y nginx
```

Enable and start the service:

```bash
sudo systemctl enable --now nginx
```

Check its status:

```bash
sudo systemctl status nginx
```

Test the response:

```bash
curl -i http://localhost/
```

If the intended application page is missing, configure the appropriate website content and verify that the health-check path returns the expected success code.

**Important:** These commands install Nginx on the current instance. To ensure future replacement instances are configured correctly, include the required installation and startup steps in the launch template's user data or use a controlled image/configuration-management process.

## 7. Verify the Fix

1. Confirm Nginx is active on the affected EC2 instance.
2. Confirm `curl -i http://localhost/` returns the expected response.
3. Open **EC2 → Target Groups → `high-traffic-tg` → Targets**.
4. Wait for the health-check evaluation to complete.
5. Confirm the target becomes healthy.
6. Test the application through the ALB and public domain.

A successful local curl is necessary but not sufficient: the ALB must also be able to reach the instance on the health-check port.

## 8. Prevention

- Install and configure Nginx through launch-template user data, a baked AMI, or configuration management.
- Validate user data before launching a new Auto Scaling instance.
- Ensure the instance security group allows inbound HTTP from the ALB security group.
- Confirm the health-check path returns the configured success code.
- Monitor target health and investigate unhealthy-host alarms.
- Test a replacement instance before relying on the configuration in production.

## 9. Troubleshooting Decision Table

| Symptom | Likely area to investigate |
|---|---|
| `Unit nginx.service could not be found` | Nginx installation |
| Nginx is inactive | Service startup or configuration |
| Nothing listens on port 80 | Service, listener port, or configuration |
| Local curl fails | Web-server configuration or application |
| Local curl works, ALB health check fails | Security groups, routing, health-check settings, or response codes |
| Target is healthy, but the public page is old | CloudFront caching or origin behavior |

## 10. Interview Explanation

**Question: How did you troubleshoot unhealthy ALB targets?**

I checked the target group's health status and investigated the affected EC2 instance. The Nginx service was missing on an instance, and `systemctl status nginx` reported that the service unit could not be found. I installed and started Nginx, validated the local HTTP response, and then checked the target group's health status. I also identified the importance of automating web-server installation in the launch template so that replacement instances are configured consistently.
