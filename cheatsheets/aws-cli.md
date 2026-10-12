# AWS CLI Cheat Sheet — High-Traffic Web Application

## 1. Overview

This cheat sheet contains useful AWS CLI commands for inspecting the resources used in the high-traffic web application project.

**Default region:** `ap-south-1` — Mumbai

**AWS CLI version:** AWS CLI v2

> Run commands from a terminal where AWS CLI is installed and configured. Use an IAM identity with the required permissions. Do not store access keys, secret keys, or session tokens in this repository.

## 2. Verify AWS CLI Access

### Check the installed version

```bash
aws --version
```

### Check the active AWS identity

```bash
aws sts get-caller-identity
```

This returns the AWS account ID, caller ARN, and user or role ID associated with the active credentials.

### Set the default region for a command

```bash
aws ec2 describe-instances --region ap-south-1
```

Using `--region` explicitly helps avoid inspecting resources in the wrong region.

## 3. EC2 Instances

### List instances

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType,PrivateIP:PrivateIpAddress}' \
  --output table
```

### Inspect a specific instance

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --instance-ids YOUR_INSTANCE_ID
```

Replace `YOUR_INSTANCE_ID` with the actual instance ID.

### Check instance status

```bash
aws ec2 describe-instance-status \
  --region ap-south-1 \
  --instance-ids YOUR_INSTANCE_ID \
  --include-all-instances
```

This reports instance and system status checks. It does not prove that Nginx or the application is healthy.

## 4. Application Load Balancer

### List load balancers

```bash
aws elbv2 describe-load-balancers \
  --region ap-south-1 \
  --query 'LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,State:State.Code,Type:Type}' \
  --output table
```

### List target groups

```bash
aws elbv2 describe-target-groups \
  --region ap-south-1 \
  --query 'TargetGroups[].{Name:TargetGroupName,ARN:TargetGroupArn,Port:Port,Protocol:Protocol}' \
  --output table
```

### Check target health

First, find the ARN for `high-traffic-tg` from the target-group output. Then run:

```bash
aws elbv2 describe-target-health \
  --region ap-south-1 \
  --target-group-arn YOUR_TARGET_GROUP_ARN \
  --output table
```

Check the target state and reason fields for unhealthy targets.

**Troubleshooting:** If a target is unhealthy, inspect the target health reason, Nginx service status, listening port, security-group rules, and health-check path.

## 5. EC2 Auto Scaling

### List Auto Scaling Groups

```bash
aws autoscaling describe-auto-scaling-groups \
  --region ap-south-1 \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:length(Instances)}' \
  --output table
```

### Inspect the project Auto Scaling Group

```bash
aws autoscaling describe-auto-scaling-groups \
  --region ap-south-1 \
  --auto-scaling-group-names high-traffic-asg
```

### Check recent scaling activities

```bash
aws autoscaling describe-scaling-activities \
  --region ap-south-1 \
  --auto-scaling-group-name high-traffic-asg \
  --max-items 10
```

Inspect activity descriptions and status messages for launch failures, quota issues, and scaling events.

### Check scaling policies

```bash
aws autoscaling describe-policies \
  --region ap-south-1 \
  --auto-scaling-group-name high-traffic-asg
```

These commands inspect configuration and activity. They do not, by themselves, prove that scale-out was successful.

## 6. CloudWatch Monitoring

### List alarms

```bash
aws cloudwatch describe-alarms \
  --region ap-south-1 \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table
```

### Inspect one alarm

```bash
aws cloudwatch describe-alarms \
  --region ap-south-1 \
  --alarm-names high-traffic-low-healthy-hosts-1a
```

### Retrieve EC2 CPU metrics

Replace the instance ID with an actual instance managed by the Auto Scaling Group.

```bash
aws cloudwatch get-metric-statistics \
  --region ap-south-1 \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=YOUR_INSTANCE_ID \
  --start-time 2026-10-12T00:00:00Z \
  --end-time 2026-10-12T01:00:00Z \
  --period 300 \
  --statistics Average
```

Replace the example timestamps with the UTC time range you want to investigate.

### Retrieve ALB healthy-host metrics

```bash
aws cloudwatch list-metrics \
  --region ap-south-1 \
  --namespace AWS/ApplicationELB \
  --metric-name HealthyHostCount
```

For metric retrieval, use the actual load balancer and target group dimensions returned by CloudWatch. `list-metrics` helps discover available dimension combinations; it does not return the healthy-host value itself.

## 7. CloudFront

### List distributions

```bash
aws cloudfront list-distributions \
  --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Status:Status,Enabled:Enabled}' \
  --output table
```

CloudFront is a global service, so these commands do not need `--region ap-south-1`.

### Inspect a distribution

```bash
aws cloudfront get-distribution \
  --id YOUR_DISTRIBUTION_ID
```

### Create an invalidation

Use this only when a cached response needs to be invalidated.

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

An invalidation request is not the same as completed invalidation. Check its status before concluding that the cache update is complete.

## 8. Route 53

### List hosted zones

```bash
aws route53 list-hosted-zones \
  --query 'HostedZones[].{Name:Name,ID:Id,Private:Config.PrivateZone}' \
  --output table
```

### Inspect DNS records

Replace the hosted-zone ID with the actual ID. The ID may include a `/hostedzone/` prefix; use the ID format accepted by the CLI.

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id YOUR_HOSTED_ZONE_ID
```

### Test DNS resolution from your terminal

```bash
dig techwithshiva.in
```

This tests DNS resolution from the current environment. It does not confirm that CloudFront's origin or the application is healthy.

## 9. Common Troubleshooting Commands

### Check the website response

```bash
curl -sSI https://techwithshiva.in
```

Inspect the HTTP status and headers such as `X-Cache`, `Age`, and `Cache-Control`.

### Check Nginx on an EC2 instance

Run after connecting to the instance through an approved access method:

```bash
sudo systemctl status nginx
```

### Check whether port 80 is listening

```bash
sudo ss -lntp | grep ':80'
```

### Test the local web server

```bash
curl -i http://localhost/
```

A successful local response does not by itself prove that the ALB can reach the instance. Check network rules and target health as well.

## 10. Common CLI Errors

| Error or symptom | What to check |
|---|---|
| `Unable to locate credentials` | Active AWS CLI credentials or role configuration |
| `AccessDenied` | IAM permissions for the requested operation |
| Resource not found | Correct resource ID and AWS region |
| Empty result | Region, filters, resource names, and permissions |
| Target unhealthy | Target health reason, application service, port, and security groups |
| Auto Scaling launch failure | Scaling activity status message and applicable EC2 quota |
| Old website content | CloudFront cache headers, selected origin, and invalidation status |

## 11. Best Practices

- Use IAM roles or another approved credential provider instead of hardcoding secrets.
- Confirm the account and region before making changes.
- Prefer read-only inspection commands during initial troubleshooting.
- Use `--query` and `--output table` to make results easier to read.
- Avoid destructive commands until the target resource has been verified.
- Never commit credentials or sensitive account information into public repositories.
- Verify the outcome of an operation instead of assuming a successful command means the application is healthy.

## 12. Key Takeaway

AWS CLI helps inspect resources, retrieve metrics, and investigate operational problems quickly. Combine CLI output with CloudWatch metrics, target health, service logs, and application tests to identify root causes.
