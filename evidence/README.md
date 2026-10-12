# Project Evidence — AWS High-Traffic Web Application

## Overview

This directory contains screenshots captured during the implementation and validation of the AWS High-Traffic Web Application project.

The evidence demonstrates the deployed website, CloudFront configuration, load-balancer target health, Auto Scaling configuration, and monitoring setup.

## Evidence Checklist

### 1. Website Homepage

**File:** `01-website-homepage.png`

Shows the professional webpage served through the public domain.

**What it demonstrates:**
- Custom-domain access over HTTPS
- Successful webpage delivery
- End-to-end application accessibility

### 2. CloudFront Distribution

**File:** `02-cloudfront-distribution.png`

Shows the CloudFront distribution configuration.

**What it demonstrates:**
- CDN configuration
- Distribution domain and status
- Origin and cache behavior configuration, where visible

### 3. Healthy Target Group

**File:** `03-target-group-healthy-targets.png`

Shows the EC2 targets registered with the Application Load Balancer.

**What it demonstrates:**
- Registered target instances
- Target health status
- Health-check validation

### 4. Auto Scaling Group

**File:** `04-auto-scaling-group.png`

Shows the Auto Scaling Group configuration.

**What it demonstrates:**
- Minimum, desired, and maximum capacity
- Instance management configuration
- Scaling policy configuration, if included in the screenshot

### 5. CloudWatch Alarms

**File:** `05-cloudwatch-alarms.png`

Shows the configured CloudWatch alarms.

**What it demonstrates:**
- Monitoring configuration
- Alarm thresholds and metrics, where visible
- Operational monitoring setup

## Screenshot Standards

For a professional portfolio:

- Capture screenshots directly from the AWS Console or deployed application.
- Keep resource names and relevant configuration visible.
- Avoid exposing access keys, passwords, session tokens, or other sensitive information.
- Use readable screenshots instead of heavily cropped images.
- Capture the actual resource state; do not edit screenshots to imply a test succeeded when it did not.

## Evidence Status

The filenames above are the planned evidence set. Add each screenshot to this directory before treating the corresponding evidence item as complete.

