# Incident Report: CloudFront Serving an Old Webpage

## 1. Incident Summary

| Field | Details |
|---|---|
| Incident | Old webpage displayed through the custom domain |
| Services | Amazon CloudFront, Application Load Balancer, EC2, Nginx |
| Environment | AWS Mumbai — `ap-south-1` |
| Impact | Users could receive stale website content |
| Resolution | Investigated the origin and CDN responses; completed a CloudFront invalidation |
| Status | Resolved during the hands-on project |

## 2. Problem Statement

The updated professional webpage was available through the Application Load Balancer, but the public website continued to display an older version.

This indicated that the response delivered through the custom domain differed from the response available at the origin.

## 3. Architecture Involved

```text
Browser
   |
   v
Route 53
   |
   v
CloudFront
   |
   | Configured origin
   v
Application Load Balancer
   |
   v
Target Group
   |
   v
EC2 Instance
   |
   v
Nginx / HTML
```

## 4. Symptoms Observed

- The updated webpage was present on the EC2 instance.
- The ALB returned the updated HTML.
- The custom domain returned an older webpage.
- CloudFront response headers indicated a cache hit during the earlier investigation.
- A CloudFront invalidation was completed, after which the website displayed the updated content.

## 5. Investigation Steps

### Step 1 — Verify the application on EC2

Run this command on an EC2 instance with access to its shell:

```bash
sudo systemctl status nginx
```

Check the locally served page:

```bash
curl -sS http://localhost/ | head -30
```

**Purpose:** Confirm that Nginx is running and inspect the content being served locally.

### Step 2 — Check the ALB response

Run this from a terminal that can reach the public ALB:

```bash
curl -sS http://high-traffic-alb-1560459725.ap-south-1.elb.amazonaws.com/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

**Purpose:** Verify whether the ALB returns the updated webpage.

### Step 3 — Check the custom domain

```bash
curl -sS https://techwithshiva.in/ \
  | grep -E '<title>|AWS High-Traffic|OK - High Traffic'
```

**Purpose:** Compare the content delivered to users with the content served by the ALB.

### Step 4 — Inspect CloudFront response headers

```bash
curl -sSI https://techwithshiva.in/
```

Review these headers:

| Header | Diagnostic use |
|---|---|
| `X-Cache` | Indicates whether CloudFront reports a cache hit or miss |
| `Age` | Helps identify the age of a cached response |
| `ETag` | Helps compare response versions |
| `Last-Modified` | Helps inspect the origin content's modification time |

A cache hit alone does not prove that the content is incorrect. Compare the actual response body and relevant headers.

## 6. Root-Cause Analysis

**Observed cause:** CloudFront was delivering stale content while the ALB already returned the updated webpage.

The investigation pointed to CDN caching as the issue. The available observations do not independently establish the exact cache-policy or object-level reason the old response persisted.

### Why this matters

Updating a file on EC2 does not automatically guarantee that every cached copy of that file is refreshed immediately.

When the origin and public domain return different versions, investigate the delivery path before rebuilding the application or replacing healthy instances.

## 7. Resolution

1. Confirmed that the updated HTML was served by the EC2/Nginx origin.
2. Verified that the ALB returned the updated webpage.
3. Investigated the custom-domain response and CloudFront headers.
4. Created a CloudFront invalidation for the relevant content.
5. Waited for the invalidation to complete.
6. Retested the public domain and confirmed that the updated webpage was displayed.

For a website where the root page is affected, an invalidation such as `/*` can refresh cached objects across the distribution. Use a narrower path when only specific objects need refreshing.

## 8. Prevention and Best Practices

- Set cache-control headers according to the type of content.
- Use versioned filenames for frequently updated static assets.
- Invalidate affected paths after deployments when required.
- Validate the origin response before investigating the CDN.
- Compare response bodies and headers instead of relying on a single header.
- Automate post-deployment checks against both the origin and public domain.

## 9. Verification Checklist

- [ ] Nginx is running.
- [ ] The EC2 instance serves the intended webpage.
- [ ] The ALB returns the intended webpage.
- [ ] The CloudFront invalidation is complete.
- [ ] The custom domain displays the updated webpage.
- [ ] The response headers have been reviewed.

## 10. Interview Explanation

**Question: Describe a production-style issue you troubleshot in AWS.**

During my AWS web application project, the ALB was returning the updated webpage, but the custom domain continued to display an older version. I compared the origin and CDN responses, inspected CloudFront headers, and identified stale content in the delivery path. I completed a CloudFront invalidation and verified that the updated page appeared through the public domain. This taught me to isolate the problem layer by layer before changing healthy infrastructure.
