# AWS Break-Fix Troubleshooting Simulation

A hands-on simulation project where I intentionally broke and then diagnosed/fixed real AWS misconfigurations — practicing the exact troubleshooting workflow used by Cloud Support Engineers: **reproduce → diagnose via logs/metrics → fix → document.**

## Why this project

Real support work isn't about knowing AWS services in theory — it's about tracing a broken system back to its root cause under time pressure, using logs and evidence rather than guesswork. This project simulates six production-style incidents across permissions, networking, performance, and monitoring, each resolved using the same process a real support ticket would require.

The first three incidents focus on IAM and policy misconfigurations, diagnosed through CloudTrail. The last three go further — a networking outage, a CPU performance issue, and a fully monitoring-driven incident where a CloudWatch alarm triggered the investigation instead of me noticing the problem myself.

## Environment Setup

- **Compute:** EC2 (Amazon Linux 2023, t3.micro) running Apache
- **Storage:** S3 bucket for project assets
- **Identity:** Dedicated IAM user (`breakfix-admin`) created following least-privilege practice
- **Logging:** CloudTrail trail (management events, all regions) — used to trace every incident back to the exact API call, user, and timestamp
- **Monitoring:** CloudWatch alarm on CPU utilization, connected to an SNS topic for email alerts

Foundation setup screenshots: [`phase1-foundation/`](./phase1-foundation)

## Incidents Simulated

| # | Scenario | Root Cause | Diagnosed Via | Resolution | Time to Fix |
|---|----------|-----------|----------------|------------|--------------|
| 001 | Website unreachable (HTTP outage) | Security Group HTTP (80) inbound rule removed | CloudTrail — `RevokeSecurityGroupIngress` | Re-added HTTP inbound rule | ~5 min |
| 002 | S3 upload failing ("Access Denied") | Bucket policy explicitly denied `s3:*` to all principals | CloudTrail — `PutBucketPolicy` | Deleted the Deny policy (explicit Deny overrides even AdministratorAccess) | ~10 min |
| 003 | EC2 console access denied | Inline IAM policy explicitly denied `ec2:DescribeInstances` | CloudTrail — `PutUserPolicy` | Removed the inline Deny policy from the IAM user | ~15 min |
| 004 | Website/SSH unreachable (connection timeout) | Route table's `0.0.0.0/0` route to the Internet Gateway was removed | CloudTrail — `DeleteRoute`, plus route table inspection | Re-added the `0.0.0.0/0` route pointing to the Internet Gateway | ~10 min |
| 005 | Server slow / high CPU | Runaway process (`yes`) consuming a full CPU core | `top` | Identified the PID and force-killed it with `kill -9` | ~10 min |
| 006 | High CPU detected via CloudWatch alarm | Runaway processes (`yes`) pushing instance-wide CPU above 70% | CloudWatch alarm notification (SNS email), then `top` to investigate | Killed the processes; alarm cleared automatically once CPU dropped | ~20 min |

Full write-ups with screenshots for each:
[`ticket-001.md`](./phase2-security-group-breakfix/ticket-001.md) · [`ticket-002.md`](./phase3-s3-policy-breakfix/ticket-002.md) · [`ticket-003.md`](./phase4-iam-breakfix/ticket-003.md) · [`ticket-004.md`](./phase5-networking-breakfix/ticket-004.md) · [`ticket-005.md`](./phase6-performance-breakfix/ticket-005.md) · [`ticket-006.md`](./phase7-monitoring-breakfix/ticket-006.md)

## Key Learnings

- **Explicit Deny always wins** — an IAM or bucket policy with `Effect: Deny` overrides any Allow, including full AdministratorAccess. Fixing an outage caused by an explicit Deny means removing the Deny, not adding a broader Allow.
- **Write actions log more reliably than read actions** — CloudTrail consistently captured every policy *change* (`Put`/`Revoke`/`Delete` events), which turned out to be the more valuable diagnostic signal than the resulting read-call errors.
- **Resource policies can lock out the owner** — an S3 bucket policy with `Principal: "*"` can deny access to the bucket owner too; recovery requires the AWS root user, not just an admin IAM user.
- **A timeout is a different clue than "connection refused"** — a timeout means traffic never reached the instance at all (a routing/security-group/NACL issue), while a refused connection means it arrived but was rejected. Knowing the difference narrows down where to look before opening a single console tab.
- **Per-process CPU and instance-wide CPU are not the same number** — a single process pegged at 100% in `top` might still show as only 50% in CloudWatch on a multi-core instance, since CloudWatch reports the average across all vCPUs. Reproducing a real alarm threshold sometimes means loading more than one core.
- **Monitoring changes how an incident starts** — the first five incidents were found by manually noticing something was broken; the sixth started from a CloudWatch alarm email instead. That's a meaningfully different (and more realistic) starting point for a support ticket, and it's how issues get caught in production before a customer ever complains.

## Repository Structure

```
aws-breakfix-troubleshooting/
├── README.md
├── phase1-foundation/
│   ├── 01-ec2-running.png
│   ├── 02-security-group-rules.png
│   ├── 03-apache-page-live.png
│   ├── 04-s3-bucket.png
│   ├── 05-iam-user.png
│   └── 06-cloudtrail-enabled.png
├── phase2-security-group-breakfix/
│   ├── 01-error-site-unreachable.png
│   ├── 02-cloudtrail-revoke-event.png
│   ├── 03-fixed-page-live.png
│   └── ticket-001.md
├── phase3-s3-policy-breakfix/
│   ├── 01-access-denied-upload.png
│   ├── 02-cloudtrail-putbucketpolicy-event.png
│   ├── 03-upload-succeeded.png
│   └── ticket-002.md
├── phase4-iam-breakfix/
│   ├── 01-unauthorized-error.png
│   ├── 02-cloudtrail-putuserpolicy-event.png
│   ├── 03-ec2-console-fixed.png
│   └── ticket-003.md
├── phase5-networking-breakfix/
│   ├── 01-route-table-healthy.png
│   ├── 02-error-timeout.png
│   ├── 03-route-table-broken.png
│   ├── 04-cloudtrail-deleteroute-event.png
│   ├── 05-route-table-fixed.png
│   ├── 06-site-restored.png
│   └── ticket-004.md
├── phase6-performance-breakfix/
│   ├── 01-cpu-normal.png
│   ├── 02-cpu-spike-top.png
│   ├── 04-cpu-fixed.png
│   └── ticket-005.md
└── phase7-monitoring-breakfix/
    ├── 01-alarm-ok-state.png
    ├── 02-alarm-in-alarm-state.png
    ├── 03-sns-email-alert.png
    ├── 04-investigation-top.png
    ├── 05-alarm-ok-again.png
    └── ticket-006.md
```

## Tools Used

AWS EC2, S3, IAM, CloudTrail, CloudWatch, SNS, VPC (default + custom route tables), Security Groups · Bash · Linux (Amazon Linux 2023)
