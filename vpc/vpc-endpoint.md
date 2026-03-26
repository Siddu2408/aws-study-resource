# AWS VPC Endpoints – Notes

## What is a VPC Endpoint?
- Allows you to **privately connect** your VPC to AWS services without using the internet
- Traffic stays within the **AWS network** — never leaves Amazon's backbone
- No need for IGW, NAT Gateway, or VPN to access AWS services
- Powered by **AWS PrivateLink**

---

## Why Use VPC Endpoints?
- Instances in **private subnets** need to access S3, DynamoDB, SQS, etc.
- Without endpoint: Private Instance → NAT Gateway → IGW → Internet → S3 (costly, slower)
- With endpoint: Private Instance → VPC Endpoint → S3 (free/cheaper, faster, private)

---

## Two Types of VPC Endpoints

| Feature              | Gateway Endpoint                  | Interface Endpoint                    |
|----------------------|-----------------------------------|---------------------------------------|
| Supports             | **S3 and DynamoDB** only          | Most other AWS services (100+)        |
| How it works         | Entry in route table              | ENI with private IP in your subnet    |
| Cost                 | **Free**                          | ~$0.01/hr + data processing           |
| Security             | VPC Endpoint Policy               | Security Groups + Endpoint Policy     |
| Access from on-prem  | Not directly                      | Yes (via VPN / Direct Connect)        |
| DNS                  | Uses public DNS                   | Private DNS (optional)                |

---

## Gateway Endpoint – Deep Dive

### How It Works
```
EC2 (Private Subnet) → Route Table → Gateway Endpoint → S3 / DynamoDB
```
- AWS adds a **prefix list** route in your route table
- Traffic to S3/DynamoDB is routed through the endpoint automatically
- No ENI, no IP address — just a route table entry

### Example: Private EC2 Accessing S3

#### Step 1: Create Gateway Endpoint
- VPC → Endpoints → **Create Endpoint**

| Setting            | Value                              |
|--------------------|------------------------------------|
| Name               | s3-gateway-endpoint                |
| Service category   | AWS services                       |
| Service            | com.amazonaws.**region**.s3 (Gateway type) |
| VPC                | my-vpc                             |
| Route tables       | Select **private subnet** route table(s) |

#### Step 2: Verify Route Table
After creation, AWS auto-adds a route:

| Destination                  | Target       |
|------------------------------|--------------|
| pl-xxxxxxxx (S3 prefix list) | vpce-xxxxxxx |

> `pl-xxxxxxxx` is the managed prefix list containing S3 IP ranges

#### Step 3: (Optional) Add Endpoint Policy
Default policy allows full access. Restrict to a specific bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ]
    }
  ]
}
```

#### Step 4: (Optional) S3 Bucket Policy – Restrict to Endpoint Only
Force S3 access only through the VPC endpoint:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-xxxxxxxx"
        }
      }
    }
  ]
}
```

#### Step 5: Test from Private EC2
```bash
# SSH into private instance (via bastion or Session Manager)
aws s3 ls s3://my-company-bucket
```
- Works without NAT Gateway or internet access
- Traffic goes through the gateway endpoint privately

---

## Interface Endpoint – Deep Dive

### How It Works
```
EC2 (Private Subnet) → ENI (Interface Endpoint) → AWS Service (e.g., SQS, SNS, SSM)
```
- Creates an **ENI** (Elastic Network Interface) with a private IP in your subnet
- Your application connects to the service using the endpoint's private DNS or endpoint-specific DNS
- Supports **security groups** for access control

### Example: Private EC2 Accessing SSM (Systems Manager)

> Needed for Session Manager to work on private instances without NAT

#### Step 1: Create Interface Endpoints
You need **3 endpoints** for SSM:

- VPC → Endpoints → **Create Endpoint** (repeat for each)

**Endpoint 1:**

| Setting            | Value                                        |
|--------------------|----------------------------------------------|
| Name               | ssm-endpoint                                 |
| Service            | com.amazonaws.**region**.ssm                 |
| VPC                | my-vpc                                       |
| Subnets            | Select private subnet(s)                     |
| Security group     | endpoint-sg (allow HTTPS 443 from VPC CIDR)  |
| Private DNS        | Enable                                       |

