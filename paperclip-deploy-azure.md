# Paperclip Deployment Guide — Azure Cloud Services

> **Target:** Microsoft Azure — VM + Azure Database for PostgreSQL + Blob Storage + Key Vault + Application Gateway + Entra ID
> **Mode:** `authenticated + public`
> **Effort:** Higher setup — enterprise features, excellent SSO/Entra ID integration for human logins

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Azure Services Used and Why](#2-azure-services-used-and-why)
3. [Prerequisites](#3-prerequisites)
4. [Resource Group and Networking](#4-resource-group-and-networking)
5. [Database — Azure Database for PostgreSQL](#5-database--azure-database-for-postgresql)
6. [File Storage — Azure Blob Storage](#6-file-storage--azure-blob-storage)
7. [Secrets — Azure Key Vault and MCP/OAuth Credentials](#7-secrets--azure-key-vault-and-mcpoauth-credentials)
8. [Compute — Azure VM](#8-compute--azure-vm)
9. [HTTPS — Application Gateway + Managed Certificate](#9-https--application-gateway--managed-certificate)
10. [DNS — Azure DNS or External](#10-dns--azure-dns-or-external)
11. [Install and Configure Paperclip](#11-install-and-configure-paperclip)
12. [First Boot and Admin Claim](#12-first-boot-and-admin-claim)
13. [Run as a Systemd Service](#13-run-as-a-systemd-service)
14. [Create Companies, Agents, and Users](#14-create-companies-agents-and-users)
15. [Secrets for Agent Tools and MCP Connectors](#15-secrets-for-agent-tools-and-mcp-connectors)
16. [Entra ID / SSO Integration](#16-entra-id--sso-integration-for-human-logins)
17. [Monitoring — Azure Monitor](#17-monitoring--azure-monitor)
18. [Backups and Disaster Recovery](#18-backups-and-disaster-recovery)
19. [Cost Estimate](#19-cost-estimate)

---

## 1. Architecture Overview

```
Internet
    │
    ▼
[Azure DNS] → paperclip.yourcompany.com
    │
    ▼
[Azure Application Gateway — HTTPS :443]
    │  TLS terminated using Azure-managed certificate
    │  WAF (Web Application Firewall) optional
    ▼
[Azure VM — Standard_B2s]  ← in private subnet
    │  Paperclip Node.js server on port 3100
    │  Managed Identity attached (no credentials in code)
    │
    ├── [Azure Database for PostgreSQL Flexible Server]
    │    Managed, automated backups, high availability optional
    │
    ├── [Azure Blob Storage]
    │    Agent work products, attachments, DB backups
    │
    └── [Azure Key Vault]
         All secrets: auth tokens, API keys, MCP credentials,
         OAuth tokens — auto-rotatable, RBAC-controlled
         
    ── [Entra ID (Azure AD)] (optional but powerful)
         SSO for human Paperclip logins
         OAuth flow broker for agent tool connectors
```

**Why Azure is especially good here:**
- **Entra ID (Azure AD)** is the gold standard for corporate SSO — if your company already uses Microsoft 365, your users can log into Paperclip with their existing corporate accounts
- **Key Vault** handles OAuth token rotation with native support for secret versioning
- **Managed Identity** means zero credentials in `.env` — the VM proves its identity to Azure services automatically
- **Application Gateway WAF** protects the public endpoint from web attacks

---

## 2. Azure Services Used and Why

| Service | Purpose | Estimated cost |
|---|---|---|
| VM Standard_B2s | Paperclip Node.js server | ~$35/month |
| Azure Database for PostgreSQL Flexible | Managed database | ~$25/month |
| Blob Storage (LRS) | File storage, backups | ~$2/month |
| Application Gateway v2 (WAF_v2) | HTTPS, WAF, routing | ~$35/month |
| Key Vault | Secret storage, rotation | ~$0.03/10k ops |
| Azure DNS | DNS hosting | ~$0.90/month |
| Azure Monitor + Log Analytics | Logs and monitoring | ~$5/month |
| Entra ID | SSO (P1 license per user) | ~$6/user/month (or included in M365) |

**Total: approximately $100–120/month (without Entra ID P1).**

---

## 3. Prerequisites

- Azure subscription with Owner or Contributor role
- Azure CLI installed: `az login`
- A domain name
- Basic Azure Portal familiarity

All steps show the Azure Portal approach with CLI equivalents.

```bash
# Login and set your subscription
az login
az account set --subscription "your-subscription-id"

# Set a variable for your resource group name (used throughout)
export RG="paperclip-rg"
export LOCATION="southindia"   # or centralindia, eastus, westeurope, etc.
```

---

## 4. Resource Group and Networking

### 4.1 Create a Resource Group

```bash
az group create --name $RG --location $LOCATION
```

Or in the Portal: **Resource Groups → Create**

### 4.2 Create a Virtual Network

```bash
az network vnet create \
  --resource-group $RG \
  --name paperclip-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name vm-subnet \
  --subnet-prefix 10.0.1.0/24
```

Create additional subnets:

```bash
# Subnet for Application Gateway (requires a /24 or larger)
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name paperclip-vnet \
  --name appgw-subnet \
  --address-prefix 10.0.2.0/24

# Subnet for PostgreSQL Flexible Server
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name paperclip-vnet \
  --name db-subnet \
  --address-prefix 10.0.3.0/24 \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers
```

### 4.3 Network Security Groups

**NSG for VM (`nsg-paperclip-vm`):**

```bash
az network nsg create --resource-group $RG --name nsg-paperclip-vm

# Allow Paperclip port from App Gateway subnet only
az network nsg rule create \
  --resource-group $RG --nsg-name nsg-paperclip-vm \
  --name allow-appgw-to-paperclip \
  --priority 100 --direction Inbound \
  --source-address-prefix 10.0.2.0/24 \
  --destination-port-ranges 3100 --protocol Tcp --access Allow

# Allow SSH from your IP only
az network nsg rule create \
  --resource-group $RG --nsg-name nsg-paperclip-vm \
  --name allow-ssh \
  --priority 200 --direction Inbound \
  --source-address-prefix "YOUR_IP/32" \
  --destination-port-ranges 22 --protocol Tcp --access Allow

# Deny all other inbound
az network nsg rule create \
  --resource-group $RG --nsg-name nsg-paperclip-vm \
  --name deny-all \
  --priority 4096 --direction Inbound \
  --source-address-prefix "*" \
  --destination-port-ranges "*" --protocol "*" --access Deny
```

---

## 5. Database — Azure Database for PostgreSQL

### 5.1 Create Flexible Server (private, VNet-integrated)

In the Portal: **Azure Database for PostgreSQL → Flexible Server → Create**:
- Server name: `paperclip-db-server`
- Region: same as your other resources
- PostgreSQL version: 16
- Compute tier: Burstable (for cost), `Standard_B1ms`
- Authentication: PostgreSQL only
- Admin username: `paperclip_admin`
- Password: generate a strong password, save it for Key Vault
- Networking: Private access (VNet integration)
- VNet: `paperclip-vnet`, Subnet: `db-subnet`
- High availability: optional (adds ~2× cost)

```bash
# CLI equivalent
az postgres flexible-server create \
  --resource-group $RG \
  --name paperclip-db-server \
  --location $LOCATION \
  --admin-user paperclip_admin \
  --admin-password "YourStrongPassword123!" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --version 16 \
  --vnet paperclip-vnet \
  --subnet db-subnet \
  --yes
```

### 5.2 Create the database

```bash
az postgres flexible-server db create \
  --resource-group $RG \
  --server-name paperclip-db-server \
  --database-name paperclip
```

Note the connection string:
```
postgresql://paperclip_admin:password@paperclip-db-server.postgres.database.azure.com:5432/paperclip?sslmode=require
```

---

## 6. File Storage — Azure Blob Storage

### 6.1 Create a storage account and container

```bash
# Storage account name must be globally unique, 3-24 lowercase alphanumeric
az storage account create \
  --resource-group $RG \
  --name paperclipstorageyou \
  --location $LOCATION \
  --sku Standard_LRS \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

az storage container create \
  --account-name paperclipstorageyou \
  --name paperclip-data \
  --auth-mode login
```

> Note: Paperclip uses S3-compatible storage. Azure Blob Storage is S3-compatible when accessed via the Azure Storage SDK — set `PAPERCLIP_STORAGE_PROVIDER=azure` or use an S3-to-Azure bridge. Check the latest Paperclip release notes for native Azure Blob support.

---

## 7. Secrets — Azure Key Vault and MCP/OAuth Credentials

### 7.1 Create the Key Vault

```bash
az keyvault create \
  --resource-group $RG \
  --name paperclip-kv-yourdomain \
  --location $LOCATION \
  --sku standard \
  --enable-rbac-authorization true
```

### 7.2 Store all credentials

```bash
# Paperclip core secrets
az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "BETTER-AUTH-SECRET" --value "your-64-char-random-string"

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "PAPERCLIP-AGENT-JWT-SECRET" --value "another-64-char-random-string"

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "DATABASE-URL" \
  --value "postgresql://paperclip_admin:password@paperclip-db-server.postgres.database.azure.com:5432/paperclip?sslmode=require"

# AI provider keys
az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "ANTHROPIC-API-KEY" --value "sk-ant-..."

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "OPENAI-API-KEY" --value "sk-..."

# Tool connector tokens
az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-GITHUB-TOKEN" --value "ghp_..."

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-SLACK-BOT-TOKEN" --value "xoxb-..."

# OAuth credentials (Google example)
az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-GOOGLE-ACCESS-TOKEN" --value "ya29.xxx"

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-GOOGLE-REFRESH-TOKEN" --value "1//xxx"
```

### 7.3 Assign Key Vault access to the VM's Managed Identity

After creating the VM (next section), assign it RBAC access to Key Vault:

```bash
# Get the VM's principal ID
VM_PRINCIPAL=$(az vm identity show \
  --resource-group $RG \
  --name paperclip-vm \
  --query principalId --output tsv)

# Grant Key Vault Secrets User role
az role assignment create \
  --assignee $VM_PRINCIPAL \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/paperclip-kv-yourdomain"
```

### 7.4 OAuth token refresh with Azure Function

For auto-refreshing OAuth access tokens (Google, Salesforce, etc.):

Create an **Azure Function App** (Consumption plan, Node.js 20):

```javascript
// function-app/refresh-token/index.js
import { SecretClient } from "@azure/keyvault-secrets";
import { DefaultAzureCredential } from "@azure/identity";

const credential = new DefaultAzureCredential();
const client = new SecretClient(
  "https://paperclip-kv-yourdomain.vault.azure.net/",
  credential
);

export default async function (context, timer) {
  // Get current OAuth credentials
  const accessToken = await client.getSecret("COMPANY-A-GOOGLE-ACCESS-TOKEN");
  const refreshToken = await client.getSecret("COMPANY-A-GOOGLE-REFRESH-TOKEN");
  const clientId = await client.getSecret("COMPANY-A-GOOGLE-CLIENT-ID");
  const clientSecret = await client.getSecret("COMPANY-A-GOOGLE-CLIENT-SECRET");
  const expiry = await client.getSecret("COMPANY-A-GOOGLE-TOKEN-EXPIRY");

  // Check if expiring within 10 minutes
  if (Date.now() / 1000 < parseInt(expiry.value) - 600) {
    context.log("Token still valid, no refresh needed");
    return;
  }

  // Refresh the token
  const resp = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: clientId.value,
      client_secret: clientSecret.value,
      refresh_token: refreshToken.value,
      grant_type: "refresh_token"
    })
  });
  const tokens = await resp.json();

  // Update secrets
  await client.setSecret("COMPANY-A-GOOGLE-ACCESS-TOKEN", tokens.access_token);
  await client.setSecret("COMPANY-A-GOOGLE-TOKEN-EXPIRY",
    String(Math.floor(Date.now() / 1000) + tokens.expires_in));

  context.log("Token refreshed successfully");
}
```

Schedule the function to run every 30 minutes via **Timer Trigger**: `0 */30 * * * *`.

Give the Function App a **Managed Identity** and assign it **Key Vault Secrets Officer** role so it can write updated tokens back to the vault.

### 7.5 Key Vault secret versioning — safe token updates

Azure Key Vault keeps all versions of every secret. If a token refresh produces a bad token, you can roll back:

```bash
# List all versions of the Google access token
az keyvault secret list-versions \
  --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-GOOGLE-ACCESS-TOKEN"

# Restore a previous version
az keyvault secret set-attributes \
  --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-GOOGLE-ACCESS-TOKEN" \
  --version "previous-version-id" \
  --enabled true
```

---

## 8. Compute — Azure VM

### 8.1 Create the VM

```bash
az vm create \
  --resource-group $RG \
  --name paperclip-vm \
  --image Ubuntu2404 \
  --size Standard_B2s \
  --admin-username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub \
  --vnet-name paperclip-vnet \
  --subnet vm-subnet \
  --nsg nsg-paperclip-vm \
  --public-ip-address "" \
  --assign-identity
```

The `--assign-identity` flag creates a **System-assigned Managed Identity** — the VM can authenticate to Key Vault and Blob Storage without any credentials in code.

### 8.2 Connect to the VM

Since the VM has no public IP, use Azure Bastion or the Serial Console:

```bash
# Option A: Azure Bastion (set up a Bastion host in a BastionSubnet)
az network bastion create \
  --resource-group $RG \
  --name paperclip-bastion \
  --vnet-name paperclip-vnet \
  --public-ip-address paperclip-bastion-ip \
  --location $LOCATION

az network bastion ssh \
  --resource-group $RG \
  --name paperclip-bastion \
  --target-resource-id $(az vm show -g $RG -n paperclip-vm --query id -o tsv) \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/id_rsa

# Option B: If you gave the VM a public IP (not recommended for prod),
# just SSH normally:
# ssh azureuser@vm-public-ip
```

---

## 9. HTTPS — Application Gateway + Managed Certificate

### 9.1 Create a public IP for Application Gateway

```bash
az network public-ip create \
  --resource-group $RG \
  --name paperclip-appgw-ip \
  --allocation-method Static \
  --sku Standard \
  --dns-label paperclip-appgw
```

### 9.2 Create the Application Gateway with SSL

In the Portal: **Application Gateway → Create**:
- Name: `paperclip-appgw`
- Tier: Standard V2 (or WAF V2 for extra security)
- VNet: `paperclip-vnet`, Subnet: `appgw-subnet`
- Public IP: `paperclip-appgw-ip`
- Listener: HTTPS :443
- SSL Certificate: Add a certificate — choose "Key Vault certificate" and select your Key Vault, or upload a PFX
- Backend pool: Add the VM's private IP with port 3100
- HTTP settings: HTTP, port 3100, path `/api/health` for probe

**For a free managed certificate via App Gateway:**
```bash
# Create the certificate resource referencing your Key Vault SSL cert
# (You'd first store the TLS cert in Key Vault, or use Azure-managed certs)
```

> Tip: For simplest TLS, create a free certificate via **Let's Encrypt** using the `certbot` DNS challenge, upload the PFX to Key Vault, and reference it from Application Gateway.

### 9.3 HTTP to HTTPS redirect

In Application Gateway → Listeners, add:
- HTTP listener on port 80
- Routing rule: redirect HTTP → HTTPS

---

## 10. DNS — Azure DNS or External

### 10.1 If using Azure DNS

```bash
# Create DNS zone
az network dns zone create \
  --resource-group $RG \
  --name yourcompany.com

# Create A record pointing to App Gateway's public IP
APPGW_IP=$(az network public-ip show \
  --resource-group $RG \
  --name paperclip-appgw-ip \
  --query ipAddress --output tsv)

az network dns record-set a add-record \
  --resource-group $RG \
  --zone-name yourcompany.com \
  --record-set-name paperclip \
  --ipv4-address $APPGW_IP
```

### 10.2 If using an external DNS provider

Add an A record manually:
- Name: `paperclip`
- Value: the public IP of Application Gateway

---

## 11. Install and Configure Paperclip

SSH into the VM (via Bastion or your chosen method).

### 11.1 Install dependencies

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs git
sudo npm install -g pnpm@latest

# Install Azure CLI (needed for Key Vault script)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### 11.2 Clone and build Paperclip

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

### 11.3 Fetch secrets from Key Vault into .env

Create a secret-refresh script. Because the VM has a Managed Identity, `az` CLI commands work without any credentials:

```bash
sudo nano /opt/paperclip-env-refresh.sh
```

```bash
#!/bin/bash
# Fetches secrets from Azure Key Vault and writes .env for Paperclip
# Uses VM Managed Identity — no credentials required

KV="paperclip-kv-yourdomain"

get_kv() {
  az keyvault secret show --vault-name $KV --name "$1" \
    --query "value" --output tsv 2>/dev/null
}

cat > /home/paperclip/paperclip/.env << EOF
# Auto-generated by env-refresh.sh — do not edit manually
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_PUBLIC_URL=https://paperclip.yourcompany.com
NODE_ENV=production
PAPERCLIP_AUTH_DISABLE_SIGN_UP=true

BETTER_AUTH_SECRET=$(get_kv "BETTER-AUTH-SECRET")
PAPERCLIP_AGENT_JWT_SECRET=$(get_kv "PAPERCLIP-AGENT-JWT-SECRET")
DATABASE_URL=$(get_kv "DATABASE-URL")

ANTHROPIC_API_KEY=$(get_kv "ANTHROPIC-API-KEY")
OPENAI_API_KEY=$(get_kv "OPENAI-API-KEY")

GITHUB_TOKEN=$(get_kv "COMPANY-A-GITHUB-TOKEN")
SLACK_BOT_TOKEN=$(get_kv "COMPANY-A-SLACK-BOT-TOKEN")

GOOGLE_ACCESS_TOKEN=$(get_kv "COMPANY-A-GOOGLE-ACCESS-TOKEN")
GOOGLE_REFRESH_TOKEN=$(get_kv "COMPANY-A-GOOGLE-REFRESH-TOKEN")
EOF

chmod 600 /home/paperclip/paperclip/.env
chown paperclip:paperclip /home/paperclip/paperclip/.env
echo "✓ .env refreshed from Key Vault"
```

```bash
sudo chmod +x /opt/paperclip-env-refresh.sh
sudo /opt/paperclip-env-refresh.sh
```

Schedule to run every 30 minutes to pick up rotated OAuth tokens:
```bash
echo "*/30 * * * * root /opt/paperclip-env-refresh.sh" | sudo tee /etc/cron.d/paperclip-secrets
```

---

## 12. First Boot and Admin Claim

```bash
# Temporarily enable sign-up
sudo -u paperclip sed -i 's/PAPERCLIP_AUTH_DISABLE_SIGN_UP=true/PAPERCLIP_AUTH_DISABLE_SIGN_UP=false/' \
  /home/paperclip/paperclip/.env

# Start Paperclip and watch for the claim URL
sudo -u paperclip bash -c "cd ~/paperclip && pnpm paperclipai start 2>&1 | tee /tmp/boot.log"

# In another terminal, watch for the claim URL:
grep "board-claim" /tmp/boot.log
```

1. Open `https://paperclip.yourcompany.com` and register with email + password
2. Open the claim URL — you become instance admin
3. Stop Paperclip
4. Re-lock sign-up:
```bash
sudo -u paperclip sed -i 's/PAPERCLIP_AUTH_DISABLE_SIGN_UP=false/PAPERCLIP_AUTH_DISABLE_SIGN_UP=true/' \
  /home/paperclip/paperclip/.env
sudo /opt/paperclip-env-refresh.sh   # also re-write from Key Vault to be safe
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
sudo systemctl status paperclip
```

---

## 14. Create Companies, Agents, and Users

Same process as custom server:
1. Create 3 companies in the Paperclip UI
2. Add agents to each company, referencing the secrets injected via `.env`
3. Invite users via invite links (toggle sign-up temporarily as needed)

For Azure-specific best practice: invite team members using their **corporate email addresses** (e.g. `user@yourcompany.com`). When Entra ID SSO is enabled (next section), these users will be able to log in with their existing Microsoft credentials instead of a separate password.

---

## 15. Secrets for Agent Tools and MCP Connectors

When configuring an agent in Paperclip UI → **Environment Variables**, reference the values from `.env` which were pulled from Key Vault:

```
ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
GITHUB_TOKEN=$GITHUB_TOKEN
SLACK_BOT_TOKEN=$SLACK_BOT_TOKEN
GOOGLE_ACCESS_TOKEN=$GOOGLE_ACCESS_TOKEN
```

**For MCP servers on Azure:**

The cleanest architecture is to run your MCP servers as **Azure Container Instances** or **Azure Container Apps** in the same VNet. They are private (no public exposure), and agents reach them via private IP. Store the MCP service endpoint and auth token in Key Vault:

```bash
az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-SLACK-MCP-URL" \
  --value "http://10.0.5.4:3001"    # private IP of your Slack MCP container

az keyvault secret set --vault-name paperclip-kv-yourdomain \
  --name "COMPANY-A-SLACK-MCP-TOKEN" \
  --value "bearer-token-for-slack-mcp"
```

Add to the env-refresh script:
```bash
SLACK_MCP_URL=$(get_kv "COMPANY-A-SLACK-MCP-URL")
SLACK_MCP_TOKEN=$(get_kv "COMPANY-A-SLACK-MCP-TOKEN")
```

Agents then call the Slack MCP at `$SLACK_MCP_URL` using `$SLACK_MCP_TOKEN`.

---

## 16. Entra ID / SSO Integration for Human Logins

This is Azure's biggest advantage for enterprise teams. If your organization uses Microsoft 365, your users can sign into Paperclip with their existing work accounts — no separate Paperclip password needed.

> Note: Native OIDC/SSO in Paperclip is in active development (see GitHub Issue #3028). When it ships, Entra ID will be one of the first supported providers. The steps below show what to prepare now.

### 16.1 Register an App in Entra ID

Go to **Entra ID → App Registrations → New Registration**:
- Name: `Paperclip`
- Supported account types: Accounts in this organizational directory only (Single tenant)
- Redirect URI: `https://paperclip.yourcompany.com/api/auth/callback/microsoft`

After registration:
- Copy the **Application (client) ID** → store as Key Vault secret `ENTRA-CLIENT-ID`
- Go to **Certificates & Secrets** → New client secret → store as `ENTRA-CLIENT-SECRET`
- Copy the **Directory (tenant) ID** → store as Key Vault secret `ENTRA-TENANT-ID`

### 16.2 Configure permissions

In the App Registration → **API Permissions**:
- Add `openid`, `profile`, `email` (Microsoft Graph, Delegated)
- For connecting agent tools (Outlook, SharePoint, Teams MCPs):
  - Add `Mail.Read`, `Files.Read`, `Calendars.Read` as needed
- Click **Grant admin consent**

### 16.3 Add Entra ID to Paperclip's auth config

When SSO ships in Paperclip, add to `.env` (pull from Key Vault):
```env
BETTER_AUTH_MICROSOFT_CLIENT_ID=$(get_kv "ENTRA-CLIENT-ID")
BETTER_AUTH_MICROSOFT_CLIENT_SECRET=$(get_kv "ENTRA-CLIENT-SECRET")
BETTER_AUTH_MICROSOFT_TENANT_ID=$(get_kv "ENTRA-TENANT-ID")
```

Users will then see a "Sign in with Microsoft" button on the Paperclip login page.

### 16.4 Using Entra ID for agent tool OAuth (On-Behalf-Of flow)

For agents that need to call Microsoft APIs (Outlook, SharePoint, Teams) on behalf of a user:

1. Use the Entra ID **On-Behalf-Of (OBO) flow**: a user grants consent once via browser
2. The resulting `access_token` + `refresh_token` are stored in Key Vault per-company
3. The OAuth refresh Azure Function (section 7.4) handles token renewal using the Microsoft refresh endpoint:
   ```
   POST https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
   grant_type=refresh_token
   refresh_token=...
   client_id=...
   client_secret=...
   ```

This means your agents can read/write Outlook emails, SharePoint documents, and Teams messages using corporate identities — without users ever sharing passwords.

---

## 17. Monitoring — Azure Monitor

### 17.1 Send Paperclip logs to Log Analytics

```bash
# Install Azure Monitor agent on the VM
az vm extension set \
  --resource-group $RG \
  --vm-name paperclip-vm \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0
```

Create a **Data Collection Rule** in Azure Monitor to forward systemd journal logs from the VM to a Log Analytics Workspace.

### 17.2 Create alerts

In **Azure Monitor → Alerts → Create**:

| Alert | Condition | Severity |
|---|---|---|
| Paperclip process down | Log query: `Syslog` contains "FATAL" | Sev1 |
| VM CPU high | CPU > 85% for 10 min | Sev2 |
| DB connection errors | PostgreSQL error logs | Sev2 |
| App Gateway unhealthy backend | Health probe failures > 0 | Sev1 |
| Key Vault access denied | `AuditEvent` with `ResultType=Unauthorized` | Sev1 |

Send alerts to email, Teams webhook, or PagerDuty via **Action Groups**.

---

## 18. Backups and Disaster Recovery

| Layer | Azure backup method | RPO |
|---|---|---|
| PostgreSQL | Automated backups (7–35 days) + point-in-time restore | 5 minutes |
| Blob Storage | Versioning + soft delete (7-day retention) | Immediate |
| VM | Azure Backup vault, daily snapshots | 24 hours |
| Key Vault | Soft delete + purge protection (90-day retention) | Immediate |
| Entra ID app reg | ARM template exported to Blob | Daily |

### Enable PostgreSQL backups

In Portal: Azure Database for PostgreSQL → **Backup and Restore**:
- Backup retention: 7 days (default)
- Enable geo-redundant backup: Yes (adds ~20% cost, protects against region failure)

### Point-in-time restore

```bash
az postgres flexible-server restore \
  --resource-group $RG \
  --name paperclip-db-restored \
  --source-server paperclip-db-server \
  --restore-time "2026-05-01T03:00:00Z"
```

---

## 19. Cost Estimate

| Service | Spec | Monthly Cost |
|---|---|---|
| VM Standard_B2s | Pay-as-you-go | ~$35 |
| PostgreSQL Standard_B1ms | Single server | ~$25 |
| Application Gateway v2 | Fixed + 1 CU | ~$35 |
| Blob Storage | 50 GB LRS | ~$2 |
| Key Vault | 20 secrets, 10k ops | ~$1 |
| Azure Bastion | Basic | ~$10 |
| Azure DNS | 1 zone | ~$1 |
| Azure Monitor | Log Analytics 5 GB/day | ~$5 |
| **Total** | | **~$114/month** |

**Cost reduction tips:**
- Use **Reserved VM Instances** (1-year commitment): saves ~40% on VM
- Use **Burstable PostgreSQL** (`Standard_B1ms`): half the cost of General Purpose
- Use Application Gateway **Standard_v2** instead of WAF_v2: saves ~$30/month
- Skip Azure Bastion for dev/small teams, use a jump VM instead: saves ~$10/month

---

## Quick Reference: Key Azure Resources

| Resource | Name | Purpose |
|---|---|---|
| Resource Group | `paperclip-rg` | Container for all resources |
| VNet | `paperclip-vnet` | Private network |
| VM | `paperclip-vm` | Paperclip server |
| PostgreSQL | `paperclip-db-server` | Database |
| Storage Account | `paperclipstorageyou` | File storage |
| Key Vault | `paperclip-kv-yourdomain` | All secrets |
| App Gateway | `paperclip-appgw` | HTTPS + routing |
| Entra App Reg | `Paperclip` | SSO + OAuth for agents |

---

*Last updated for Paperclip v2026.428.0+*
