# AWS Direct Connect (DX) – Notes

## What is Direct Connect?
- A **dedicated private network connection** from your on-premises data center to AWS
- Traffic does **not** go over the public internet
- Provides **consistent low latency** and **high throughput**
- Physical connection established at an **AWS Direct Connect location** (partner facility)

---

## Architecture
```
On-Premises Data Center          DX Location              AWS Cloud
┌──────────────────┐      ┌──────────────────┐     ┌──────────────────┐
│                  │      │                  │     │                  │
│  Corporate       │──────│  Your Router /   │─────│  AWS Direct      │
│  Network         │ Dark │  Partner Router  │ DX  │  Connect Router  │
│                  │Fiber │                  │Link │       │          │
└──────────────────┘      └──────────────────┘     │       ▼          │
                                                   │  Virtual Private │
                                                   │  Gateway (VGW)   │
                                                   │       │          │
                                                   │       ▼          │
                                                   │  VPC             │
                                                   └──────────────────┘
```

---

## Key Components

| Component                     | Description                                              |
|-------------------------------|----------------------------------------------------------|
| **Direct Connect Location**   | Physical facility where AWS and your network meet        |
| **Connection**                | Physical link (1 Gbps, 10 Gbps, or 100 Gbps)            |
| **Virtual Interface (VIF)**   | Logical connection over the physical link                |
| **Virtual Private Gateway**   | VPN endpoint on VPC side (for Private VIF)               |
| **Direct Connect Gateway**    | Connects DX to multiple VPCs across regions              |
| **Letter of Authorization (LOA-CFA)** | Document to authorize cross-connect at DX location |

---

## Connection Speeds

| Type              | Speeds                          | How to get                     |
|-------------------|---------------------------------|--------------------------------|
| Dedicated         | 1 Gbps, 10 Gbps, 100 Gbps     | Request directly from AWS      |
| Hosted            | 50 Mbps – 10 Gbps              | Through AWS Direct Connect Partner |

> **Dedicated** = physical port reserved for you
> **Hosted** = shared infrastructure via partner, faster to provision

---

## Virtual Interfaces (VIFs)

| VIF Type          | Purpose                                    | Connects to                    |
|-------------------|--------------------------------------------|--------------------------------|
| **Private VIF**   | Access VPC resources (EC2, RDS, etc.)      | Virtual Private Gateway / DX Gateway |
| **Public VIF**    | Access AWS public services (S3, DynamoDB)  | AWS public endpoints           |
| **Transit VIF**   | Access multiple VPCs via Transit Gateway   | Transit Gateway                |

---

## Step-by-Step Configuration

### Example Scenario
> **Goal**: Connect corporate data center (10.0.0.0/16) to AWS VPC (172.16.0.0/16) using Direct Connect

### Step 1: Request a Direct Connect Connection
- AWS Console → Direct Connect → **Create Connection**

| Setting          | Value                                |
|------------------|--------------------------------------|
| Name             | corp-dx-connection                   |
| Location         | Nearest DX location to your DC       |
| Port speed       | 1 Gbps                              |
| Service provider | Your partner (or AWS for dedicated)  |

> AWS provides a **LOA-CFA** document — give this to your data center / colocation provider to set up the physical cross-connect

### Step 2: Wait for Physical Connection
- Takes **days to weeks** for dedicated connections
- Hosted connections through partners can be faster
- Connection status: `ordering` → `pending` → `available`

### Step 3: Create Virtual Private Gateway (VGW)
- VPC → Virtual Private Gateways → **Create**

| Setting | Value    |
|---------|----------|
| Name    | my-vgw   |

- **Attach to VPC** (172.16.0.0/16)

### Step 4: Create Private Virtual Interface (Private VIF)
- Direct Connect → Virtual Interfaces → **Create Private Virtual Interface**

| Setting              | Value                              |
|----------------------|------------------------------------|
| Name                 | corp-private-vif                   |
| Connection           | corp-dx-connection                 |
| Virtual interface owner | My AWS account                  |
| Virtual Private Gateway | my-vgw                          |
| VLAN                 | 100 (must match on-prem config)    |
| BGP ASN              | 65000 (your on-prem ASN)          |
| BGP Auth Key         | Optional (recommended)             |
| Your router peer IP  | 169.254.x.x/30                    |
| Amazon router peer IP| 169.254.x.y/30                    |
| Prefixes to advertise| 10.0.0.0/16                        |

> BGP is **required** for Direct Connect — no static routing option

### Step 5: Configure On-Premises Router
Set up on your router:
- VLAN tagging (802.1Q) matching the VLAN ID
- BGP peering with Amazon's peer IP
- Advertise your on-prem CIDR (10.0.0.0/16)

