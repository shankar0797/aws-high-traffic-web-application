# EC2 Auto Scaling — Configuration and Validation

## 1. Overview

Amazon EC2 Auto Scaling helps maintain the desired number of EC2 instances and adjusts capacity according to configured scaling policies.

In this project, the Auto Scaling Group manages EC2 instances that run Nginx and serve the website behind an Application Load Balancer.

**AWS Region:** `ap-south-1` — Mumbai

**Auto Scaling Group:** `high-traffic-asg`

## 2. Architecture

```text
       Auto Scaling Group
       Min: 2 | Desired: 2 | Max: 4
                    |
                    v
             Launch Template
                    |
                    v
               EC2 Instances
                /         \
               v           v
           EC2 - A       EC2 - B
            Nginx         Nginx
               \           /
                v         v
                ALB Target Group
                       |
                       v
                Application Traffic

      CloudWatch Metrics → Scaling Policy
```

The launch template provides the instance configuration. The Auto Scaling Group manages the instances, while the ALB target group evaluates whether instances are healthy enough to receive application traffic.

## 3. Key Components

### Launch template

The launch template defines how EC2 instances should be created.

Typical settings include:

- Amazon Machine Image (AMI)
- Instance type
- Security groups
- IAM instance profile, if required
- Storage configuration
- User data for installing and configuring Nginx

This project used the launch template `high-traffic-launch-template-v2`.

A launch template can define an instance, but it does not guarantee that the application is correctly installed. User data or a prepared AMI should configure the web server consistently.

### Auto Scaling Group

The Auto Scaling Group maintains the desired capacity and can launch or terminate instances according to its configuration and scaling policies.

The project's configured capacity was:

| Setting | Value |
|---|---:|
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |

- **Minimum:** The group should not normally scale below this configured capacity.
- **Desired:** The capacity the group attempts to maintain.
- **Maximum:** The upper capacity limit for scaling activities.

### Target tracking policy

The group was configured with a CPU target tracking policy targeting **50% CPU utilization**.

Target tracking evaluates the policy's metric and adjusts capacity to move that metric toward the target. It does not guarantee an immediate launch whenever a single CPU reading exceeds 50%.

### Instance warm-up

The configured instance warm-up was **180 seconds**.

Warm-up allows a newly launched instance time to initialize before its metrics are fully considered for certain scaling decisions.

## 4. How Scale-Out Works

```text
Workload increases on ASG instances
                 |
                 v
       CloudWatch CPU metrics
                 |
                 v
        Target tracking policy
                 |
                 v
       Scaling decision evaluated
                 |
                 v
       Auto Scaling launches EC2
                 |
                 v
       Instance initializes Nginx
                 |
                 v
       Instance registers as target
                 |
                 v
       Health check succeeds
```

The exact timing depends on metric collection, policy evaluation, instance launch time, warm-up settings, and application initialization.

## 5. Validate Auto Scaling Configuration

### Step 1 — Inspect the group

In the AWS Console:

1. Open **EC2 → Auto Scaling Groups**.
2. Select `high-traffic-asg`.
3. Review minimum, desired, and maximum capacity.
4. Review the associated launch template.
5. Open the **Automatic scaling** tab and inspect the target tracking policy.

### Step 2 — Review scaling activities

Open the group's **Activity** tab.

Look for:

- Launch requests
- Successful instance launches
- Failed launch attempts
- Scale-out or scale-in activities
- Quota-related failures

The Activity tab helps explain what the Auto Scaling service attempted and why a launch may have failed.

### Step 3 — Review CloudWatch metrics

Open **CloudWatch → Metrics → EC2** and inspect CPU utilization for the instances managed by the group.

Use a time range that includes the test. Check that the instance generating load is actually a member of `high-traffic-asg`.

### Step 4 — Confirm instance health

Verify that new instances reach `InService` and that the ALB target group reports them as healthy.

An instance launch alone does not prove that the website is working correctly.

## 6. CPU Stress-Test Validation

During the initial project test, CPU load was generated on a manually created EC2 instance rather than an instance managed by the Auto Scaling Group. This did not reliably validate the group's scaling policy.

For a controlled lab test:

1. Identify an instance managed by `high-traffic-asg`.
2. Confirm the scaling policy and CloudWatch metric.
3. Generate a controlled workload only on the lab instance.
4. Monitor CPU utilization and scaling activities.
5. Verify whether the group launches an additional instance.
6. Confirm the new instance becomes healthy in the target group.
7. Stop the test and allow the system to stabilize.

If `stress-ng` is installed, a short lab test could use:

```bash
stress-ng --cpu 2 --cpu-load 90 --timeout 360s
```

Run this only in an authorized test environment. It attempts to load two CPU workers for six minutes; the actual utilization depends on the instance and workload.

**Important:** Do not claim scale-out succeeded unless the Activity tab, instance count, and target health confirm it.

## 7. Troubleshooting

| Symptom | Investigation |
|---|---|
| No scale-out occurs | Verify the policy metric and load on ASG-managed instances |
| CPU stays below target | Inspect CloudWatch metrics and workload |
| Launch request fails | Read the Auto Scaling Activity status message |
| vCPU quota exceeded | Review the applicable EC2 quota in the region |
| Instance launches but remains unhealthy | Check Nginx, security groups, port 80, and health checks |
| Group cannot exceed its capacity | Verify maximum capacity |
| New instance takes time to stabilize | Review instance initialization and warm-up settings |

## 8. Production Best Practices

- Keep launch-template configuration versioned and reviewed.
- Automate application installation and startup.
- Use health checks that reflect application readiness.
- Monitor failed launches and scaling activities.
- Review EC2 quotas before planned scaling tests.
- Test scale-out and scale-in behavior in a controlled environment.
- Confirm that the ALB has healthy targets after deployments.
- Review capacity settings and cost implications regularly.

## 9. Interview Questions

### What is the difference between a launch template and an Auto Scaling Group?

The launch template defines how an instance should be configured. The Auto Scaling Group manages the number of instances and their lifecycle.

### What does target tracking do?

It adjusts capacity based on a configured metric and target value, attempting to keep the metric near the target.

### Why might an Auto Scaling Group fail to launch an instance?

Possible reasons include EC2 quotas, invalid launch-template configuration, insufficient capacity, subnet constraints, or other EC2 launch errors. The Activity tab provides the first place to investigate.

### How did you test scaling?

The initial test was run on a manually created instance, so it did not reliably test the group's scaling policy. The correct validation is to generate controlled load on an ASG-managed instance, monitor CloudWatch, inspect scaling activities, and verify the health of any new target.

## 10. Key Takeaway

Auto Scaling is not just about adding instances. Reliable scaling requires a valid launch template, appropriate capacity limits, a correctly configured policy, successful EC2 launches, and healthy application targets.
