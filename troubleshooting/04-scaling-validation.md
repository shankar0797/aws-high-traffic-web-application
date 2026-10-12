# Troubleshooting: Auto Scaling Did Not Scale Out

## 1. Incident Summary

| Field | Details |
|---|---|
| AWS service | EC2 Auto Scaling |
| Auto Scaling Group | `high-traffic-asg` |
| Region | `ap-south-1` — Mumbai |
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| Scaling policy | Target tracking |
| CPU target | 50% |
| Instance warm-up | 180 seconds |
| Observed issue | Expected scale-out did not occur during the initial stress test |

## 2. Problem Statement

During the project, a CPU stress test was attempted, but the Auto Scaling Group did not launch additional instances as expected.

The initial test was run on a manually created EC2 instance rather than an instance managed by `high-traffic-asg`.

This meant the test did not necessarily generate load on the instances whose CPU metrics were used by the Auto Scaling policy.

## 3. How CPU-Based Auto Scaling Works

```text
CPU load generated on ASG instances
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
       ASG requests new instance
                |
                v
      EC2 instance launches
                |
                v
       Target becomes healthy
```

Target tracking does not simply launch an instance whenever any EC2 instance experiences high CPU. It evaluates the metrics associated with the scaling policy and the Auto Scaling Group.

## 4. Investigation

### Step 1 — Verify Auto Scaling configuration

In the AWS Console:

1. Open **EC2 → Auto Scaling Groups**.
2. Select `high-traffic-asg`.
3. Verify minimum, desired, and maximum capacity.
4. Open the **Automatic scaling** tab.
5. Review the target tracking policy and its metric.

Confirm that the policy is attached to the intended group.

### Step 2 — Identify the test instance

Check whether the instance generating CPU load belongs to `high-traffic-asg`.

An independently launched EC2 instance is not automatically part of the group's scaling decision.

### Step 3 — Check CloudWatch metrics

Open **CloudWatch → Metrics → EC2**.

Review the CPU utilization metric for the actual Auto Scaling instances during the test period.

Compare the timestamps with the stress test. A low average CPU metric or a short-lived spike may not produce the scaling behavior you expected.

### Step 4 — Review scaling activities

Open the Auto Scaling Group's **Activity** tab.

Look for:

- Successful launch activities
- Failed launch attempts
- Scaling cooldown or warm-up effects
- Capacity limits
- Quota-related errors

A scaling policy can request more capacity, but instance launch failures can prevent the desired result.

## 5. Root-Cause Analysis

**Confirmed test-design issue:** The initial stress test ran on a manually created instance rather than an instance managed by the Auto Scaling Group.

Therefore, that test did not reliably validate CPU-based scaling for `high-traffic-asg`.

The available observations do not establish that this was the only possible issue. The policy metric, CloudWatch data, scaling activities, and launch errors must also be reviewed.

## 6. Correct Validation Procedure

### Step 1 — Confirm the target tracking policy

Verify that the policy targets CPU utilization at 50% and is attached to `high-traffic-asg`.

### Step 2 — Connect to an ASG-managed instance

Identify an instance registered with the group and connect using your approved access method.

**Safety:** Run a controlled test only on your lab environment. Do not generate heavy load on production systems without authorization.

### Step 3 — Generate controlled CPU load

If `stress-ng` is installed on the test instance:

```bash
stress-ng --cpu 2 --cpu-load 90 --timeout 360s
```

This attempts to load two CPU workers for six minutes. Confirm the number of vCPUs and monitor the instance while testing.

### Step 4 — Observe CloudWatch

Monitor the CPU utilization metric for the test instance and the Auto Scaling policy's configured metric.

Allow time for metric publication and scaling evaluation. Target tracking is not instantaneous.

### Step 5 — Review Auto Scaling activity

Check whether the group starts a scale-out activity.

If it does not, review the metric, policy configuration, group limits, warm-up behavior, and activity messages before changing the configuration.

### Step 6 — Verify the outcome

A successful scale-out test should show:

- A scaling activity indicating additional capacity was requested.
- A new EC2 instance launched successfully.
- The new instance associated with the Auto Scaling Group.
- The instance reaching `InService`.
- The target group reporting the instance as healthy.

Do not report scale-out as successful until these checks pass.

## 7. Why Scaling May Not Occur

| Possible cause | What to check |
|---|---|
| Load generated on the wrong instance | Confirm the instance belongs to the ASG |
| CPU metric remains below target | Inspect the correct CloudWatch metric and time range |
| Scaling policy misconfigured | Verify the target tracking policy and metric |
| Maximum capacity reached | Check the ASG maximum |
| Instance launch failure | Review activity history and EC2 errors |
| EC2 quota exceeded | Check the applicable regional vCPU quota |
| Evaluation or warm-up delay | Review policy timing and recent scaling activities |

## 8. Prevention

- Generate test load only on instances managed by the intended ASG.
- Confirm the scaling policy and metric before testing.
- Record the start and end time of the test.
- Monitor CPU metrics and scaling activities together.
- Check EC2 quotas before increasing capacity.
- Keep a before-and-after record of desired capacity, instance count, and target health.
- Clean up test processes after validation.

## 9. Interview Explanation

**Question: Why did your Auto Scaling Group not scale out during a CPU test?**

During my initial validation, I generated CPU load on a manually created EC2 instance instead of an instance managed by the Auto Scaling Group. That did not reliably test the group's CPU-based scaling policy. I identified the test-design issue and established a better validation process: generate controlled load on an ASG-managed instance, inspect CloudWatch metrics, review scaling activities, and verify that any newly launched instance becomes healthy in the target group.
