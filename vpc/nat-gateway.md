# AWS NAT Gateway – Configuration Notes

## What is NAT Gateway?
- Allows instances in **private subnets** to access the internet (outbound only)
- Managed by AWS — no patching, high availability within AZ
- Deployed in a **public subnet** with an **Elastic IP**

---

## Architecture
```
Private Instance → NAT Gateway (Public Subnet) → Internet Gateway → Internet
```

---

## Step-by-Step Setup

### Step 1: Ensure VPC Has an Internet Gateway
- VPC → Internet Gateways → Verify IGW is attached
- If not: Create IGW → Attach to VPC

### Step 2: Allocate an Elastic IP
- VPC → Elastic IPs → **Allocate Elastic IP address**
- This will be assigned to the NAT Gateway

### Step 3: Create NAT Gateway
- VPC → NAT Gateways → **Create NAT Gateway**

| Setting         | Value                          |
|-----------------|--------------------------------|
| Name            | my-nat-gw                      |
| Subnet          | **Public subnet**              |
| Connectivity    | Public                         |
| Elastic IP      | Select the allocated EIP       |

> Create one NAT Gateway **per AZ** for high availability

### Step 4: Update Private Subnet Route Table
- VPC → Route Tables → Select **private subnet's route table**
- Edit Routes → Add:

| Destination   | Target       |
|---------------|--------------|
| 0.0.0.0/0     | nat-gw-id    |

> Do **not** add this route to the public subnet route table (it should point to IGW)

### Step 5: Verify
```bash
# SSH into private instance (via bastion or Session Manager)
ping google.com
curl https://ifconfig.me
```
- If you get a response, NAT Gateway is working
- The public IP shown should be the NAT Gateway's Elastic IP

---

## Multi-AZ High Availability Setup
```
AZ-a: Public Subnet → NAT-GW-a (EIP-a)
       Private Subnet → Route Table → 0.0.0.0/0 → NAT-GW-a

AZ-b: Public Subnet → NAT-GW-b (EIP-b)
       Private Subnet → Route Table → 0.0.0.0/0 → NAT-GW-b
```
- Each AZ's private subnet routes through its **own** NAT Gateway
- If one AZ goes down, the other AZ is unaffected

---

## Pricing
- **Per hour**: ~$0.045/hr (varies by region)
- **Data processed**: ~$0.045/GB
- Elastic IP is free while associated with NAT Gateway

---

## Key Points
- NAT Gateway must be in a **public subnet**
- Private subnet route table points `0.0.0.0/0` → NAT Gateway
- NAT Gateway is **AZ-specific** — not cross-AZ resilient by default
- No security group on NAT Gateway (use NACLs on subnet)
- Supports up to 100 Gbps burst
- Cannot be used by instances in the same subnet as the NAT Gateway
