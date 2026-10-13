# Amazon IAM — Practical Guide

A GitHub-friendly guide to AWS Identity and Access Management: who you are, what you can do, and how AWS decides.

## Contents

- [1. What is IAM?](#1-what-is-iam)
- [2. How access works (the big picture)](#2-how-access-works-the-big-picture)
- [3. IAM entities: users, groups, roles](#3-iam-entities-users-groups-roles)
- [4. Policy types](#4-policy-types)
- [5. Anatomy of a policy](#5-anatomy-of-a-policy)
- [6. Policy evaluation logic](#6-policy-evaluation-logic)
- [7. IAM roles in depth](#7-iam-roles-in-depth)
- [8. Federation and IAM Identity Center](#8-federation-and-iam-identity-center)
- [9. MFA and access security](#9-mfa-and-access-security)
- [10. Tools: CLI, simulator, Access Analyzer](#10-tools-cli-simulator-access-analyzer)
- [11. Troubleshooting AccessDenied](#11-troubleshooting-accessdenied)
- [12. IAM best practices checklist](#12-iam-best-practices-checklist)

## 1. What is IAM?

AWS Identity and Access Management (IAM) controls **who can call AWS** and **what they are allowed to do**. It is global (not per-region) and free to use.

> **Mental model:** IAM answers three questions for every API call:
> 1. **Who is asking?** (principal: user, role, or AWS service)
> 2. **What do they want to do?** (action, e.g. `s3:GetObject`)
> 3. **On which resource?** (e.g. `arn:aws:s3:::my-bucket/*`)
>
> If an explicit **Deny** matches anywhere, the request is denied. Otherwise an explicit **Allow** is needed. No allow = deny by default.

| Term | Meaning |
|---|---|
| Principal | An identity that can make requests: IAM user, IAM role, or an AWS service. |
| Permission | One API action, such as `ec2:StartInstances`. |
| Policy | A JSON document that lists permissions (Allow/Deny on actions/resources). |
| ARN | Amazon Resource Name — unique address of a resource, e.g. `arn:aws:s3:::my-bucket`. |
| Root account | The account owner email; full access, should rarely be used. |

## 2. How access works (the big picture)

```text
Principal (user / role / service)
        |
        v
Policies attached to the principal  ----+
        |                               |
        v                               v
Service control policies (org)    Resource-based policies
        |                               |
        +---------------+---------------+
                        v
              Evaluation: Deny wins,
              else Allow needed
                        |
                        v
              AWS API call succeeds / fails
```

Two ways to grant access:

| Style | Where the policy lives | Example |
|---|---|---|
| Identity-based | Attached to a user, group, or role | "This role may read from bucket X." |
| Resource-based | Attached to the resource itself | "Bucket X allows this account/role to write." |

Most permissions are identity-based. Resource policies are common for S3 bucket policies, Lambda function policies, KMS key policies, and cross-account access.

## 3. IAM entities: users, groups, roles

| Entity | What it is | When to use |
|---|---|---|
| **IAM user** | A long-lived identity with a password (console) and/or access keys (CLI/API). | Humans that need steady access; legacy cases. Prefer roles + SSO now. |
| **IAM group** | A collection of users that share the same policies. Groups cannot be principals in policies. | Applying the same permissions to many users (e.g. `Developers`, `Admins`). |
| **IAM role** | A set of permissions that **anyone/something can temporarily assume**. Has a trust policy (who may assume) + permissions policy (what they can do). | EC2/Lambda service access, cross-account access, SSO/federation, switching contexts. |
| **Instance profile** | A wrapper that delivers an IAM role's credentials to an EC2 instance. | Attaching a role to EC2 (created automatically when you attach a role in the console). |

### Root user vs IAM user

| | Root user | IAM user / role |
|---|---|---|
| Identity | Account root email + password | Named identity inside the account |
| Permissions | Everything (no restrictions) | Only what policies allow |
| Recommended use | Few account-level tasks only | All daily work |
| MFA | Strongly required | Required by best practice |

> **Golden rule:** Do not use the root account for everyday work. Do not create long-lived IAM users with access keys when a role (via IAM Identity Center) will do.

### Access methods compared

| Method | Credential | Typical use |
|---|---|---|
| Console | Username + password (+ MFA) | Humans in the browser |
| AWS CLI / SDK | Access key ID + secret access key | Scripts, CI/CD, local dev (prefer roles) |
| Assumed role | Temporary STS credentials | EC2, Lambda, cross-account, SSO |
| IAM Identity Center | SSO portal / AWS CLI SSO | Humans, org-wide access |

## 4. Policy types

| Policy type | Attached to | Purpose |
|---|---|---|
| **AWS managed policy** | Users, groups, roles | Ready-made policies maintained by AWS (e.g. `AmazonS3ReadOnlyAccess`). Convenient but often broad. |
| **Customer managed policy** | Users, groups, roles | Your own reusable policy; versioned, editable — preferred for production. |
| **Inline policy** | One specific user/group/role | Embedded in the entity; deleted with the entity. Use sparingly (harder to audit/reuse). |
| **Permissions boundary** | IAM entity | A **maximum** permission limit. An entity can never exceed the boundary, even with broader policies. Great for limiting roles created by developers. |
| **Service control policy (SCP)** | OU / account (AWS Organizations) | Org-wide guardrail. Does **not** grant permissions by itself — it filters what member accounts can ever do. |
| **Session policy** | Passed at assume-role time | Extra limit applied to the session; used by advanced role-assumption flows. |

> SCPs **limit**, they do not **grant**. An identity still needs an Allow in its own policy; the SCP just removes what is outside the org guardrail.

```text
Effective permissions =
    Identity policy  ∩  Permissions boundary  ∩  SCP  ∩  (resource policy allow)
    ...unless an explicit Deny applies anywhere → Deny.
```

## 5. Anatomy of a policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadFromOneBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `Version` | Policy language version; use `"2012-10-17"` for all modern policies. |
| `Statement` | Array of permission rules. |
| `Sid` | Optional label to identify a statement. |
| `Effect` | `Allow` or `Deny`. |
| `Action` | API actions, e.g. `s3:GetObject`, or wildcards `s3:*`. |
| `Resource` | Target ARNs. Use `*` for "all resources" carefully. |
| `Condition` | Extra tests: region, MFA, source IP, tags, time, etc. |
| `Principal` | **Only in resource-based policies** — who may use the resource (account, role ARN, `*`). |

### Action wildcards

```text
s3:*            all S3 actions          (too broad for production)
s3:Get*         all actions starting with Get
s3:GetObject    one precise action      (prefer this)
```

### Conditions you will actually use

| Condition key | Example | Effect |
|---|---|---|
| `aws:MultiFactorAuthPresent` | `"Bool": {"aws:MultiFactorAuthPresent": "true"}` | Require MFA for the action. |
| `aws:SourceIp` | `"IpAddress": {"aws:SourceIp": "203.0.113.0/24"}` | Restrict by caller IP. |
| `aws:RequestedRegion` | `"StringEquals": ...` | Restrict to specific regions. |
| `aws:PrincipalTag/Team` | Tag-based access control | Allow only if the caller's role has a matching tag. |

## 6. Policy evaluation logic

AWS evaluates every request. Memorize this order:

```text
1. Start: request is denied by default (implicit deny).
2. Any explicit Deny anywhere (identity, resource, SCP, boundary)
   matching the request?  ---- YES ----> DENIED (final).
3. Any explicit Allow matching?  ------- NO ----> still denied (implicit deny).
4. Also check permissions boundaries and SCPs:
   the action must be allowed by identity policy
   AND not blocked by boundary/SCP.
5. ALLOW.
```

| Situation | Result |
|---|---|
| Identity Allow + no Deny | **Allowed** |
| Identity Allow + explicit Deny on the same action | **Denied** (Deny wins) |
| No Allow anywhere | **Denied** (implicit deny) |
| Allow, but action outside permissions boundary | **Denied** |
| Allow, but SCP blocks the action in the org | **Denied** |
| Resource policy grants Allow to this principal | **Allowed** (can be enough even without identity Allow, for same-account/resource cases) |

> **Interview/real-world tip:** "Deny if not MFA" is implemented as an explicit Deny statement with a condition — explicit Deny beats any Allow.

Example — force MFA for dangerous actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnlessMFA",
      "Effect": "Deny",
      "Action": [
        "iam:Delete*",
        "iam:Create*",
        "iam:Attach*",
        "iam:Detach*"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

## 7. IAM roles in depth

A role is the most important IAM concept in practice. It has two policies:

| Piece | Question it answers |
|---|---|
| **Trust policy** (principal) | *Who may assume this role?* (account, user, service, external account) |
| **Permissions policy** | *What can they do once assumed?* |

```text
Caller ---- assume-role (STS) ----> Role
                                      |
                                      +--> Trust policy: is caller allowed in?
                                      +--> Permissions: what actions result?
                                      |
                                      v
                            Temporary credentials
                            (AccessKey, SecretKey, SessionToken)
                            expire (15 min – 12 hours)
```

### Common role use cases

| Use case | Trust policy principal | Notes |
|---|---|---|
| EC2 instance role | `ec2.amazonaws.com` | Delivered via instance profile; safest way for apps on EC2. |
| Lambda execution role | `lambda.amazonaws.com` | Permissions for logs, S3, DynamoDB, etc. |
| ECS task role | `ecs-tasks.amazonaws.com` | Per-task permissions, distinct from the execution role. |
| Cross-account access | The other account/role ARN | Requires a resource-based policy allowing the assume + matching identity policy. |
| Human access (SSO) | IAM Identity Center permission sets | Preferred over long-lived IAM users. |
| Automation / CI | GitHub OIDC, Jenkins, etc. | Federated assume-role; no static keys stored. |

### Example: EC2 instance role trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Example: minimal S3-read permissions policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::example-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::example-bucket"
    }
  ]
}
```

> Note: `ListBucket` targets the **bucket ARN** (no `/*`); `GetObject` targets **object ARNs** (with `/*`). Mixing these up is a classic bug.

### Credentials on EC2 (instance metadata)

Applications on EC2 retrieve temporary credentials from instance metadata. **Never bake access keys into AMIs or user data.**

```text
EC2 instance
    |
    v
Instance profile -> IAM role -> least-privilege policies -> AWS services
```

Require **IMDSv2** (session tokens; blocks simple SSRF credential theft):

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-endpoint enabled
```

### Cross-account access (pattern)

```text
Account A (trusted)                    Account B (owns the role)
Role: DataReader  <--- trust ---       Trust policy: allow Account A
Permissions: read S3 in Account B
```

1. In Account B, create role `DataReader` with a trust policy allowing Account A's root/role.
2. In Account A, the caller runs `sts:AssumeRole` on `arn:aws:iam::B:role/DataReader`.
3. Temporary credentials for the role are used for subsequent calls.

## 8. Federation and IAM Identity Center

Federation = use an external identity (company IdP, Active Directory, Google, etc.) to get AWS access without long-lived IAM users.

| Option | What it is | When |
|---|---|---|
| **IAM Identity Center** (successor to AWS SSO) | Central portal: one login → many AWS accounts/permission sets. Integrates with IdPs (Okta, Azure AD, Google, AD). | **Recommended for humans in orgs.** |
| SAML 2.0 federation | Direct SAML integration with your IdP to assume roles. | Custom integrations; often backs Identity Center. |
| Cognito | Public user sign-in for apps (social login, user pools). | Application end-users, not AWS admin access. |
| Web identity (OIDC) | Roles trusted by an OIDC provider (e.g. GitHub Actions). | CI/CD pipelines without static keys. |

```text
Human -> IdP (Okta/AD/Google)
            |
            v
    IAM Identity Center
            |
            v
    Permission set (policy) assumed into Target AWS account
```

**Permission sets** are templates (e.g. `AdministratorAccess`, `PowerUser`, `ReadOnly`) applied to users/groups per account.

## 9. MFA and access security

| Control | Why |
|---|---|
| **MFA on root + all IAM users** | Blocks password-only takeover. |
| **Virtual / hardware MFA** | Authenticator app (virtual), hardware key, or FIDO2 security key. |
| **No access keys for humans** | Humans should use Identity Center; keys are for machines and should rotate. |
| **Password policy** | Min length, complexity, expiry, no reuse. |
| **Condition: require MFA** | For sensitive actions (billing, IAM changes). |

### Setting a strong password policy (CLI sketch)

```bash
aws iam update-account-password-policy \
  --minimum-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --max-password-age 90 \
  --password-reuse-prevention 24
```

### Access keys hygiene

| Do | Don't |
|---|---|
| One key pair per application, per environment | Share keys between apps or people |
| Rotate regularly (90 days) or use roles | Commit keys to git |
| Store in Secrets Manager / Parameter Store | Put keys in user data or AMIs |
| Delete unused keys immediately | Leave departed employees' keys active |

List your own keys and last-used info:

```bash
aws iam list-access-keys --user-name my-user
aws iam get-access-key-last-used --access-key-id AKIA...
```

## 10. Tools: CLI, simulator, Access Analyzer

### Useful IAM CLI commands

```bash
# Who am I?
aws sts get-caller-identity

# List users, groups, roles, policies
aws iam list-users
aws iam list-groups-for-user --user-name alice
aws iam list-attached-role-policies --role-name app-role
aws iam list-policies --scope Local          # customer-managed only

# Inline policies
aws iam list-role-policies --role-name app-role
aws iam get-role-policy --role-name app-role --policy-name my-inline

# Attach / detach managed policy
aws iam attach-role-policy \
  --role-name app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam detach-role-policy \
  --role-name app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Assume a role and use it
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/DataReader \
  --role-session-name my-session
```

### IAM policy simulator

Web tool: **IAM → Policy simulator** (or `aws iam simulate-principal-policy`). Answers "would this identity be allowed to do X on resource Y?" without executing the action. Excellent for debugging before granting more permissions.

### IAM Access Analyzer

- Finds **external sharing** (roles trust outside accounts, public S3 buckets, unrestricted KMS keys, etc.).
- Can generate policies from **CloudTrail activity** (only actions the workload actually used).
- Validates policies against IAM best practices and AWS service limits.

Enable it in every region you use. Treat findings as a to-do list: every external share should be intentional.

### CloudTrail for auditing

Every IAM API call (`CreateUser`, `AttachRolePolicy`, `AssumeRole`, …) appears in CloudTrail. Alert on:

- Root account usage
- IAM policy/role/user changes
- Failed `AccessDenied` spikes (possible probing)
- New access keys created

## 11. Troubleshooting AccessDenied

| Symptom | Likely cause | Fix |
|---|---|---|
| `AccessDenied` on S3 `GetObject` | Missing object permission or wrong resource ARN (`/*` needed) | Check both bucket and object statements. |
| `AccessDenied` on `ListBucket` | Statement targets `bucket/*` instead of `bucket` | Split List vs Get resources. |
| Role cannot be assumed | Trust policy does not include caller, or caller lacks `sts:AssumeRole` | Fix trust policy + identity policy. |
| Works in console, fails in CLI | CLI uses different identity (keys vs SSO session) | `aws sts get-caller-identity` first. |
| Region errors | Policy has `Condition` on region | Align `aws:RequestedRegion`. |
| EC2 app denied | Instance has no instance profile / wrong role | Attach profile; wait ~minutes for credentials. |
| Works alone, fails with boundary/SCP | Action outside boundary or blocked by org SCP | Check boundary and SCPs. |
| "Still denied" after adding Allow | Explicit Deny exists elsewhere | Search all policies for `Deny` on that action. |

**Debugging order:**

```bash
aws sts get-caller-identity          # 1. Who am I?
aws iam simulate-principal-policy \  # 2. Would this identity pass?
  --policy-source-arn arn:aws:iam::123456789012:role/app-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::example-bucket/*
```

Then read the error code carefully: `AccessDenied`, `AccessDeniedException`, `UnauthorizedOperation`, `AccessDenied` from STS — each points at a different layer (identity policy vs resource policy vs service auth).

## 12. IAM best practices checklist

- [ ] Never use the root account for daily work; secure it with hardware MFA and no access keys.
- [ ] Enable MFA for root and every IAM user (prefer hardware/FIDO2 for root).
- [ ] Prefer **IAM Identity Center** + roles over long-lived IAM users with access keys.
- [ ] Grant least privilege: precise actions, precise resources; avoid `*:*` and `Resource: *`.
- [ ] Use **customer managed policies** (versioned, reusable) over inline and broad AWS managed policies.
- [ ] Use **permissions boundaries** for roles that developers create.
- [ ] Use **SCPs** in AWS Organizations as org guardrails.
- [ ] EC2/Lambda/ECS always use **roles**, never embedded keys.
- [ ] Require **IMDSv2** on all EC2 instances.
- [ ] Store secrets in **Secrets Manager** or **Parameter Store (SecureString)**, not in code or user data.
- [ ] Rotate access keys automatically; delete unused keys and stale roles.
- [ ] Enable **IAM Access Analyzer** in all regions; act on findings.
- [ ] Enable **CloudTrail** and alarm on IAM changes and root usage.
- [ ] Review permissions quarterly; remove unused users, roles, and policies.
- [ ] Tag IAM entities (`Team`, `Environment`) for ownership and automation.

> **Remember:** IAM is allow-by-default-deny. Start from zero permissions, add the minimum Allow needed, test with the Policy Simulator, and use explicit Deny + conditions for guardrails. Roles over users; temporary credentials over permanent keys.

[Back to IAM topic index](index.md) · [Back to AWS index](../index.md) · [Back to site home](../../index.md)
