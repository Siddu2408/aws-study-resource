# AWS NAT Instance Setup - Reference Notes

## What is a NAT Instance?
- An EC2 instance that allows private subnet instances to access the internet
- Placed in a public subnet, forwards traffic from private subnet to the internet
- Alternative to AWS NAT Gateway (managed service)

---

## Prerequisites
- A VPC with at least 1 public subnet and 1 private subnet
- An Internet Gateway attached to the VPC
- A key pair for SSH access

---

## Step 1: Find NAT AMI
- Go to EC2 → Launch Instance
- Search "amzn-ami-vpc-nat" under Community AMIs
- AMI name example: `amzn-ami-vpc-nat-2018.03.0.20230322.0-x86_64-ebs`
- Or use Amazon Linux 2023 and configure NAT manually

## Step 2: Launch NAT Instance
- Instance type: t2.micro / t3.micro
- Subnet: Public subnet
- Auto-assign public IP: Enable
- Key pair: Select existing

## Step 3: Configure Security Group
### Inbound Rules:
| Type  | Port | Source              |
|-------|------|---------------------|
| HTTP  | 80   | Private subnet CIDR |
| HTTPS | 443  | Private subnet CIDR |
| SSH   | 22   | Your IP             |
| ICMP  | All  | Private subnet CIDR |

### Outbound Rules:
- All traffic → 0.0.0.0/0

## Step 4: Disable Source/Destination Check (CRITICAL)
- EC2 → Select NAT Instance
- Actions → Networking → Change source/destination check
- Stop (disable) → Save

## Step 5: Update Private Subnet Route Table
- VPC → Route Tables → Select private subnet route table
- Add route:
  - Destination: 0.0.0.0/0
  - Target: NAT Instance ID

## Step 6: Test from Private Instance
```bash
curl http://google.com
ping 8.8.8.8
```

---

## Key Points to Remember
- NAT instance → Public subnet with public IP
- Source/dest check → MUST be disabled
- Private route table → 0.0.0.0/0 points to NAT instance
- No built-in HA — instance down = no internet for private subnet
- For production → Use NAT Gateway instead (managed, HA, auto-scaling)

## NAT Instance vs NAT Gateway
| Feature         | NAT Instance        | NAT Gateway          |
|-----------------|---------------------|----------------------|
| Managed         | No (self-managed)   | Yes (AWS managed)    |
| HA              | No (single AZ)      | Yes (per AZ)         |
| Cost            | EC2 instance cost   | Hourly + data charge |
| Bandwidth       | Depends on instance | Up to 100 Gbps       |
| Src/Dest check  | Must disable        | Not needed           |


# AWS NAT Instance – High Availability Notes

## The Problem
- NAT Instance = single EC2 instance
- If it dies or AZ goes down → **all private subnet instances lose internet**
- Unlike NAT Gateway, AWS does not manage failover for you

---

## The Solution: ASG + Multi-AZ + Resilient User-Data

### Architecture
```
                        Auto Scaling Group (Multi-AZ)
                    ┌──────────────────────────────────┐
                    │                                  │
              ┌─────┴─────┐                  ┌─────────┴───┐
              │   AZ-a    │                  │    AZ-b     │
              │           │                  │             │
              │  NAT      │   (failover)     │  NAT        │
              │  Instance │ ──────────────►  │  Instance   │
              │  (active) │   ASG replaces   │  (new)      │
              └─────┬─────┘                  └──────┬──────┘
                    │                               │
                    ▼                               ▼
            Private Subnet                  Private Subnet
            (routes via NAT)                (routes via NAT)
```

---

## ASG Configuration

| Setting          | Value                              |
|------------------|------------------------------------|
| Min              | 1                                  |
| Max              | 1                                  |
| Desired          | 1                                  |
| AZs              | Multiple (e.g., us-east-1a, 1b)   |
| Health check     | EC2 (or custom)                    |
| AMI              | Amazon Linux with NAT enabled      |
| Instance type    | t3.micro or as needed              |
| IAM Role         | Needs EC2 + route table permissions |

---

## IAM Role for NAT Instance
The instance needs permission to modify itself and update route tables:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:ModifyInstanceAttribute",
        "ec2:ReplaceRoute",
        "ec2:DescribeRouteTables"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Resilient User-Data Script
This runs automatically when ASG launches a new instance:

