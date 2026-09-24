# Amazon EC2 — Practical Guide

A GitHub-friendly guide to EC2 instances, networking, security, SSH, storage, IAM, deployment, and troubleshooting.

## Contents

- [1. What is EC2?](#1-what-is-ec2)
- [2. Core architecture](#2-core-architecture)
- [3. Instance types and pricing](#3-instance-types-and-pricing)
- [4. Launching an instance](#4-launching-an-instance)
- [5. VPC, subnets, and IP addresses](#5-vpc-subnets-and-ip-addresses)
- [6. Security groups and network ACLs](#6-security-groups-and-network-acls)
- [7. SSH and Session Manager](#7-ssh-and-session-manager)
- [8. Ports and port forwarding](#8-ports-and-port-forwarding)
- [9. IAM roles](#9-iam-roles)
- [10. Storage](#10-storage)
- [11. Web-server demo](#11-web-server-demo)
- [12. Production design](#12-production-design)
- [13. Monitoring, backup, and troubleshooting](#13-monitoring-backup-and-troubleshooting)
- [14. Final production checklist](#14-final-production-checklist)

## 1. What is EC2?

Amazon Elastic Compute Cloud (EC2) provides virtual servers in AWS. An instance is a server on which you choose the operating system, CPU, memory, storage, network, security rules, and permissions.

> **Mental model:** EC2 is the server; the VPC is the network; security groups are the firewall; IAM roles provide AWS permissions; EBS is the disk; CloudWatch monitors the server.

## 2. Core architecture

```text
User
  |
  v
Route 53 / Load Balancer
  |
  v
Public subnet
  |
  v
Private EC2 application instances
  |
  v
Private database
```

| Component | Purpose |
|---|---|
| Region | Geographic AWS location, such as `us-east-1`. |
| Availability Zone | Isolated data center inside a Region. |
| AMI | Template containing an operating system and software. |
| VPC | Private network containing subnets and routes. |
| Subnet | IP range inside a VPC. |
| Security group | Stateful firewall attached to an instance network interface. |
| IAM role | Temporary AWS permissions for applications on the instance. |

## 3. Instance types and pricing

An instance type determines vCPUs, memory, network performance, EBS bandwidth, and sometimes GPUs or local disks.

| Family | Use | Examples |
|---|---|---|
| General purpose | Balanced CPU and memory for web servers and APIs. | `t3`, `t4g`, `m7i` |
| Burstable | Low average CPU with occasional bursts. | `t3.micro`, `t4g.small` |
| Compute optimized | CPU-heavy APIs, encoding, and batch jobs. | `c7i`, `c7g` |
| Memory optimized | In-memory databases, caches, and analytics. | `r7i`, `x2idn` |
| Storage optimized | High local storage throughput and I/O. | `i4i`, `d3` |
| Accelerated | GPU, machine learning, and rendering. | `g`, `p`, `inf`, `trn` |

Pricing options:

- **On-Demand:** Flexible, no long-term commitment.
- **Reserved Instances / Savings Plans:** Discounted for predictable usage.
- **Spot:** Discounted but interruptible; use for fault-tolerant work.

## 4. Launching an instance

Configure the AMI, instance type, key pair, VPC, subnet, public-IP behavior, security group, EBS volumes, IAM role, user data, and tags.

Example:

```text
AMI: Ubuntu Server LTS
Type: t3.micro
Subnet: public-subnet-a
Public IP: enabled for a temporary demo
Security group: SSH from my IP, HTTP from the Internet
Root volume: 20 GiB gp3
IAM role: application-role
```

### User data

User data runs during first boot and is useful for repeatable setup. Do not put long-term secrets in it.

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl enable --now nginx
echo "Hello from EC2" > /var/www/html/index.html
```

### Lifecycle

| Action | Result |
|---|---|
| Start | Starts a stopped instance. A normal public IP may change. |
| Stop | Stops compute; EBS data normally remains. |
| Reboot | Restarts the operating system. |
| Hibernate | Saves memory to EBS, then stops. |
| Terminate | Deletes the instance; the root volume may also be deleted. |

## 5. VPC, subnets, and IP addresses

```text
VPC: 10.0.0.0/16
  Public subnet:   10.0.1.0/24  -- route to Internet Gateway
  Private subnet:  10.0.2.0/24  -- no direct inbound Internet route
  Database subnet: 10.0.3.0/24
```

A public subnet has a route to an Internet Gateway. A private instance can make outbound connections through a NAT Gateway, but inbound Internet traffic is not directly routed to it.

| Address | Meaning |
|---|---|
| Private IPv4 | Used inside the VPC, for example `10.0.2.25`. |
| Public IPv4 | Internet address that may change after stop/start. |
| Elastic IP | Static public IPv4 address; use sparingly. |
| IPv6 | Optional address family requiring IPv6 routes and firewall rules. |

## 6. Security groups and network ACLs

Security groups are stateful: if a connection is allowed in, its response is automatically allowed. They allow traffic; they do not provide deny rules.

| Service | Protocol / port | Recommended source |
|---|---|---|
| SSH | TCP 22 | Your IP, VPN, or bastion security group |
| HTTP | TCP 80 | Internet only for a public website |
| HTTPS | TCP 443 | Internet only for a public website |
| Application | TCP 8080 | Load balancer security group |
| PostgreSQL | TCP 5432 | Application security group |

> Never expose SSH or a database to `0.0.0.0/0` unless there is a specific, controlled reason. Prefer security-group references instead of fixed private IP addresses.

Network ACLs apply at the subnet level and are stateless. Return traffic must be allowed explicitly, including ephemeral ports.

## 7. SSH and Session Manager

For SSH you need a running instance, a reachable address, the correct AMI username, private key, port 22 access, routes, and network ACL rules.

| AMI | Typical username |
|---|---|
| Amazon Linux | `ec2-user` |
| Ubuntu | `ubuntu` |
| Debian | `admin` or `debian` |
| CentOS | `centos` |

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ubuntu@PUBLIC_IP
```

Through a bastion:

```bash
ssh -i private-key.pem \
  -J ubuntu@BASTION_PUBLIC_IP \
  ubuntu@PRIVATE_EC2_IP
```

### Session Manager

Systems Manager Session Manager provides shell access without a public IP, inbound SSH, or a bastion. The instance needs SSM Agent, an IAM role containing `AmazonSSMManagedInstanceCore`, and connectivity to Systems Manager endpoints.

```bash
aws ssm start-session --target i-0123456789abcdef0
```

## 8. Ports and port forwarding

Opening a port requires the application, security group, operating-system firewall, routes, and NACL to agree.

```bash
sudo ss -tulpn
sudo ss -tulpn | grep :8080
sudo systemctl status nginx
```

An application bound to `127.0.0.1` is local-only. To accept network traffic it normally must bind to `0.0.0.0` and be secured appropriately.

### Local forwarding to a private database

```bash
ssh -i bastion.pem \
  -L 15432:10.0.3.25:5432 \
  ec2-user@BASTION_PUBLIC_IP

psql -h 127.0.0.1 -p 15432 -U appuser appdb
```

```text
Your computer:15432
        |
   encrypted SSH tunnel
        v
Bastion host ----> Private database:5432
```

### Session Manager forwarding

```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

## 9. IAM roles

An instance profile provides an IAM role whose temporary credentials are available to applications through instance metadata. This is safer than storing AWS access keys on the server.

```text
EC2 instance
    |
    v
Instance profile -> IAM role -> Least-privilege policies -> AWS services
```

Example policy allowing reads from one S3 bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::example-bucket/*"
  }]
}
```

Use Secrets Manager or Systems Manager Parameter Store for secrets. Require IMDSv2:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-endpoint enabled
```

## 10. Storage

EBS is persistent block storage. Common types are `gp3` for general use, `io2` for high I/O, and `st1`/`sc1` for HDD workloads.

Attach and mount a volume:

```bash
lsblk
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
df -h
sudo blkid /dev/nvme1n1
```

Use the UUID in `/etc/fstab` for a persistent mount:

```text
UUID=your-volume-uuid /data xfs defaults,nofail 0 2
```

EBS snapshots are point-in-time backups and can create new volumes or AMIs. Instance store is fast but ephemeral; do not use it as the only copy of important data.

## 11. Web-server demo

1. Launch Ubuntu with a public IP.
2. Allow TCP 22 from your IP and TCP 80 from the Internet.
3. Connect and install Nginx:

```bash
ssh -i my-key.pem ubuntu@PUBLIC_IP
sudo apt update
sudo apt install -y nginx
echo '<h1>Hello from EC2</h1>' | sudo tee /var/www/html/index.html
sudo systemctl status nginx
curl http://localhost
```

Open `http://PUBLIC_IP` in a browser. If local curl works but the browser does not, check the security group, route, NACL, OS firewall, and public IP.

## 12. Production design

```text
Users
  |
  v
Route 53
  |
  v
Application Load Balancer (public subnets)
  |
  +-------------------+
  |                   |
  v                   v
EC2 A (private)    EC2 B (private)
  |                   |
  +---------+---------+
            v
       Database (private)
```

Use an Auto Scaling group to maintain instances across Availability Zones. Use a launch template to define the AMI, instance type, security groups, IAM role, storage, and user data.

Typical rules: the ALB allows 80/443 from users; EC2 allows the application port from the ALB security group; the database allows its port from the EC2 security group.

## 13. Monitoring, backup, and troubleshooting

CloudWatch provides CPU, network, disk I/O, and status checks. Install the CloudWatch Agent for memory, disk-space, process, and application-log metrics.

Use automated EBS snapshots, AMIs, application-level database backups, encryption, retention rules, and restoration tests.

| Symptom | Check first |
|---|---|
| SSH timeout | Instance state, address, route, security group, NACL, and local network. |
| SSH permission denied | Username, private key, key permissions, and selected key pair. |
| Connection refused | Whether the service is running and listening on the expected port. |
| Works on localhost only | Application binding; change from `127.0.0.1` to an appropriate network binding. |
| Public IP changed | Use DNS, a load balancer, or an Elastic IP where necessary. |

## 14. Final production checklist

- [ ] Use private subnets for applications and databases where possible.
- [ ] Restrict SSH or use Session Manager.
- [ ] Use IAM roles instead of access keys.
- [ ] Require IMDSv2.
- [ ] Use least-privilege security groups and IAM policies.
- [ ] Encrypt EBS volumes and backups.
- [ ] Keep the operating system and packages updated.
- [ ] Use HTTPS and a load balancer for public applications.
- [ ] Use multiple Availability Zones and Auto Scaling for critical services.
- [ ] Configure CloudWatch alarms and centralized logs.
- [ ] Automate backups and test recovery.
- [ ] Tag resources and review cost regularly.

[Back to EC2 topic index](README.md) · [Back to repository home](../../README.md)
