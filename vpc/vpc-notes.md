# AWS VPC – IPv4 Notes

## VPC Basics
- VPC = Virtual Private Cloud
- Max 5 VPCs per region (soft limit)
- Max 5 CIDRs per VPC
  - Min size: /28 (16 IPs)
  - Max size: /16 (65,536 IPs)
- Only private IPv4 ranges allowed:
  - `10.0.0.0/8` → 10.0.0.0 – 10.255.255.255
  - `172.16.0.0/12` → 172.16.0.0 – 172.31.255.255
  - `192.168.0.0/16` → 192.168.0.0 – 192.168.255.255
- VPC CIDR should **not overlap** with other networks (e.g., corporate)

---

## Subnet – Reserved IPs
- AWS reserves **5 IPs** per subnet (first 4 + last 1)
- Example for `10.0.0.0/24`:

| Reserved IP   | Purpose                              |
|---------------|--------------------------------------|
| 10.0.0.0      | Network address                      |
| 10.0.0.1      | VPC router                           |
| 10.0.0.2      | Amazon-provided DNS                  |
| 10.0.0.3      | Reserved for future use              |
| 10.0.0.255    | Broadcast (not supported, reserved)  |

---

## Exam Tip – Subnet Sizing
If you need **29 IPs** for EC2 instances:
- /27 = 32 IPs → 32 - 5 = **27 usable** ❌ (not enough)
- /26 = 64 IPs → 64 - 5 = **59 usable** ✅

---

## Internet Gateway (IGW)
- Allows VPC resources to connect to the internet
- One IGW per VPC (1:1 relationship)
- Must also update route tables to use IGW
- IGW itself is horizontally scaled, redundant, and highly available

---

## Route Tables
- Each subnet must be associated with a route table
- A subnet can only be associated with **one** route table at a time
- Public subnet → route table has `0.0.0.0/0 → IGW`
- Private subnet → no route to IGW (or routes through NAT)

---

## NAT Gateway / NAT Instance
- Allows private subnet instances to access the internet (outbound only)
- **NAT Gateway** (managed by AWS):
  - Deployed in a **public subnet**
  - Uses an Elastic IP
  - 5 Gbps bandwidth, auto-scales up to 100 Gbps
  - No security group management needed
  - Pay per hour + data processed
  - AZ-specific — create one per AZ for high availability
- **NAT Instance** (self-managed, legacy):
  - EC2 instance in public subnet
  - Must disable source/destination check
  - Must manage security groups, patching, failover yourself

| Feature              | NAT Gateway       | NAT Instance        |
|----------------------|-------------------|---------------------|
| Managed              | Yes               | No                  |
| High availability    | Within AZ         | Manual (scripting)  |
| Bandwidth            | Up to 100 Gbps    | Depends on instance |
| Security groups      | Not applicable    | Must configure      |
| Cost                 | Per hour + data   | EC2 cost            |

---

## Network ACL (NACL) vs Security Groups

| Feature            | Security Group          | NACL                     |
|--------------------|-------------------------|--------------------------|
| Level              | Instance (ENI)          | Subnet                   |
| Rules              | Allow only              | Allow & Deny             |
| Stateful/Stateless | Stateful                | Stateless                |
| Rule evaluation    | All rules evaluated     | Rules evaluated in order |
| Default            | Denies all inbound, allows all outbound | Allows all in & out |
| Applies to         | Assigned to instance    | All instances in subnet  |

- NACL rules have a **number** (1–32766) — lower number = higher priority
- Default NACL allows all traffic; custom NACL denies all by default
- Ephemeral ports (1024–65535) must be allowed in NACL for return traffic

---

## VPC Peering
- Connect two VPCs privately using AWS network
- Works **cross-account** and **cross-region**
- Not transitive (A↔B, B↔C does **not** mean A↔C)
- CIDR ranges must **not overlap**
- Must update route tables in **both** VPCs

---

## VPC Endpoints
- Allow private access to AWS services without going through the internet
- **Gateway Endpoint** (free):
  - S3 and DynamoDB only
  - Configured in route table
- **Interface Endpoint** (powered by PrivateLink):
  - Most other AWS services
  - Uses ENI with private IP
  - Costs per hour + data processed
  - Supports security groups

---

## VPC Flow Logs
- Capture IP traffic information in your VPC
- Can be set at: **VPC** / **Subnet** / **ENI** level
- Logs go to: CloudWatch Logs, S3, or Kinesis Data Firehose
- Captures accepted, rejected, or all traffic
- Helps troubleshoot connectivity issues & monitor traffic

---

## Default VPC
- Created by AWS in every region by default
- Has a default subnet in each AZ (public, with public IP auto-assign)
- Has an IGW, default NACL, default security group, default route table
- New EC2 instances launch into default VPC if no VPC is specified

---

## IPv6 in VPC
- Every IPv6 address is public (no private range)
- IPv6 CIDR can be added alongside IPv4 (dual-stack)
- Egress-Only Internet Gateway — allows outbound IPv6 traffic from private subnets (like NAT for IPv6)
