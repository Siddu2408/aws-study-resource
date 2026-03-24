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
