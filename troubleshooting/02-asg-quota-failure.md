# Incident Report: EC2 vCPU Quota Prevented Instance Launch

## 1. Incident Summary

| Field | Details |
|---|---|
| Service | Amazon EC2 / Auto Scaling |
| Region | Asia Pacific (Mumbai) — `ap-south-1` |
| Component | `high-traffic-asg` |
| Instance type | `t3.micro` |
| Problem | Replacement instance launch failed because of an EC2 vCPU quota limit |
| Impact | Scaling or instance replacement could be delayed |
| Resolution path | Review the quota, request an increase, and verify launch activity |

## 2. Problem Statement

During the project, a replacement EC2 instance launch failed because the account had reached its applicable On-Demand Standard vCPU quota.

The account's existing quota was reported as 8 vCPUs, and a request was submitted to increase it to 16 vCPUs.

**Important:** A submitted quota request does not mean that the increase has been approved. Verify its status in AWS Service Quotas before retrying.

## 3. How Auto Scaling Is Affected

```text
Auto Scaling Group
        |
        | Needs an instance
        v
EC2 Launch Request
        |
        v
EC2 vCPU Quota Check
        |
        +----------------------+
        |                      |
        v                      v
Quota available          Quota exceeded
        |                      |
        v                      v
Instance launches        Launch fails
                               |
                               v
                     Desired capacity may
                     remain unmet
```

Auto Scaling can retry launches, but retries cannot overcome an unchanged quota limit.

## 4. Investigation

### Step 1 — Inspect the Auto Scaling Group

In the AWS Console:

1. Open **EC2 → Auto Scaling Groups**.
2. Select `high-traffic-asg`.
3. Review the desired, minimum, and maximum capacity.
4. Open the **Activity** tab.
5. Find failed launch or replacement activities and read their status messages.

**What to look for:** Messages indicating that the requested instance could not launch because of a service quota.

### Step 2 — Check the EC2 quota

1. Open **AWS Service Quotas**.
2. Select **AWS services → Amazon Elastic Compute Cloud (Amazon EC2)**.
3. Search for the applicable On-Demand Standard vCPU quota.
4. Check the applied quota value and current utilization.

The quota applies across relevant instance usage in the account and region. Do not assume that the vCPU requirement of one `t3.micro` instance is the only usage that counts.

### Step 3 — Check the launch template

Open the launch template used by `high-traffic-asg` and verify:

- Instance type
- AMI
- Security group
- Subnet and Availability Zone configuration
- IAM instance profile, if required

This helps distinguish a quota failure from other launch errors.

## 5. Root-Cause Analysis

**Root cause:** The EC2 launch request exceeded the applicable account-level On-Demand Standard vCPU quota.

The quota is an AWS account and regional capacity limit, not an application error or an Nginx health-check problem.

A quota failure can prevent Auto Scaling from launching the instances required to restore or increase capacity.

## 6. Resolution Procedure

### Step 1 — Request a quota increase

In AWS Service Quotas, request an increase for the applicable On-Demand Standard vCPU quota in `ap-south-1`.

For this project, a request to increase the reported limit from 8 to 16 vCPUs was submitted.

### Step 2 — Verify approval

Check the quota request status.

- **Pending:** Wait for AWS to process the request.
- **Approved:** Verify the new applied quota.
- **Denied:** Review the request details and available quota options before deciding on next steps.

Do not assume approval until the applied quota confirms the change.

### Step 3 — Recheck Auto Scaling activity

After the quota issue is resolved:

1. Return to the Auto Scaling Group's **Activity** tab.
2. Check whether new launch attempts succeed.
3. Confirm that the required instances reach the `InService` state.
4. Verify that the target group reports healthy targets.

### Step 4 — Validate the application

Check the target group in the EC2 console and confirm that registered targets are healthy.

Finally, open the public website and verify that the application remains accessible.

## 7. Prevention

- Review EC2 quota utilization before capacity-intensive tests.
- Account for other running instances in the same quota category and region.
- Monitor Auto Scaling activities for failed launches.
- Set realistic minimum, desired, and maximum capacities.
- Request quota increases before planned load tests or scaling demonstrations.
- Keep launch-template configuration and instance requirements documented.

## 8. Verification Checklist

- [ ] Applicable EC2 vCPU quota reviewed.
- [ ] Quota request status checked.
- [ ] Applied quota verified if approved.
- [ ] Auto Scaling launch activity reviewed.
- [ ] Required instances reached `InService`.
- [ ] Target group health verified.
- [ ] Public application tested.

## 9. Interview Explanation

**Question: What happens when an Auto Scaling Group cannot launch an EC2 instance because of a quota limit?**

Auto Scaling requests an instance using its launch template, but EC2 rejects the launch when the applicable account-level quota has been exceeded. Auto Scaling may retry, but it cannot resolve the quota restriction itself. I investigate the Activity tab, identify the quota-related failure, review the applicable regional vCPU quota, and request an increase when necessary. After the quota is resolved, I verify successful instance launches, target health, and application availability.