```
# Example BGP config (Cisco-style)
router bgp 65000
  neighbor 169.254.x.y remote-as 7224
  network 10.0.0.0 mask 255.255.0.0
```

### Step 6: Update VPC Route Table
- Enable **route propagation** for the VGW
- Routes from on-prem (10.0.0.0/16) auto-populate via BGP

Or manually add:

| Destination   | Target   |
|---------------|----------|
| 10.0.0.0/16   | vgw-xxxx |

### Step 7: Update Security Groups
Allow traffic from on-prem:

| Type        | Port | Source        |
|-------------|------|---------------|
| All Traffic | All  | 10.0.0.0/16   |

### Step 8: Verify
- Direct Connect → Connections → Status: **available**
- Virtual Interfaces → BGP Status: **up**

```bash
# From on-prem
ping 172.16.1.10  # EC2 private IP

# From EC2
ping 10.0.1.50    # On-prem server
```

---

## Direct Connect Gateway

### What is it?
- Connects a Direct Connect connection to **multiple VPCs across different regions**
- Without it: one DX connection → one VGW → one VPC in one region
- With it: one DX connection → DX Gateway → multiple VGWs in multiple regions

### Architecture
```
On-Prem → DX → DX Gateway → VGW (us-east-1 VPC)
                            → VGW (eu-west-1 VPC)
                            → VGW (ap-south-1 VPC)
```

### Setup
1. Direct Connect → Direct Connect Gateways → **Create**
2. Associate VGWs from different regions
3. Create Private VIF pointing to DX Gateway (instead of VGW)

> VPC CIDRs must **not overlap** when using DX Gateway

---

## Direct Connect + VPN (Encrypted DX)
- Direct Connect is **not encrypted** by default
- For encryption: run a **Site-to-Site VPN over Direct Connect**

```
On-Prem → DX (private link) → VPN (IPsec encryption) → VPC
```

- Provides both **private dedicated connection** + **encryption**
- Use Public VIF to establish VPN tunnel endpoints

---

## High Availability Architectures

### Single DX (No Redundancy) — NOT recommended
```
On-Prem → DX → VPC
```

### Two DX Connections at Same Location
```
On-Prem → DX-1 → VPC
        → DX-2 → VPC (backup)
```

### Two DX Connections at Different Locations (Maximum Resiliency)
```
On-Prem → DX Location A → DX-1 → VPC
        → DX Location B → DX-2 → VPC (backup)
```
> AWS recommended for **critical production** workloads

### DX + VPN Backup (Cost-Effective HA)
```
On-Prem → DX (primary) ──────→ VPC
        → VPN (backup, over internet) → VPC
```
> VPN kicks in if DX fails — cheaper than two DX connections

---

## Direct Connect vs Site-to-Site VPN

| Feature            | Direct Connect              | Site-to-Site VPN           |
|--------------------|-----------------------------|----------------------------|
| Connection         | Dedicated private line      | Over public internet       |
| Encryption         | Not encrypted by default    | IPsec encrypted            |
| Setup time         | Weeks to months             | Minutes                    |
| Bandwidth          | 50 Mbps – 100 Gbps         | Up to 1.25 Gbps/tunnel     |
| Latency            | Consistent, low             | Variable (internet-based)  |
| Cost               | Higher (port + data)        | Lower                      |
| Redundancy         | Need 2 connections          | 2 tunnels built-in         |
| Use case           | High throughput, hybrid     | Quick setup, backup for DX |

---

## Pricing
- **Port hours**: charged per hour based on port speed
  - 1 Gbps: ~$0.30/hr
  - 10 Gbps: ~$2.25/hr
- **Data transfer out**: ~$0.02/GB (cheaper than internet transfer)
- **Data transfer in**: Free
- DX Gateway: No additional charge

---

## Key Points
- Direct Connect provides **dedicated private** connection — not over internet
- Setup takes **weeks to months** (physical infrastructure)
- **BGP is mandatory** — no static routing
- Not encrypted by default — use **VPN over DX** for encryption
- Use **DX Gateway** to connect to multiple VPCs across regions
- **Private VIF** → VPC access, **Public VIF** → AWS public services, **Transit VIF** → Transit Gateway
- For HA: use **2 DX connections at different locations** or **DX + VPN backup**
- Data transfer over DX is **cheaper** than over the internet
- Supports **jumbo frames** (MTU 9001) for better throughput
- Use **Link Aggregation Group (LAG)** to bundle multiple DX connections into one logical connection