**Endpoint 2:**

| Setting            | Value                                        |
|--------------------|----------------------------------------------|
| Name               | ssmmessages-endpoint                         |
| Service            | com.amazonaws.**region**.ssmmessages         |

**Endpoint 3:**

| Setting            | Value                                        |
|--------------------|----------------------------------------------|
| Name               | ec2messages-endpoint                         |
| Service            | com.amazonaws.**region**.ec2messages         |

> Use same VPC, subnets, security group, and enable private DNS for all three

#### Step 2: Create Security Group for Endpoints (endpoint-sg)
Inbound:

| Type  | Port | Source           |
|-------|------|------------------|
| HTTPS | 443  | 172.16.0.0/16 (VPC CIDR) |

> Interface endpoints communicate over HTTPS (port 443)

#### Step 3: Ensure EC2 Has SSM IAM Role
- Attach `AmazonSSMManagedInstanceCore` policy to the EC2 instance role
- Without this, Session Manager won't work even with endpoints

#### Step 4: Test
- Systems Manager → Session Manager → Start Session → Select your private instance
- No NAT Gateway, no public IP, no SSH — fully private connection

---

## Another Example: Interface Endpoint for SQS

#### Step 1: Create Endpoint

| Setting            | Value                                |
|--------------------|--------------------------------------|
| Name               | sqs-endpoint                         |
| Service            | com.amazonaws.**region**.sqs         |
| VPC                | my-vpc                               |
| Subnets            | Private subnet(s)                    |
| Security group     | endpoint-sg (HTTPS 443 from VPC)     |
| Private DNS        | Enable                               |

#### Step 2: Test from Private EC2
```bash
# With private DNS enabled, standard AWS CLI works
aws sqs list-queues --region us-east-1

# Without private DNS, use endpoint URL
aws sqs list-queues --endpoint-url https://vpce-xxxxx.sqs.us-east-1.vpce.amazonaws.com
```

---

## Private DNS – How It Works
- When **enabled**, the service's default public DNS resolves to the endpoint's **private IP**
- Example: `sqs.us-east-1.amazonaws.com` → resolves to `172.16.1.55` (ENI private IP)
- No code changes needed — existing AWS SDK/CLI calls work automatically
- Requires **DNS hostnames** and **DNS support** enabled on the VPC

---

## Endpoint Policy vs Security Group vs IAM

| Layer            | Controls                                    | Applies to          |
|------------------|---------------------------------------------|---------------------|
| Endpoint Policy  | What actions/resources via this endpoint     | Gateway & Interface |
| Security Group   | Who can reach the endpoint ENI               | Interface only      |
| IAM Policy       | What the user/role is allowed to do          | Both                |

> All three are evaluated — request must pass **all** of them

---

## Common Interface Endpoint Services

| Service                | Endpoint Service Name                    |
|------------------------|------------------------------------------|
| SSM (Session Manager)  | com.amazonaws.region.ssm                 |
| SSM Messages           | com.amazonaws.region.ssmmessages         |
| EC2 Messages           | com.amazonaws.region.ec2messages         |
| SQS                    | com.amazonaws.region.sqs                 |
| SNS                    | com.amazonaws.region.sns                 |
| CloudWatch Logs        | com.amazonaws.region.logs                |
| CloudWatch Monitoring  | com.amazonaws.region.monitoring          |
| Secrets Manager        | com.amazonaws.region.secretsmanager      |
| KMS                    | com.amazonaws.region.kms                 |
| ECR (Docker)           | com.amazonaws.region.ecr.dkr            |
| API Gateway            | com.amazonaws.region.execute-api         |

---

## Key Points
- **Gateway Endpoint** = free, route table based, S3 & DynamoDB only
- **Interface Endpoint** = paid, ENI based, most AWS services
- Enable **private DNS** so existing code works without changes
- Interface endpoints need **security groups** allowing HTTPS (443)
- Use **endpoint policies** to restrict what can be accessed through the endpoint
- Gateway endpoints cannot be accessed from on-prem; interface endpoints **can** (via VPN/DX)
- VPC must have **DNS hostnames** and **DNS support** enabled for private DNS to work
- Prefer **gateway endpoint** for S3/DynamoDB (free and simpler)