```bash
#!/bin/bash

# 1. Enable IP forwarding (makes instance act as a router)
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 2. Get instance ID
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

# 3. Disable source/destination check (required for NAT)
aws ec2 modify-instance-attribute \
    --instance-id $INSTANCE_ID \
    --no-source-dest-check

# 4. Update private subnet route table to point to this instance
aws ec2 replace-route \
    --route-table-id rtb-xxxxxxxx \
    --destination-cidr-block 0.0.0.0/0 \
    --instance-id $INSTANCE_ID
```

### What Each Step Does

| Step                        | Why it's needed                                         |
|-----------------------------|---------------------------------------------------------|
| Enable IP forwarding        | Allows instance to forward packets (act as NAT)        |
| iptables MASQUERADE         | Rewrites source IP of outgoing packets                 |
| Disable source/dest check   | EC2 drops traffic not meant for it by default          |
| Update route table          | Points private subnet's `0.0.0.0/0` to new instance   |

---

## Failover Flow

```
NAT Instance dies in AZ-a
         │
         ▼
ASG detects unhealthy instance
         │
         ▼
Terminates failed instance
         │
         ▼
Launches new instance in AZ-b
         │
         ▼
User-data script runs automatically:
    ├── Enables IP forwarding
    ├── Configures iptables for NAT
    ├── Disables source/dest check
    └── Updates route table → points to new instance
         │
         ▼
Private subnets regain internet access ✅
```

---

## Step-by-Step Setup

### Step 1: Create Launch Template
- EC2 → Launch Templates → **Create**

| Setting            | Value                                |
|--------------------|--------------------------------------|
| Name               | nat-instance-template                |
| AMI                | Amazon Linux 2023                    |
| Instance type      | t3.micro                             |
| Key pair           | Your key pair                        |
| Subnet             | Don't include (ASG handles this)     |
| Security group     | NAT-SG (see below)                   |
| IAM instance profile | NAT-Instance-Role                  |
| User data          | Paste the script above               |

### Step 2: NAT Instance Security Group (NAT-SG)
Inbound:

| Type         | Port    | Source                  |
|--------------|---------|-------------------------|
| All Traffic  | All     | Private subnet CIDR     |

Outbound:

| Type         | Port    | Destination             |
|--------------|---------|-------------------------|
| All Traffic  | All     | 0.0.0.0/0               |

### Step 3: Create Auto Scaling Group
- EC2 → Auto Scaling Groups → **Create**

| Setting              | Value                              |
|----------------------|------------------------------------|
| Name                 | nat-instance-asg                   |
| Launch template      | nat-instance-template              |
| VPC                  | Your VPC                           |
| Subnets              | **Public subnets** in multiple AZs |
| Min / Max / Desired  | 1 / 1 / 1                         |
| Health check type    | EC2                                |
| Health check grace   | 120 seconds                        |

### Step 4: Create Route in Private Subnet Route Table
- Initial route (will be auto-updated by user-data on failover):

| Destination   | Target                    |
|---------------|---------------------------|
| 0.0.0.0/0     | NAT instance ID           |

---

## Downtime During Failover
- ASG takes **2–5 minutes** to detect failure and launch replacement
- During this time, private instances have **no internet access**
- Not zero-downtime — acceptable for non-critical workloads

---

## NAT Instance HA vs NAT Gateway

| Feature              | NAT Instance + ASG       | NAT Gateway              |
|----------------------|--------------------------|--------------------------|
| Managed              | No (self-managed)        | Yes (AWS managed)        |
| Failover             | 2–5 min (ASG)            | Automatic, within AZ     |
| Multi-AZ HA          | ASG handles              | Create one per AZ        |
| Cost                 | EC2 cost (cheaper)       | Per hour + data          |
| Complexity           | High (scripts, IAM)      | Low (click and go)       |
| Use case             | Budget-conscious, small  | Production workloads     |

> **Bottom line**: Use NAT Instance + ASG only if cost is a concern. For production, use **NAT Gateway**.

---

## Key Points
- ASG with **Min=1, Max=1** ensures exactly one NAT Instance is always running
- User-data script must be **resilient** — fully configures NAT on every boot
- Instance needs **IAM permissions** to modify itself and update route tables
- NAT Instance must be in a **public subnet**
- There is **downtime during failover** (2–5 min) — not zero-downtime
- This is a common **exam topic** — understand why each user-data step is needed
