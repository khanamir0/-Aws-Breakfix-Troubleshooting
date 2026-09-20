# AWS Break-Fix Troubleshooting Simulation

A hands-on simulation project where I intentionally broke and then diagnosed/fixed real AWS misconfigurations — practicing the exact troubleshooting workflow used by Cloud Support Engineers: **reproduce → diagnose via logs → fix → document.**

## Why this project

Real support work isn't about knowing AWS services in theory — it's about tracing a broken system back to its root cause under time pressure, using logs and evidence rather than guesswork. This project simulates three common production incidents, each resolved using the same process a support ticket would require.

## Environment Setup

- **Compute:** EC2 (Amazon Linux 2023, t3.micro) running Apache
- **Storage:** S3 bucket for project assets
- **Identity:** Dedicated IAM user (`breakfix-admin`) created following least-privilege practice
- **Logging:** CloudTrail trail (management events, all regions) — used to trace every incident back to the exact API call, user, and timestamp

Foundation setup screenshots: [`phase1-foundation/`](./phase1-foundation)

## Incidents Simulated

| # | Scenario | Root Cause | Diagnosed Via | Resolution | Time to Fix |
|---|----------|-----------|----------------|------------|--------------|
| 001 | Website unreachable (HTTP outage) | Security Group HTTP (80) inbound rule removed | CloudTrail — `RevokeSecurityGroupIngress` | Re-added HTTP inbound rule | ~5 min |
| 002 | S3 upload failing ("Access Denied") | Bucket policy explicitly denied `s3:*` to all principals | CloudTrail — `PutBucketPolicy` | Deleted the Deny policy (explicit Deny overrides even AdministratorAccess) | ~10 min |
| 003 | EC2 console access denied | Inline IAM policy explicitly denied `ec2:DescribeInstances` | CloudTrail — `PutUserPolicy` | Removed the inline Deny policy from the IAM user | ~15 min |

Full write-ups with screenshots for each: [`ticket-001.md`](./phase2-security-group-breakfix/Ticket-001.md) · [`ticket-002.md`](./phase3-s3-policy-breakfix/Ticket-002.md) · [`ticket-003.md`](./phase4-iam-breakfix/Ticket-003.md)

## Key Learnings

- **Explicit Deny always wins** — an IAM or bucket policy with `Effect: Deny` overrides any Allow, including full AdministratorAccess. Fixing an outage caused by an explicit Deny means removing the Deny, not adding a broader Allow.
- **Write actions log more reliably than read actions** — CloudTrail consistently captured every policy *change* (`Put`/`Revoke`/`Delete` events), which turned out to be the more valuable diagnostic signal than the resulting read-call errors.
- **Resource policies can lock out the owner** — an S3 bucket policy with `Principal: "*"` can deny access to the bucket owner too; recovery requires the AWS root user, not just an admin IAM user.

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
└── phase4-iam-breakfix/
    ├── 01-unauthorized-error.png
    ├── 02-cloudtrail-putuserpolicy-event.png
    ├── 03-ec2-console-fixed.png
    └── ticket-003.md
```

## Tools Used

AWS EC2, S3, IAM, CloudTrail, VPC (default), Security Groups · Bash · Linux (Amazon Linux 2023)
