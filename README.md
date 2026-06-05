# ICT171 Cloud Server Project

**Student Name:** Subha Islam 
**Student Number:** 35854246
**GitHub Repository:** [https://github.com/suuiisparkle/ICT171_cloud_server_project](https://github.com/suuiisparkle/ICT171_cloud_server_project)  
**Server IP:** 20.213.11.20  
**Domain:** suuiisparkle.com  
**Video Explainer:** [YouTube / OneDrive link to your recorded walkthrough]

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Phase 1 — Azure VM Setup](#phase-1--azure-vm-setup)
4. [Phase 2 — DNS Configuration (GoDaddy)](#phase-2--dns-configuration-godaddy)
5. [Phase 3 — Nginx Reverse Proxy and SSL](#phase-3--nginx-reverse-proxy-and-ssl)
6. [Phase 4 — Main Website](#phase-4--main-website)
7. [Phase 5 — Wiki.js Knowledge Base](#phase-5--wikijs-knowledge-base)
8. [Phase 6 — FileBrowser File Storage](#phase-6--filebrowser-file-storage)
9. [Phase 7 — Static Status Page](#phase-7--static-status-page)
10. [Phase 8 — WireGuard VPN](#phase-8--wireguard-vpn)
11. [Phase 9 — Automation Script](#phase-9--automation-script)
12. [Problems Encountered and Solutions](#problems-encountered-and-solutions)
13. [Troubleshooting Reference](#troubleshooting-reference)
14. [References](#references)

---

## Project Overview

This project builds a multi-purpose Ubuntu 24.04 cloud server hosted on Microsoft Azure. One virtual machine (1 GB RAM, 2 vCPU) runs all of the following services simultaneously:

| Service | Purpose | URL |
|---|---|---|
| Static website | Personal landing page linking all services | `https://suuiisparkle.com` |
| Wiki.js | K-pop knowledge base and wiki | `https://wiki.suuiisparkle.com` |
| FileBrowser | Personal cloud file storage and sharing | `https://cloud.suuiisparkle.com` |
| Status page | Live service monitoring dashboard | `https://status.suuiisparkle.com` |
| WireGuard VPN | Encrypted private remote access | `vpn.suuiisparkle.com:51820` |
| PostgreSQL | Database backend for Wiki.js | Internal only |

All web services are reverse-proxied through Nginx with HTTPS via Let's Encrypt certificates. DNS is managed through GoDaddy.

> **Note on VM size:** The Azure for Students subscription assigned a Standard_B2ats v2 VM with only 1 GB RAM. A 2 GB swap file was added to prevent out-of-memory crashes. Nextcloud was replaced with FileBrowser (~20 MB RAM) because Nextcloud requires significantly more memory. Uptime Kuma was replaced with a lightweight static status page for the same reason. See [Problems Encountered](#problems-encountered-and-solutions) for full details.

---

## Architecture

```
Your Browser / VPN Client
         |
         ▼
   GoDaddy DNS
   A records → 20.213.11.20
         |
         ▼
   Azure Public IP: 20.213.11.20 (static)
   Network Security Group
   Ports: 22 TCP, 80 TCP, 443 TCP, 51820 UDP
         |
         ▼
   Ubuntu 24.04 VM (1 GB RAM + 2 GB swap)
         |
    ┌────┴────┐
    ▼         ▼
  Nginx     WireGuard
  :80/:443   UDP :51820
    |
    ├──► /var/www/main        (static website)
    ├──► /var/www/status      (static status page)
    ├──► localhost:3000        (Wiki.js)
    └──► localhost:8080        (FileBrowser)
              |
              ▼
         PostgreSQL :5432
         (wikijs database)
```

**Subdomain routing:**

| Subdomain | Service | How served |
|---|---|---|
| `suuiisparkle.com` | Main website | Static files via Nginx |
| `wiki.suuiisparkle.com` | Wiki.js | Proxied to port 3000 |
| `cloud.suuiisparkle.com` | FileBrowser | Proxied to port 8080 |
| `status.suuiisparkle.com` | Status page | Static files via Nginx |
| `vpn.suuiisparkle.com` | WireGuard | UDP port 51820 (bypasses Nginx) |

**Why Nginx as a reverse proxy?**
One server with one IP can host multiple websites. Nginx reads the subdomain in every HTTPS request and forwards it to the correct internal application. Internal apps are never directly exposed to the internet.

---

## Phase 1 — Azure VM Setup

### 1.1 Create the Virtual Machine

1. Go to [portal.azure.com](https://portal.azure.com) and sign in with your university Azure for Students account.
2. Search for **Virtual Machines** in the top bar and click **Create → Azure Virtual Machine**.
3. Fill in the settings:

| Setting | Value | Why |
|---|---|---|
| Subscription | Azure for Students | Student subscription |
| Resource group | `ict171-rg` → **Create new** | Groups all project resources for easy deletion later |
| VM name | `ict171-server` | Descriptive name |
| Region | **(Asia Pacific) Australia East** | Lowest latency from Perth |
| Availability options | No infrastructure redundancy | Not needed for a student project |
| Image | **Ubuntu Server 24.04 LTS — x64 Gen2** | Required OS |
| Size | **Standard_B2s** (2 vCPU, 4 GB RAM) recommended | See note below |
| Authentication type | **SSH public key** | More secure than password |
| Username | `azureuser` | Default admin username |
| SSH public key source | **Generate new key pair** | Azure generates it for you |
| Key pair name | `ict171-server-key` | |
| Public inbound ports | **Allow selected → SSH (22)** | Other ports added in step 1.2 |
| OS disk type | **Standard SSD** | |
| OS disk size | **30 GB** | Enough for all services |

> **Important — VM size:** If Azure assigns you a Standard_B2ats v2 (1 GB RAM), add a swap file in step 1.6 before installing anything. Without swap, npm and other package managers will exhaust RAM and hang the server.

4. Click **Review + Create** → **Create**.
5. When prompted **"Download private key and create resource"**, click it immediately. Save the `.pem` file — you cannot download it again.

### 1.2 Configure the Network Security Group

The NSG controls which ports the internet can reach. By default only SSH (22) is open.

1. Open your VM in Azure Portal → click **Networking** in the left sidebar.
2. Click **Add inbound port rule** for each row:

| Priority | Name | Port | Protocol | Action | Purpose |
|---|---|---|---|---|---|
| 100 | Allow-SSH | 22 | TCP | Allow | Remote terminal access |
| 110 | Allow-HTTP | 80 | TCP | Allow | Let's Encrypt certificate verification |
| 120 | Allow-HTTPS | 443 | TCP | Allow | All web traffic |
| 130 | Allow-WireGuard | 51820 | UDP | Allow | VPN connections |

### 1.3 Assign a Static Public IP

By default Azure gives a dynamic IP that changes on restart — that breaks all DNS records.

1. Open your VM → click **Overview**.
2. Click the **Public IP address** blue link.
3. In the left sidebar click **Configuration**.
4. Change **Assignment** from Dynamic to **Static** → click **Save**.
5. Note this IP — this project uses `20.213.11.20`.

### 1.4 Connect via SSH

**On macOS / Linux:**
```bash
chmod 400 ~/Downloads/ict171-server-key.pem
ssh -i ~/Downloads/ict171-server-key.pem azureuser@20.213.11.20
```

**On Windows (PowerShell):**
```powershell
icacls "$env:USERPROFILE\Downloads\ict171-server-key.pem" /inheritance:r /grant:r "$env:USERNAME:R"
ssh -i "$env:USERPROFILE\Downloads\ict171-server-key.pem" azureuser@20.213.11.20
```

> **Paste tip:** In CMD, Ctrl+V does not work by default. Use **right-click** to paste, or switch to Windows Terminal which supports Ctrl+V natively. Run every command one at a time — never paste multiple lines at once as they merge together.

### 1.5 Initial System Configuration

**Update all packages:**
```bash
sudo apt update && sudo apt upgrade -y
```

**Install essential tools:**
```bash
sudo apt install -y curl wget git ufw unzip software-properties-common apt-transport-https ca-certificates gnupg lsb-release
```

**Configure UFW firewall — run one line at a time:**
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 51820/udp
sudo ufw enable
sudo ufw status
```

When `sudo ufw enable` asks `Proceed with operation (y|n)?` type `y`.

Expected output of `sudo ufw status`:
```
Status: active
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
51820/udp                  ALLOW       Anywhere
```

**Set timezone to Perth:**
```bash
sudo timedatectl set-timezone Australia/Perth
```

### 1.6 Add a Swap File (Required for 1 GB RAM VMs)

Add 2 GB of swap before installing anything else. Without this, npm and other installers will exhaust RAM and hang.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify:
```bash
free -h
```

The Swap row should show `2.0G` total. The system now has effectively 3 GB of memory available.

---

## Phase 2 — DNS Configuration (GoDaddy)

DNS translates domain names into your server's IP address. We need one A record per subdomain, all pointing to the same Azure IP.

### 2.1 Log In to GoDaddy DNS

1. Go to [godaddy.com](https://godaddy.com) and sign in.
2. Click your account name (top right) → **My Products**.
3. Find your domain → click **DNS**.
4. You will see a table of existing records — add to it.

### 2.2 Add the A Records

Click **Add New Record** for each row below. Select Type **A**, fill in Name and Value:

| Type | Name | Value | TTL |
|---|---|---|---|
| A | `@` | `20.213.11.20` | 600 seconds |
| A | `www` | `20.213.11.20` | 600 seconds |
| A | `wiki` | `20.213.11.20` | 600 seconds |
| A | `cloud` | `20.213.11.20` | 600 seconds |
| A | `status` | `20.213.11.20` | 600 seconds |
| A | `vpn` | `20.213.11.20` | 600 seconds |

**What each record does:**
- `@` — root domain (`suuiisparkle.com`)
- `www` — `www.suuiisparkle.com`
- `wiki` — `wiki.suuiisparkle.com` → Wiki.js
- `cloud` — `cloud.suuiisparkle.com` → FileBrowser
- `status` — `status.suuiisparkle.com` → Status page
- `vpn` — `vpn.suuiisparkle.com` → WireGuard

> **Why TTL 600?** TTL (Time To Live) tells DNS resolvers how long to cache the record in seconds. 600 = 10 minutes. A low TTL means changes propagate quickly during setup.

> **WireGuard note:** WireGuard works with a raw IP. The `vpn` record is a convenience so clients can use `vpn.suuiisparkle.com:51820` instead of remembering the IP number.

### 2.3 Verify DNS Propagation

Run these from your Windows machine in CMD or PowerShell — no SSH needed:

```
nslookup suuiisparkle.com
nslookup wiki.suuiisparkle.com
nslookup cloud.suuiisparkle.com
nslookup status.suuiisparkle.com
```

Each should return `Address: 20.213.11.20`. If it returns nothing, wait 5 minutes and try again. GoDaddy typically propagates within 10 minutes.

> **Do not proceed to Phase 3 until all four return your IP.** Certbot verifies domain ownership over HTTP — it will fail if DNS is not propagated.

---

## Phase 3 — Nginx Reverse Proxy and SSL

### 3.1 Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

Test: visit `http://20.213.11.20` in your browser — you should see the default Nginx welcome page.

### 3.2 Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 3.3 Obtain SSL Certificates

DNS must be propagated before this step. Run once for all subdomains:

```bash
sudo certbot --nginx \
  -d suuiisparkle.com \
  -d www.suuiisparkle.com \
  -d wiki.suuiisparkle.com \
  -d cloud.suuiisparkle.com \
  -d status.suuiisparkle.com \
  --email your@email.com \
  --agree-tos \
  --no-eff-email
```

Certbot contacts Let's Encrypt, verifies each domain, issues the certificate and saves it to `/etc/letsencrypt/live/suuiisparkle.com/`.

Verify auto-renewal works (certificates expire every 90 days):
```bash
sudo certbot renew --dry-run
```

Expected: `Congratulations, all simulated renewals succeeded`.

### 3.4 Create Nginx Virtual Host Configs

Remove the default placeholder:
```bash
sudo rm /etc/nginx/sites-enabled/default
```

**Create each config file using nano. Run `sudo nano [filename]`, paste the content, save with Ctrl+X → Y → Enter.**

---

**Main website** — `/etc/nginx/sites-available/main`:

```nginx
server {
    listen 80;
    server_name suuiisparkle.com www.suuiisparkle.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name suuiisparkle.com www.suuiisparkle.com;

    ssl_certificate     /etc/letsencrypt/live/suuiisparkle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/suuiisparkle.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;

    root  /var/www/main;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
        error_page 404 /404.html;
    }
    location = /404.html { internal; }
}
```

---

**Wiki.js** — `/etc/nginx/sites-available/wiki`:

```nginx
server {
    listen 80;
    server_name wiki.suuiisparkle.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name wiki.suuiisparkle.com;

    ssl_certificate     /etc/letsencrypt/live/suuiisparkle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/suuiisparkle.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

---

**FileBrowser** — `/etc/nginx/sites-available/cloud`:

```nginx
server {
    listen 80;
    server_name cloud.suuiisparkle.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cloud.suuiisparkle.com;

    ssl_certificate     /etc/letsencrypt/live/suuiisparkle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/suuiisparkle.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;

    client_max_body_size 512M;

    location / {
        proxy_pass         http://localhost:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600;
    }
}
```

---

**Status page** — `/etc/nginx/sites-available/status`:

```nginx
server {
    listen 80;
    server_name status.suuiisparkle.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name status.suuiisparkle.com;

    ssl_certificate     /etc/letsencrypt/live/suuiisparkle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/suuiisparkle.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;

    root  /var/www/status;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

**IP redirect** — `/etc/nginx/sites-available/default-ip`:

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;

    ssl_certificate     /etc/letsencrypt/live/suuiisparkle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/suuiisparkle.com/privkey.pem;

    return 301 https://suuiisparkle.com$request_uri;
}
```

This redirects anyone visiting the raw IP `20.213.11.20` to `suuiisparkle.com` automatically.

### 3.5 Enable All Sites and Reload Nginx

```bash
sudo ln -s /etc/nginx/sites-available/main       /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/wiki       /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/cloud      /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/status     /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/default-ip /etc/nginx/sites-enabled/
```

Always test before reloading — a syntax error will take down all websites:
```bash
sudo nginx -t
```

Must say `syntax is ok`. Then reload:
```bash
sudo systemctl reload nginx
```

> **Common mistake:** If any config file still contains `yourdomain.com` the test will fail with a certificate error. Fix all four files at once:
> ```bash
> sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/main
> sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/wiki
> sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/cloud
> sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/status
> ```
> Verify nothing remains: `grep -r "yourdomain" /etc/nginx/sites-available/`

---

## Phase 4 — Main Website

### 4.1 Create the Web Root

```bash
sudo mkdir -p /var/www/main
sudo chown -R azureuser:www-data /var/www/main
sudo chmod -R 755 /var/www/main
```

### 4.2 Create index.html

```bash
nano /var/www/main/index.html
```

Paste:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Suuiisparkle — ICT171 Server</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'Segoe UI', sans-serif;
      background: #0f172a; color: #e2e8f0;
      min-height: 100vh; display: flex;
      flex-direction: column; align-items: center;
      justify-content: center; padding: 40px 20px;
    }
    h1  { font-size: 2.5rem; margin-bottom: 8px; }
    .sub { color: #94a3b8; margin-bottom: 48px; font-size: 1.1rem; }
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 20px; width: 100%; max-width: 900px;
    }
    .card {
      background: #1e293b; border: 1px solid #334155;
      border-radius: 12px; padding: 24px;
      text-decoration: none; color: inherit;
      transition: border-color 0.2s, transform 0.2s;
    }
    .card:hover { border-color: #3b82f6; transform: translateY(-2px); }
    .card h3 { font-size: 1.1rem; margin-bottom: 6px; }
    .card p  { color: #94a3b8; font-size: 0.9rem; }
    footer   { margin-top: 60px; color: #475569; font-size: 0.8rem; }
  </style>
</head>
<body>
  <h1>Suuiisparkle</h1>
  <p class="sub">ICT171 Cloud Server Project — Murdoch University</p>
  <div class="grid">
    <a class="card" href="https://wiki.suuiisparkle.com">
      <h3>📖 Wiki</h3>
      <p>K-pop knowledge base powered by Wiki.js</p>
    </a>
    <a class="card" href="https://cloud.suuiisparkle.com">
      <h3>📁 File Storage</h3>
      <p>Personal files via FileBrowser</p>
    </a>
    <a class="card" href="https://status.suuiisparkle.com">
      <h3>📡 Status</h3>
      <p>Live service monitoring</p>
    </a>
    <div class="card">
      <h3>🔒 VPN</h3>
      <p>Private access via WireGuard</p>
    </div>
  </div>
  <footer>Hosted on Microsoft Azure · Ubuntu 24.04 · Nginx</footer>
</body>
</html>
```

### 4.3 Create a 404 Page

```bash
nano /var/www/main/404.html
```

Paste:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>404 — Not Found</title>
  <style>
    body {
      font-family: 'Segoe UI', sans-serif; background: #0f172a;
      color: #e2e8f0; display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      height: 100vh; margin: 0;
    }
    h1 { font-size: 6rem; color: #3b82f6; margin-bottom: 0; }
    p  { color: #94a3b8; margin: 12px 0 32px; }
    a  { color: #3b82f6; text-decoration: none; }
  </style>
</head>
<body>
  <h1>404</h1>
  <p>That page doesn't exist.</p>
  <a href="/">← Back to home</a>
</body>
</html>
```

### 4.4 Verify

Visit `https://suuiisparkle.com` — landing page with HTTPS padlock should appear. Visit `https://suuiisparkle.com/nonexistent` — the 404 page should appear.

---

## Phase 5 — Wiki.js Knowledge Base

### 5.1 Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version
```

Must print `v20.x.x` — Wiki.js requires Node 18+.

### 5.2 Install PostgreSQL

PostgreSQL is the database for Wiki.js.

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```

**Create the Wiki.js database:**

```bash
sudo -u postgres psql << 'EOF'
CREATE USER wikijs WITH PASSWORD 'your_wikijs_password';
CREATE DATABASE wikijs OWNER wikijs;
GRANT ALL PRIVILEGES ON DATABASE wikijs TO wikijs;
EOF
```

Verify:
```bash
sudo -u postgres psql -c "\l"
```

`wikijs` should appear in the list.

**Change the password immediately** — replace `your_wikijs_password` with something strong. Use the same password in config.yml below.

To change an existing password:
```bash
sudo -u postgres psql -c "ALTER USER wikijs WITH PASSWORD 'yournewpassword';"
```

### 5.3 Download Wiki.js

```bash
sudo mkdir -p /opt/wikijs
sudo chown azureuser:azureuser /opt/wikijs
cd /opt/wikijs
wget https://github.com/requarks/wiki/releases/latest/download/wiki-js.tar.gz
tar xzf wiki-js.tar.gz
rm wiki-js.tar.gz
```

### 5.4 Create the Config File

```bash
nano /opt/wikijs/config.yml
```

Paste:

```yaml
db:
  type: postgres
  host: localhost
  port: 5432
  user: wikijs
  pass: your_wikijs_password
  db: wikijs
  ssl: false

bindIP: 127.0.0.1
port: 3000
logLevel: info
logFormat: default
```

`bindIP: 127.0.0.1` means Wiki.js only listens locally — all public traffic goes through Nginx.

### 5.5 Create a systemd Service

```bash
sudo nano /etc/systemd/system/wikijs.service
```

Paste:

```ini
[Unit]
Description=Wiki.js
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
ExecStart=/usr/bin/node server
Restart=always
RestartSec=5
User=azureuser
WorkingDirectory=/opt/wikijs
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wikijs
sudo systemctl start wikijs
sudo systemctl status wikijs
```

If it fails check logs: `sudo journalctl -u wikijs -n 50`

### 5.6 Complete Web Setup

Wait 30–60 seconds then visit `https://wiki.suuiisparkle.com`. Complete the setup wizard and create an admin account.

### 5.7 Create the Wiki Home Page

In Wiki.js, edit the home page and paste this Markdown:

```markdown
# 🎵 K-pop Chronicles Wiki

> Your ultimate encyclopedia for K-pop groups, discographies, fandoms, and history.

---

## 📂 Browse by Category

- 👥 **[Groups](/groups)** — Idol groups by generation and agency
- 🎤 **[Solo Artists](/solos)** — Soloists and solo debuts
- 💿 **[Discographies](/discographies)** — Albums, EPs, and singles
- 💗 **[Fandoms](/fandoms)** — Fandom names, colors, and culture
- 📅 **[History](/history)** — 1st gen through 5th gen timeline
- 🏢 **[Agencies](/agencies)** — HYBE, SM, JYP, YG and more

---

## 🕐 Recently Updated

| Artist | Generation | Agency | Debut |
|---|---|---|---|
| BTS | 3rd gen | HYBE | 2013 |
| BLACKPINK | 3rd gen | YG | 2016 |
| TWICE | 3rd gen | JYP | 2015 |
| aespa | 4th gen | SM | 2020 |
| Stray Kids | 4th gen | JYP | 2018 |
| NewJeans | 4th gen | ADOR | 2022 |

---

*Built with ❤️ by Suuiisparkle · Hosted on suuiisparkle.com*
```

---

## Phase 6 — FileBrowser File Storage

FileBrowser is a lightweight, open-source file manager with a web UI. It uses approximately 20 MB of RAM — far less than Nextcloud — making it ideal for a 1 GB VM. It supports file upload, download, folder management, and public share links.

### 6.1 Install FileBrowser

FileBrowser is a single binary — no database, no PHP, no Apache needed.

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
```

Verify the install:
```bash
filebrowser version
```

### 6.2 Create the FileBrowser Directory

```bash
sudo mkdir -p /var/www/files
sudo chown azureuser:azureuser /var/www/files
```

This is the root directory where all uploaded files will be stored.

### 6.3 Configure FileBrowser

```bash
sudo mkdir -p /etc/filebrowser
```

Create the config file:
```bash
sudo nano /etc/filebrowser/filebrowser.json
```

Paste:

```json
{
  "port": 8080,
  "baseURL": "",
  "address": "127.0.0.1",
  "log": "stdout",
  "database": "/etc/filebrowser/filebrowser.db",
  "root": "/var/www/files"
}
```

`address: 127.0.0.1` means FileBrowser only listens locally — all public traffic goes through Nginx.

### 6.4 Create a systemd Service

```bash
sudo nano /etc/systemd/system/filebrowser.service
```

Paste:

```ini
[Unit]
Description=FileBrowser
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/filebrowser -c /etc/filebrowser/filebrowser.json
Restart=always
RestartSec=5
User=azureuser

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable filebrowser
sudo systemctl start filebrowser
sudo systemctl status filebrowser
```

### 6.5 Verify

Visit `https://cloud.suuiisparkle.com` — you should see the FileBrowser login page.

Default credentials:
- Username: `admin`
- Password: `admin`

**Change the password immediately** after first login — click the top right avatar → Settings → change password.

### 6.6 Enable File Sharing

1. Log in to FileBrowser.
2. Right-click any file → **Share**.
3. Set an expiry if desired → click **Copy link**.
4. Share the link with anyone — no account needed to download.

---

## Phase 7 — Static Status Page

> **Why not Uptime Kuma?** Uptime Kuma requires Node.js and ~150 MB RAM. On a 1 GB VM already running Wiki.js, FileBrowser, PostgreSQL and Nginx this causes out-of-memory crashes. The static status page below uses zero extra server RAM — it runs entirely in the visitor's browser using JavaScript to check if each service is reachable.

### 7.1 Create the Status Page Directory

```bash
sudo mkdir -p /var/www/status
sudo chown azureuser:www-data /var/www/status
sudo chmod 755 /var/www/status
```

### 7.2 Create the Status Page

```bash
nano /var/www/status/index.html
```

Paste:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="refresh" content="60">
  <title>Status — suuiisparkle.com</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Segoe UI', sans-serif; background: #0f172a; color: #e2e8f0; min-height: 100vh; padding: 40px 20px; }
    .container { max-width: 700px; margin: 0 auto; }
    header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; padding-bottom: 1rem; border-bottom: 1px solid #1e293b; }
    header h1 { font-size: 1.4rem; font-weight: 500; }
    header p  { font-size: 0.85rem; color: #64748b; margin-top: 4px; }
    .overall  { display: flex; align-items: center; gap: 8px; font-size: 0.85rem; font-weight: 500; padding: 6px 14px; border-radius: 20px; }
    .ok   { background: #14532d; color: #86efac; }
    .warn { background: #78350f; color: #fcd34d; }
    .down { background: #7f1d1d; color: #fca5a5; }
    .dot  { width: 8px; height: 8px; border-radius: 50%; }
    .dot-ok   { background: #22c55e; }
    .dot-warn { background: #f59e0b; }
    .dot-down { background: #ef4444; }
    .dot-info { background: #3b82f6; }
    .services { display: flex; flex-direction: column; gap: 10px; margin-bottom: 2rem; }
    .card { background: #1e293b; border: 1px solid #334155; border-radius: 10px; padding: 1rem 1.25rem; display: flex; align-items: center; justify-content: space-between; }
    .card-left  { display: flex; align-items: center; gap: 12px; }
    .icon { font-size: 1.2rem; width: 28px; text-align: center; }
    .name { font-size: 0.95rem; font-weight: 500; }
    .url  { font-size: 0.75rem; color: #64748b; margin-top: 2px; }
    .card-right { display: flex; align-items: center; gap: 10px; }
    .ping  { font-size: 0.75rem; color: #475569; min-width: 60px; text-align: right; }
    .label { font-size: 0.8rem; font-weight: 500; }
    .label-ok   { color: #4ade80; }
    .label-warn { color: #fbbf24; }
    .label-down { color: #f87171; }
    .label-info { color: #60a5fa; }
    .divider { border: none; border-top: 1px solid #1e293b; margin: 1.5rem 0; }
    .section-label { font-size: 0.75rem; color: #475569; text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 0.75rem; }
    .stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-bottom: 2rem; }
    .stat { background: #1e293b; border-radius: 8px; padding: 1rem; text-align: center; }
    .stat-num   { font-size: 1.5rem; font-weight: 500; }
    .stat-label { font-size: 0.75rem; color: #64748b; margin-top: 4px; }
    .vpn-card { background: #1e293b; border: 1px solid #1d4ed8; border-radius: 10px; padding: 1rem 1.25rem; display: flex; align-items: center; justify-content: space-between; }
    .vpn-detail { font-size: 0.75rem; color: #64748b; margin-top: 2px; }
    footer { text-align: center; font-size: 0.75rem; color: #334155; margin-top: 1rem; }
  </style>
</head>
<body>
<div class="container">
  <header>
    <div>
      <h1>suuiisparkle.com — status</h1>
      <p id="last-checked">Checking services...</p>
    </div>
    <div class="overall ok" id="overall">
      <div class="dot dot-ok" id="overall-dot"></div>
      <span id="overall-text">Checking...</span>
    </div>
  </header>

  <div class="section-label">Web services</div>
  <div class="services" id="services"></div>

  <hr class="divider">

  <div class="section-label">VPN</div>
  <div class="vpn-card">
    <div class="card-left">
      <div class="icon">🔒</div>
      <div>
        <div class="name">WireGuard VPN</div>
        <div class="vpn-detail">vpn.suuiisparkle.com — UDP port 51820</div>
      </div>
    </div>
    <div class="card-right">
      <span class="ping">UDP</span>
      <span class="label label-warn" id="vpn-label">Not configured</span>
      <div class="dot dot-warn" id="vpn-dot"></div>
    </div>
  </div>

  <hr class="divider">

  <div class="stats">
    <div class="stat"><div class="stat-num" id="s-up">—</div><div class="stat-label">Up</div></div>
    <div class="stat"><div class="stat-num">3</div><div class="stat-label">Total</div></div>
    <div class="stat"><div class="stat-num" id="s-pct">—</div><div class="stat-label">Uptime</div></div>
    <div class="stat"><div class="stat-num" id="s-avg">—</div><div class="stat-label">Avg ms</div></div>
  </div>

  <footer>Auto-refreshes every 60 seconds · Microsoft Azure · Ubuntu 24.04 · Nginx</footer>
</div>

<script>
var svcs = [
  { name: 'Main website', url: 'https://suuiisparkle.com',       icon: '🏠' },
  { name: 'Wiki.js',      url: 'https://wiki.suuiisparkle.com',  icon: '📖' },
  { name: 'FileBrowser',  url: 'https://cloud.suuiisparkle.com', icon: '📁' }
];
var res = svcs.map(function() { return { s: 'checking', ms: null }; });
function render() {
  document.getElementById('services').innerHTML = svcs.map(function(v, i) {
    var r = res[i];
    var dc = r.s==='ok' ? 'dot-ok' : r.s==='down' ? 'dot-down' : 'dot-warn';
    var lc = r.s==='ok' ? 'label-ok' : r.s==='down' ? 'label-down' : 'label-warn';
    var lt = r.s==='ok' ? 'Operational' : r.s==='down' ? 'Unreachable' : 'Checking...';
    return '<div class="card"><div class="card-left"><div class="icon">'+v.icon+'</div>'+
      '<div><div class="name">'+v.name+'</div><div class="url">'+v.url+'</div></div></div>'+
      '<div class="card-right"><span class="ping">'+(r.ms!==null?r.ms+'ms':'—')+'</span>'+
      '<span class="label '+lc+'">'+lt+'</span>'+
      '<div class="dot '+dc+'"></div></div></div>';
  }).join('');
  var up   = res.filter(function(r){return r.s==='ok';}).length;
  var chk  = res.filter(function(r){return r.s==='checking';}).length;
  var pings = res.filter(function(r){return r.ms!==null;}).map(function(r){return r.ms;});
  document.getElementById('s-up').textContent  = up;
  document.getElementById('s-pct').textContent = chk ? '—' : Math.round(up/svcs.length*100)+'%';
  document.getElementById('s-avg').textContent = pings.length ? Math.round(pings.reduce(function(a,b){return a+b;},0)/pings.length)+'' : '—';
  var ob=document.getElementById('overall');
  var od=document.getElementById('overall-dot');
  var ot=document.getElementById('overall-text');
  ob.className='overall';
  if(chk>0)              { ob.classList.add('warn'); od.className='dot dot-warn'; ot.textContent='Checking...'; }
  else if(up===svcs.length){ ob.classList.add('ok');   od.className='dot dot-ok';   ot.textContent='All systems operational'; }
  else                   { ob.classList.add('down'); od.className='dot dot-down'; ot.textContent=(svcs.length-up)+' service(s) down'; }
}
function check(i) {
  var t=Date.now();
  var img=new Image();
  img.onload=img.onerror=function(){res[i]={s:'ok',ms:Date.now()-t};render();};
  setTimeout(function(){if(res[i].s==='checking'){res[i]={s:'down',ms:null};render();}},8000);
  img.src=svcs[i].url+'/favicon.ico?'+t;
}
function run() {
  res=svcs.map(function(){return{s:'checking',ms:null};});
  render();
  document.getElementById('last-checked').textContent='Last checked: '+new Date().toLocaleTimeString();
  svcs.forEach(function(_,i){setTimeout(function(){check(i);},i*400);});
}
run();
</script>
</body>
</html>
```

### 7.3 Automated VPN Status Updates

The server_monitor.sh script (Phase 9) checks WireGuard every minute and updates the VPN card on the status page automatically. Once WireGuard is installed and running, the card will change from "Not configured" to "Active" or "Connected" within one minute.

### 7.4 Verify

Visit `https://status.suuiisparkle.com` — the dashboard should show live status for all three web services and the VPN card.

---

## Phase 8 — WireGuard VPN

WireGuard runs directly on UDP port 51820, bypassing Nginx entirely. Clients connect to `vpn.suuiisparkle.com:51820`.

### 8.1 Install WireGuard

```bash
sudo apt install -y wireguard
```

### 8.2 Generate Server Keys

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key \
           | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
cat /etc/wireguard/server_public.key
```

Copy the public key output — looks like `AbC123...XyZ=`.

### 8.3 Enable IP Forwarding

This allows VPN clients to route internet traffic through the server:

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify: `sysctl net.ipv4.ip_forward` should return `1`.

### 8.4 Find Your Network Interface Name

```bash
ip link show | grep -E "^[0-9]+:" | grep -v lo
```

Look for the interface that is UP — usually `eth0` on Azure. Use that name in PostUp/PostDown below.

### 8.5 Create the Server Config

```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste (replace `[SERVER_PRIVATE_KEY]` with the output of `cat /etc/wireguard/server_private.key`):

```ini
[Interface]
Address    = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [SERVER_PRIVATE_KEY]

PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Add one [Peer] block per client device
# [Peer]
# PublicKey  = [CLIENT_PUBLIC_KEY]
# AllowedIPs = 10.0.0.2/32
```

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
wg show
```

### 8.6 Add a Client Device

On your client machine generate keys:

```bash
wg genkey | tee ~/client_private.key | wg pubkey > ~/client_public.key
cat ~/client_private.key
cat ~/client_public.key
```

Create the client config file (`wg0.conf` on your device):

```ini
[Interface]
Address    = 10.0.0.2/24
PrivateKey = [CLIENT_PRIVATE_KEY]
DNS        = 1.1.1.1

[Peer]
PublicKey           = [SERVER_PUBLIC_KEY]
Endpoint            = vpn.suuiisparkle.com:51820
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

Add the client's public key to the server:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Uncomment and fill in the `[Peer]` block:

```ini
[Peer]
PublicKey  = [CLIENT_PUBLIC_KEY]
AllowedIPs = 10.0.0.2/32
```

Apply without dropping existing connections:

```bash
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)
```

Test the connection: from the client, ping `10.0.0.1`. On the server, `wg show` should show a recent handshake timestamp. The status page will automatically update the VPN card to "Connected" within 1 minute.

---

## Phase 9 — Automation Script

A single combined script handles both disk monitoring and WireGuard status updates. It runs every minute via cron — disk usage is logged every hour, VPN status is checked every minute.

### 9.1 Create the Script

```bash
nano /home/azureuser/server_monitor.sh
```

Paste:

```bash
#!/bin/bash
# =============================================================================
# server_monitor.sh
# Part 1: Checks disk usage every hour — logs warnings to disk_monitor.log
# Part 2: Checks WireGuard status every minute — updates the status page
#
# Usage: ./server_monitor.sh [threshold_percent]
# Default threshold: 80%
# Scheduled: * * * * * (every minute via cron)
# =============================================================================

THRESHOLD=${1:-80}
LOG="/var/log/disk_monitor.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
STATUS_FILE="/var/www/status/index.html"
ALERT=0

# =============================================================================
# PART 1 — Disk usage (only logs at the top of each hour)
# =============================================================================

MINUTE=$(date '+%M')
if [ "$MINUTE" = "00" ]; then

    echo "[$TIMESTAMP] === Disk check (threshold: ${THRESHOLD}%) ===" >> "$LOG"

    while IFS= read -r line; do
        USE=$(echo "$line"   | awk '{print $5}' | tr -d '%')
        MOUNT=$(echo "$line" | awk '{print $6}')
        FS=$(echo "$line"    | awk '{print $1}')

        if [ "$USE" -gt "$THRESHOLD" ]; then
            echo "[$TIMESTAMP] WARNING: $MOUNT ($FS) at ${USE}% — EXCEEDS ${THRESHOLD}%" >> "$LOG"
            ALERT=1
        else
            echo "[$TIMESTAMP] OK:      $MOUNT ($FS) at ${USE}%" >> "$LOG"
        fi
    done < <(df -h --output=source,size,used,avail,pcent,target \
             | tail -n +2 \
             | grep -Ev '^(tmpfs|overlay|devtmpfs|udev)')

    if [ "$ALERT" -eq 0 ]; then
        echo "[$TIMESTAMP] All partitions within threshold." >> "$LOG"
    else
        echo "[$TIMESTAMP] ALERT: One or more partitions exceeded threshold!" >> "$LOG"
    fi

    echo "[$TIMESTAMP] ---" >> "$LOG"

fi

# =============================================================================
# PART 2 — WireGuard VPN status (updates status page every minute)
# =============================================================================

if sudo wg show wg0 > /dev/null 2>&1; then
    HANDSHAKE=$(sudo wg show wg0 latest-handshakes | awk '{print $2}')
    NOW=$(date +%s)

    if [ -z "$HANDSHAKE" ] || [ "$HANDSHAKE" = "0" ]; then
        LABEL="label-warn"
        DOT="dot-warn"
        TEXT="No peers connected"
    else
        DIFF=$((NOW - HANDSHAKE))
        if [ "$DIFF" -lt 180 ]; then
            LABEL="label-ok"
            DOT="dot-ok"
            TEXT="Connected"
        else
            LABEL="label-info"
            DOT="dot-info"
            TEXT="Active"
        fi
    fi
else
    LABEL="label-down"
    DOT="dot-down"
    TEXT="Offline"
fi

sudo sed -i \
  "s|<span class=\"label label-[a-z]*\" id=\"vpn-label\">[^<]*</span>|<span class=\"label $LABEL\" id=\"vpn-label\">$TEXT</span>|" \
  "$STATUS_FILE"

sudo sed -i \
  "s|<div class=\"dot dot-[a-z]*\" id=\"vpn-dot\"></div>|<div class=\"dot $DOT\" id=\"vpn-dot\"></div>|" \
  "$STATUS_FILE"
```

### 9.2 Set Up Permissions and Log File

```bash
chmod +x /home/azureuser/server_monitor.sh
sudo touch /var/log/disk_monitor.log
sudo chown azureuser:azureuser /var/log/disk_monitor.log
```

Allow the script to run `wg` without a password prompt:
```bash
sudo visudo
```

Add this line at the very bottom:
```
azureuser ALL=(ALL) NOPASSWD: /usr/bin/wg, /usr/sbin/wg, /usr/bin/sed, /usr/bin/tee
```

### 9.3 Schedule with Cron

```bash
crontab -e
```

Add this line:
```
* * * * * /home/azureuser/server_monitor.sh 80
```

Verify:
```bash
crontab -l
```

### 9.4 Test Manually

Run the script once immediately without waiting for cron:

```bash
/home/azureuser/server_monitor.sh 80
```

View the disk log:
```bash
cat /var/log/disk_monitor.log
```

Sample output:
```
[2026-06-05 01:00:32] === Disk check (threshold: 80%) ===
[2026-06-05 01:00:32] OK:      / (/dev/root) at 14%
[2026-06-05 01:00:32] OK:      /boot/efi (/dev/sda15) at 6%
[2026-06-05 01:00:32] All partitions within threshold.
[2026-06-05 01:00:32] ---
```

---

## Problems Encountered and Solutions

This section documents every real issue encountered during the build, and exactly how each was resolved.

---

### Problem 1 — UFW commands merging when pasted

**What happened:** Pasting multiple commands at once into CMD caused them to merge into a single malformed command (e.g. `sudo ufsudo ufw allow 80/tcp`). The firewall ended up inactive — `sudo ufw status` showed `Status: inactive`.

**Root cause:** CMD does not handle multi-line pastes correctly. Commands ran into each other.

**Solution:** Run every command individually — one line at a time, pressing Enter after each. Never paste a block of commands into CMD.

**Verify fix:**
```bash
sudo ufw status
```
Must show `Status: active`.

---

### Problem 2 — Ctrl+V not working in CMD

**What happened:** Standard Ctrl+V paste did not work in Windows Command Prompt even with QuickEdit Mode enabled.

**Root cause:** CMD's paste shortcut with QuickEdit Mode is Ctrl+Shift+V, not Ctrl+V.

**Solution:** Use **right-click** to paste (always works). Or switch to **Windows Terminal** which supports Ctrl+V natively.

---

### Problem 3 — Nginx config files still had `yourdomain.com` placeholder

**What happened:** After creating all four Nginx config files, `sudo nginx -t` returned:
```
cannot load certificate "/etc/letsencrypt/live/yourdomain.com/fullchain.pem": No such file or directory
```

**Root cause:** The SSL certificate paths still contained `yourdomain.com` instead of `suuiisparkle.com`. The `server_name` lines had been updated but the certificate paths were missed.

**Solution:** Use `sed` to replace all occurrences automatically:
```bash
sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/main
sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/wiki
sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/cloud
sudo sed -i 's/yourdomain.com/suuiisparkle.com/g' /etc/nginx/sites-available/status
```

Verify: `grep -r "yourdomain" /etc/nginx/sites-available/` — no output means all fixed.

---

### Problem 4 — "File exists" error when creating Nginx symlinks

**What happened:** Running `sudo ln -s` returned `ln: failed to create symbolic link: File exists`.

**Root cause:** An earlier failed paste attempt had partially created some symlinks. Running the commands again hit the already-existing ones.

**Solution:** Remove all symlinks first, then recreate cleanly:
```bash
sudo rm -f /etc/nginx/sites-enabled/main
sudo rm -f /etc/nginx/sites-enabled/wiki
sudo rm -f /etc/nginx/sites-enabled/cloud
sudo rm -f /etc/nginx/sites-enabled/status
sudo ln -s /etc/nginx/sites-available/main   /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/wiki   /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/cloud  /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/status /etc/nginx/sites-enabled/
```

---

### Problem 5 — Raw IP showing 502 Bad Gateway

**What happened:** Visiting `https://20.213.11.20` showed a 502 Bad Gateway error.

**Root cause:** All Nginx configs were set up for domain names only. Nginx had no `default_server` block to handle requests arriving via raw IP.

**Solution:** Created a dedicated catch-all config at `/etc/nginx/sites-available/default-ip` that redirects any raw IP visit to `https://suuiisparkle.com`.

---

### Problem 6 — bzip2 not installed, Nextcloud extraction failed

**What happened:**
```
tar (child): bzip2: Cannot exec: No such file or directory
tar: Error is not recoverable: exiting now
```

**Root cause:** `bzip2` was not installed on the base Ubuntu image. The `-j` flag in tar requires bzip2 to decompress `.bz2` archives.

**Solution:**
```bash
sudo apt install -y bzip2
sudo tar -xjf latest.tar.bz2 -C /var/www/
```

---

### Problem 7 — VM ran out of RAM, npm install hung (Uptime Kuma)

**What happened:** Running `npm install uptime-kuma` caused the terminal to hang for over 10 minutes. The VM became completely unresponsive and the Azure Portal showed "Virtual machine agent status is not ready."

**Root cause:** The VM is a Standard_B2ats v2 with only 1 GB RAM. npm requires significantly more memory to resolve and install Uptime Kuma's dependency tree. The system ran out of RAM and froze.

**Solution — Part 1:** VM was restored using Azure Portal → Redeploy + Reapply.

**Solution — Part 2:** Added a 2 GB swap file to prevent future out-of-memory crashes:
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Solution — Part 3:** Uptime Kuma was removed from the project entirely and replaced with a lightweight static HTML status page that uses zero extra RAM.

---

### Problem 8 — VM agent not ready, Restart button greyed out

**What happened:** After the VM froze, the Azure Portal showed "Virtual machine agent status is not ready" and the Restart button was greyed out.

**Root cause:** The VM OS was completely unresponsive, so the Azure agent could not report back to Azure, disabling the standard restart button.

**Solution:** Used **Redeploy + Reapply** from the VM's left sidebar under Support + Troubleshooting. This moves the VM to a fresh physical host in Azure while preserving all data. VM came back online after ~5 minutes.

> **Note:** The `az vm restart --force` command via Azure Cloud Shell returned AuthorizationFailed — the Azure for Students subscription does not grant redeploy permissions via CLI.

---

### Problem 9 — Nextcloud replaced with FileBrowser

**What happened:** After restoring the VM and adding swap, Nextcloud was evaluated again. Even with 2 GB swap, Nextcloud's Apache + PHP stack (~300 MB RAM) combined with Wiki.js (~200 MB), PostgreSQL (~80 MB) and Nginx (~30 MB) left the 1 GB VM critically low on real memory causing slow performance.

**Solution:** Nextcloud was replaced with **FileBrowser** — a single Go binary using ~20 MB RAM. It provides the same core functionality (file upload, download, folder management, public share links) with a fraction of the resource usage. This kept the server stable and responsive.

---

### Problem 10 — Status page showing wrong VPN status

**What happened:** After removing Uptime Kuma and creating the static status page, the WireGuard VPN card was hardcoded as "Active" even though WireGuard was not installed.

**Root cause:** The initial HTML had a static label that was never updated by any script.

**Solution:** The `server_monitor.sh` script was updated with a WireGuard check that uses `wg show` to detect the real state and rewrites the VPN card label in the HTML file every minute via cron. The label now accurately reflects the real state:
- `Offline` — WireGuard not installed or stopped
- `No peers connected` — WireGuard running but no client connected
- `Active` — WireGuard running, client configured but idle
- `Connected` — Client actively connected within last 3 minutes

---

## Troubleshooting Reference

**Check all services at once:**
```bash
for svc in nginx wikijs postgresql filebrowser wg-quick@wg0; do
    echo -n "$svc: "
    sudo systemctl is-active $svc
done
```

**Check server memory:**
```bash
free -h
```

**Nginx fails to start:**
```bash
sudo nginx -t
sudo journalctl -u nginx --since "5 minutes ago"
```

**Certbot fails — DNS problem:**
Run `nslookup suuiisparkle.com` — if it doesn't return `20.213.11.20`, DNS hasn't propagated. Wait 5 minutes and retry.

**Wiki.js not loading:**
```bash
sudo systemctl status wikijs
sudo journalctl -u wikijs -n 50
sudo systemctl status postgresql
```

**FileBrowser not loading:**
```bash
sudo systemctl status filebrowser
sudo journalctl -u filebrowser -n 30
```

**WireGuard client connected but no internet:**
```bash
sudo wg show
sysctl net.ipv4.ip_forward
sudo ufw status
```

**View disk monitor log:**
```bash
tail -30 /var/log/disk_monitor.log
```

**VM running out of memory:**
```bash
free -h
sudo swapon --show
```
If swap is not showing, re-run the swap file setup from Phase 1.6.

---

## References

- Microsoft Azure — Create a Linux VM: https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal
- Microsoft Azure — Static Public IP: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/configure-public-ip-vm
- GoDaddy — Add DNS A Records: https://au.godaddy.com/help/add-an-a-record-19238
- Nginx — Reverse Proxy Guide: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- Let's Encrypt / Certbot: https://certbot.eff.org/instructions?os=ubuntufocal&webserver=nginx
- Wiki.js — Linux Installation: https://docs.requarks.io/install/linux
- FileBrowser — Official Documentation: https://filebrowser.org/installation
- FileBrowser — Configuration: https://filebrowser.org/configuration
- WireGuard — Quick Start: https://www.wireguard.com/quickstart/
- WireGuard — Ubuntu Setup: https://ubuntu.com/server/docs/wireguard-vpn-peer2site-router
- PostgreSQL — Ubuntu Installation: https://www.postgresql.org/download/linux/ubuntu/
- Ubuntu UFW Guide: https://help.ubuntu.com/community/UFW
- Ubuntu Swap File: https://ubuntu.com/server/docs/swap-space
