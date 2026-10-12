# Amazon CloudWatch Monitoring and Alarms

## 1. Overview

Amazon CloudWatch helps monitor AWS resources, collect metrics, visualize performance, and trigger alarms when defined conditions occur.

In this project, CloudWatch is used to monitor the health and behavior of the high-traffic web application infrastructure.

**AWS Region:** `ap-south-1` — Mumbai

**Auto Scaling Group:** `high-traffic-asg`

**Target Group:** `high-traffic-tg`

## 2. Monitoring Architecture

```text
        EC2 Instances
        Nginx Web Servers
               |
               v
        CloudWatch Metrics
               |
               v
         CloudWatch Alarm
               |
               v
       Investigate the Alert
               |
               v
       Troubleshoot the Cause
```

CloudWatch collects metrics from supported AWS services. Alarms evaluate metric conditions and change state when those conditions are met.

An alarm does not automatically fix an issue unless an action is configured to respond to it.

## 3. Metrics Used in This Project

### HealthyHostCount

The `AWS/ApplicationELB` namespace includes the `HealthyHostCount` metric.

It indicates the number of healthy targets registered with a load balancer target group, subject to the selected metric dimensions and statistic.

If the healthy-host count falls to zero, the ALB may have no healthy targets available to serve traffic for that target group.

### CPUUtilization

CPU utilization helps identify compute load on EC2 instances.

In this project, CPU utilization is relevant to the Auto Scaling target tracking policy configured with a 50% target.

A single high CPU reading does not guarantee an immediate scale-out. The scaling policy evaluates its configured metric over time.

### Additional useful metrics

| Metric | Purpose |
|---|---|
| `HealthyHostCount` | Checks the number of healthy load-balancer targets |
| `UnHealthyHostCount` | Helps identify targets failing health checks |
| `CPUUtilization` | Measures EC2 CPU utilization |
| `RequestCount` | Helps understand load-balancer request volume |
| `TargetResponseTime` | Helps investigate application response latency |
| `HTTPCode_Target_5XX_Count` | Helps identify HTTP 5xx responses returned by targets |

Metric availability and dimensions depend on the AWS service and configuration.

## 4. Healthy-Host Alarm

An alarm named `high-traffic-low-healthy-hosts-1a` was created to detect low healthy-target capacity.

The configured alarm settings were:

| Setting | Value |
|---|---|
| Metric | `HealthyHostCount` |
| Namespace | `AWS/ApplicationELB` |
| Statistic | Minimum |
| Period | 5 minutes |
| Condition | Less than 1 |
| Evaluation | 1 datapoint within 5 minutes |

The alarm is intended to detect when the healthy-host count drops below one.

**Important:** The configuration describes what the alarm evaluates. It does not establish the alarm's current state. Check the CloudWatch console to confirm whether it is `OK`, `ALARM`, or `INSUFFICIENT_DATA`.

## 5. How to Inspect the Alarm

1. Open **AWS Console → CloudWatch → Alarms**.
2. Select `high-traffic-low-healthy-hosts-1a`.
3. Review the current state.
4. Check the metric graph and evaluation periods.
5. Open the alarm history to see when its state changed.
6. Review the associated target group's health status in the EC2 console.

Use the alarm history and metric graph together to determine when the problem began.

## 6. Troubleshooting: HealthyHostCount Falls to Zero

### Step 1 — Check the target group

Open **EC2 → Target Groups → `high-traffic-tg` → Targets**.

Check whether targets are healthy, unhealthy, or still registering.

### Step 2 — Inspect health-check failures

Review the target health reason and description.

Possible causes include:

- Nginx is not installed or running.
- The application is not listening on port 80.
- The security group blocks traffic from the ALB.
- The configured health-check path does not return an expected response.
- The instance is still initializing.

### Step 3 — Check Nginx on the affected instance

Connect using your approved access method and run:

```bash
sudo systemctl status nginx
```

If Nginx is missing on an Amazon Linux 2023 instance, install and start it:

```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

Test the local web server:

```bash
curl -i http://localhost/
```

Confirm that the response matches the target group's expected success code.

### Step 4 — Check security groups

Verify that the EC2 instance security group allows inbound HTTP traffic on port 80 from the ALB security group.

Avoid opening port 80 to all internet traffic unless the architecture specifically requires it.

### Step 5 — Recheck target health

Return to the target group and wait for health checks to run.

Confirm that the targets become healthy and that the alarm state reflects the updated metric data.

## 7. Understanding Alarm States

| State | Meaning |
|---|---|
| `OK` | The alarm condition is not currently breached |
| `ALARM` | The configured alarm condition has been breached |
| `INSUFFICIENT_DATA` | CloudWatch does not have enough data to evaluate the alarm |

An alarm in `INSUFFICIENT_DATA` is not necessarily evidence of an infrastructure failure. Check metric availability, dimensions, recent activity, and the alarm configuration.

## 8. Production Best Practices

- Monitor both infrastructure health and application behavior.
- Configure notification actions when alerts require operational response.
- Use meaningful alarm names and document their purpose.
- Review alarm history during incident investigation.
- Avoid relying on a single metric to diagnose complex failures.
- Use dashboards to correlate CPU, request volume, target health, and response latency.
- Periodically test alarm conditions in a controlled environment.
- Review missing metrics and `INSUFFICIENT_DATA` states.

## 9. Interview Questions

### What is Amazon CloudWatch?

Amazon CloudWatch is an AWS monitoring and observability service for metrics, logs, alarms, and dashboards.

### What does HealthyHostCount indicate?

It indicates the number of healthy targets associated with the relevant load-balancer dimensions.

### What happens when a CloudWatch alarm enters ALARM state?

The alarm condition has been met. Any configured alarm actions may run, such as sending a notification or invoking a supported action.

### Does a CloudWatch alarm automatically repair an unhealthy EC2 instance?

Not by itself. An alarm evaluates a condition. Remediation requires an appropriate configured action or an operational response.

### How would you investigate an alarm showing zero healthy hosts?

I would inspect the alarm graph and history, check target health reasons, verify Nginx and port 80 on the instances, validate security-group rules, and confirm that the health-check path returns the expected response.

## 10. Key Takeaway

CloudWatch helps detect abnormal conditions, but effective monitoring requires investigation and remediation. Correlating CloudWatch metrics with ALB target health and EC2 application status helps identify the root cause.
