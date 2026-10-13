# AWS Getting Started — Zero to Deployed

First-day path for a brand-new AWS account: sign up, secure it, set budgets, launch a server, connect from your laptop, and deploy a web application.

## Learning path

1. [From Zero to Deployed Guide](aws-from-zero-to-deployed.md)
2. [Formatted HTML version](aws-from-zero-to-deployed.html)

## What the guide covers

- Creating and securing an AWS account (root + MFA)
- Billing budgets and free-tier alerts (day-one safety)
- Console tour: regions, services, CloudShell
- IAM admin user and AWS CLI setup on your PC
- Launching your first free-tier EC2 instance
- SSH from Windows, macOS, and Linux
- Running Nginx / a Node.js app on the server
- Security groups, ports, and public access
- SSH port forwarding for private services
- Elastic IPs, EBS snapshots, monitoring
- Stop vs terminate, cleanup, and cost traps
- Troubleshooting the classic beginner errors

## Quick navigation

| Topic | Section |
|---|---|
| Account creation | [Guide](aws-from-zero-to-deployed.md#3-create-your-aws-account) |
| Root MFA | [Guide](aws-from-zero-to-deployed.md#4-lock-down-the-root-account) |
| Budgets and alerts | [Guide](aws-from-zero-to-deployed.md#5-billing-and-budgets-do-this-on-day-one) |
| IAM admin + CLI | [Guide](aws-from-zero-to-deployed.md#7-create-your-daily-use-admin-identity) |
| First EC2 launch | [Guide](aws-from-zero-to-deployed.md#10-launch-your-first-ec2-instance) |
| SSH connection | [Guide](aws-from-zero-to-deployed.md#11-connect-from-your-pc-with-ssh) |
| Run a web app | [Guide](aws-from-zero-to-deployed.md#13-run-your-first-web-application) |
| Ports and security groups | [Guide](aws-from-zero-to-deployed.md#14-open-the-app-to-the-world-ports-and-security-groups) |
| Port forwarding | [Guide](aws-from-zero-to-deployed.md#15-port-forwarding-explained) |
| Avoiding surprise bills | [Guide](aws-from-zero-to-deployed.md#21-stop-vs-terminate-avoiding-surprise-bills) |
| Troubleshooting | [Guide](aws-from-zero-to-deployed.md#23-troubleshooting-common-errors) |

## Recommended order

```text
Create account + MFA
   ↓
Budgets and free-tier alerts
   ↓
IAM admin user + AWS CLI
   ↓
Launch free-tier EC2 + security group
   ↓
SSH from your PC + update server
   ↓
Install Nginx / run your app
   ↓
Open port 80 -> browser test
   ↓
Learn port forwarding + stop instance
   ↓
Deepen with EC2 and IAM guides
```

> New to AWS entirely? Start here. Already have an account and want deeper networking or permissions? Go to the [EC2](../EC2-Instance/index.md) or [IAM](../IAM/index.md) guides.

[Back to AWS index](../index.md) · [Back to site home](../../index.md)
