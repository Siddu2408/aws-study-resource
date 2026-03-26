# AWS VPC Peering – Configuration Notes

## What is VPC Peering?
- A networking connection between two VPCs using AWS private network
- Traffic never goes over the public internet
- Works **cross-account** and **cross-region**
- Behaves as if both VPCs are in the same network

---

## Architecture
```
VPC-A (10.0.0.0/16) ←— Peering Connection —→ VPC-B (172.16.0.0/16)
```

---

## Rules & Limitations
- CIDRs must **not overlap**
- **Not transitive**: A↔B and B↔C does NOT mean A↔C
- Only **one** peering connection between any two VPCs
- Must update route tables on **both** sides
- Security groups can reference peering VPC's SG (same region only)

---

## Step-by-Step Setup

### Step 1: Create Peering Connection (Requester VPC)
- VPC → Peering Connections → **Create Peering Connection**

| Setting                | Value                              |
|------------------------|------------------------------------|
| Name                   | vpc-a-to-vpc-b                     |
| Requester VPC          | VPC-A                              |
| Account                | My account / Another account       |
| Region                 | This region / Another region       |
| Accepter VPC           | VPC-B (or VPC ID if cross-account) |

### Step 2: Accept Peering Connection (Accepter VPC)
- VPC → Peering Connections → Select pending request → **Actions → Accept**
- If cross-account: log into the accepter account to accept

> Status changes from `pending-acceptance` → `active`

### Step 3: Update Route Tables – VPC-A Side
- VPC → Route Tables → Select VPC-A's route table(s)
- Edit Routes → Add:

| Destination        | Target              |
|--------------------|---------------------|
| 172.16.0.0/16      | pcx-xxxxxxxx        |

> Add VPC-B's CIDR pointing to the peering connection ID

### Step 4: Update Route Tables – VPC-B Side
- VPC → Route Tables → Select VPC-B's route table(s)
- Edit Routes → Add:

| Destination        | Target              |
|--------------------|---------------------|
| 10.0.0.0/16        | pcx-xxxxxxxx        |

> Add VPC-A's CIDR pointing to the same peering connection ID

### Step 5: Update Security Groups
- Allow traffic from the peered VPC's CIDR or reference its security group

Example — VPC-B instance SG inbound rule:

| Type  | Port | Source           |
|-------|------|------------------|
| SSH   | 22   | 10.0.0.0/16      |
| HTTPS | 443  | sg-xxxxx (VPC-A) |

> SG referencing by ID works only for **same-region** peering

### Step 6: Verify Connectivity
```bash
# From instance in VPC-A
ping <private-ip-of-vpc-b-instance>

# From instance in VPC-B
ping <private-ip-of-vpc-a-instance>
```

---

## Cross-Account Peering – Extra Steps
1. Requester creates peering using accepter's **Account ID + VPC ID**
2. Accepter logs into their account → accepts the request
3. Both sides update route tables and security groups

## Cross-Region Peering – Notes
- Supported but with some limitations:
  - Cannot reference SG by ID (use CIDR instead)
  - Inter-region data transfer charges apply
  - DNS resolution for private hostnames requires enabling in peering settings

---

## Enable DNS Resolution (Optional)
- Peering Connections → Select → **Actions → Edit DNS Settings**
- Enable: **Allow DNS resolution from VPC-A / VPC-B**
- Allows private DNS hostnames to resolve to private IPs across the peering

---

## Key Points
- Peering is **not transitive** — use Transit Gateway for hub-and-spoke
- CIDR **must not overlap**
- Route tables must be updated on **both** sides
- No single point of failure — peering uses AWS backbone
- No bandwidth bottleneck — traffic stays on AWS network
- Free for same-AZ traffic, charges apply for cross-AZ and cross-region
