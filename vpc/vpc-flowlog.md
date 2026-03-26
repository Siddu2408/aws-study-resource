# AWS VPC Flow Logs – Notes

## What are VPC Flow Logs?
- Capture information about **IP traffic** going to and from network interfaces in your VPC
- Helps monitor, troubleshoot, and analyze network connectivity issues
- Logs are not real-time — there is a delay of a few minutes

---

## Flow Log Levels
Can be created at three levels:

| Level   | What it captures                        |
|---------|-----------------------------------------|
| VPC     | All traffic in the entire VPC           |
| Subnet  | All traffic in a specific subnet        |
| ENI     | Traffic on a specific network interface |

---

## What Gets Logged?
Each flow log record contains:

| Field            | Description                          |
|------------------|--------------------------------------|
| version          | Flow log version                     |
| account-id       | AWS account ID                       |
| interface-id     | ENI ID                               |
| srcaddr          | Source IP                            |
| dstaddr          | Destination IP                       |
| srcport          | Source port                          |
| dstport          | Destination port                     |
| protocol         | IANA protocol number (6=TCP, 17=UDP) |
| packets          | Number of packets                    |
| bytes            | Number of bytes                      |
| start            | Start time of capture window         |
| end              | End time of capture window           |
| action           | **ACCEPT** or **REJECT**             |
| log-status       | OK, NODATA, or SKIPDATA              |

---

## Flow Log Destinations

| Destination              | Use Case                              |
|--------------------------|---------------------------------------|
| CloudWatch Logs          | Real-time monitoring, metric filters  |
| S3 Bucket                | Long-term storage, Athena queries     |
| Kinesis Data Firehose    | Streaming to third-party tools        |

---

## Step-by-Step Configuration

### Option A: Send to CloudWatch Logs

#### Step 1: Create IAM Role for Flow Logs
- IAM → Roles → Create Role
- Trusted entity: **VPC Flow Logs** (Service: `vpc-flow-logs.amazonaws.com`)
- Attach policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Step 2: Create CloudWatch Log Group
- CloudWatch → Log Groups → **Create log group**
- Name: `vpc-flow-logs`
- Set retention period (e.g., 30 days)

#### Step 3: Create Flow Log
- VPC → Your VPC → **Flow Logs** tab → **Create flow log**

| Setting              | Value                        |
|----------------------|------------------------------|
| Name                 | my-vpc-flow-log              |
| Filter               | All / Accept / Reject        |
| Max aggregation      | 1 minute or 10 minutes       |
| Destination          | Send to CloudWatch Logs      |
| Log group            | vpc-flow-logs                |
| IAM Role             | Role created in Step 1       |

---

### Option B: Send to S3 Bucket

#### Step 1: Create S3 Bucket
- S3 → Create bucket → Name: `my-vpc-flow-logs-bucket`
- Bucket policy is auto-updated by AWS when flow log is created

#### Step 2: Create Flow Log
- VPC → Your VPC → **Flow Logs** tab → **Create flow log**

| Setting              | Value                            |
|----------------------|----------------------------------|
| Name                 | my-vpc-flow-log-s3               |
| Filter               | All / Accept / Reject            |
| Max aggregation      | 1 minute or 10 minutes           |
| Destination          | Send to S3 bucket                |
| S3 bucket ARN        | arn:aws:s3:::my-vpc-flow-logs-bucket |

> No IAM role needed — AWS manages permissions via bucket policy

---

### Option C: Send to Kinesis Data Firehose
- Select **Send to Kinesis Data Firehose** as destination
- Choose delivery stream (must be created beforehand)
- Useful for streaming to Splunk, Datadog, or OpenSearch

---

## Querying Flow Logs with Athena (S3)
```sql
-- Find rejected traffic
SELECT srcaddr, dstaddr, dstport, action
FROM vpc_flow_logs
WHERE action = 'REJECT'
ORDER BY dstport;
```
- Use AWS-provided Athena table template for flow logs
- Great for analyzing large volumes of log data

---

## Flow Log Filter Examples

| Filter   | What it captures                     |
|----------|--------------------------------------|
| All      | Both accepted and rejected traffic   |
| Accept   | Only accepted (allowed) traffic      |
| Reject   | Only rejected (denied) traffic       |

---

## What Flow Logs Do NOT Capture
- DNS traffic to Amazon-provided DNS (Route 53 Resolver)
- DHCP traffic
- Traffic to instance metadata (`169.254.169.254`)
- Traffic to Amazon Time Sync Service (`169.254.169.123`)
- Traffic between an ENI and a Network Load Balancer

---

## Troubleshooting with Flow Logs

| Scenario                                  | Look for                          |
|-------------------------------------------|-----------------------------------|
| Instance can't reach internet             | REJECT on outbound to 0.0.0.0/0  |
| SSH not working                           | REJECT on port 22                 |
| SG allows but NACL blocks                 | Inbound ACCEPT + Outbound REJECT  |
| NACL allows but SG blocks                 | Inbound REJECT                    |

> If inbound shows **ACCEPT** but outbound shows **REJECT** → NACL issue (stateless)
> If inbound shows **REJECT** → Security Group or NACL issue

---

## Key Points
- Flow logs **cannot be modified** after creation — delete and recreate
- Flow logs do not affect network throughput or latency
- Can be created at VPC, subnet, or ENI level
- Use **Reject** filter to quickly find blocked traffic
- S3 + Athena is the most cost-effective way to analyze large logs
- Flow logs capture metadata only — **not packet contents**
