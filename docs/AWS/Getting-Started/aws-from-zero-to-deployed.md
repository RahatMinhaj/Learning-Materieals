# AWS From Zero to Deployed — Beginner Guide

Start with an empty email inbox and end with a web application running on a public URL. This guide walks a complete beginner through the full path: create an AWS account, secure it, set billing alarms, launch a server, connect from your own PC, run an app, open ports safely, and avoid surprise bills.

> Console screenshots go stale quickly as AWS updates its UI. This guide uses **exact click-paths, what-you-will-see descriptions, and diagrams** instead — they stay accurate longer.

## What this guide will build

```text
Your laptop
   |
   |  1) SSH (port 22) with your private key
   v
EC2 instance (t2.micro, free tier)
   |
   |  2) runs Nginx or your app (port 80 / 8080)
   v
Your browser  <--- http://PUBLIC_IP  ---  anyone on the Internet
```

By the end you will have:

1. A secured AWS account with MFA and budget alerts
2. AWS CLI installed and configured on your PC
3. An EC2 server you can SSH into
4. A web application serving traffic on a public IP
5. A cleanup plan so your experiments do not cost money

## Contents

- [1. What you need before starting](#1-what-you-need-before-starting)
- [2. Cloud and AWS in plain English](#2-cloud-and-aws-in-plain-english)
- [3. Create your AWS account](#3-create-your-aws-account)
- [4. Lock down the root account](#4-lock-down-the-root-account)
- [5. Billing and budgets (do this on day one)](#5-billing-and-budgets-do-this-on-day-one)
- [6. Console tour: regions, services, CloudShell](#6-console-tour-regions-services-cloudshell)
- [7. Create your daily-use admin identity](#7-create-your-daily-use-admin-identity)
- [8. Install the AWS CLI on your PC](#8-install-the-aws-cli-on-your-pc)
- [9. Mental map of core services](#9-mental-map-of-core-services)
- [10. Launch your first EC2 instance](#10-launch-your-first-ec2-instance)
- [11. Connect from your PC with SSH](#11-connect-from-your-pc-with-ssh)
- [12. First ten minutes on the server](#12-first-ten-minutes-on-the-server)
- [13. Run your first web application](#13-run-your-first-web-application)
- [14. Open the app to the world (ports and security groups)](#14-open-the-app-to-the-world-ports-and-security-groups)
- [15. Port forwarding explained](#15-port-forwarding-explained)
- [16. Elastic IPs and stable addresses](#16-elastic-ips-and-stable-addresses)
- [17. Storage and snapshots](#17-storage-and-snapshots)
- [18. Monitoring and billing checks while you work](#18-monitoring-and-billing-checks-while-you-work)
- [19. Deploying updates to your app](#19-deploying-updates-to-your-app)
- [20. What you built: architecture recap](#20-what-you-built-architecture-recap)
- [21. Stop vs terminate: avoiding surprise bills](#21-stop-vs-terminate-avoiding-surprise-bills)
- [22. Cleanup checklist](#22-cleanup-checklist)
- [23. Troubleshooting: common errors](#23-troubleshooting-common-errors)
- [24. What is free and what is not](#24-what-is-free-and-what-is-not)
- [25. Where to go next](#25-where-to-go-next)
- [26. Final beginner checklist](#26-final-beginner-checklist)

---

## 1. What you need before starting

| Item | Notes |
|---|---|
| Email address | Preferably one dedicated to AWS; the root account will use it. |
| Phone number | AWS sends a verification code during signup. |
| Credit or debit card | Required for identity verification. You are not charged if you stay inside the free tier and stop unused resources. |
| A PC with internet | Windows 10/11, macOS, or Linux. |
| Terminal access | PowerShell/CMD on Windows, Terminal on macOS/Linux. |
| ~2 hours | Account setup + first deployment in one sitting. |

No prior cloud or Linux experience is required. Every command is provided.

---

## 2. Cloud and AWS in plain English

Traditionally, to run a web app you buy a server, put it in a data center, and pay for it whether you use it or not. **Cloud computing** means renting computers, storage, and networking over the internet, minute by minute, and paying only for what you use.

**Amazon Web Services (AWS)** is the largest cloud provider. You create an account, log into a web console, click "create server," and minutes later you have a machine on the internet.

| Traditional hosting | AWS cloud |
|---|---|
| Buy hardware weeks in advance | Server in ~2 minutes |
| Fixed monthly cost | Pay per second/hour of use |
| One data center | Regions worldwide |
| You fix broken disks | AWS manages the hardware; you manage your data and config |

> **Mental model:** AWS is a giant vending machine for infrastructure. You pick compute (EC2), storage (S3/EBS), network (VPC), and identity (IAM) — assembled by you, billed as you go.

---

## 3. Create your AWS account

1. Open **https://aws.amazon.com/free/** (or aws.amazon.com → **Create an AWS Account**).
2. Enter:
   - **Email address** → **Verify email** → enter the code AWS emails you.
   - **Password** — choose a strong unique password (this is the **root** account).
   - **Account name** — your name or company.
3. **Contact information**
   - Select **Personal** or **Business**.
   - Enter your full name, phone number, country, address.
4. **Payment information**
   - Card details for identity verification.
   - AWS may place a small temporary authorization hold; it is refunded.
5. **Identity verification**
   - Phone verification by SMS or voice call.
6. **Support plan**
   - Choose **Basic Support — Free**. You can upgrade later.

```text
Signup flow
    |
    v
Email + password  -->  Contact info  -->  Card  -->  Phone OTP
                                                      |
                                                      v
                                            Support plan (Basic/Free)
                                                      |
                                                      v
                                            Root console access
```

### What happens next

- You land in the **AWS Management Console** at `https://console.aws.amazon.com/`.
- Your account has a unique **Account ID** (12 digits) — find it under the top-right account menu. You will need it for ARNs: `arn:aws:iam::123456789012:role/...`.
- New accounts get **12 months of selected free-tier usage** (for example, 750 hours/month of a `t2.micro` or `t3.micro` instance in most regions). Limits and eligible services change — bookmark **https://aws.amazon.com/free/** and read current terms.

### First habits

- Note your root email + password in a **password manager** immediately.
- Write down (password manager) that the root user must never be used for daily work — you will create an IAM admin next.

---

## 4. Lock down the root account

The **root account** = your signup email + password. It can do everything, including deleting your whole account. Secure it before creating anything.

### Enable MFA (multi-factor-authentication) on root — today

1. Log in as root (use the account email → **Root user**).
2. Top-right account menu → **Security credentials** (or search "IAM" → **Users** → root is not listed; use the account dropdown).
3. Find **Multi-factor authentication (MFA)** → **Assign MFA device**.
4. Choose **Virtual MFA device** → next.
5. Install an authenticator app (Google Authenticator, Authy, Microsoft Authenticator).
6. Scan the QR code (or paste the secret key) → enter two consecutive codes → **Activate**.

```text
Login:  email + password  +  6-digit code from phone
              \______________/   \___________________/
                   something        something
                     you know         you have
```

| Factor | Example |
|---|---|
| Something you know | Password |
| Something you have | Phone authenticator / hardware key |

Without MFA, a leaked password is game over. With MFA, an attacker needs your phone too.

### Root hygiene rules

- Enable MFA (virtual is fine; hardware key is better).
- Do **not** create access keys for root (console only).
- Use root once a quarter for account-level tasks only (billing, closing account, some support cases).
- Everything else happens as an IAM user/role you create in section 7.

---

## 5. Billing and budgets (do this on day one)

Nothing in this guide should cost real money if you stay on free-tier shapes and **stop or terminate** unused resources. Budgets are your safety net for the day you forget.

### 5.1 Open the billing console

Console home → search **Billing** → **Billing Dashboard**. You see current month-to-date cost, forecasts, and free-tier usage.

### 5.2 Create a cost budget with alerts

Goal: get an email if spend crosses small thresholds (e.g. $1, $3, $5) so a forgotten resource never becomes a $200 surprise.

1. **Billing** → **Budgets** → **Create budget**.
2. Choose **Use a template** → **Zero-spend budget** *or* **Custom cost budget** (custom lets you pick amounts).
3. For a custom budget:
   - **Budget name:** `monthly-learning`
   - **Budgeting amount:** e.g. `5` (USD per month)
   - **Alert threshold 1:** 80% → your email
   - **Alert threshold 2 (optional):** 100% → your email
4. Confirm the email address (AWS may ask to verify SNS/email).
5. **Create budget.**

```text
Spend $0.00          $4.00 (80%)           $5.00 (100%)
   |--------------------|---------------------|
   OK                 Warning email        Critical email
                                          (consider stopping all
                                           non-free resources)
```

### 5.3 Create a free-tier usage alert

1. **Billing** → **Preferences** (or **Billing preferences**).
2. Enable **Receive Free Tier Usage Alerts**.
3. Confirm your email.

You get notified as free-tier allowances get consumed (for example EC2 hours).

### 5.4 Turn on Cost Explorer (usually on by default)

**Billing** → **Cost Explorer** → if prompted, **Enable Cost Explorer**. Use it later to see which service drove a charge (EC2 vs EBS vs data transfer).

### 5.5 Optional but recommended

| Setting | Where | Why |
|---|---|---|
| Billing alerts via SNS | Billing → Budgets → alert actions | Faster than waiting to check email |
| Region in budget scope | Budget settings | Global services (Route 53, IAM) still bill to your home region view |
| Close unused regions habit | Just don't create resources there | Most people only need 1–2 regions |

> Budget alerts are **not a hard cap**. AWS does not auto-shutdown at 100%. They notify you — you still act. (Organizations can use SCPs to hard-cap; solo accounts rely on you.)

### 5.6 Know your free-tier clock

The 12-month free tier starts at account creation. After 12 months, `t2.micro`-class instances bill hourly. Build the habit: **stop instances when idle** (section 21).

---

## 6. Console tour: regions, services, CloudShell

### Regions and Availability Zones

```text
Region: us-east-1 (N. Virginia)     <-- billing "home region" for most new accounts
   |
   +-- AZ: us-east-1a   (data center cluster)
   +-- AZ: us-east-1b
   +-- AZ: us-east-1c
```

| Concept | Meaning | Beginner rule |
|---|---|---|
| **Region** | Geographic area (`us-east-1`, `eu-west-1`, `ap-southeast-1`). | Pick one region and keep everything there while learning. Closer = lower latency. |
| **Availability Zone (AZ)** | Isolated set of data centers inside a region. | Default single-AZ is fine for learning; production uses 2+ AZs. |

**Region selector:** top-right of the console. Resources are regional — an EC2 instance in `us-east-1` is invisible from `eu-west-1`.

> Global services (IAM, Route 53, Organizations) appear the same from every region, but some resources (S3 buckets, budgets view) are regional or have a home region.

### Services search

Top navigation search bar: type `EC2`, `IAM`, `S3`... The console pins recent services. Almost everything lives here — no separate apps needed.

### AWS CloudShell (browser terminal)

Top bar has a **CloudShell** icon (looks like `>_`). Click it → a free-ish terminal in the browser with AWS CLI pre-authenticated as your current console user. Great for copy-pasting CLI commands without installing anything yet.

```text
Console (GUI)  <-->  CloudShell (CLI in browser)  <-->  AWS CLI on your laptop (CLI local)
```

---

## 7. Create your daily-use admin identity

Never work as root. Create an IAM user (or use IAM Identity Center — see the [IAM guide](../IAM/amazon-iam-complete-guide.md) for the org-grade approach). For your first week, the classic path is fine:

### Create an admin user

1. Search **IAM** → open **IAM dashboard**.
2. **Users** → **Create user**.
3. Username: `admin` (or `rahat-admin`).
4. **Permissions options**:
   - Select **Attach policies directly**.
   - Search and check **`AdministratorAccess`**.
   - (Later, replace with least-privilege policies.)
5. **Create user** without enabling console access yet.
6. Click the user → **Security credentials** tab → **Create access key**:
   - Use case: **Command Line Interface (CLI)**.
   - Confirm the recommendation.
   - **Download the `.csv`** (Access key ID + Secret access key). The secret is shown **only once**.
7. Also create a console password for this user (**Security credentials** → **Console sign-in** → **Assign password**) + **assign MFA** the same way as root.

### Recommended account map

```text
Root (email + MFA)
  |-- used rarely: billing changes, account close, some support
  |
  +-- IAM user: admin (AdministratorAccess + MFA + access keys for CLI)
        |-- daily console + CLI work while learning
        |
        +-- later: narrow policies / roles per project (see IAM guide)
```

ARNs you will see: `arn:aws:iam::123456789012:user/admin`

---

## 8. Install the AWS CLI on your PC

The CLI lets you create resources from the terminal and is essential for scripting.

### Install

| OS | Method |
|---|---|
| Windows | Download the MSI from https://aws.amazon.com/cli/ → run installer → reopen PowerShell. |
| macOS | `curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o AWSCLIV2.pkg` then `sudo installer -pkg AWSCLIV2.pkg -target /` (or `brew install awscli`). |
| Linux (Debian/Ubuntu) | `sudo apt update && sudo apt install -y awscli` (or use the official `.zip` bundle from AWS docs). |
| Linux (Amazon Linux) | `sudo yum install -y awscli` (or `dnf`). |

Verify:

```bash
aws --version
# aws-cli/2.x.x Python/3.x.x Linux/x86_64
```

### Configure with the admin access keys

```bash
aws configure
```

Enter:

```text
AWS Access Key ID:     AKIA...          (from the CSV)
AWS Secret Access Key: wJal...          (from the CSV; typing is hidden)
Default region name:   us-east-1        (or your chosen region)
Default output format: json
```

Confirm identity — this should print **your admin user**, not root:

```bash
aws sts get-caller-identity
```

```json
{
  "UserId": "AIDA...",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/admin"
}
```

> If the `Arn` says `root`, you configured root keys — stop and use IAM user keys instead.

Files created on your machine:

```text
~/.aws/credentials   (keys — protect this file)
~/.aws/config        (region, output)
```

---

## 9. Mental map of core services

You do not need all of AWS. For your first app you touch a handful:

| Service | What it does | Used in this guide? |
|---|---|---|
| **IAM** | Identities and permissions | Yes (setup) |
| **EC2** | Virtual servers | Yes (the app host) |
| **VPC** | Private network (auto-created defaults are fine) | Yes (defaults) |
| **Security Groups** | Virtual firewall around instances | Yes (SSH + HTTP) |
| **EBS** | Disk volumes for EC2 | Yes (root volume) |
| **CloudWatch** | Metrics, logs, alarms | Glance |
| **S3** | Object storage (files, websites, backups) | Not yet |
| **RDS** | Managed databases | Not yet |
| **Route 53** | DNS | Not yet |
| **ALB** | Load balancer | Not yet (later architecture) |

```text
Your future production picture (simplified):

  Users --> Route 53 (DNS) --> Load Balancer --> EC2 in private subnets --> RDS
                                                      |
                                               (you are here today:
                                                single EC2 + public IP)
```

---

## 10. Launch your first EC2 instance

Console search **EC2** → make sure the **region** (top right) is the one you chose. If you have never used EC2 in this region, you may see a prompt to enable **EC2-Classic**-era defaults — just continue; modern accounts create a default VPC automatically.

### Decision cheat-sheet

| Setting | First-time choice | Why |
|---|---|---|
| Name | `first-server` | Tag name; easy to spot in the list. |
| AMI | **Amazon Linux 2023** or **Ubuntu Server 24.04 LTS** | Both free-tier eligible; Ubuntu commands below assume Ubuntu — see note. |
| Instance type | **t3.micro** or **t2.micro** | Free-tier eligible (750 hrs/month for 12 months, check current terms). |
| Key pair | Create new, download `.pem` | Your SSH password file. **Do not lose it.** |
| Network | Default VPC + default subnet | Works out of the box with internet access. |
| Auto-assign public IP | **Enable** | So you can reach it from your laptop. |
| Storage | 8–30 GiB **gp3** | Free tier includes up to 30 GB EBS. |
| IAM instance profile | None for now | Optional; add later (see [IAM guide](../IAM/amazon-iam-complete-guide.md)). |
| Firewall (security group) | Allow **SSH (22)** from **My IP** only | Safer than 0.0.0.0/0. |
| User data | Leave empty for now | We configure the server manually first. |

### Step-by-step (console)

1. EC2 Dashboard → **Launch instance** (orange button).
2. **Name and tags** → Name: `first-server`.
3. **Application and OS Images (AMI)**
   - Quickly find: **Amazon Linux** or **Ubuntu**.
   - Note the **Free tier eligible** badge when present.
4. **Instance type** → `t3.micro` (or `t2.micro`).
5. **Key pair (login)**
   - **Create new key pair** → name `first-server-key` → type **RSA** → `.pem` format.
   - **Download** — the browser saves `first-server-key.pem`. Move it to a safe folder, e.g. `~/aws-keys/` (macOS/Linux) or `C:\aws-keys\` (Windows).
6. **Network settings** → **Edit**
   - VPC: default.
   - Subnet: leave **No preference** or pick one AZ.
   - **Auto-assign public IP: Enable**.
   - **Create security group**:
     - Name: `first-server-sg`
     - Rule 1: **SSH**, source **My IP** (console fills your current public IP — this changes if your ISP rotates addresses).
     - (We will add HTTP later, in section 14.)
7. **Configure storage** → keep root volume `gp3`, size 8 GiB is enough to start.
8. **Advanced details** → collapse/ignore for now.
9. **Launch instance** → success screen → click the instance ID.

### What just happened

```text
Click "Launch"
     |
     +--> EC2 boots your AMI on a hypervisor host in the chosen AZ
     +--> AWS attaches an EBS root volume
     +--> Places it in your subnet, assigns a private IP
     +--> With "auto-assign public IP": maps a public IPv4 to the instance
     +--> Security group acts as firewall (only port 22 from your IP so far)
     |
     v
Instance state: running    Status checks: 2/2 checks passed  (wait ~1–3 min)
```

Watch the **Instance state** column: `Pending` → `Running`. **Status checks** must show `2/2 checks passed` before SSH will work reliably.

### Key pair rules (read twice)

| Rule | Reason |
|---|---|
| `.pem` file is the only way in (without password) | Lose it = rebuild instance or use AWS recovery tricks; simplest is stop, detach volume, rebuild. |
| Never commit `.pem` to git | Add to `.gitignore`. |
| File permissions must be private | `chmod 400` on macOS/Linux (next section). |
| One key per instance or per team pattern | Rotation and accountability. |

---

## 11. Connect from your PC with SSH

**SSH** = encrypted remote terminal. You need five things:

```text
1. Instance is running + 2/2 status checks
2. A reachable public IPv4 (console: select instance -> Public IPv4 address)
3. Correct username for the AMI
4. Private key file (.pem) matching the key pair chosen at launch
5. Security group allows TCP 22 from YOUR current IP
```

### Default usernames by AMI

| AMI | Username |
|---|---|
| Amazon Linux 2023 / AL2 | `ec2-user` |
| Ubuntu | `ubuntu` |
| Debian | `admin` |
| RHEL | `ec2-user` |
| Fedora | `ec2-user` |
| SUSE | `ec2-user` |

### macOS / Linux terminal

```bash
# Put the key somewhere safe, then lock permissions (required)
mkdir -p ~/aws-keys
mv ~/Downloads/first-server-key.pem ~/aws-keys/
chmod 400 ~/aws-keys/first-server-key.pem

# Connect (replace with your Public IPv4 and AMI username)
ssh -i ~/aws-keys/first-server-key.pem ubuntu@3.94.12.34
```

First connection asks: `Are you sure you want to continue connecting (yes/no)?` → type `yes` (the host fingerprint is cached).

### Windows

**Option A — OpenSSH (built into Windows 10/11):**

```powershell
mkdir C:\aws-keys
move $env:USERPROFILE\Downloads\first-server-key.pem C:\aws-keys\
# OpenSSH requires strict ACLs on the key:
icacls C:\aws-keys\first-server-key.pem /inheritance:r /grant:r "$env:USERNAME:R"

ssh -i C:\aws-keys\first-server-key.pem ubuntu@3.94.12.34
```

**Option B — PuTTY:** convert `.pem` to `.ppk` with PuTTYgen, put the `.ppk` in Connection → SSH → Auth.

### Success looks like

```text
Ubuntu 24.04.1 LTS first-server tty1
ubuntu@ip-10-0-1-45:~$
```

`10.0.1.45` is the **private IP** — normal. Your public IP is the NAT mapping in front.

### If it fails

Jump to the [troubleshooting table](#23-troubleshooting-common-errors) — the big four are: wrong username, wrong key, security group source IP changed, instance not fully started.

---

## 12. First ten minutes on the server

Run these over SSH. We update packages and understand the environment. (Commands below assume **Ubuntu**; for Amazon Linux replace with `sudo dnf update -y`.)

```bash
# Who am I, where am I
whoami
hostname
curl -s http://169.254.169.254/latest/meta-data/ 2>/dev/null | head

# Update package index and upgrade
sudo apt update
sudo apt upgrade -y

# Basic tools
sudo apt install -y curl git jq ufw
```

### Create a non-root user (good habit)

```bash
# Replace 'deploy' with your preferred username
sudo adduser deploy
sudo usermod -aG sudo deploy

# Optional: allow this user to run sudo without password prompts
echo 'deploy ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/deploy
sudo chmod 440 /etc/sudoers.d/deploy
```

Later you can SSH as `deploy` the same way (`ssh -i key.pem deploy@PUBLIC_IP`). Many teams keep `ubuntu`/`ec2-user` for admin and a separate app user for runtime.

### Local firewall note (UFW on Ubuntu)

Ubuntu often ships with UFW inactive. For the simple public-web-server pattern, AWS **security groups** are the primary firewall. You *may* enable UFW defensively — but allow SSH first or you will lock yourself out:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw --force enable
sudo ufw status
```

On Amazon Linux, `firewalld` may exist; beginners can rely on security groups first and add host firewalls later.

---

## 13. Run your first web application

Two paths: (A) static site with Nginx — fastest; (B) a tiny Node.js app — shows app servers. Do A first, then B if you write JavaScript.

### Path A — Nginx serving a static page (Ubuntu)

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx

# Replace the default page
echo '<h1>Hello from my first AWS server</h1><p>Deployed from scratch.</p>' \
  | sudo tee /var/www/html/index.html

# Test locally on the server
curl -I http://localhost
# HTTP/1.1 200 OK
```

### Path B — Node.js app on port 8080

```bash
# Install Node.js 20.x (example using NodeSource; check current instructions)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

mkdir -p ~/myapp && cd ~/myapp
cat > server.js <<'EOF'
const http = require('http');
const port = process.env.PORT || 8080;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <h1>It works!</h1>
    <p>Host: ${require('os').hostname()}</p>
    <p>Path: ${req.url}</p>
    <p>Time: ${new Date().toISOString()}</p>
  `);
});

server.listen(port, '0.0.0.0', () => {
  console.log(\`Listening on 0.0.0.0:\${port}\`);
});
EOF

# Run in foreground for the test
node server.js
```

> **`0.0.0.0` vs `127.0.0.1`:** binding to `127.0.0.1` makes the app reachable only from the server itself (fine for local testing). Binding to `0.0.0.0` accepts connections from the network — required for public access. Binding to `0.0.0.0` without a security-group lock is dangerous — keep port 22 restricted and only open what you need.

### Keep the app running after you disconnect

Foreground `node server.js` dies when SSH closes. For learning, use `tmux` or `nohup`:

```bash
sudo apt install -y tmux
tmux new -s app
node server.js
# Detach: Ctrl+b then d    Reattach: tmux attach -t app
```

or:

```bash
nohup node server.js > ~/myapp/app.log 2>&1 &
```

Production apps use **systemd services** or containers — covered in deeper guides later.

### Test from the server

```bash
curl http://127.0.0.1:8080
```

Works? Great — now open the security group so outsiders can reach it.

---

## 14. Open the app to the world (ports and security groups)

### Port cheat-sheet

| Port | Service | Open to Internet? |
|---|---|---|
| 22 | SSH | **No** — only your IP or VPN |
| 80 | HTTP | Yes, if public website |
| 443 | HTTPS | Yes, for public website (set up certs later) |
| 8080 | Common app port | Prefer keeping private; put Nginx on 80 in front |
| 5432 / 3306 | Postgres / MySQL | **Never** open to 0.0.0.0/0 — app tier only |

### Add an HTTP rule

1. EC2 → **Instances** → select `first-server`.
2. **Security** tab → click the security group ID (`first-server-sg`).
3. **Edit inbound rules** → **Add rule**:
   - Type: **HTTP** (TCP 80)
   - Source: **Anywhere-IPv4** (`0.0.0.0/0`) — appropriate for a public demo site.
4. **Save rules.**

If you ran the Node app on 8080 instead of Nginx, add **Custom TCP 8080** from `0.0.0.0/0` for the demo (remember: 80 via Nginx is the cleaner pattern).

### Browse

```text
http://<Public IPv4 address>
```

```text
Browser  --HTTP:80-->  Public IPv4  --NAT-->  EC2 private IP :80
                           ^
                           |
              Security group must allow 80 from 0.0.0.0/0
              (SSH rule stays limited to your IP)
```

**If curl on the server works but the browser fails:** 95% security group, 4% app bound to 127.0.0.1, 1% UFW/NACL. Work the list.

### Security group mindset

| Rule style | When |
|---|---|
| SSH: My IP | Always, unless you have a bastion/VPN |
| HTTP/HTTPS: Anywhere | Public sites |
| DB port: only app security group | Production pattern (reference SG IDs, not raw IPs) |
| Everything: Anywhere | Almost never |

---

## 15. Port forwarding explained

You published the app directly. For private services (databases, admin dashboards, internal APIs) you instead **tunnel** through SSH. Port forwarding = "carry traffic for port X on the server to port Y on my laptop (or vice versa) inside the encrypted SSH session."

### Local port forwarding (most common)

Situation: app runs on the EC2 **private** interface (or a private-subnet box) on 8080; you want `http://localhost:8080` on your laptop to reach it — without opening 8080 to the internet.

```bash
# On YOUR laptop (not on the server):
ssh -i ~/aws-keys/first-server-key.pem \
    -L 8080:127.0.0.1:8080 \
    ubuntu@3.94.12.34
```

```text
Laptop browser: http://127.0.0.1:8080
        |
        v
Local port 8080  ===SSH tunnel===  EC2  -->  app on server 127.0.0.1:8080
```

Then open `http://127.0.0.1:8080` locally — traffic never crosses the public internet except the SSH session itself. **No security-group hole for 8080.**

### Forward to a database on a private host

```bash
ssh -i ~/aws-keys/key.pem -L 15432:10.0.3.25:5432 ubuntu@BASTION_PUBLIC_IP
psql -h 127.0.0.1 -p 15432 -U appuser appdb
```

```text
Your laptop :15432
      |
   SSH encrypted tunnel
      v
Bastion EC2  ------->  Private DB 10.0.3.25:5432
```

### Remote port forwarding (less common for beginners)

Expose your laptop's service to the server:

```bash
ssh -i key.pem -R 9000:127.0.0.1:3000 ubuntu@SERVER
# On server: curl http://127.0.0.1:9000  reaches your laptop's :3000
```

### Session Manager alternative (no SSH key, no open port 22)

Attach an instance profile with `AmazonSSMManagedInstanceCore`, then:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

Excellent production practice — details in the [EC2 guide](../EC2-Instance/amazon-ec2-complete-guide.md#7-ssh-and-session-manager).

| Method | Needs public SSH port? | Needs key on disk? | Skill curve |
|---|---|---|---|
| Direct SSH | Yes (restricted by IP) | Yes | Low |
| SSH + port forward | Yes (SSH only) | Yes | Low |
| SSM Session Manager | No | No | Medium setup, easiest daily use |

---

## 16. Elastic IPs and stable addresses

A normal **public IPv4** on EC2 changes when you **stop/start** the instance. That breaks DNS entries and saved bookmarks.

**Elastic IP** = a static public IPv4 you own in your account; reassign it to any instance in the region.

1. EC2 → **Elastic IPs** → **Allocate Elastic IP address**.
2. Select it → **Associate** → choose `first-server`.
3. Update your security group / bookmarks to the new fixed IP.

| Topic | Note |
|---|---|
| Idle cost | As of Feb 2024 AWS charges for **all** public IPv4 addresses (including attached ones) at a small hourly rate — check current pricing. Detach/delete Elastic IPs you do not use. |
| Stop/start | With Elastic IP associated, the address survives stop/start. |
| Better long term | Put a DNS name (Route 53) in front instead of sharing raw IPs. |

---

## 17. Storage and snapshots

Your instance has an **EBS** root volume (the disk). It persists when you **stop** the instance; it is deleted on **terminate** if "delete on termination" is checked (default for root).

```text
Instance lifecycle
   running <----start---- stopped
      |                    |
   reboot              EBS volume persists
      |                    |
      +---- terminate --> volume deleted (default)
                           (or retained if you unchecked delete-on-termination)
```

### Snapshot = point-in-time backup

EC2 → **Volumes** → select volume → **Actions** → **Create snapshot**. Snapshots live in S3 (managed by AWS), are incremental, and can create new volumes or AMIs.

**Beginner habit:** snapshot before big upgrades (kernel, major app migrations). Snapshots are not free forever — delete old ones you do not need.

Instance store (local NVMe on some types) is **ephemeral** — wiped on stop/terminate. Never put your only copy of data there.

---

## 18. Monitoring and billing checks while you work

### CloudWatch (built-in)

EC2 → **Monitoring** tab: CPU, network, disk, status checks — free basic metrics every 5 minutes.

```text
CloudWatch metrics (auto):  CPUUtilization, NetworkIn/Out, DiskRead/Write
Custom metrics/logs:        need CloudWatch agent (later topic)
```

Useful alarms later: high CPU, status check failed, free-tier usage limit.

### Cost check habit (weekly, 2 minutes)

1. **Billing Dashboard** → month-to-date.
2. **Cost Explorer** → group by **Service**.
3. **EC2 → Instances** → sort by state; stop anything `Running` you forgot.
4. **Elastic IPs** → release unattached.
5. **EBS → Snapshots** → delete experiments.
6. **S3** → buckets may hold old artifacts (later).

---

## 19. Deploying updates to your app

| Method | Command sketch | When |
|---|---|---|
| Edit over SSH + vim/nano | `nano ~/myapp/server.js` | Tiny fixes |
| `scp` copy a file | `scp -i key.pem app.js ubuntu@IP:~/myapp/` | Single file from laptop |
| `rsync` folder | `rsync -avz -e "ssh -i key.pem" ./app/ ubuntu@IP:~/myapp/` | Sync a project tree |
| Git clone/pull on server | `git clone ...` / `git pull` | Recommended habit |
| CI/CD pipeline | GitHub Actions → build → deploy | Production (later) |

```bash
# Example: push local changes via rsync, restart app
rsync -avz -e "ssh -i ~/aws-keys/first-server-key.pem" \
  ./myapp/ ubuntu@3.94.12.34:~/myapp/
ssh -i ~/aws-keys/first-server-key.pem ubuntu@3.94.12.34 \
  "tmux send-keys -t app C-c; tmux send-keys -t app 'node server.js' Enter"
```

Prefer **immutable deploys** later (AMI or containers) over hand-editing servers — but for week one, rsync + restart is normal.

---

## 20. What you built: architecture recap

```text
                    +---------------- AWS Account (12-digit ID) ----------------+
                    |  Root (MFA)          IAM admin user (MFA, CLI keys)      |
                    |  Budgets: $5 alert   Free-tier email alerts               |
                    |                                                           |
Region (e.g. us-east-1)                                                         |
  |                                                                            |
  +-- Default VPC 10.0.0.0/16                                                  |
  |     +-- Public subnet (e.g. 10.0.0.0/24)                                   |
  |           +-- EC2 t3.micro "first-server"                                  |
  |                 |  AMI: Ubuntu 24.04                                       |
  |                 |  EBS gp3 root volume                                     |
  |                 |  Public IPv4 (or Elastic IP)                             |
  |                 |  SG: 22 from YourIP, 80 from Anywhere                    |
  |                 |  App: Nginx :80  or  Node :8080                          |
  |                 |                                                          |
  |                 +-- SSH key pair on your laptop (~/.aws-keys/*.pem)        |
  |                                                                            |
  +-- (unused for now: S3, RDS, ALB, multi-AZ...)                             |
```

You now understand the entire beginner loop: **account → identity → budget → server → key → firewall → app → port → browser.**

---

## 21. Stop vs terminate: avoiding surprise bills

| Action | Compute billing | Disk (EBS) | Public IPv4 | Data on instance |
|---|---|---|---|---|
| **Running** | Yes (per second/hour) | Yes (small $/GB-month) | Yes | Live |
| **Stop** | No | Yes (still pays for disk) | Released (unless Elastic IP kept+associated differently — check console) | EBS persists; instance store lost |
| **Reboot** | Yes | Yes | Kept | Kept |
| **Terminate** | No | Root deleted if delete-on-termination | Released | **Gone** (unless AMI/snapshot) |

### Beginner policy

- Learning session over for the day → **Stop** the instance.
- Experiment finished for good → **Terminate** + delete snapshots/keys you no longer need.
- Never leave forgotten `Running` instances for weeks — that is how surprise bills happen. Your budget alert is the seatbelt; stopping is the brakes.

```text
End of day:  Stop instance  ->  pay only for EBS pennies
End of project: Terminate + cleanup  ->  pay ~zero
```

---

## 22. Cleanup checklist

When you are done experimenting (or starting fresh):

- [ ] EC2 → Instances → select all learning instances → **Terminate**.
- [ ] EC2 → Elastic IPs → release any unassociated addresses.
- [ ] EC2 → Volumes → delete orphaned volumes (state `available`, not attached).
- [ ] EC2 → Snapshots → delete old experiment snapshots.
- [ ] EC2 → Key Pairs → delete keys for terminated instances.
- [ ] EC2 → Security Groups → delete unused custom SGs (keep `default`).
- [ ] IAM → Users → delete access keys you rotated away; review unused users.
- [ ] Billing → Budgets → keep the budget (it protects future work).
- [ ] Local machine → keep `.pem` only if the instance still exists; otherwise shred it.

---

## 23. Troubleshooting common errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong username, wrong `.pem`, bad permissions, wrong key pair | Verify AMI username; `chmod 400`; confirm key pair name matches launch. |
| `Connection timed out` | Security group blocks your IP (ISP change), wrong public IP, instance stopped | Re-check SG source = current IP (`curl https://checkip.amazonaws.com`); instance Running; correct IP. |
| `Connection refused` | Nothing listening on that port; app bound to 127.0.0.1 | `ss -tulpn` on server; bind `0.0.0.0`; open SG port. |
| Browser spins; server curl works | SG missing HTTP/8080 rule; UFW; wrong public IP | Add inbound rule; `sudo ufw status`; re-copy Public IPv4 from console. |
| Browser fails; server curl to localhost fails | App not running / crashed | `systemctl status nginx` or check `app.log`; `journalctl -xe` for systemd. |
| `Host key verification failed` | Server IP was reused; stale known_hosts entry | `ssh-keygen -R 3.94.12.34` then reconnect. |
| AWS CLI `ExpiredToken` / `InvalidClientTokenId` | Wrong/expired keys; keys disabled | `aws configure` again; IAM → access keys; root keys not recommended. |
| `AccessDenied` from CLI | Identity lacks permission | You are the wrong principal (`sts get-caller-identity`); attach policy (see [IAM guide](../IAM/amazon-iam-complete-guide.md)). |
| Outbound internet fails on instance | No route/NAT (unusual in default VPC public subnet) | Default public subnet has IGW route; check instance has public IP; `curl https://checkip.amazonaws.com` from server. |
| Cannot SSH after enabling UFW | UFW blocked SSH | Use **EC2 Instance Connect** or serial console if available; or stop instance, detach volume, mount elsewhere to fix rules — or rebuild. Prefer allowing OpenSSH before enabling UFW. |

### Three diagnostic commands (memorize)

```bash
# On your laptop
aws sts get-caller-identity          # who does AWS think you are?
curl https://checkip.amazonaws.com   # your current public IP for SG rules

# On the server
ss -tulpn                            # what ports are listening?
```

---

## 24. What is free and what is not

**Always verify current terms at https://aws.amazon.com/free/** — AWS changes limits.

| Typically free-tier (12 months) | Not free / easy to forget |
|---|---|
| 750 hrs/month `t2.micro`/`t3.micro` (one instance, ~31 days) | A **second** running instance doubles hours |
| 30 GB EBS volume storage | Snapshots beyond free limits |
| 1 million S3 requests / 5 GB storage (limited) | Large S3 data + transfer |
| Data transfer out (limited GB/month) | Large downloads, video, backups |
| — | Elastic IP / all public IPv4 (small hourly) |
| — | Elastic Load Balancing, NAT Gateway (real $/hr) |
| — | RDS, Route 53 queries, non-free AMIs |

```text
Common beginner bill sources
  EC2 left running            ████████████
  EBS volumes/snapshots       ████
  Public IPv4 charges         ██
  NAT/ALB (if you "advance")  ██████████   <- learn these AFTER free tier habits
```

**Rule of thumb:** if you did not click "launch" on it this week, it should not be `Running`.

---

## 25. Where to go next

You have the beginner loop. Deepen each pillar:

| Topic | Guide |
|---|---|
| EC2 in depth (types, storage, bastions, production layout) | [EC2 Instance guide](../EC2-Instance/amazon-ec2-complete-guide.md) |
| IAM policies, roles, SSO, least privilege | [IAM guide](../IAM/amazon-iam-complete-guide.md) |
| Networking: VPC, subnets, NAT, NACLs | *(add a VPC guide next in this repo)* |
| Object storage for files/backups/CDN | *(add S3 guide next)* |
| Managed databases | *(add RDS guide next)* |
| HTTPS for your public IP (Let's Encrypt on Nginx) | *(add TLS guide next)* |

Suggested progression:

```text
This guide (account + first server + first app)
        |
        v
EC2 guide (production patterns, Session Manager, storage)
        |
        +--> VPC guide (private subnets, NAT) --> ALB/ASG
        |
        +--> IAM guide (roles for EC2/Lambda, boundaries)
        |
        +--> S3 + RDS guides
        |
        v
CI/CD deploy to AWS  +  HTTPS  +  backups/alerts
```

---

## 26. Final beginner checklist

- [ ] AWS account created; card verified; root password in a password manager.
- [ ] MFA enabled on root.
- [ ] Budget ($ small) + free-tier email alerts enabled; test that emails arrive.
- [ ] IAM admin user created with MFA + CLI access keys stored safely.
- [ ] AWS CLI installed; `aws sts get-caller-identity` shows the IAM user.
- [ ] Region chosen and consistent.
- [ ] EC2 launched: free-tier type, key pair downloaded (`chmod 400`), SG = SSH from My IP.
- [ ] SSH works from laptop; packages updated.
- [ ] Web app runs (Nginx or Node); bound to `0.0.0.0`.
- [ ] SG allows HTTP 80 (or 8080) as needed; page loads in browser via public IP.
- [ ] Understood port forwarding for private services.
- [ ] Instance **stopped** at end of day; terminate + cleanup when done.
- [ ] Billing dashboard checked at least once after first day of experiments.

> **The whole AWS journey in one sentence:** prove who you are (IAM), put a machine or service where users can reach it (network + compute), open only the necessary doors (security groups), watch the money (budgets), and clean up like someone else is paying — because future-you is.

[Back to Getting Started index](index.md) · [Back to AWS index](../index.md) · [Back to site home](../../index.md)
