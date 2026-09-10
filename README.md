# 🚀 Production-Grade Self-Hosted n8n on Google Cloud Platform (GCP)

A secure, cost-optimized, and fully automated self-hosted [n8n](https://n8n.io/) automation platform tailored for running a private, flexible, and free-forever workflow automation environment on **Google Cloud Platform**.

---

## 📋 Table of Contents

- [Executive Summary & Architecture](#-executive-summary--architecture)
- [GCP Always Free Tier Cost Breakdown ($0 / Month)](#-gcp-always-free-tier-cost-breakdown-0--month)
- [System Architecture](#-system-architecture)
- [Step 1: Create GCP Compute Engine Instance (Always Free)](#step-1-create-gcp-compute-engine-instance-always-free)
- [Step 2: Server Prep, Swap Space & Docker Setup](#step-2-server-prep-swap-space--docker-setup)
- [Step 3: Repository & Environment Configuration](#step-3-repository--environment-configuration)
- [Step 4: Launching n8n with Docker Compose](#step-4-launching-n8n-with-docker-compose)
- [Step 5: Cloudflare Tunnel Setup (Zero Open Ports & Free SSL)](#step-5-cloudflare-tunnel-setup-zero-open-ports--free-ssl)
- [Step 6: Security, Search Engine & AI Cloaking](#step-6-security-search-engine--ai-cloaking)
  - [1. Cloudflare Access (Protecting the Admin UI)](#1-cloudflare-access-protecting-the-admin-ui)
  - [2. Webhook & Health Check Bypass (Two-Application Architecture)](#2-webhook--health-check-bypass-two-application-architecture)
  - [3. Search Engine & AI Crawler Cloaking](#3-search-engine--ai-crawler-cloaking)
  - [4. Enable 2FA in n8n](#4-enable-2fa-in-n8n)
- [Step 7: Third-Party Integrations & Webhooks](#step-7-third-party-integrations--webhooks)
  - [Connecting External Services (OAuth 2.0 & API Keys)](#1-connecting-external-services-oauth-20--api-keys)
  - [Inbound Webhooks](#2-inbound-webhooks)
- [Step 8: Automated Uptime & Downtime Email Alerts](#step-8-automated-uptime--downtime-email-alerts)
  - [Multi-Layered Automatic Reboot & Self-Healing](#-multi-layered-automatic-reboot--self-healing)
- [Step 9: Intrusion Detection & Security Alerts](#step-9-intrusion-detection--security-alerts)
  - [1. Instant SSH Terminal Login Alerts](#1-instant-ssh-terminal-login-alerts)
  - [2. Cloudflare Tunnel Health & Security Monitoring](#2-cloudflare-tunnel-health--security-monitoring)
  - [3. Automated Intrusion Prevention (fail2ban)](#3-automated-intrusion-prevention-fail2ban)
- [Operational Runbook (Backups, Updates, Logs)](#-operational-runbook)

---

## 💡 Executive Summary & Architecture

This self-hosted setup gives you:
- **Unlimited Executions & Workflows**: No subscription tier limits, task-count throttling, or per-execution charges.
- **Unified GCP Cloud Project**: Your compute VM, OAuth credentials, and integrations live under one tidy cloud project.
- **Enterprise-Grade Database**: PostgreSQL 16 (optimized for GCP `e2-micro` with tuned buffers and 2GB swap).
- **Maximum Security**: Zero open firewall ports on your GCP VPC; web UI shielded behind Cloudflare Zero Trust (Email OTP / Google SSO) and Two-Factor Authentication.
- **Search Engine & AI Cloaking**: Hidden from all web crawlers, search engines, and AI scrapers.
- **24/7 Health Monitoring & Auto-Heal**: Instant email alerts when the system goes down and when it recovers, plus automated self-restart if frozen.

---

## 💰 GCP Always Free Tier Cost Breakdown ($0 / Month)

Google Cloud offers an **Always Free** tier that includes enough resources to run this stack 24/7 at no cost:

| Component | GCP Always Free Allowance | Your Allocation | Cost |
| :--- | :--- | :--- | :--- |
| **Compute Instance** | 1 non-preemptible `e2-micro` VM per month in US regions | 1 `e2-micro` (2 vCPUs, 1 GB RAM) | **$0.00** |
| **Storage (Disk)** | 30 GB Standard Persistent Disk per month | 30 GB Standard Persistent Disk (Ubuntu OS) | **$0.00** |
| **Network Egress** | 1 GB egress per month to worldwide destinations | Used for API calls & tunnel traffic | **$0.00** |
| **SSL & Routing** | Cloudflare Free Tier + Cloudflare Tunnel | Handles public HTTPS edge traffic | **$0.00** |
| **Access Protection** | Cloudflare Zero Trust (up to 50 users) | One-Time PIN / Google Login | **$0.00** |
| **Uptime Monitoring** | UptimeRobot Free Tier (50 monitors) | 5-minute interval health checks | **$0.00** |
| **TOTAL** | | | **$0.00 / month** |

> [!IMPORTANT]
> **Eligible Always Free Regions**: You must place your VM in one of these three regions to qualify for $0/month:
> - `us-central1` (Iowa)
> - `us-east1` (South Carolina)
> - `us-west1` (Oregon)

> [!NOTE]
> **Console Price Estimator Notice**: When creating the VM, GCP's estimator will display gross pricing (~`US$6.11/mo`). This is normal—the Always Free credit is applied automatically at billing invoice calculation, reducing the net charge to $0.00.

---

## 🏗️ System Architecture

```text
                                  +---------------------------------------+
                                  |         Internet / Third Parties      |
                                  +---------------------------------------+
                                        |                         |
                           Admin UI Access (Browser)      Inbound Webhooks
                                        |                  (e.g., Stripe, Form)
                                        v                         v
                       +------------------------------------------------------+
                       |                 Cloudflare Edge                      |
                       |  - Free SSL / TLS Certificate                        |
                       |  - Cloudflare Access: Email OTP / Google Login       |
                       |    (App 1: Protect Admin UI | App 2: Bypass Webhook) |
                       |  - X-Robots-Tag: noindex, nofollow (Zero indexing)  |
                       |  - Block AI scrapers and crawlers: Enabled           |
                       +------------------------------------------------------+
                                                  |
                                Encrypted Cloudflare Tunnel (Outbound only!)
                                                  |
                                                  v
                       +------------------------------------------------------+
                       |         GCP Compute Engine (e2-micro, Always Free)   |
                       |         (Zero Inbound Firewall Ports Needed!)        |
                       |                                                      |
                       |  [cloudflared daemon]                                |
                       |            |                                         |
                       |            v (localhost:5678)                        |
                       |  [n8n Container] <====> [PostgreSQL 16 Container]     |
                       |  (max memory: 512MB)    (shared buffers: 64MB)       |
                       |       ^                                              |
                       |  [autoheal container] (Watches /healthz, restarts)   |
                       |            |                    |                    |
                       |      Persistent Data      Persistent DB Data         |
                       |  +------------------------------------------------+  |
                       |  |        2 GB Linux Swap File (/swapfile)         |  |
                       |  +------------------------------------------------+  |
                       +------------------------------------------------------+
                                |                           |
                                v                           v
                     [External APIs & Services]     [SaaS Tools & AI APIs]
```

---

## Step 1: Create GCP Compute Engine Instance (Always Free)

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Create or select a project (e.g. `n8n-hobby-automation`).
3. Navigate to **Compute Engine > VM instances**.
4. Click **Create Instance** and configure:
   - **Name**: `n8n-server`
   - **Region**: Select `us-central1`, `us-east1`, or `us-west1` (Must be one of these for Always Free).
   - **Zone**: Any (e.g. `us-central1-a`).
   - **Machine configuration**:
     - Series: **E2**
     - Machine type: **e2-micro** (2 vCPU, 1 GB memory).
   - **Boot disk**:
     - Click **Change**.
     - Operating system: **Ubuntu**.
     - Version: **Ubuntu 24.04 LTS** (or 22.04 LTS).
     - Boot disk type: **Standard persistent disk** (Do not choose SSD to stay in Always Free).
     - Size: **30 GB** (Maximum free tier allowance).
     - Click **Select**.
   - **Firewall**:
     - Leave **Allow HTTP traffic** and **Allow HTTPS traffic** **UNCHECKED**. (Cloudflare Tunnel makes outbound connections; you do **not** need any inbound firewall ports open!).
5. Click **Create**.

---

## Step 2: Server Prep, Swap Space & Docker Setup

Connect to your instance using the GCP web SSH button in the console (or your local terminal via `gcloud compute ssh n8n-server`).

### 1. Configure 2 GB Swap File (Crucial for 1GB RAM)
An `e2-micro` has 1 GB of physical RAM. Adding a 2 GB swap file prevents Out-Of-Memory (OOM) errors during heavy workflow processing or node executions:

```bash
# Create a 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persist swap across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap is active
free -h
```

### 2. Install Docker & Docker Compose Plugin
Run the standard Docker installation:

```bash
# 1. Update packages
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release git

# 2. Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine and Docker Compose
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker
```

---

## Step 3: Repository & Environment Configuration

Clone this repository on your GCP VM:

```bash
git clone https://github.com/yusuke-iw/n8n-selfhost-setup.git ~/n8n-selfhost-setup
cd ~/n8n-selfhost-setup

# Copy the environment file template
cp .env.example .env
```

Generate a secure 32-character encryption key on the server:

```bash
openssl rand -hex 24
```

Edit your `.env` file (`nano .env`):

```bash
nano .env
```

Set the values:
- `DOMAIN_NAME`: e.g. `n8n.yourdomain.com`
- `GENERIC_TIMEZONE`: your local timezone (e.g., `Asia/Tokyo`, `America/New_York`, or `UTC`)
- `WEBHOOK_URL`: `https://n8n.yourdomain.com/`
- `N8N_EDITOR_BASE_URL`: `https://n8n.yourdomain.com/`
- `N8N_ENCRYPTION_KEY`: paste the generated random key.
- `POSTGRES_USER`: `n8n_admin`
- `POSTGRES_PASSWORD`: a strong password (generate using `openssl rand -hex 16`)
- `POSTGRES_DB`: `n8n_db`

> [!TIP]
> **Password Best Practice**: Use alphanumeric passwords generated via `openssl rand -hex 16`. Avoid using `$` in passwords inside `.env`, as Docker Compose interprets `$word` as an environment variable interpolation (which blanks out that part of the password).

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

---

## Step 4: Launching n8n with Docker Compose

Start the services in the background:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

You will see:
- `n8n_postgres`: Status `Up (healthy)` (tuned memory: 64MB shared buffers)
- `n8n_app`: Status `Up (healthy)` (bound to `127.0.0.1:5678` with 512MB Node heap limit)
- `n8n_autoheal`: Status `Up` (monitors `/healthz` and auto-restarts if frozen)

To view live startup logs:

```bash
docker compose logs -f n8n
```

---

## Step 5: Cloudflare Tunnel Setup (Zero Open Ports & Free SSL)

Instead of managing Let's Encrypt or opening GCP firewall ingress rules, Cloudflare Tunnel securely bridges your custom domain directly to `127.0.0.1:5678`.

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/).
2. Navigate to **Networks > Tunnels** and click **Create a tunnel**.
3. Select **Cloudflared** and name it `gcp-n8n-tunnel`.
4. Choose operating system:
   - Select: **`Debian`** *(Ubuntu is Debian-based and 100% compatible)*.
   - Architecture: **`64-bit`**.
5. Cloudflare will display installation commands. Copy and execute them in your VM SSH terminal:
   - **Step A (Install package)**: Run the script to add the Cloudflare repository and install `cloudflared`.
   - **Step B (Connect tunnel)**: Run the command containing your unique token:
     ```bash
     sudo cloudflared service install <YOUR_TOKEN>
     ```
6. Once connected (green checkmark in dashboard), click **Next** to configure the **Public Hostname**:
   - **Subdomain**: `n8n`
   - **Domain**: `yourdomain.com`
   - **Type**: `HTTP`
   - **URL**: `localhost:5678`
7. Click **Save tunnel**.

Your n8n instance is now live securely at `https://n8n.yourdomain.com` with zero open ports on GCP!

---

## Step 6: Security, Search Engine & AI Cloaking

To ensure that only you can access the admin UI while allowing webhooks to function and completely blocking search engines and AI bots:

### 1. Cloudflare Access (Protecting the Admin UI)
In [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/):
1. Navigate to **Access > Applications > Add an application**.
2. Select **Self-hosted**.
3. **Application Name**: `n8n Workspace (Admin)`.
4. **Application Domain**:
   - Subdomain: `n8n`
   - Domain: `yourdomain.com`
   - Path: *(Leave empty)*
5. Under **Policies**:
   - **Policy Name**: `Allow Owner Only`.
   - **Action**: `Allow`.
   - **Include**: Rule `Emails` -> enter your personal email address.
6. Click **Save application**.
*(Now, anyone visiting `https://n8n.yourdomain.com/` is challenged with a one-time login PIN sent to your email).*

### 2. Webhook & Health Check Bypass (Two-Application Architecture)
Because Cloudflare evaluates applications from most specific to least specific, we create a second Access application specifically to allow inbound webhooks and uptime checks:
1. In **Access > Applications**, click **Add an application** (Self-hosted).
2. **Application Name**: `n8n Webhook & Healthz Bypass`.
3. **Application Domain**:
   - Subdomain: `n8n`
   - Domain: `yourdomain.com`
   - **Path**: `webhook*`
4. Under **Policies**:
   - **Policy Name**: `Bypass Webhooks`.
   - **Action**: `Bypass`.
   - **Include**: Select **`Everyone`**.
5. Click **Save application**.
6. *(Optional)*: Create an identical Bypass application with Path: `healthz` for UptimeRobot monitoring.

### 3. Search Engine & AI Crawler Cloaking
1. **Search Engine Cloaking (`X-Robots-Tag`)**:
   - In Cloudflare Dashboard, go to **Rules > Overview > Transform Rules (Modify Response Header)**:
   - Click **Create rule**.
   - Rule name: `Block Search Indexing`
   - When incoming requests match: `Hostname equals n8n.yourdomain.com`
   - Then: Set static header `X-Robots-Tag` = `noindex, nofollow, noarchive, nosnippet`
   - Click **Deploy**.
2. **Block AI Scrapers and Crawlers (1-Click)**:
   - In Cloudflare Dashboard, navigate to **Security > Bots**.
   - Toggle **"Block AI scrapers and crawlers"** to **ON**.
   - This automatically blocks OpenAI (GPTBot), Anthropic (ClaudeBot), Perplexity, Bytespider, and all known AI training bots at the edge.

### 4. Enable 2FA in n8n
1. Open `https://n8n.yourdomain.com` and log in via your email PIN.
2. Complete initial registration to create your n8n Admin account.
3. Navigate to **Settings > Personal > Security** and turn on **Two-Factor Authentication (2FA)** using your authenticator app (Google Authenticator, 1Password, Bitwarden).

---

## Step 7: Third-Party Integrations & Webhooks

### 1. Connecting External Services (OAuth 2.0 & API Keys)
In n8n, credentials for external services (APIs, SaaS tools, cloud storage, databases) are encrypted and stored in your PostgreSQL database:

#### For OAuth 2.0 Integrations:
1. In the target service's developer portal, create an OAuth app.
2. Set the **Authorized Redirect URI** (Callback URL) to:
   ```text
   https://n8n.yourdomain.com/rest/oauth2-credential/callback
   ```
3. Copy the **Client ID** and **Client Secret** into the corresponding n8n Credential modal and click **Connect**.

#### For API Key Integrations:
1. Generate an API Key / Bearer Token from your external provider.
2. In n8n, add the relevant node, select **Create New Credential**, and paste your API key.

### 2. Inbound Webhooks
1. In any n8n workflow, drag a **Webhook** trigger node onto the canvas.
2. Select your HTTP Method (`GET`, `POST`, etc.).
3. n8n exposes two URLs:
   - **Test URL**: Used during workflow development (`https://n8n.yourdomain.com/webhook-test/...`).
   - **Production URL**: Used when the workflow is activated (`https://n8n.yourdomain.com/webhook/...`).
4. Thanks to the Cloudflare Bypass rule configured in Step 6, external services can trigger these endpoints directly without encountering an authentication prompt.

---

## Step 8: Automated Uptime & Downtime Email Alerts

Get instant email alerts if n8n ever goes down or comes back up:

1. Sign up for free at [UptimeRobot](https://uptimerobot.com/).
2. Click **Add New Monitor**:
   - **Monitor Type**: `HTTP(s)`
   - **Friendly Name**: `n8n GCP Instance`
   - **URL**: `https://n8n.yourdomain.com/healthz`
   - **Monitoring Interval**: `5 minutes`
   - **Alert Contacts**: Select your email.
3. Click **Create Monitor**.

### Alert behavior:
- When n8n is running, `/healthz` returns HTTP `200 OK`.
- If the VM restarts or n8n crashes, `/healthz` returns an error or timeout.
- You receive an immediate email:
  - 🔴 **DOWN Alert**: *"Monitor is DOWN: n8n GCP Instance"*
  - 🟢 **UP Alert**: *"Monitor is UP: n8n GCP Instance (Recovery detected)"*

### 🔄 Multi-Layered Automatic Reboot & Self-Healing

The stack is architected with three levels of automated recovery so it heals itself without manual intervention:

1. **Process Crash Recovery (`restart: unless-stopped`)**:
   - If n8n or PostgreSQL encounters an unhandled exception or crashes, the Docker daemon automatically restarts the container immediately.
2. **Hung / Frozen State Recovery (`autoheal`)**:
   - A built-in container health check pings `http://127.0.0.1:5678/healthz` every 30 seconds.
   - If n8n locks up or freezes silently (failing 3 consecutive health checks), the included `autoheal` companion container automatically force-restarts `n8n_app`.
3. **Host VM Reboot Recovery**:
   - GCP Compute Engine features **Automatic Restart** on host maintenance.
   - When the VM boots back up, the Ubuntu `docker` service and `cloudflared` systemd daemon start automatically, restoring all containers and the encrypted tunnel with zero manual commands.

---

## Step 9: Intrusion Detection & Security Alerts

Because Cloudflare Tunnel keeps all web ports closed, attackers cannot scan or access n8n directly from the internet. However, to stay alerted in real-time against unauthorized logins, configure these automated alert channels:

### 1. Instant SSH Terminal Login Alerts
If anyone (including you or an attacker) successfully accesses the GCP VM via SSH, receive an immediate notification via Email, Discord, or Telegram.

#### Option A: Email Alerts via Python (Zero Dependencies)
Ubuntu includes Python 3 by default. You can send email alerts directly to your inbox using a Gmail App Password:

1. Create a script `/usr/local/bin/send-alert-email.py`:
   ```bash
   sudo tee /usr/local/bin/send-alert-email.py > /dev/null << 'EOF'
   import sys, smtplib
   from email.message import EmailMessage

   GMAIL_USER = "your_email@gmail.com"
   GMAIL_APP_PASS = "abcdefghijklmnop"  # 16-character Google App Password (no spaces)
   TO_EMAIL = "your_email@gmail.com"

   subject = sys.argv[1] if len(sys.argv) > 1 else "Security Alert"
   body = sys.argv[2] if len(sys.argv) > 2 else "SSH login detected."

   msg = EmailMessage()
   msg['Subject'] = subject
   msg['From'] = f"Server Monitor <{GMAIL_USER}>"
   msg['To'] = TO_EMAIL
   msg.set_content(body)

   try:
       with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp:
           smtp.login(GMAIL_USER, GMAIL_APP_PASS)
           smtp.send_message(msg)
   except Exception:
       pass
   EOF
   sudo chmod +x /usr/local/bin/send-alert-email.py
   ```
2. Trigger it on SSH login in `/etc/profile.d/ssh-login-alert.sh`:
   ```bash
   sudo tee /etc/profile.d/ssh-login-alert.sh > /dev/null << 'EOF'
   #!/bin/bash
   if [ -n "$SSH_CLIENT" ]; then
       CLIENT_IP=$(echo $SSH_CLIENT | awk '{print $1}')
       HOSTNAME=$(hostname)
       TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S %Z')
       SUBJECT="🚨 [Security Alert] SSH Login on ${HOSTNAME}"
       BODY="SSH login detected!\n\nUser: ${USER}\nIP: ${CLIENT_IP}\nTimestamp: ${TIMESTAMP}"
       python3 /usr/local/bin/send-alert-email.py "$SUBJECT" "$BODY" > /dev/null 2>&1 &
   fi
   EOF
   sudo chmod +x /etc/profile.d/ssh-login-alert.sh
   ```

#### Option B: Discord Webhook Alert
If you prefer Discord notifications:
```bash
sudo tee /etc/profile.d/ssh-login-alert.sh > /dev/null << 'EOF'
#!/bin/bash
if [ -n "$SSH_CLIENT" ]; then
    CLIENT_IP=$(echo $SSH_CLIENT | awk '{print $1}')
    HOSTNAME=$(hostname)
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S %Z')
    MESSAGE="🚨 Security Alert: SSH login on ${HOSTNAME} by user '${USER}' from IP: ${CLIENT_IP} at ${TIMESTAMP}"
    curl -s -H "Content-Type: application/json" -X POST -d "{\"content\":\"$MESSAGE\"}" "YOUR_DISCORD_WEBHOOK_URL" > /dev/null 2>&1 &
fi
EOF
sudo chmod +x /etc/profile.d/ssh-login-alert.sh
```

---

### 2. Cloudflare Tunnel Health & Security Monitoring
1. **Tunnel Health Alert (Free)**:
   - In Cloudflare Dashboard, go to **Notifications > Add**.
   - Filter Product by **Tunnel** and select **Tunnel Health Alert**.
   - Receive an immediate email if your VM tunnel drops, goes offline, or disconnects.
2. **Security & WAF Analytics (Free)**:
   - Under your domain, navigate to **Security > Events**.
   - Cloudflare logs every blocked request, crawler rejection, and Access challenge in real-time.

---

### 3. Automated Intrusion Prevention (`fail2ban`)
Install `fail2ban` on your Ubuntu VM to automatically detect repeated failed SSH attempts, block the attacker's IP at the Linux firewall level, and log intrusion attempts:

```bash
# Install and activate fail2ban
sudo apt-get install -y fail2ban
sudo systemctl enable --now fail2ban

# Check banned IPs anytime
sudo fail2ban-client status sshd
```

---

## 🛠️ Operational Runbook

### Check Logs
```bash
cd ~/n8n-selfhost-setup
docker compose logs -f n8n
```

### Restart n8n
```bash
docker compose restart n8n
```

### One-Command Upgrade to Latest n8n
```bash
cd ~/n8n-selfhost-setup
docker compose pull
docker compose up -d
```

### Automated Database Backup
Create a snapshot of your PostgreSQL database:

```bash
docker exec -t n8n_postgres pg_dump -U n8n_admin n8n_db > ~/n8n_backup_$(date +%Y%m%d_%H%M%S).sql
```

Add a daily automated backup cron job (`crontab -e`):
```bash
0 3 * * * docker exec -t n8n_postgres pg_dump -U n8n_admin n8n_db | gzip > ~/n8n_backup_$(date +\%Y\%m\%d).sql.gz
```

---

## 📁 Repository Structure

```text
n8n-selfhost-setup/
├── .env.example          # Environment template with explanations
├── .gitignore            # Protects credentials and local storage
├── docker-compose.yml    # n8n + PostgreSQL 16 (GCP e2-micro optimized + autoheal)
└── README.md             # Complete step-by-step setup guide for GCP
```

---

## 📄 License
MIT License. Feel free to use and customize for your personal projects.
