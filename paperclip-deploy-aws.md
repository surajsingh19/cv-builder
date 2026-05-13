# Paperclip Deployment Guide — AWS Cloud Services

> **Target:** Amazon Web Services — EC2 + RDS + S3 + Secrets Manager + ALB + Route 53
> **Mode:** `authenticated + public`
> **Effort:** Higher setup effort — enterprise-grade reliability, scalability, and managed services

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [AWS Services Used and Why](#2-aws-services-used-and-why)
3. [Prerequisites](#3-prerequisites)
4. [Networking — VPC Setup](#4-networking--vpc-setup)
5. [Database — RDS PostgreSQL](#5-database--rds-postgresql)
6. [File Storage — S3](#6-file-storage--s3)
7. [Secrets — AWS Secrets Manager](#7-secrets--aws-secrets-manager-and-mcp-oauth-credentials)
8. [Compute — EC2 Instance](#8-compute--ec2-instance)
9. [HTTPS — Application Load Balancer + ACM](#9-https--application-load-balancer--acm)
10. [DNS — Route 53](#10-dns--route-53)
11. [Install and Configure Paperclip](#11-install-and-configure-paperclip)
12. [First Boot and Admin Claim](#12-first-boot-and-admin-claim)
13. [Run as a Systemd Service](#13-run-as-a-systemd-service)
14. [Create Companies, Agents, and Users](#14-create-companies-agents-and-users)
15. [Secrets for Agent Tools and MCP Connectors](#15-secrets-for-agent-tools-and-mcp-connectors)
16. [Monitoring — CloudWatch](#16-monitoring--cloudwatch)
17. [Backups and Disaster Recovery](#17-backups-and-disaster-recovery)
18. [Cost Estimate](#18-cost-estimate)

---

## 1. Architecture Overview

```
Internet
    │
    ▼
[Route 53] → paperclip.yourcompany.com
    │
    ▼
[AWS Application Load Balancer — HTTPS :443]
    │  TLS terminated here using ACM certificate (free)
    │  Forwards HTTP to EC2 on port 3100
    ▼
[EC2 Instance — t3.medium]  ← in private subnet
    │  Paperclip Node.js server on port 3100
    │
    ├── [RDS PostgreSQL — db.t3.micro]  ← in private subnet
    │    Managed, automated backups, Multi-AZ optional
    │
    ├── [S3 Bucket]
    │    Agent work products, attachments, DB backups
    │
    └── [AWS Secrets Manager]
         All secrets: auth tokens, API keys, MCP credentials,
         OAuth tokens for connected tools
```

**Why AWS over a plain VPS:**
- RDS handles PostgreSQL backups, patches, and failover automatically
- Secrets Manager rotates and audits credentials — agents never touch raw secrets
- ACM gives you free TLS certificates that auto-renew
- ALB gives you health checks and easy scaling
- IAM Roles mean EC2 never needs hardcoded AWS credentials in `.env`

---

## 2. AWS Services Used and Why

| Service | Purpose | Estimated cost |
|---|---|---|
| EC2 t3.medium | Paperclip Node.js server | ~$30/month |
| RDS db.t3.micro PostgreSQL | Managed database | ~$15/month |
| S3 | File storage, backups | ~$1–5/month |
| ALB | HTTPS termination, health checks | ~$16/month |
| ACM | Free TLS certificate | $0 |
| Route 53 | DNS | ~$0.50/month |
| Secrets Manager | Encrypted credential storage | ~$0.40/secret/month |
| CloudWatch | Logs and monitoring | ~$3/month |

**Total: approximately $65–80/month for a production setup.**

---

## 3. Prerequisites

- AWS account with billing set up
- AWS CLI installed and configured: `aws configure`
- A domain name (can be registered in Route 53 or any registrar)
- Basic familiarity with the AWS Console or Terraform

All steps below use the AWS Console. CLI equivalents are noted where helpful.

---

## 4. Networking — VPC Setup

### 4.1 Create a VPC

Go to **VPC → Create VPC**:
- Name: `paperclip-vpc`
- IPv4 CIDR: `10.0.0.0/16`
- Enable DNS hostnames: Yes
- Enable DNS resolution: Yes

### 4.2 Create subnets

Create two public subnets (for ALB — ALB requires 2 AZs) and one private subnet (for EC2 and RDS):

| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| `paperclip-public-1` | `10.0.1.0/24` | `ap-south-1a` | ALB |
| `paperclip-public-2` | `10.0.2.0/24` | `ap-south-1b` | ALB (second AZ) |
| `paperclip-private-1` | `10.0.10.0/24` | `ap-south-1a` | EC2 + RDS |

### 4.3 Internet Gateway and routing

- Create an **Internet Gateway**, attach it to `paperclip-vpc`
- Create a **route table** for public subnets: add route `0.0.0.0/0 → igw-xxxxx`
- Associate the route table with both public subnets

For the private subnet, add a **NAT Gateway** (in a public subnet) so EC2 can reach the internet for npm installs and GitHub pulls, without being directly internet-accessible.

### 4.4 Security Groups

Create these security groups in `paperclip-vpc`:

**`sg-alb` (for the Application Load Balancer):**
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | TCP | 80 | 0.0.0.0/0 |
| Inbound | TCP | 443 | 0.0.0.0/0 |

**`sg-paperclip` (for the EC2 instance):**
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | TCP | 3100 | sg-alb (only ALB can reach Paperclip) |
| Inbound | TCP | 22 | Your IP address only |
| Outbound | All | All | 0.0.0.0/0 |

**`sg-rds` (for RDS):**
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | TCP | 5432 | sg-paperclip (only EC2 can reach DB) |

---

## 5. Database — RDS PostgreSQL

### 5.1 Create a DB Subnet Group

Go to **RDS → Subnet Groups → Create**:
- Name: `paperclip-db-subnet-group`
- VPC: `paperclip-vpc`
- Add subnets: both private subnets (create a second private subnet in `ap-south-1b` if needed for multi-AZ)

### 5.2 Create the RDS Instance

Go to **RDS → Create Database**:
- Engine: PostgreSQL 16
- Template: Free tier (dev) or Production
- DB instance identifier: `paperclip-db`
- Master username: `paperclip_user`
- Master password: generate a strong password, save it
- Instance class: `db.t3.micro` (scale up as needed)
- Storage: 20 GB gp3
- VPC: `paperclip-vpc`
- Subnet group: `paperclip-db-subnet-group`
- Security group: `sg-rds`
- Public access: **No**
- Enable automatic backups: Yes, retention 7 days

Note the **Endpoint** (something like `paperclip-db.xxx.ap-south-1.rds.amazonaws.com`) — you'll need this for `DATABASE_URL`.

---

## 6. File Storage — S3

### 6.1 Create the bucket

Go to **S3 → Create Bucket**:
- Name: `paperclip-storage-yourdomain` (must be globally unique)
- Region: same as your EC2
- Block all public access: **Yes** (Paperclip generates signed URLs for access, never public)
- Versioning: optional but recommended
- Encryption: SSE-S3 (default)

### 6.2 Create an IAM policy for Paperclip

Go to **IAM → Policies → Create Policy**, use JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::paperclip-storage-yourdomain",
        "arn:aws:s3:::paperclip-storage-yourdomain/*"
      ]
    }
  ]
}
```

Name it `paperclip-s3-policy`.

---

## 7. Secrets — AWS Secrets Manager and MCP/OAuth Credentials

This is the most important section for agent tool connectivity.

### 7.1 Why Secrets Manager instead of `.env`

With AWS Secrets Manager:
- Secrets are **never written to disk** on the EC2 instance
- All secret access is logged in CloudTrail (audit trail)
- Secrets can be **auto-rotated** (critical for OAuth tokens)
- IAM Role on EC2 grants access — no hardcoded credentials anywhere
- Company admins can update secrets via AWS Console without SSHing into the server

### 7.2 Store all credentials as Secrets Manager secrets

For each credential, go to **Secrets Manager → Store a new secret**:

**Paperclip core secrets:**
```
Secret name: paperclip/core/better-auth-secret
Value: {"BETTER_AUTH_SECRET": "your-generated-value"}

Secret name: paperclip/core/agent-jwt-secret
Value: {"PAPERCLIP_AGENT_JWT_SECRET": "your-generated-value"}

Secret name: paperclip/core/db-url
Value: {"DATABASE_URL": "postgresql://paperclip_user:password@rds-endpoint:5432/paperclip"}
```

**AI provider keys:**
```
Secret name: paperclip/agents/anthropic
Value: {"ANTHROPIC_API_KEY": "sk-ant-..."}

Secret name: paperclip/agents/openai
Value: {"OPENAI_API_KEY": "sk-..."}
```

**MCP and tool connector tokens (per company):**
```
Secret name: paperclip/company-a/github-token
Value: {"GITHUB_TOKEN": "ghp_..."}

Secret name: paperclip/company-a/slack-bot-token
Value: {"SLACK_BOT_TOKEN": "xoxb-..."}

Secret name: paperclip/company-a/notion-token
Value: {"NOTION_TOKEN": "secret_..."}
```

**OAuth tokens for connected services:**
```
Secret name: paperclip/company-a/google-oauth
Value: {
  "GOOGLE_ACCESS_TOKEN": "ya29.xxx",
  "GOOGLE_REFRESH_TOKEN": "1//xxx",
  "GOOGLE_CLIENT_ID": "xxx.apps.googleusercontent.com",
  "GOOGLE_CLIENT_SECRET": "GOCSPX-xxx",
  "GOOGLE_TOKEN_EXPIRY": "1720000000"
}
```

**Why store OAuth tokens in Secrets Manager:**
Secrets Manager supports **automatic rotation**. You can write a Lambda function that:
1. Calls the OAuth token refresh endpoint using the refresh token
2. Stores the new access token back to Secrets Manager
3. Paperclip picks up the rotated value on next agent heartbeat

See section 7.4 for the rotation Lambda.

### 7.3 Create an IAM Role for the EC2 instance

Go to **IAM → Roles → Create Role**:
- Trusted entity: EC2
- Attach policies:
  - `paperclip-s3-policy` (created above)
  - A new inline policy for Secrets Manager access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:s3:::arn:aws:secretsmanager:ap-south-1:ACCOUNT_ID:secret:paperclip/*"
    }
  ]
}
```

Name the role `paperclip-ec2-role`.

When you launch the EC2 instance (next section), attach this IAM role to it. The EC2 instance can then read secrets without any AWS credentials in `.env`.

### 7.4 OAuth token refresh Lambda (optional but recommended)

For Google/Salesforce/etc. OAuth tokens that expire hourly:

Go to **Lambda → Create Function**:
- Runtime: Node.js 20.x
- Name: `paperclip-oauth-refresher`

```javascript
// Lambda function: refreshes OAuth access tokens
import { SecretsManagerClient, GetSecretValueCommand, UpdateSecretCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "ap-south-1" });

export const handler = async (event) => {
  const secretName = event.secretName || "paperclip/company-a/google-oauth";
  
  // Get current secret
  const { SecretString } = await client.send(new GetSecretValueCommand({ SecretId: secretName }));
  const creds = JSON.parse(SecretString);
  
  // Check if token is expiring within 10 minutes
  const expiryTime = parseInt(creds.GOOGLE_TOKEN_EXPIRY);
  if (Date.now() / 1000 < expiryTime - 600) return { status: "not_needed" };
  
  // Refresh via Google OAuth endpoint
  const resp = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: creds.GOOGLE_CLIENT_ID,
      client_secret: creds.GOOGLE_CLIENT_SECRET,
      refresh_token: creds.GOOGLE_REFRESH_TOKEN,
      grant_type: "refresh_token"
    })
  });
  const tokens = await resp.json();
  
  // Update the secret with new access token
  creds.GOOGLE_ACCESS_TOKEN = tokens.access_token;
  creds.GOOGLE_TOKEN_EXPIRY = String(Math.floor(Date.now() / 1000) + tokens.expires_in);
  
  await client.send(new UpdateSecretCommand({
    SecretId: secretName,
    SecretString: JSON.stringify(creds)
  }));
  
  return { status: "refreshed" };
};
```

Schedule it with **EventBridge** to run every 30 minutes. Now your Google OAuth token is always valid when agents need it.

---

## 8. Compute — EC2 Instance

### 8.1 Launch the EC2 instance

Go to **EC2 → Launch Instance**:
- Name: `paperclip-server`
- AMI: Ubuntu Server 24.04 LTS
- Instance type: `t3.medium` (2 vCPU, 4 GB RAM)
- Key pair: create or use existing (save the `.pem` file)
- Network: `paperclip-vpc`, subnet: `paperclip-private-1`
- Auto-assign public IP: **Disable** (it's behind ALB, doesn't need a public IP)
- Security group: `sg-paperclip`
- IAM instance profile: `paperclip-ec2-role`
- Storage: 30 GB gp3

### 8.2 SSH to the instance (via bastion or SSM)

Since the instance has no public IP, use **AWS Systems Manager Session Manager** (no need for a bastion host):

```bash
# Install SSM plugin for AWS CLI if not already installed
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx --region ap-south-1
```

Or add a bastion host in a public subnet for SSH access.

### 8.3 Install Node.js and pnpm

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs git
npm install -g pnpm@latest
```

---

## 9. HTTPS — Application Load Balancer + ACM

### 9.1 Request a certificate via ACM

Go to **ACM → Request Certificate**:
- Type: Public
- Domain: `paperclip.yourcompany.com`
- Validation: DNS validation (ACM provides a CNAME record to add to your DNS)
- Add the CNAME to Route 53 (or your DNS provider) and wait for validation (~2 minutes)

### 9.2 Create a Target Group

Go to **EC2 → Target Groups → Create**:
- Target type: Instances
- Name: `paperclip-tg`
- Protocol: HTTP, Port: 3100
- VPC: `paperclip-vpc`
- Health check path: `/api/health`
- Register the EC2 instance as a target

### 9.3 Create the Application Load Balancer

Go to **EC2 → Load Balancers → Create ALB**:
- Name: `paperclip-alb`
- Scheme: Internet-facing
- IP type: IPv4
- VPC: `paperclip-vpc`
- Subnets: both public subnets (`paperclip-public-1` and `paperclip-public-2`)
- Security group: `sg-alb`

**Listeners:**
- HTTP :80 → redirect to HTTPS :443
- HTTPS :443 → forward to `paperclip-tg`
  - SSL certificate: select the ACM certificate you just created

Note the **ALB DNS name** (e.g. `paperclip-alb-xxx.ap-south-1.elb.amazonaws.com`).

---

## 10. DNS — Route 53

Go to **Route 53 → Hosted Zones** → your domain → **Create Record**:
- Record name: `paperclip`
- Record type: A
- Alias: Yes
- Route traffic to: Alias to Application and Classic Load Balancer
- Select: your ALB

The record will look like: `paperclip.yourcompany.com → ALB DNS name`

---

## 11. Install and Configure Paperclip

SSH into the EC2 instance (via SSM or bastion).

### 11.1 Clone and build

```bash
sudo useradd --system --create-home --shell /bin/bash paperclip
sudo -u paperclip bash
cd ~
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
pnpm install
pnpm build
exit
```

### 11.2 Pull secrets from Secrets Manager into .env

Create a helper script that fetches secrets from Secrets Manager and writes `.env`:

```bash
sudo nano /opt/paperclip-env-refresh.sh
```

```bash
#!/bin/bash
# Fetches secrets from AWS Secrets Manager and writes .env for Paperclip

REGION="ap-south-1"

get_secret() {
  aws secretsmanager get-secret-value \
    --region $REGION \
    --secret-id "$1" \
    --query SecretString \
    --output text | python3 -c "import sys, json; d=json.load(sys.stdin); [print(f'{k}={v}') for k,v in d.items()]"
}

cat > /home/paperclip/paperclip/.env << EOF
# Auto-generated by env-refresh.sh — do not edit manually
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_PUBLIC_URL=https://paperclip.yourcompany.com
NODE_ENV=production
PAPERCLIP_AUTH_DISABLE_SIGN_UP=true
PAPERCLIP_STORAGE_PROVIDER=s3
AWS_REGION=${REGION}
PAPERCLIP_S3_BUCKET=paperclip-storage-yourdomain

$(get_secret "paperclip/core/better-auth-secret")
$(get_secret "paperclip/core/agent-jwt-secret")
$(get_secret "paperclip/core/db-url")
EOF

chmod 600 /home/paperclip/paperclip/.env
chown paperclip:paperclip /home/paperclip/paperclip/.env
echo "✓ .env refreshed from Secrets Manager"
```

```bash
sudo chmod +x /opt/paperclip-env-refresh.sh
sudo /opt/paperclip-env-refresh.sh
```

> Note: Because the EC2 has an IAM Role attached, `aws` CLI calls work without any credentials in `.env`. This is the secure pattern — no AWS access keys ever written to disk.

---

## 12. First Boot and Admin Claim

### 12.1 Temporarily enable sign-up

Edit the `.env` script or directly:
```bash
sudo -u paperclip sed -i 's/PAPERCLIP_AUTH_DISABLE_SIGN_UP=true/PAPERCLIP_AUTH_DISABLE_SIGN_UP=false/' /home/paperclip/paperclip/.env
```

### 12.2 Start Paperclip manually, watch for claim URL

```bash
sudo -u paperclip bash -c "cd ~/paperclip && NODE_ENV=production pnpm paperclipai start 2>&1 | tee /tmp/paperclip-boot.log"
```

Watch for the claim URL in the output. Open it in your browser.

### 12.3 Lock sign-up back down

```bash
sudo -u paperclip sed -i 's/PAPERCLIP_AUTH_DISABLE_SIGN_UP=false/PAPERCLIP_AUTH_DISABLE_SIGN_UP=true/' /home/paperclip/paperclip/.env
```

---

## 13. Run as a Systemd Service

```bash
sudo nano /etc/systemd/system/paperclip.service
```

```ini
[Unit]
Description=Paperclip AI Agent Orchestration
After=network.target

[Service]
Type=simple
User=paperclip
WorkingDirectory=/home/paperclip/paperclip
EnvironmentFile=/home/paperclip/paperclip/.env
ExecStart=/usr/bin/pnpm paperclipai start
Restart=always
RestartSec=15
StandardOutput=journal
StandardError=journal
SyslogIdentifier=paperclip
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable paperclip
sudo systemctl start paperclip
```

---

## 14. Create Companies, Agents, and Users

Same as the custom server guide. Log in at `https://paperclip.yourcompany.com`:

1. Create 3 companies (one per initiative)
2. Add agents to each company, referencing company secrets for their credentials
3. Invite users:
   - Set `PAPERCLIP_AUTH_DISABLE_SIGN_UP=false` temporarily
   - Create invite links (with or without `admin` permission)
   - Send links, users register
   - Set `PAPERCLIP_AUTH_DISABLE_SIGN_UP=true` again

---

## 15. Secrets for Agent Tools and MCP Connectors

When adding an agent in the Paperclip UI, in the **Environment Variables** section:
- Add `ANTHROPIC_API_KEY` and point it to the company secret `$ANTHROPIC_API_KEY`
- Add `GITHUB_TOKEN=$GITHUB_TOKEN` (maps to the company secret you stored)
- For OAuth tools: `GOOGLE_ACCESS_TOKEN=$GOOGLE_ACCESS_TOKEN`

At agent run time, Paperclip fetches the current value from its encrypted secrets store (which you populate from Secrets Manager via the env-refresh script). For OAuth tokens that rotate, run the env-refresh script on a cron:

```bash
# /etc/cron.d/paperclip-secrets
*/30 * * * * root /opt/paperclip-env-refresh.sh && systemctl reload paperclip
```

Or, for a cleaner approach, configure Paperclip to call the Secrets Manager SDK directly at run time (advanced — requires custom adapter code).

**MCP Gateway on AWS:**
To run an MCP gateway (for OAuth-backed MCPs), deploy it as a separate Lambda function or ECS task, and store only the gateway's API key in Secrets Manager as `paperclip/company-a/mcp-gateway-key`. Agents call the gateway endpoint; the gateway handles OAuth internally.

---

## 16. Monitoring — CloudWatch

### 16.1 Application logs

Paperclip logs to systemd journal. Forward to CloudWatch:

```bash
sudo apt install -y amazon-cloudwatch-agent
```

Configure `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/journal/*/",
            "log_group_name": "paperclip",
            "log_stream_name": "{instance_id}",
            "journal_path": "/var/log/journal"
          }
        ]
      }
    }
  }
}
```

### 16.2 Alerting

In CloudWatch → Alarms, create alerts for:
- EC2 CPU > 80% for 10 minutes
- ALB Target health check failures
- RDS storage < 20% free
- Paperclip logs containing "FATAL" or "crash"

---

## 17. Backups and Disaster Recovery

| Layer | Backup method | RPO |
|---|---|---|
| RDS | Automated daily snapshots (7-day retention) + point-in-time recovery | 5 minutes |
| S3 | Versioning enabled | Immediate |
| EC2 | AMI snapshots weekly | 1 week |
| Secrets Manager | Versioned automatically | Immediate |

### Manual RDS restore

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier paperclip-db-restored \
  --db-snapshot-identifier rds:paperclip-db-2026-05-01
```

---

## 18. Cost Estimate

| Service | Spec | Monthly Cost (ap-south-1) |
|---|---|---|
| EC2 t3.medium | On-Demand | ~$30 |
| RDS db.t3.micro | Single-AZ | ~$15 |
| ALB | 1 LCU assumed | ~$16 |
| S3 | 50 GB + requests | ~$2 |
| Secrets Manager | 10 secrets | ~$4 |
| NAT Gateway | 1 | ~$32 |
| Route 53 | 1 hosted zone | ~$0.50 |
| CloudWatch | Logs + metrics | ~$3 |
| **Total** | | **~$100–105/month** |

**Cost reduction tips:**
- Use a Reserved Instance for EC2 (1-year): saves ~40%
- Replace NAT Gateway with a small `t3.nano` NAT instance: saves ~$25/month
- Use RDS Graviton instance (`db.t4g.micro`): saves ~20%

---

*Last updated for Paperclip v2026.428.0+*
