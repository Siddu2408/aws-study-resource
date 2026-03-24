# AWS Bastion Host (Jump Server) - Reference Notes

## What is a Bastion Host?
- A publicly accessible EC2 instance in a public subnet
- Acts as a gateway to SSH into private subnet instances
- Only entry point to private resources — reduces attack surface

## Architecture
```
You → (SSH) → Bastion Host (Public Subnet) → (SSH) → Private Instance (Private Subnet)
```

---

## Prerequisites
- A VPC with public and private subnets
- Internet Gateway attached to VPC
- Key pairs for SSH access

---

## Step 1: Launch Bastion Host Instance
- EC2 → Launch Instance
- Name: Bastion-Host
- AMI: Amazon Linux 2023 / Ubuntu
- Instance type: t2.micro
- Key pair: Select or create (e.g., bastion-key.pem)
- Subnet: **Public subnet**
- Auto-assign public IP: **Enable**

## Step 2: Bastion Security Group (Bastion-SG)
### Inbound:
| Type | Port | Source            |
|------|------|-------------------|
| SSH  | 22   | Your IP (x.x.x.x/32) |

### Outbound:
| Type | Port | Destination              |
|------|------|--------------------------|
| SSH  | 22   | Private subnet CIDR      |

> Never use 0.0.0.0/0 for inbound SSH

## Step 3: Private Instance Security Group (Private-SG)
### Inbound:
| Type | Port | Source        |
|------|------|---------------|
| SSH  | 22   | Bastion-SG ID |

This allows SSH only from the Bastion Host.

## Step 4: Connect to Private Instance via Bastion

### Method 1: SSH Agent Forwarding (Recommended)
```bash
ssh-add -K private-instance-key.pem
ssh -A -i bastion-key.pem ec2-user@<bastion-public-ip>
# From bastion:
ssh ec2-user@<private-instance-private-ip>
```

### Method 2: ProxyJump (Single Command)
```bash
ssh -i private-instance-key.pem \
    -J ec2-user@<bastion-public-ip> \
    ec2-user@<private-instance-private-ip>
```

### Method 3: SSH Config File (~/.ssh/config)
```
Host bastion
    HostName <bastion-public-ip>
    User ec2-user
    IdentityFile ~/bastion-key.pem

Host private-instance
    HostName <private-instance-private-ip>
    User ec2-user
    IdentityFile ~/private-instance-key.pem
    ProxyJump bastion
```
Then: `ssh private-instance`

## Step 5: Harden the Bastion
```bash
# Disable root login
sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Disable password authentication
sudo sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

# Restart SSH
sudo systemctl restart sshd
```

---

## Key Points to Remember
- Bastion → Public subnet with public IP
- Private instances → Accept SSH only from Bastion SG
- Use SSH agent forwarding — never copy keys to bastion
- Restrict SSH to your IP only
- Use Elastic IP so bastion IP doesn't change on reboot
- Enable CloudWatch Logs for audit logging
- Keep bastion patched and minimal

## Bastion Host vs AWS Session Manager
| Feature            | Bastion Host      | Session Manager     |
|--------------------|-------------------|---------------------|
| Requires public IP | Yes               | No                  |
| Port 22 open       | Yes               | No                  |
| Key management     | Manual            | IAM-based           |
| Audit logging      | Manual setup      | Built-in (CloudTrail)|
| Cost               | EC2 cost          | Free (with SSM agent)|

> For production: Consider AWS Systems Manager Session Manager
> — no open ports, no key management, built-in logging.
