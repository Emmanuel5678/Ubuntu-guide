# Ubuntu Production Deployment Guide: Nginx + Systemd + Troubleshooting

A comprehensive guide for deploying web applications to production on Ubuntu with Nginx as reverse proxy and systemd for process management.

---

## 📑 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Nginx Deployment Guide](#nginx-deployment-guide)
3. [Systemd Service Configuration](#systemd-service-configuration)
4. [Common Blockers & Fixes](#common-blockers--fixes)
5. [Troubleshooting Toolkit](#troubleshooting-toolkit)
6. [Maintenance Best Practices](#maintenance-best-practices)

---

## 🏗️ Architecture Overview

The core principle is using Nginx as a **gateway** that:
- Accepts web traffic from the internet
- Manages HTTPS encryption (SSL termination)
- Forwards requests securely to your internal application running on a local port (e.g., 3000)

This keeps your application isolated while Nginx efficiently serves static files and handles high concurrency.

```
Internet → Nginx (Port 80/443) → Systemd Service (Port 3000) → Your App
```

---

## 🚀 Nginx Deployment Guide

### Step 1: Install Nginx & Configure Firewall

```bash
# Update and install Nginx
sudo apt update
sudo apt install nginx -y

# Enable Nginx to start on boot
sudo systemctl enable nginx

# Configure firewall
sudo ufw allow 'Nginx Full'  # HTTP (80) and HTTPS (443)
sudo ufw allow OpenSSH       # Keep SSH access
sudo ufw enable
```

**Verify installation:** Visit your server's public IP in a browser - you should see the Nginx welcome page.

### Step 2: Create Nginx Server Block

```bash
# Create directory structure (replace your-domain.com)
sudo mkdir -p /var/www/your-domain.com/html
sudo chown -R $USER:$USER /var/www/your-domain.com/html

# Create test page
echo "<h1>Hello from your-domain.com</h1>" | sudo tee /var/www/your-domain.com/html/index.html

# Create configuration file
sudo nano /etc/nginx/sites-available/your-domain.com
```

Add this configuration:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;
    root /var/www/your-domain.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/your-domain.com /etc/nginx/sites-enabled/
sudo nginx -t                    # Test syntax
sudo systemctl reload nginx      # Apply changes
```

### Step 3: Configure Reverse Proxy

Create a configuration for your application (e.g., `/etc/nginx/sites-available/global`):

```nginx
server {
    listen 80;
    server_name api.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;  # Your app's port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/global /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Step 4: Secure with SSL/TLS (Let's Encrypt)

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain and install certificate
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Test automatic renewal
sudo certbot renew --dry-run
```

### Step 5: Production Optimization

Edit `/etc/nginx/nginx.conf`:

```nginx
# Enable Gzip compression
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml text/javascript application/javascript application/json;

# Set worker processes to auto
worker_processes auto;

# Add browser caching for static assets inside server block
location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
}
```

---

## ⚙️ Systemd Service Configuration

### Basic Service File

Create `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Production Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node /var/www/myapp/server.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT=3000

# Security hardening
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```
Here are the list of services we have on the server
 c2c-iat-customer-backend.service
c2c-iat-customer.service
c2c-iat-staff-backend.service
c2c-iat.service
c2c-sit-customer-backend.service
c2c-sit-staff-backend.service
```


### Service Management Commands

```bash
# Reload systemd to recognize new service
sudo systemctl daemon-reload

# Enable auto-start on boot
sudo systemctl enable myapp.service

# Start/stop/restart
sudo systemctl start myapp.service
sudo systemctl stop myapp.service
sudo systemctl restart myapp.service

# Check status
sudo systemctl status myapp.service

# View logs
sudo journalctl -u myapp.service -f
```

### Production-Ready Service Example

```ini
[Unit]
Description=My Production Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp

# Startup
ExecStart=/usr/bin/node /var/www/myapp/server.js
ExecStartPre=/bin/bash -c 'test -f /var/www/myapp/server.js'
Restart=on-failure
RestartSec=10
TimeoutStartSec=30
TimeoutStopSec=30

# Environment
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/www/myapp/tmp /var/log/myapp
ProtectHome=true
PrivateDevices=true

# Resources
MemoryMax=512M
CPUQuota=200%
TasksMax=50

[Install]
WantedBy=multi-user.target
```

---

## ⚠️ Common Blockers & Fixes

### Nginx Issues

| Issue | Likely Cause | Fix |
|-------|--------------|-----|
| **502 Bad Gateway** | Backend app not running or wrong port | Check `systemctl status myapp`; verify `proxy_pass` URL |
| **403 Forbidden** | Incorrect file permissions | `sudo chown -R www-data:www-data /var/www/`<br>`sudo chmod -R 755 /var/www/` |
| **Address already in use** | Another service on port 80/443 | `sudo netstat -tulnp \| grep ':80\|:443'`<br>`sudo systemctl stop apache2` |
| **Config changes not applying** | Nginx not reloaded or site not enabled | Run `sudo nginx -t && sudo systemctl reload nginx`<br>Check `ls -la /etc/nginx/sites-enabled/` |
| **Nginx fails to start** | Syntax error in config | Run `sudo nginx -t` to find exact error |

### Systemd Issues

| Issue | Likely Cause | Fix |
|-------|--------------|-----|
| **Permission denied on port 80/443** | Binding to privileged port | Use port 3000 + Nginx proxy (recommended) |
| **Service won't start at boot** | Not enabled | `sudo systemctl enable myapp.service` |
| **"Failed at step USER spawning"** | User doesn't exist | `sudo useradd -r -s /bin/false www-data` or use `DynamicUser=yes` |
| **Can't write files** | Permission issues | `sudo chown -R www-data:www-data /var/www/myapp` |
| **App crashes on startup** | Missing dependencies or syntax error | Check `journalctl -u myapp.service -n 50` |
| **Port conflict** | Another process using same port | `sudo lsof -i :3000` then kill or change port |
| **Slow shutdown** | Timeout too short | Add `TimeoutStopSec=30` and `KillMode=mixed` |
| **Environment variables missing** | Not loaded correctly | Use `EnvironmentFile=` or inline `Environment=` |

---

## 🛠️ Troubleshooting Toolkit

### Essential Commands Quick Reference

| Tool | Primary Use | Example |
|------|-------------|---------|
| **grep** | Pattern searching in logs | `grep -i error /var/log/nginx/error.log` |
| **journalctl** | Systemd service logs | `journalctl -u myapp -f` |
| **systemctl** | Service management | `systemctl status myapp` |
| **ss/netstat** | Port/network inspection | `ss -tulpn \| grep :3000` |
| **lsof** | Open files and ports | `lsof -i :3000` |
| **curl** | HTTP endpoint testing | `curl -v http://localhost:3000/health` |
| **df/du** | Disk usage analysis | `df -h` and `du -sh /var/log/` |
| **top/htop** | Real-time process monitoring | `htop` |
| **ps** | Process listing | `ps aux \| grep node` |

### Detailed Tool Usage

#### grep - Pattern Searching
```bash
# Find errors with context (3 lines before/after)
grep -B 3 -A 3 "ERROR" /var/log/myapp.log

# Search recursively in all logs
grep -r "OutOfMemory" /var/log/

# Count occurrences
grep -c "500" /var/log/nginx/access.log

# Multiple patterns (OR)
grep -E "error|failed|critical" /var/log/syslog

# Live log monitoring with filtering
journalctl -u myapp -f | grep -i error
```

#### journalctl - Systemd Logs
```bash
# Time-based filtering
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp --since yesterday --until today

# Priority filtering
journalctl -u myapp -p err          # Errors only
journalctl -u myapp -p warning..emerg  # Warnings and above

# JSON output for parsing
journalctl -u myapp -o json-pretty

# Monitor multiple services
journalctl -u nginx -u myapp -f
```

#### ss/netstat - Network Inspection
```bash
# Show all listening ports (modern ss is faster)
sudo ss -tulpn

# Find what's using port 3000
sudo ss -tulpn | grep :3000

# Show all TCP connections with process IDs
sudo netstat -tpn
```

#### lsof - List Open Files
```bash
# Find what's using a specific port
sudo lsof -i :3000

# List open files for a user
sudo lsof -u www-data

# Show network connections by state
sudo lsof -i | grep ESTABLISHED
```

#### curl - HTTP Testing
```bash
# Test local endpoint with timing
curl -w "Connect: %{time_connect}s TTFB: %{time_starttransfer}s Total: %{time_total}s\n" \
     -o /dev/null -s http://localhost:3000/health

# Show response headers
curl -I https://your-domain.com

# Detailed connection information
curl -v http://localhost:3000
```

### Quick Diagnostic Workflow

When something breaks, run this sequence:

```bash
# 1. Check service status
sudo systemctl status myapp.service

# 2. View recent logs
sudo journalctl -u myapp.service -n 50

# 3. Check if port is listening
sudo ss -tulpn | grep 3000

# 4. Test the endpoint locally
curl -v http://localhost:3000/health

# 5. Check system resources
df -h && free -h && top -b -n 1 | head -20

# 6. Check Nginx (if applicable)
sudo nginx -t && systemctl status nginx
sudo tail -20 /var/log/nginx/error.log

# 7. Check permissions
ls -la /var/www/myapp/
sudo -u www-data test -w /var/log/myapp/

# 8. Monitor in real-time during issue
sudo journalctl -u myapp -f &
curl http://localhost:3000/trigger-error
fg  # bring journalctl back
```

### Health Check One-Liners

```bash
# Check if service is running
systemctl is-active myapp && echo "OK" || echo "DOWN"

# Check if port responds
nc -zv localhost 3000 && echo "Port open" || echo "Port closed"

# Alert if disk >85%
[ $(df -h / | awk 'NR==2 {print $5}' | sed 's/%//') -gt 85 ] && echo "ALERT: Disk >85%"

# Memory usage percentage
free -h | awk 'NR==2{printf "Memory Usage: %.2f%%\n", $3*100/$2}'

# Top 5 CPU-consuming processes
ps aux --sort=-%cpu | head -6

# Top 5 memory-consuming processes
ps aux --sort=-%mem | head -6
```

### Log Analysis Patterns

```bash
# Extract unique error messages with counts
sudo grep -i error /var/log/nginx/error.log | cut -d: -f4- | sort | uniq -c | sort -nr

# Find IPs making too many requests (potential DDoS)
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Find slow requests (>1 second)
sudo awk '$NF > 1' /var/log/nginx/access.log

# Find stack traces (multi-line)
sudo grep -A 20 "Exception:" /var/log/myapp.log
```

---

## 📈 Maintenance Best Practices

### Monitoring & Logs

```bash
# Watch logs regularly
sudo tail -f /var/log/nginx/error.log
sudo journalctl -u myapp -f

# Check log rotation (prevents disk full)
ls -la /etc/logrotate.d/
sudo logrotate -d /etc/logrotate.d/nginx  # Test rotation
```

### Security Hardening

```bash
# Disable password SSH authentication (use keys only)
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no

# Set up automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Regular security audit
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

### Essential Habits

1. **Always test configs before reloading:**
   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

2. **Save logs before major changes:**
   ```bash
   sudo journalctl -u myapp > /tmp/app-before.log
   ```

3. **Create aliases for common commands:**
   ```bash
   alias app-logs='sudo journalctl -u myapp -f'
   alias nginx-logs='sudo tail -f /var/log/nginx/error.log'
   alias check-port='sudo ss -tulpn | grep'
   alias nginx-test='sudo nginx -t && sudo systemctl reload nginx'
   ```

4. **Use watch for real-time monitoring:**
   ```bash
   watch -n 1 'systemctl status myapp | grep Active'
   watch -n 5 'ss -tulpn | grep :3000'
   ```

---

## 📋 Quick Reference Card

### Service States & Commands
| State | Command |
|-------|---------|
| Check status | `systemctl status myapp` |
| View logs | `journalctl -u myapp -f` |
| Restart | `systemctl restart myapp` |
| Reload config | `systemctl reload nginx` |
| Enable on boot | `systemctl enable myapp` |

### Port Troubleshooting
| Task | Command |
|------|---------|
| Find what's on port 3000 | `lsof -i :3000` |
| Kill process on port | `sudo kill -9 $(lsof -t -i:3000)` |
| Check listening ports | `ss -tulpn` |
| Test port connectivity | `nc -zv localhost 3000` |

### Common Error Fixes
| Error | Fix |
|-------|-----|
| 502 Bad Gateway | Restart app service |
| 403 Forbidden | Fix file permissions |
| EADDRINUSE | Change port or kill process |
| EACCES on port 80 | Use port 3000 + Nginx proxy |

---

This guide provides a complete foundation for deploying and maintaining production applications on Ubuntu with Nginx and systemd. Bookmark this reference for quick troubleshooting during incidents.
