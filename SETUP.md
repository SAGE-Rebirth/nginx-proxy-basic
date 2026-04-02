# NGINX Reverse Proxy — Complete Setup & Troubleshooting Guide

> **Purpose:** Step-by-step reference for setting up NGINX as a reverse proxy with SSL (Certbot), so you never miss a step even after months away.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Plan Your Configuration](#2-plan-your-configuration)
3. [Create the NGINX Config File](#3-create-the-nginx-config-file)
4. [Validate the Config](#4-validate-the-config)
5. [Enable the Site](#5-enable-the-site)
6. [Restart NGINX](#6-restart-nginx)
7. [Install SSL with Certbot](#7-install-ssl-with-certbot)
8. [Restart NGINX Again](#8-restart-nginx-again)
9. [Restart Your App (PM2)](#9-restart-your-app-pm2)
10. [Verify Everything Works](#10-verify-everything-works)
11. [Troubleshooting](#11-troubleshooting)
12. [Nuclear Option — Full Reinstall](#12-nuclear-option--full-reinstall)
13. [Quick Reference — Command Cheatsheet](#13-quick-reference--command-cheatsheet)

---

## 1. Prerequisites

Before you begin, make sure the following are in place:

```bash
# NGINX is installed
sudo apt update && sudo apt install nginx -y

# Certbot is installed (for SSL)
sudo apt install certbot python3-certbot-nginx -y

# PM2 is installed (for Node.js process management)
npm install -g pm2

# Your app is already running on a local port (e.g., 3002)
# Verify with:
curl http://127.0.0.1:3002
```

**Checklist before proceeding:**

- [ ] NGINX installed and running (`sudo systemctl status nginx`)
- [ ] Certbot installed (`certbot --version`)
- [ ] Your app is running and accessible on `127.0.0.1:<PORT>`
- [ ] Your domain's DNS A record points to this server's public IP
- [ ] If using a subdomain (e.g., `api.clearmymind.com`), its A record also points to this server

---

## 2. Plan Your Configuration

Before creating the config file, **sort out your ports and domains** to avoid conflicts.

### 2a. List all your existing configs and the ports they use

```bash
# See all existing site configs
ls -la /etc/nginx/sites-available/
ls -la /etc/nginx/sites-enabled/

# Check what ports are already mapped to which domains
grep -r "server_name\|proxy_pass" /etc/nginx/sites-available/
```

### 2b. Map out your setup on paper first

| Domain / Subdomain          | Local Port | Config File Name         |
|-----------------------------|------------|--------------------------|
| `clearmymind.com`           | 3002       | `clearmymind.com`        |
| `api.clearmymind.com`       | 3003       | `api.clearmymind.com`    |
| *(add more rows as needed)* |            |                          |

**Rules to avoid problems:**

- **One port per app** — never map two domains to the same port unless they truly serve the same app.
- **One `server_name` per domain** — never declare the same domain in two different config files. This causes silent conflicts.
- **File naming convention** — name the config file after the domain (e.g., `api.clearmymind.com`). This makes it instantly clear what each file is for.

---

## 3. Create the NGINX Config File

### 3a. Create the file in `sites-available`

```bash
sudo nano /etc/nginx/sites-available/api.clearmymind.com
```

> **Why `sites-available` and not `sites-enabled`?**
> `sites-available` holds all configs. `sites-enabled` holds symlinks to the ones you want active. This way you can disable a site without deleting the config — just remove the symlink.

### 3b. Paste the server block

```nginx
server {
    listen 80;
    server_name www.api.clearmymind.com api.clearmymind.com;

    location / {
        proxy_pass http://127.0.0.1:3003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3c. Understand each line

| Line | What it does |
|------|-------------|
| `listen 80;` | Listens on port 80 (HTTP). Certbot will add port 443 (HTTPS) later. |
| `server_name ...;` | The domain(s) this block responds to. Include both `www` and non-`www`. |
| `proxy_pass http://127.0.0.1:3003;` | Forwards requests to your app running locally on port 3003. |
| `Host $host` | Passes the original domain name to your app. |
| `X-Real-IP` | Passes the real client IP (not NGINX's IP) to your app. |
| `X-Forwarded-For` | Appends the client IP to the forwarding chain (useful if behind multiple proxies). |
| `X-Forwarded-Proto` | Tells your app whether the original request was HTTP or HTTPS. |

### 3d. Save and exit

- In `nano`: `Ctrl + O` (save) → `Enter` (confirm) → `Ctrl + X` (exit)

---

## 4. Validate the Config

**Always test before restarting.** A bad config will take down ALL your sites, not just the new one.

```bash
sudo nginx -t
```

**Expected output (good):**

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**If it fails:**

- Read the error message carefully — it tells you the exact file and line number.
- Common mistakes:
  - Missing semicolon at the end of a line.
  - Mismatched braces `{ }`.
  - Duplicate `server_name` — the same domain declared in another config file.

**Do NOT proceed to the next step if `nginx -t` fails.**

---

## 5. Enable the Site

Create a symbolic link from `sites-available` to `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/api.clearmymind.com /etc/nginx/sites-enabled/
```

**Verify the symlink was created:**

```bash
ls -la /etc/nginx/sites-enabled/
```

You should see something like:

```
api.clearmymind.com -> /etc/nginx/sites-available/api.clearmymind.com
```

**Common mistake:** Running `ln -s` with a relative path. Always use the full path for the source, or run the command from the correct directory. If the symlink is broken (red in `ls`), remove it and recreate:

```bash
sudo rm /etc/nginx/sites-enabled/api.clearmymind.com
sudo ln -s /etc/nginx/sites-available/api.clearmymind.com /etc/nginx/sites-enabled/
```

**Run `nginx -t` again** after enabling the site to catch any issues:

```bash
sudo nginx -t
```

---

## 6. Restart NGINX

```bash
sudo systemctl restart nginx
```

**Verify it's running:**

```bash
sudo systemctl status nginx
```

Look for `Active: active (running)`. If it says `failed`, go to [Troubleshooting](#11-troubleshooting).

**Quick test — does the domain respond over HTTP?**

```bash
curl -I http://api.clearmymind.com
```

You should get a response from your app (or a redirect). If you get `connection refused` or `timeout`, check:

- DNS is pointing to this server.
- Port 80 is open in your cloud provider's security group.
- UFW (if enabled) allows port 80: `sudo ufw allow 80/tcp`.

---

## 7. Install SSL with Certbot

```bash
sudo certbot --nginx -d www.api.clearmymind.com -d api.clearmymind.com
```

**What this does:**

1. Requests a free SSL certificate from Let's Encrypt.
2. Automatically modifies your NGINX config to add HTTPS (port 443).
3. Sets up a redirect from HTTP to HTTPS.

**During the process:**

- Enter your email when prompted (for renewal notifications).
- Agree to terms of service.
- Choose whether to redirect HTTP to HTTPS (recommended: **yes**).

**If Certbot fails:**

- **"Could not find a matching server block"** — your `server_name` in the config doesn't match the `-d` domains you passed to Certbot. They must match exactly.
- **"DNS problem"** — the domain's A record doesn't point to this server. Verify with: `dig api.clearmymind.com +short` (should show your server's public IP).
- **"Connection refused on port 80"** — NGINX isn't running, or port 80 is blocked. Check security groups and UFW.

**Verify the SSL cert was installed:**

```bash
sudo certbot certificates
```

---

## 8. Restart NGINX Again

After Certbot modifies the config, restart NGINX to apply:

```bash
sudo systemctl restart nginx
```

**Verify:**

```bash
sudo systemctl status nginx
sudo nginx -t
```

---

## 9. Restart Your App (PM2)

```bash
pm2 restart all
```

Or restart a specific app:

```bash
pm2 list                    # see all running apps
pm2 restart <app-name>      # restart a specific one
```

**Verify your app is running:**

```bash
pm2 status
```

---

## 10. Verify Everything Works

Run these checks in order:

```bash
# 1. Is NGINX running?
sudo systemctl status nginx

# 2. Is your app running?
pm2 status

# 3. Can you reach the app locally?
curl http://127.0.0.1:3003

# 4. Does the domain resolve to this server?
dig api.clearmymind.com +short

# 5. Does HTTP work?
curl -I http://api.clearmymind.com

# 6. Does HTTPS work?
curl -I https://api.clearmymind.com

# 7. Full verbose check (shows SSL handshake, headers, redirects)
curl -vL https://api.clearmymind.com
```

If all 7 checks pass, you're done.

---

## 11. Troubleshooting

### 11a. Check ports in security groups & firewall

Your cloud provider (AWS, GCP, DigitalOcean, etc.) has security groups / firewall rules. Make sure ports **80** and **443** are open for inbound traffic.

On the server itself, if UFW is enabled:

```bash
# Check UFW status
sudo ufw status

# Allow HTTPS if not already allowed
sudo ufw allow 443/tcp

# Allow HTTP if not already allowed
sudo ufw allow 80/tcp

# Reload UFW
sudo ufw reload
```

### 11b. Verbose domain check

```bash
curl -v http://api.clearmymind.com
curl -v https://api.clearmymind.com
```

Look for:

- `Connection refused` — NGINX isn't listening, or port is blocked.
- `SSL certificate problem` — cert not installed or expired.
- `502 Bad Gateway` — NGINX is running but can't reach your app. Check if the app is running and the port matches.
- `404 Not Found` — NGINX is running and reaching your app, but the route doesn't exist.

### 11c. Check for duplicate `server_name` blocks

This is a **very common** and sneaky issue. If two config files declare the same `server_name`, NGINX silently picks one (usually the first one alphabetically) and ignores the other.

```bash
# Find all server_name declarations across all configs
grep -r "server_name" /etc/nginx/sites-enabled/
```

**Every domain should appear only once.** If you see duplicates, remove or fix the extra one.

### 11d. Check for duplicate config files

```bash
# These two directories should have a 1:1 symlink relationship
ls /etc/nginx/sites-available/
ls /etc/nginx/sites-enabled/

# Check for broken symlinks in sites-enabled
find /etc/nginx/sites-enabled/ -xtype l
```

Remove any broken symlinks:

```bash
sudo rm /etc/nginx/sites-enabled/<broken-link>
```

### 11e. Check NGINX and system logs

```bash
# NGINX error log (most useful)
sudo tail -50 /var/log/nginx/error.log

# NGINX access log
sudo tail -50 /var/log/nginx/access.log

# Systemd journal for NGINX
sudo journalctl -u nginx --no-pager -n 50

# If NGINX failed to start, this gives the reason
sudo journalctl -u nginx --since "5 minutes ago"
```

### 11f. Check if something else is using the port

```bash
# What's listening on port 80?
sudo ss -tlnp | grep :80

# What's listening on port 443?
sudo ss -tlnp | grep :443

# What's listening on your app's port?
sudo ss -tlnp | grep :3003
```

---

## 12. Nuclear Option — Full Reinstall

**Only do this if nothing else works.** This removes NGINX and Certbot completely and starts fresh.

```bash
# Stop NGINX
sudo systemctl stop nginx

# Remove NGINX and Certbot
sudo apt purge nginx nginx-common nginx-full certbot python3-certbot-nginx -y
sudo apt autoremove -y

# Remove leftover config files (CAREFUL — this deletes all your configs)
sudo rm -rf /etc/nginx/

# Reinstall
sudo apt update
sudo apt install nginx -y
sudo apt install certbot python3-certbot-nginx -y

# Verify
nginx -v
certbot --version

# Start NGINX
sudo systemctl start nginx
sudo systemctl enable nginx
```

After reinstalling, go back to [Step 2](#2-plan-your-configuration) and recreate all your configs from scratch.

> **Tip:** Before nuking, back up your configs:
> ```bash
> sudo cp -r /etc/nginx/ ~/nginx-backup-$(date +%Y%m%d)
> ```

---

## 13. Quick Reference — Command Cheatsheet

| Step | Command |
|------|---------|
| Create config | `sudo nano /etc/nginx/sites-available/<domain>` |
| Test config | `sudo nginx -t` |
| Enable site | `sudo ln -s /etc/nginx/sites-available/<domain> /etc/nginx/sites-enabled/` |
| Restart NGINX | `sudo systemctl restart nginx` |
| Check NGINX status | `sudo systemctl status nginx` |
| Install SSL | `sudo certbot --nginx -d <domain> -d www.<domain>` |
| Check SSL certs | `sudo certbot certificates` |
| Renew SSL certs | `sudo certbot renew --dry-run` |
| Restart PM2 | `pm2 restart all` |
| Check PM2 status | `pm2 status` |
| Check open ports | `sudo ss -tlnp` |
| UFW status | `sudo ufw status` |
| NGINX error log | `sudo tail -f /var/log/nginx/error.log` |
| Journal logs | `sudo journalctl -u nginx --no-pager -n 50` |
| Find duplicate server_names | `grep -r "server_name" /etc/nginx/sites-enabled/` |
| Verbose domain check | `curl -vL https://<domain>` |

---

## Basic Server Block Template

Copy-paste this and change the domain and port:

```nginx
server {
    listen 80;
    server_name www.example.com example.com;

    location / {
        proxy_pass http://127.0.0.1:PORT;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

After Certbot runs, it will automatically add the HTTPS block. Do **not** manually add `listen 443` or SSL directives — let Certbot handle that.

---

## SSL Auto-Renewal

Certbot sets up a systemd timer for auto-renewal. Verify it's active:

```bash
sudo systemctl status certbot.timer
```

Test renewal (dry run, doesn't actually renew):

```bash
sudo certbot renew --dry-run
```

If the timer isn't active, enable it:

```bash
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

---

*Last updated: 2026-04-02*
