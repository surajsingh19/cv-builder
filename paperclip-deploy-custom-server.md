# Paperclip Deployment Guide — Custom / Self-Hosted Server

> **Target:** Any Linux VPS or bare-metal server (DigitalOcean, Hetzner, Linode, Vultr, your own hardware, etc.)
> **Mode:** `authenticated + public`
> **Effort:** Medium — full control, lowest cost

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Server Setup](#3-server-setup)
4. [HTTPS with Nginx + Certbot](#4-https-with-nginx--certbot)
5. [Install and Build Paperclip](#5-install-and-build-paperclip)
6. [Environment Configuration](#6-environment-configuration)
7. [Secrets — API Keys, MCP Tokens, and OAuth Connectors](#7-secrets--api-keys-mcp-tokens-and-oauth-connectors)
8. [First Boot and Admin Claim](#8-first-boot-and-admin-claim)
9. [Run as a Systemd Service](#9-run-as-a-systemd-service)
10. [Create Companies and Agents](#10-create-companies-and-agents)
11. [User Access Control](#11-user-access-control)
12. [External PostgreSQL (Optional but Recommended)](#12-external-postgresql-optional-but-recommended)
13. [Backups](#13-backups)
14. [Upgrades](#14-upgrades)
15. [Firewall and Security Hardening](#15-firewall-and-security-hardening)

---

## 1. Architecture Overview

```
Internet
    │
    ▼
[Your Domain: paperclip.yourcompany.com]
    │  (DNS A record → server IP)
    ▼
[Nginx — Port 443 HTTPS, TLS termination]
    │  (reverse proxy to localhost:3100)
    ▼
[Paperclip Node.js Server — Port 3100, not exposed publicly]
    │
    ├── Embedded PostgreSQL (dev/small teams)
    │   OR External PostgreSQL (recommended for production)
    │
    ├── Local file storage
    │   OR S3-compatible storage
    │
    └── Encrypted secrets store
```

**What this setup gives you:**
- Full control over hardware, data, and costs
- Cheapest option (VPS from ~$6/month)
- Manual upgrades and ops burden on you

---

## 2. Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU | 2 vCPU | 4 vCPU |
| RAM | 2 GB | 4–8 GB |
| Disk | 20 GB SSD | 50 GB SSD |
| Node.js | 20.x | 22.x (LTS) |
| pnpm | 9.15+ | latest 9.x |
| Domain | Required | Required (HTTPS mandatory for auth+public) |

---

## 3. Server Setup

SSH into your server and run all setup commands as a non-root user with `sudo` access.

### 3.1 System updates

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget unzip build-essential
```

### 3.2 Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version   # confirm: v20.x.x
```

### 3.3 Install pnpm

```bash
npm install -g pnpm@latest
pnpm --version   # confirm: 9.x.x
```

### 3.4 Create a dedicated system user

Running Paperclip as root is dangerous. Create a dedicated user:

```bash
sudo useradd --system --create-home --shell /bin/bash paperclip
```

---

## 4. HTTPS with Nginx + Certbot

`auth + public` mode **requires HTTPS**. Paperclip signs session cookies — if these travel over HTTP they can be stolen.

### 4.1 Install Nginx and Certbot

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 4.2 Point your domain to the server

In your DNS registrar's control panel, create an A record:
- **Name:** `paperclip` (or `@` for the root domain)
- **Value:** your server's public IP address
- Wait 5–30 minutes for DNS to propagate

Verify with: `dig paperclip.yourcompany.com` — should return your server IP.

### 4.3 Get a free SSL certificate

```bash
sudo certbot --nginx -d paperclip.yourcompany.com
# Follow prompts: enter email, agree to ToS, choose redirect (option 2)
```

Certbot auto-renews the certificate. Confirm renewal works:
```bash
sudo certbot renew --dry-run
```

### 4.4 Configure Nginx as a reverse proxy

Create `/etc/nginx/sites-available/paperclip`:

```nginx
server {
    listen 443 ssl http2;
    server_name paperclip.yourcompany.com;

    # Managed by Certbot:
    ssl_certificate /etc/letsencrypt/live/paperclip.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/paperclip.yourcompany.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Paperclip — forward all traffic
    location / {
        proxy_pass http://127.0.0.1:3100;
        proxy_http_version 1.1;

        # WebSocket support (required for Paperclip live updates)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Increase timeouts for long-running agent streams
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        client_max_body_size 50M;
    }
}

server {
    listen 80;
    server_name paperclip.yourcompany.com;
    return 301 https://$host$request_uri;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/paperclip /etc/nginx/sites-enabled/
sudo nginx -t          # test — must say "syntax is ok"
sudo systemctl reload nginx
```

---

## 5. Install and Build Paperclip

```bash
sudo mkdir -p /opt/paperclip
sudo chown paperclip:paperclip /opt/paperclip

sudo -u paperclip bash
cd /opt/paperclip
git clone https://github.com/paperclipai/paperclip.git .
pnpm install
pnpm build
exit
```

---

## 6. Environment Configuration

Create `/opt/paperclip/.env` (owned by the `paperclip` system user, not readable by others):

```bash
sudo -u paperclip nano /opt/paperclip/.env
sudo chmod 600 /opt/paperclip/.env
```

Paste and fill in:

```env
# ── Deployment mode ─────────────────────────────────────────────────────────
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_PUBLIC_URL=https://paperclip.yourcompany.com

# ── Auth secrets (generate with: openssl rand -hex 32) ──────────────────────
BETTER_AUTH_SECRET=<generate-a-64-char-random-string>
PAPERCLIP_AGENT_JWT_SECRET=<generate-another-64-char-random-string>

# ── Sign-up control ──────────────────────────────────────────────────────────
# Keep true normally. Set to false temporarily only when inviting new users.
PAPERCLIP_AUTH_DISABLE_SIGN_UP=true

# ── Database (leave blank to use embedded PostgreSQL) ─────────────────────────
# For external Postgres, uncomment and fill:
# DATABASE_URL=postgresql://paperclip_user:password@localhost:5432/paperclip

# ── Storage (leave blank for local disk) ──────────────────────────────────────
# For S3-compatible storage, uncomment:
# PAPERCLIP_STORAGE_PROVIDER=s3
# AWS_REGION=ap-south-1
# AWS_ACCESS_KEY_ID=...
# AWS_SECRET_ACCESS_KEY=...
# PAPERCLIP_S3_BUCKET=your-paperclip-bucket

# ── Telemetry (optional disable) ─────────────────────────────────────────────
# PAPERCLIP_TELEMETRY_DISABLED=1

# ── Node ─────────────────────────────────────────────────────────────────────
NODE_ENV=production
```

**Generate secrets:**
```bash
openssl rand -hex 32   # run twice, use one for each secret above
```

---

## 7. Secrets — API Keys, MCP Tokens, and OAuth Connectors

This is the critical section for giving your agents access to external tools.

### 7.1 Understanding Paperclip's two-tier secrets system

| Tier | What it is | Scope | How agents use it |
|---|---|---|---|
| **Instance secrets** | Shared across all companies | Whole instance | All agents on the instance |
| **Company secrets** | Scoped to one company | One company | Only agents in that company |

Secrets are encrypted at rest. They are injected into agent environment variables at run time — the agent process sees them as env vars, they never appear in task content or logs.

### 7.2 Static API keys (simplest — do this first)

For tools with simple API keys (Anthropic, OpenAI, GitHub, Slack, etc.):

1. In Paperclip UI → Company Settings → **Secrets**
2. Click **Add Secret**
3. Name it clearly: `ANTHROPIC_API_KEY`, `GITHUB_TOKEN`, `SLACK_BOT_TOKEN`
4. Paste the value
5. In your agent's configuration, reference it as `$ANTHROPIC_API_KEY`

The secret is stored encrypted in the DB. At run time, Paperclip injects it as an environment variable for that agent's execution.

### 7.3 MCP server tokens

If your agents use MCP servers (e.g. a Slack MCP, Notion MCP, browser MCP):

**Option A — static bearer token** (recommended for most MCPs):
```
# Store in Company Secrets as:
SLACK_MCP_TOKEN=<bearer-token-from-slack-mcp-setup>
NOTION_MCP_TOKEN=<notion-integration-token>

# In your agent's AGENTS.md / skill config, reference:
export MCP_SLACK_AUTH=$SLACK_MCP_TOKEN
```

**Option B — MCP Gateway** (recommended when you have 5+ MCPs or need OAuth):

Run an MCP gateway like [MintMCP](https://mintmcp.com) or a self-hosted proxy in front of your MCPs. The gateway handles OAuth and exposes a single token per agent role. Store only the gateway token in Paperclip secrets. Benefits:
- Agents use one token, gateway handles OAuth refresh
- Role-based tool access (read-only vs read-write agents)
- Audit log of every tool call

### 7.4 OAuth-based connectors (Google, Salesforce, Notion, etc.)

Paperclip does **not** have a built-in OAuth consent screen. The pattern to handle OAuth:

**Step 1 — Complete the OAuth flow externally**

Use any OAuth helper tool (e.g. `npx @modelcontextprotocol/inspector`, Postman, or a small script) to complete the consent flow and capture:
- `access_token`
- `refresh_token`
- `token_expiry`

**Step 2 — Store tokens as Company Secrets**

```
GOOGLE_ACCESS_TOKEN=ya29.xxxxx
GOOGLE_REFRESH_TOKEN=1//xxxxx
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret
```

**Step 3 — Token refresh routine**

Create a Paperclip Routine (scheduled job) with a lightweight agent whose only job is to check if `GOOGLE_ACCESS_TOKEN` is near expiry and refresh it using the `refresh_token`. Store the new access token back as a Company Secret.

```
# Example routine: runs every 50 minutes
# Agent task: "Check GOOGLE_ACCESS_TOKEN expiry. If within 10 minutes of expiry,
# call https://oauth2.googleapis.com/token with the refresh token and update
# the GOOGLE_ACCESS_TOKEN company secret via the Paperclip API."
```

**Step 4 — (Better) Use an MCP Gateway for OAuth**

For production, an MCP gateway is the right answer. It handles token refresh automatically and your agents never touch raw OAuth tokens. Store only the MCP gateway's API key in Paperclip secrets.

### 7.5 SSO for the Paperclip UI itself (human logins)

Paperclip currently supports **email + password** natively. SSO (Google, GitHub, Okta, Entra ID) is handled via the Better Auth plugin system. This is in active development — see [Issue #3028](https://github.com/paperclipai/paperclip/issues/3028) for OIDC/SSO support progress.

To enable Google SSO login when available, you would add to `.env`:
```env
BETTER_AUTH_GOOGLE_CLIENT_ID=...
BETTER_AUTH_GOOGLE_CLIENT_SECRET=...
```

Until SSO ships officially, users log in with email + password you create via the invite flow.

---

## 8. First Boot and Admin Claim

### 8.1 Temporarily enable sign-up

```bash
# Edit .env
PAPERCLIP_AUTH_DISABLE_SIGN_UP=false
```

### 8.2 Start Paperclip manually (first time)

```bash
sudo -u paperclip bash -c "cd /opt/paperclip && NODE_ENV=production pnpm paperclipai start"
```

Watch the logs. You will see a one-time **claim URL** printed prominently:
```
⚠  Board claim URL (one-time):
   https://paperclip.yourcompany.com/board-claim/abc123?code=xyz
```

### 8.3 Register and claim admin

1. Open `https://paperclip.yourcompany.com` in your browser
2. Register with your email + password (this works because sign-up is temporarily enabled)
3. Open the claim URL from the logs — this promotes your account to **instance admin**
4. Stop Paperclip (Ctrl+C)

### 8.4 Lock down sign-up

```bash
# Edit .env
PAPERCLIP_AUTH_DISABLE_SIGN_UP=true
```

---

## 9. Run as a Systemd Service

```bash
sudo nano /etc/systemd/system/paperclip.service
```

```ini
[Unit]
Description=Paperclip AI Agent Orchestration
Documentation=https://docs.paperclip.ing
After=network.target postgresql.service
Wants=network.target

[Service]
Type=simple
User=paperclip
Group=paperclip
WorkingDirectory=/opt/paperclip
EnvironmentFile=/opt/paperclip/.env
ExecStart=/usr/bin/pnpm paperclipai start
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=paperclip

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable paperclip
sudo systemctl start paperclip
sudo systemctl status paperclip   # should say "active (running)"

# View live logs:
sudo journalctl -u paperclip -f
```

---

## 10. Create Companies and Agents

Log in at `https://paperclip.yourcompany.com`.

### 10.1 Create your 3 companies

For each initiative:
1. Click **New Company** (top of sidebar or company switcher)
2. Give it a name (e.g. "Initiative Alpha")
3. Set a mission statement (this is the top-level goal all agents inherit)

### 10.2 Add agents to each company

Inside each company → **Org Chart** → **Add Agent**:

| Field | What to enter |
|---|---|
| Name | e.g. "Aria" |
| Title | e.g. "CEO", "Engineer", "Marketer" |
| Adapter type | `claude_code`, `codex`, `openclaw`, `http`, `cli` |
| Working directory | e.g. `/paperclip/workspaces/initiative-alpha` |
| Monthly budget | e.g. $50 (enforced hard stop) |
| Environment variables | Reference company secrets: `ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY` |

Repeat for all agents in all 3 companies.

---

## 11. User Access Control

### 11.1 Three user tiers

| Tier | How to set up | What they can do |
|---|---|---|
| Read-only user | Invite without `admin` grant | View company, tasks, agents — no edits |
| Company admin | Invite with `admin` grant | Full company management, can invite others |
| Instance super-admin | Set `instance_admin` flag in Instance Settings | All companies, all settings, all users |

### 11.2 Inviting a new user (any tier)

1. Edit `.env` → `PAPERCLIP_AUTH_DISABLE_SIGN_UP=false`, restart Paperclip
2. In Paperclip UI → Company Settings → **Invites** → **Create Invite**
3. Choose whether to grant `admin` permission (for company admins) or leave it unchecked (for read-only users)
4. Copy the invite link and send it to the user
5. User opens link, registers with email + password
6. User is added to that company with the permission level you set
7. Edit `.env` → `PAPERCLIP_AUTH_DISABLE_SIGN_UP=true`, restart Paperclip

### 11.3 Elevating to instance super-admin

After a user has registered:
1. Go to **Instance Settings** → **Users**
2. Find the user, toggle **Instance Admin: ON**

---

## 12. External PostgreSQL (Optional but Recommended)

For teams > 2 people, use a dedicated PostgreSQL instance for reliability and easier backups.

### 12.1 Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo -u postgres psql -c "CREATE USER paperclip_user WITH PASSWORD 'strong-password-here';"
sudo -u postgres psql -c "CREATE DATABASE paperclip OWNER paperclip_user;"
```

### 12.2 Point Paperclip to it

In `.env`:
```env
DATABASE_URL=postgresql://paperclip_user:strong-password-here@localhost:5432/paperclip
```

Restart Paperclip — it will run migrations automatically on startup.

---

## 13. Backups

### 13.1 Automatic built-in backups

Paperclip has built-in DB backup support. In `.env`:
```env
PAPERCLIP_DB_BACKUP_ENABLED=true
PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES=1440   # daily
PAPERCLIP_DB_BACKUP_RETENTION_DAYS=14
```

Backups are stored in `/opt/paperclip/backups/` by default.

### 13.2 Off-server backups (recommended)

Use a cron job to copy backups to an S3 bucket or remote server:

```bash
# /etc/cron.d/paperclip-backup
0 3 * * * paperclip aws s3 sync /opt/paperclip/backups/ s3://your-backup-bucket/paperclip/
```

---

## 14. Upgrades

```bash
sudo systemctl stop paperclip
sudo -u paperclip bash
cd /opt/paperclip
git pull origin master
pnpm install
pnpm build
exit
sudo systemctl start paperclip
```

Paperclip runs database migrations automatically on startup — no manual migration step needed.

---

## 15. Firewall and Security Hardening

```bash
# Allow only necessary ports
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP (redirect to HTTPS)
sudo ufw allow 443/tcp     # HTTPS
sudo ufw deny 3100/tcp     # Block direct Paperclip port from internet
sudo ufw enable

# Confirm status
sudo ufw status
```

### Additional hardening

```bash
# Disable SSH password auth (use SSH keys only)
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart sshd

# Install fail2ban to block brute-force attempts
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## Quick Reference: Key File Locations

| File | Purpose |
|---|---|
| `/opt/paperclip/.env` | All Paperclip configuration |
| `/etc/nginx/sites-available/paperclip` | Nginx reverse proxy config |
| `/etc/systemd/system/paperclip.service` | Systemd service definition |
| `/opt/paperclip/backups/` | Database backups |
| `~/.paperclip/instances/default/config.json` | Paperclip CLI config (generated) |

---

*Last updated for Paperclip v2026.428.0+*
