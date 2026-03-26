# AWS Site-to-Site VPN – Notes

## What is Site-to-Site VPN?
- A secure **IPsec encrypted** connection between your on-premises network and AWS VPC
- Traffic goes over the **public internet** but is encrypted
- Used to extend your corporate data center into AWS privately
- Quick to set up compared to Direct Connect

---

## Architecture
```
On-Premises Data Center                          AWS Cloud
┌──────────────────┐                    ┌──────────────────────┐
│                  │    IPsec Tunnel     │                      │
│  Corporate       │◄──────────────────►│  Virtual Private     │
│  Network         │    (Encrypted)      │  Gateway (VGW)       │
│  (10.0.0.0/16)   │                    │       │               │
│       │          │                    │       ▼               │
│  Customer        │                    │  VPC (172.16.0.0/16)  │
│  Gateway Device  │                    │                      │
└──────────────────┘                    └──────────────────────┘
```

---

## Key Components

| Component                  | Description                                              |
|----------------------------|----------------------------------------------------------|
| **Virtual Private Gateway (VGW)** | VPN endpoint on the AWS side, attached to VPC     |
| **Customer Gateway (CGW)**        | Reference to your on-premises VPN device          |
| **Customer Gateway Device**       | Physical/software VPN appliance in your data center |
| **VPN Connection**                | The actual IPsec tunnel between VGW and CGW       |

---

## Example Scenario
> **Goal**: Connect corporate office (10.0.0.0/16) to AWS VPC (172.16.0.0/16)
>
> Corporate office has a VPN router with public IP: 203.0.113.50

---

## Step-by-Step Configuration

### Step 1: Create Customer Gateway (CGW)
- VPC → Customer Gateways → **Create Customer Gateway**

| Setting       | Value                                  |
|---------------|----------------------------------------|
| Name          | corp-office-cgw                        |
| BGP ASN       | 65000 (or your ASN, use 65000 if none) |
| IP Address    | 203.0.113.50 (your router's public IP) |
| Routing       | Static or Dynamic (BGP)               |

### Step 2: Create Virtual Private Gateway (VGW)
- VPC → Virtual Private Gateways → **Create Virtual Private Gateway**

| Setting       | Value              |
|---------------|--------------------|
| Name          | my-vgw             |
| ASN           | Amazon default ASN  |

- After creation → **Actions → Attach to VPC** → Select your VPC (172.16.0.0/16)

### Step 3: Create VPN Connection
- VPC → Site-to-Site VPN Connections → **Create VPN Connection**

| Setting                  | Value                          |
|--------------------------|--------------------------------|
| Name                     | corp-to-aws-vpn                |
| Virtual Private Gateway  | my-vgw                         |
| Customer Gateway         | corp-office-cgw                |
| Routing Options          | Static                         |
| Static IP Prefixes       | 10.0.0.0/16 (on-prem CIDR)    |

> AWS creates **2 IPsec tunnels** for redundancy (across 2 AZs)

### Step 4: Download VPN Configuration
- Select VPN Connection → **Download Configuration**
- Choose your device vendor (e.g., Cisco, Palo Alto, pfSense, generic)
- Use this config to set up your on-premises VPN device

### Step 5: Configure On-Premises VPN Device
Using the downloaded config, set up on your router/firewall:
- IPsec tunnel parameters (pre-shared keys, encryption algorithms)
- Tunnel interfaces with AWS tunnel endpoint IPs
- Static routes or BGP peering
- Firewall rules to allow:

| Protocol | Port     | Purpose              |
|----------|----------|----------------------|
| UDP      | 500      | IKE (key exchange)   |
| IP       | 50 (ESP) | IPsec encrypted data |
| UDP      | 4500     | NAT-Traversal        |

### Step 6: Update VPC Route Table
- VPC → Route Tables → Select your VPC's route table
- Add route:

| Destination   | Target   |
|---------------|----------|
| 10.0.0.0/16   | vgw-xxxx |

> This tells AWS to send traffic destined for on-prem through the VPN

### Step 7: Enable Route Propagation (Optional – Recommended)
- Route Tables → Select route table → **Route Propagation** → Edit
- Enable propagation for the VGW
- Routes from on-prem are automatically added to the route table (useful with BGP)

### Step 8: Update Security Groups
- Allow inbound traffic from on-premises CIDR:

| Type       | Port  | Source         |
|------------|-------|----------------|
| All Traffic| All   | 10.0.0.0/16    |

Or restrict to specific ports as needed.

### Step 9: Verify Connection
- VPC → VPN Connections → Check tunnel status

| Tunnel   | Status |
|----------|--------|
| Tunnel 1 | **UP** |
| Tunnel 2 | **UP** |

```bash
# From on-premises, ping an EC2 private IP
ping 172.16.1.10

# From EC2, ping on-premises server
ping 10.0.1.50
```

---

## Static vs Dynamic (BGP) Routing

| Feature          | Static Routing              | Dynamic (BGP)                |
|------------------|-----------------------------|------------------------------|
| Route management | Manual on both sides        | Automatic via BGP            |
| Failover         | Manual                      | Automatic                    |
| Complexity       | Simple                      | Requires BGP support         |
| Best for         | Single office, simple setup | Multiple sites, complex nets |

> Use **BGP** if your device supports it — enables automatic failover between tunnels

---

## VPN with Transit Gateway (Scalable Option)
- Instead of attaching VGW to a single VPC, attach VPN to **Transit Gateway**
- Transit Gateway acts as a hub — connects multiple VPCs + VPN
```
On-Prem → VPN → Transit Gateway → VPC-A
                                 → VPC-B
                                 → VPC-C
```
- Better for multi-VPC architectures

---

## VPN CloudHub
- Connect **multiple on-premises sites** through a single VGW
- Sites can communicate with each other through AWS
```
Site-A ──► VGW ◄── Site-B
             ▲
             │
           Site-C
```
- Each site needs its own CGW and VPN connection
- Requires BGP with unique ASNs per site

---

## Site-to-Site VPN vs Direct Connect

| Feature          | Site-to-Site VPN         | Direct Connect            |
|------------------|--------------------------|---------------------------|
| Connection       | Over public internet     | Dedicated private line    |
| Encryption       | IPsec (encrypted)        | Not encrypted by default  |
| Setup time       | Minutes                  | Weeks to months           |
| Bandwidth        | Up to 1.25 Gbps/tunnel   | 1 Gbps / 10 Gbps / 100 Gbps |
| Cost             | Lower                    | Higher                    |
| Reliability      | Depends on internet      | More consistent           |
| Use case         | Quick, budget-friendly   | High throughput, low latency |

> You can use VPN as a **backup** for Direct Connect

---

## Key Points
- VPN creates **2 tunnels** for high availability
- VGW is attached to VPC, CGW represents your on-prem device
- Must update route tables on AWS side + configure on-prem device
- Encrypted using IPsec — secure over public internet
- Max throughput: **1.25 Gbps per tunnel**
- Use **BGP** for automatic failover between tunnels
- Enable **route propagation** to auto-populate routes from on-prem
- VPN connection charges: ~$0.05/hr + data transfer
