Deploying a web application to production requires more than just installing Nginx—it demands a secure, reliable configuration that can handle real-world traffic. This guide will walk you through a complete production deployment on Ubuntu, acting as your reverse proxy and SSL terminator.

The core principle of this setup is using Nginx as a **gateway**. It accepts web traffic from the internet, manages the HTTPS encryption, and then securely forwards requests to your internal application, which runs on a local port (like 3000 for a Node.js app). This keeps your application isolated and allows Nginx to efficiently serve static files and handle high concurrency.

### 📝 Step-by-Step Deployment Guide

Here is the complete process, from server preparation to securing your site with a free SSL certificate.

#### 1. Install Nginx and Configure the Firewall
First, update your package list and install Nginx from the official Ubuntu repository.
```bash
sudo apt update
sudo apt install nginx -y
```
Nginx will start automatically. We need to enable it to run on boot and then adjust the firewall.
```bash
sudo systemctl enable nginx
sudo ufw allow 'Nginx Full' # Opens both HTTP (80) and HTTPS (443) ports
sudo ufw allow OpenSSH      # Ensure your SSH connection is not blocked
sudo ufw enable
```
The `'Nginx Full'` profile is a convenient way to open the standard web ports without remembering the numbers. You can verify Nginx is running correctly by visiting your server's public IP address in a web browser; you should see the Nginx welcome page.

#### 2. Create an Nginx Server Block (Virtual Host)
Nginx uses "server blocks" (similar to Apache's virtual hosts) to manage different websites. Instead of modifying the default file, we create our own for better organization.

First, create a directory for your website's files and a simple `index.html` to test with. Replace `your-domain.com` with your actual domain name.
```bash
sudo mkdir -p /var/www/your-domain.com/html
sudo chown -R $USER:$USER /var/www/your-domain.com/html
echo "<h1>Hello from your-domain.com</h1>" | sudo tee /var/www/your-domain.com/html/index.html
```
Now, create a new configuration file for your site in the `sites-available` directory.
```bash
sudo nano /etc/nginx/sites-available/your-domain.com
```
Paste the following configuration. This tells Nginx to listen for requests to your domain and serve files from the directory you just created.
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name your-domain.com www.your-domain.com;

    root /var/www/your-domain.com/html;
    index index.html index.htm index.nginx-debian.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
To activate this configuration, you create a symbolic link from `sites-available` to `sites-enabled`.
```bash
sudo ln -s /etc/nginx/sites-available/your-domain.com /etc/nginx/sites-enabled/
sudo nginx -t               # Always test for syntax errors!
sudo systemctl reload nginx # Reload to apply changes
```

#### 3. Set Up a Reverse Proxy for Your Application
For a production app, you'll use Nginx as a reverse proxy. The configuration below assumes your application is running locally on port `3000`. Adjust the `proxy_pass` URL if your app uses a different port.

Create a new configuration file for your application or modify your existing one. This example is for `api.your-domain.com`.
```bash
sudo nano /etc/nginx/sites-available/api.your-domain.com
```
We already have that which you can access at this location
```bash
sudo nano /etc/nginx/sites-available/global
```
Add this configuration to proxy all incoming traffic to your backend app.
```nginx
server {
    listen 80;
    server_name api.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000; # Your app's address
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
Enable the site and reload Nginx just like in the previous step.
```bash
sudo nginx -t               # Always test for syntax errors!
sudo systemctl reload nginx # Reload to apply changes
```
#### 4. Secure Your Site with SSL/TLS (Let's Encrypt)
You should never run a production site on plain HTTP. We'll use Certbot, a tool that automates getting and installing a free SSL certificate from Let's Encrypt.

First, install Certbot and its Nginx plugin.
```bash
sudo apt install certbot python3-certbot-nginx -y
```
Then, run Certbot. It will automatically modify your Nginx configuration to enable HTTPS and set up a redirect from HTTP to HTTPS. Replace the domain names with your own.
```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```
Follow the prompts. Certbot will also set up an automatic renewal job, so you don't have to worry about the certificate expiring. You can test the renewal process with `sudo certbot renew --dry-run`.

#### 5. Optimize for Production
Before considering your server production-ready, a few performance tweaks make a big difference. Edit the main Nginx configuration file:
```bash
sudo nano /etc/nginx/nginx.conf
```
- **Enable Gzip Compression:** Inside the `http { ... }` block, add or uncomment lines to enable gzip, which compresses text-based files for faster downloads.
   ```nginx
   gzip on;
   gzip_vary on;
   gzip_proxied any;
   gzip_comp_level 6;
   gzip_types text/plain text/css text/xml text/javascript application/javascript application/json application/xml+rss;
   ```
- **Set Worker Processes:** Ensure `worker_processes` is set to `auto`, allowing Nginx to use all your server's CPU cores.
   ```nginx
   worker_processes auto;
   ```
- **Browser Caching for Static Assets:** Add this block inside a `server { ... }` block to tell browsers to cache images, CSS, and JS files locally for 30 days, drastically reducing load times for repeat visitors.
   ```nginx
   location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2)$ {
       expires 30d;
       add_header Cache-Control "public, no-transform";
   }
   ```
After making any changes to `global file`, always test and reload: `sudo nginx -t && sudo systemctl reload nginx`.

---

### ⚠️ Common Production Blockers and Fixes

Even careful setups can run into issues. Here are the most common problems and how to solve them.

| Issue | Likely Cause | How to Diagnose & Fix |
| :--- | :--- | :--- |
| **`502 Bad Gateway`** | Your backend application isn't running, or the `proxy_pass` URL is wrong. | 1. Check if your app is running (`systemctl status myapp` or `ps aux | grep node`).<br>2. Verify the port in your Nginx `proxy_pass` line (e.g., `http://localhost:3000;`) matches your app's port. |
| **`403 Forbidden`** | Incorrect file permissions for the website root directory. | Nginx runs as the `www-data` user. Ensure it has read access to your files: `sudo chown -R www-data:www-data /var/www/your-domain.com` and `sudo chmod -R 755 /var/www/your-domain.com`. |
| **`Address already in use`** | Another service (like Apache) is using port 80 or 443. | Stop the conflicting service, or change Nginx's `listen` port. Find the process with `sudo netstat -tulnp | grep ':80\|:443'` and stop it with `sudo systemctl stop apache2`. |
| **Configuration changes not applying** | Nginx wasn't reloaded, or the site isn't enabled. | Always run `sudo nginx -t` to test, then `sudo systemctl reload nginx`. Verify your site's symbolic link exists: `ls -la /etc/nginx/sites-enabled/`. |
| **Nginx fails to start/install** | A syntax error in a configuration file. | Run `sudo nginx -t`. The output will tell you exactly which file and line number the error is on, making it easy to fix. |
| **Let's Encrypt certificate fails** | Your domain's DNS doesn't point to the server's IP address. | Wait for DNS propagation to complete (can take from a few minutes to 48 hours). Ensure your domain's A record points to your server's public IP. |

### 📈 Monitoring and Maintenance for the Long Run

A production server isn't a "set and forget" machine. Incorporate these practices to keep it healthy:
- **Watch the Logs:** Logs are your best friend for debugging.
    - Access Log: `sudo tail -f /var/log/nginx/access.log`
    - Error Log: `sudo tail -f /var/log/nginx/error.log`.
- **Manage Disk Space:** Log files can grow large over time. Ubuntu's `logrotate` handles this, but you can verify your settings in `/etc/logrotate.d/nginx`.
- **Security Hardening:** Consider disabling password authentication for SSH and only using SSH keys. You can also set up `unattended-upgrades` for automatic security updates.
- **Test Config Before Every Reload:** This is the single most important habit. The command `sudo nginx -t` can save you from accidentally taking your site offline with a typo.







# Complete Systemd Service Documentation for Production Deployment

This guide covers creating a robust systemd service for your application, integrating it with Nginx, and troubleshooting common production blockers like port binding errors and permission issues.

## 📋 Table of Contents
- [Creating a Systemd Service File](#-creating-a-systemd-service-file)
- [Managing Your Service](#-managing-your-service)
- [Integrating with Nginx](#-integrating-with-nginx)
- [Common Production Blockers & Fixes](#-common-production-blockers--fixes)
- [Monitoring & Debugging](#-monitoring--debugging)

---

## 🚀 Creating a Systemd Service File

First, create a service file for your application in `/etc/systemd/system/`. Name it appropriately (e.g., `myapp.service`).

```bash
sudo nano /etc/systemd/system/myapp.service
```

### Basic Service Configuration

Here's a production-ready template:

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

### Critical Configuration Explanation

| Directive | Purpose |
|-----------|---------|
| `After=network-online.target` | Ensures network is fully configured before starting  |
| `Wants=network-online.target` | Pulls in the network-online target as a dependency  |
| `User=` & `Group=` | Runs service with least privileges - **critical for security**  |
| `Restart=on-failure` | Automatically restarts if the app crashes |
| `RestartSec=10` | Wait 10 seconds before restarting |
| `NoNewPrivileges=true` | Prevents privilege escalation  |

---

## 🔧 Managing Your Service

After creating the service file, use these commands to manage it:

```bash
# Reload systemd to recognize the new service
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable myapp.service

# Start the service
sudo systemctl start myapp.service

# Check service status
sudo systemctl status myapp.service

# View real-time logs
sudo journalctl -u myapp.service -f

# Stop the service
sudo systemctl stop myapp.service

# Restart the service
sudo systemctl restart myapp.service
```

---

## 🔄 Integrating with Nginx

Your systemd service should run your application on a high port (e.g., 3000, 5000, 8080), then Nginx acts as a reverse proxy.

### Service Configuration (Running on Port 3000)

```ini
[Service]
Environment=PORT=3000
ExecStart=/usr/bin/node /var/www/myapp/server.js
```

### Nginx Proxy Configuration

```nginx
server {
    listen 80;
    server_name api.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
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

This architecture eliminates port binding issues entirely since the app runs on an unprivileged port (>1024).

---

## ⚠️ Common Production Blockers & Fixes

### 1. **Port Binding Error: "Permission denied" or "Address already in use"**

**Symptoms:**
```
Error: listen EACCES: permission denied 0.0.0.0:80
```

**Why this happens:** Linux prevents non-root processes from binding to ports below 1024 (privileged ports) .

**Solutions (choose one):**

**Option A: Use High Port + Nginx (RECOMMENDED)**
```ini
# In your service file
Environment=PORT=3000
```
Then proxy through Nginx as shown above.

**Option B: Grant CAP_NET_BIND_SERVICE capability**
```bash
# Add capability to your binary
sudo setcap 'cap_net_bind_service+ep' /usr/bin/node

# Then in service file:
[Service]
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
```
**Note:** This doesn't work reliably across all Node.js versions .

**Option C: Run as root (NOT RECOMMENDED)**
```ini
[Service]
User=root  # Security risk - avoid in production
```

### 2. **Service Fails to Start: "Failed at step USER spawning"**

**Symptoms:**
```
myapp.service: Failed at step USER spawning /usr/bin/node: No such process
```

**Cause:** The user specified in `User=` doesn't exist or has invalid shell configuration .

**Fix:**
```bash
# Check if user exists
id www-data

# If user doesn't exist, create it
sudo useradd -r -s /bin/false www-data

# OR use DynamicUser for automatic creation
[Service]
DynamicUser=yes  # systemd creates temporary user at runtime
```

### 3. **Application Can't Write Files**

**Symptoms:**
```
Error: EACCES: permission denied, open '/var/log/myapp.log'
```

**Fix:** Ensure the running user has write permissions:
```bash
# Change ownership to the service user
sudo chown -R www-data:www-data /var/www/myapp
sudo chown -R www-data:www-data /var/log/myapp

# Or set appropriate permissions
sudo chmod 755 /var/www/myapp
sudo chmod 775 /var/log/myapp
```

### 4. **Environment Variables Not Loading**

**Symptoms:** App can't find database credentials, API keys, etc.

**Solution A: Inline Environment Variables**
```ini
[Service]
Environment=NODE_ENV=production
Environment=DB_PASSWORD=securepassword123
Environment=API_KEY=abc123def456
```

**Solution B: Environment File (RECOMMENDED for production)**
```bash
# Create environment file
sudo nano /etc/myapp/env
```
```
NODE_ENV=production
DB_PASSWORD=securepassword123
API_KEY=abc123def456
```
```ini
# In service file
[Service]
EnvironmentFile=/etc/myapp/env
```
**Note:** Set restrictive permissions on this file:
```bash
sudo chmod 600 /etc/myapp/env
sudo chown www-data:www-data /etc/myapp/env
```

### 5. **Service Starts But Immediately Exits**

**Symptoms:**
```
● myapp.service - My Application
     Active: failed (Result: exit-code)
```

**Debugging:**
```bash
# Check detailed logs
sudo journalctl -u myapp.service -n 50 --no-pager

# Common issues:
# - Missing dependencies (check WorkingDirectory)
# - Syntax errors in your application
# - Port already in use by another process
```

### 6. **Port Conflicts with Multiple Services**

**Symptoms:**
```
Error: listen EADDRINUSE: address already in use :::3000
```

**Fix:** Find and resolve the conflict:
```bash
# Check what's using the port
sudo lsof -i :3000
sudo netstat -tulpn | grep :3000

# Either kill the conflicting process
sudo kill -9 <PID>

# OR change your app's port
[Service]
Environment=PORT=3001
```

### 7. **Service Hangs on Shutdown**

**Symptoms:** `systemctl stop` takes too long or times out.

**Fix:** Add timeout and kill mode:
```ini
[Service]
TimeoutStopSec=30
KillMode=mixed
KillSignal=SIGTERM
```

---

## 📊 Monitoring & Debugging

### Essential Journalctl Commands

```bash
# View all logs for your service
sudo journalctl -u myapp.service

# Follow logs in real-time
sudo journalctl -u myapp.service -f

# Show last 100 lines
sudo journalctl -u myapp.service -n 100

# Show logs from last hour
sudo journalctl -u myapp.service --since "1 hour ago"

# Show logs with priority ERROR or higher
sudo journalctl -u myapp.service -p err

# Show logs from current boot only
sudo journalctl -u myapp.service -b

# Combine with other filters
sudo journalctl -u myapp.service -u nginx.service --since today
```

### Service Status Checklist

Run this diagnostic command regularly:
```bash
sudo systemctl status myapp.service --no-pager
```

Check for:
- **Loaded:** Should show the correct service file path
- **Active:** Should be `active (running)`
- **Main PID:** Should show a valid process ID
- **Tasks:** Should show reasonable numbers (not thousands)
- **Memory:** Should be within expected limits
- **CGroup:** Should show correct slice

### Health Check Script

Add this to your service for automatic health monitoring:

```ini
[Service]
# Health check via curl (if your app has an endpoint)
ExecStartPost=/bin/bash -c 'sleep 5 && curl -f http://localhost:3000/health || exit 1'
Restart=on-failure
RestartSec=10
StartLimitInterval=300
StartLimitBurst=5
```

---

## 🛡️ Production Security Hardening

Add these security directives to your service file:

```ini
[Service]
# Restrict system calls
SystemCallFilter=~@privileged @resources

# Limit file system access
ProtectSystem=strict
ReadWritePaths=/var/www/myapp /var/log/myapp

# Prevent network namespace manipulation
PrivateNetwork=no  # Set to 'yes' if no network access needed

# Protect kernel tunables
ProtectKernelTunables=true

# Protect kernel modules
ProtectKernelModules=true

# Restrict address families
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Memory limits
MemoryMax=512M
MemoryHigh=384M

# CPU limits
CPUQuota=200%
```

---

## 🔄 Complete Production-Ready Example

Here's a comprehensive service file combining all best practices:

```ini
[Unit]
Description=My Production Application
Documentation=https://docs.myapp.com
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=www-data
Group=www-data
DynamicUser=no
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
EnvironmentFile=-/etc/myapp/secrets.env
EnvironmentFile=-/etc/myapp/local.env

# Port binding (if needed for ports <1024)
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE
# AmbientCapabilities=CAP_NET_BIND_SERVICE

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

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

## 📝 Quick Troubleshooting Reference

| Problem | Most Likely Fix |
|---------|----------------|
| "Permission denied" on port 80/443 | Use high port (3000) + Nginx proxy |
| Service won't start at boot | Run `sudo systemctl enable myapp.service` |
| "User doesn't exist" | Use `DynamicUser=yes` or create the user |
| Environment variables missing | Use `EnvironmentFile=` or inline `Environment=` |
| App crashes on startup | Check logs: `journalctl -u myapp.service -n 50` |
| Can't write files | Fix ownership: `sudo chown -R www-data:www-data /path` |
| Slow shutdown | Add `TimeoutStopSec=30` and `KillMode=mixed` |

---

This setup provides a robust, production-ready foundation for running your application behind Nginx. The systemd service handles process management, automatic restarts, and security isolation, while Nginx manages SSL, static files, and external traffic routing.



# Ubuntu Production Troubleshooting: Essential Tools & Techniques

This guide covers the most common troubleshooting methods and command-line tools you'll need when debugging production issues on Ubuntu. Think of these as your diagnostic toolkit - each tool solves specific types of problems.

## 🔍 Core Diagnostic Tools

### 1. **grep - Pattern Searching Master**

`grep` is your primary tool for finding specific text in files or command output. It's essential for log analysis.

**Basic Syntax:**
```bash
grep [options] pattern [file]
```

**Common Production Uses:**

```bash
# Find errors in Nginx logs
sudo grep "ERROR" /var/log/nginx/error.log

# Find specific IP address in access logs
sudo grep "192.168.1.100" /var/log/nginx/access.log

# Search recursively in all log files
sudo grep -r "OutOfMemory" /var/log/

# Show line numbers (helpful for context)
sudo grep -n "failed" /var/log/syslog

# Case-insensitive search
sudo grep -i "warning" /var/log/myapp.log

# Show 3 lines before and after match
sudo grep -B 3 -A 3 "segmentation fault" /var/log/app.log

# Exclude specific patterns
sudo grep -v "INFO" /var/log/app.log | grep "ERROR"

# Count occurrences
sudo grep -c "500" /var/log/nginx/access.log

# Multiple patterns (OR)
sudo grep -E "error|failed|critical" /var/log/syslog

# Only show matching part (not entire line)
sudo grep -o "IP: [0-9]*\.[0-9]*\.[0-9]*\.[0-9]*" /var/log/nginx/access.log

# Live monitoring with grep (real-time filtering)
sudo journalctl -u myapp -f | grep -i error
```

### 2. **journalctl - Systemd Log Viewer**

Your primary source for application and system logs when using systemd.

```bash
# View all logs for a service
sudo journalctl -u myapp.service

# Follow live logs (like tail -f)
sudo journalctl -u myapp.service -f

# Show last 100 lines
sudo journalctl -u myapp.service -n 100

# Filter by priority
sudo journalctl -u myapp.service -p err   # errors only
sudo journalctl -u myapp.service -p warning..emerg  # warnings and above

# Time-based filtering
sudo journalctl -u myapp.service --since "2024-01-15 10:00:00"
sudo journalctl -u myapp.service --since "1 hour ago"
sudo journalctl -u myapp.service --since yesterday --until today

# Output as JSON (for parsing)
sudo journalctl -u myapp.service -o json-pretty

# Show logs from current boot only
sudo journalctl -b

# Combine with grep
sudo journalctl -u myapp.service | grep "database connection"

# Monitor multiple services
sudo journalctl -u nginx -u myapp -f
```

### 3. **systemctl - Service Management**

Debug service-related issues.

```bash
# Check service status (most important command)
sudo systemctl status myapp.service

# List all failed services
sudo systemctl --failed

# Show dependency tree
sudo systemctl list-dependencies myapp.service

# View service environment
sudo systemctl show myapp.service

# Check if service is enabled for boot
sudo systemctl is-enabled myapp.service

# Check if service is running
sudo systemctl is-active myapp.service

# View service file location and content
sudo systemctl cat myapp.service

# Reload systemd after service file changes
sudo systemctl daemon-reload
```

---

## 🌐 Network & Port Troubleshooting

### 4. **netstat / ss - Network Statistics**

Critical for identifying port conflicts and network connections.

```bash
# Show all listening ports (netstat)
sudo netstat -tulpn

# Show all listening ports (ss - modern replacement, faster)
sudo ss -tulpn

# Find what's using port 3000
sudo netstat -tulpn | grep :3000
sudo ss -tulpn | grep :3000

# Show all TCP connections with process IDs
sudo netstat -tpn

# Show listening ports with program names
sudo netstat -tulpn | grep LISTEN

# Show all connections for a specific process
sudo netstat -p | grep "nginx"

# Show routing table (for network issues)
ip route show
sudo netstat -rn
```

### 5. **lsof - List Open Files**

Everything in Linux is a file - including network connections.

```bash
# Find what's using a specific port
sudo lsof -i :3000

# List all open files by a process
sudo lsof -p PID

# List processes using a specific file
sudo lsof /var/log/myapp.log

# Show all network connections
sudo lsof -i

# Show TCP connections only
sudo lsof -i tcp

# List open files for a specific user
sudo lsof -u www-data

# Combine with grep for specific patterns
sudo lsof -i | grep ESTABLISHED
```

### 6. **curl / wget - HTTP Testing**

Test your web endpoints directly from the server.

```bash
# Basic HTTP GET
curl http://localhost:3000/health

# Show response headers only
curl -I http://localhost:3000

# Follow redirects
curl -L http://example.com

# Show detailed connection information
curl -v http://localhost:3000

# Test SSL/TLS
curl -vI https://your-domain.com

# Check response time
curl -w "Connect: %{time_connect}s TTFB: %{time_starttransfer}s Total: %{time_total}s\n" -o /dev/null -s http://localhost:3000

# Send POST request
curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' http://localhost:3000/api

# Test with custom User-Agent
curl -A "Mozilla/5.0" http://localhost:3000

# Check if port is accessible (telnet-style)
curl telnet://localhost:3000
```

---

## 💾 Disk & Filesystem Tools

### 7. **df / du - Disk Usage Analysis**

Prevent disk full errors that crash applications.

```bash
# Show disk usage in human-readable format
df -h

# Show inode usage (often forgotten but critical)
df -i

# Show disk usage for specific directory
du -sh /var/log/

# Show largest directories (top 10)
sudo du -ah /var/ | sort -rh | head -10

# Show filesystem type
df -T

# Show only specific filesystem
df -h /dev/sda1
```

### 8. **find - Search for Files**

Locate configuration files, logs, or problematic files.

```bash
# Find files modified in last hour
find /var/log -mmin -60

# Find files larger than 100MB
find / -type f -size +100M -exec ls -lh {} \;

# Find files by name (case-insensitive)
find /etc -iname "nginx*.conf"

# Find files with specific permissions
find /var/www -perm 777

# Find and delete old log files
find /var/log -name "*.log" -mtime +30 -delete

# Find files owned by specific user
find /home -user www-data
```

---

## 🚀 Process & Performance Monitoring

### 9. **top / htop - Real-time Process Monitoring**

```bash
# Standard top (usually pre-installed)
top
# Inside top: Press '1' for per-CPU, 'M' sort by memory, 'P' sort by CPU

# Better alternative (install with: sudo apt install htop)
htop

# Show processes for specific user
top -u www-data

# Batch mode for scripting
top -b -n 1 | grep "nginx"
```

### 10. **ps - Process Status**

```bash
# Show all processes for a user
ps aux | grep www-data

# Tree view of processes
ps auxf

# Show only specific columns
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head

# Show processes with their environment variables
ps ewww -p PID

# Show thread information
ps -Lf PID
```

### 11. **strace - System Call Tracer**

Debug what a process is actually doing (heavy but powerful).

```bash
# Trace a running process
sudo strace -p PID

# Trace file operations only
sudo strace -e trace=file -p PID

# Trace network operations only
sudo strace -e trace=network -p PID

# Trace a new process from start
sudo strace -f -o /tmp/strace.out node server.js

# Follow forks and show timestamps
sudo strace -f -t -p PID
```

---

## 🛡️ Security & Permission Tools

### 12. **ls -la / chmod / chown - Permission Inspection**

```bash
# Check file permissions
ls -la /var/www/myapp/

# Check directory permissions with context
ls -laZ /var/www/  # Shows SELinux context

# Find files with world-writable permissions
find /var/www -perm -o+w -type f

# Check file ownership
stat /var/www/myapp/config.js

# Compare expected vs actual permissions
# Good: directory 755, file 644, owned by www-data
```

### 13. **sudo -l / whoami - User Context**

```bash
# Check what sudo privileges you have
sudo -l

# Check current user
whoami

# Check which groups user belongs to
groups www-data

# Check process running context
ps aux | grep nginx | grep -v grep
```

---

## 📋 Log File Locations & Common Grep Patterns

### Nginx Logs
```bash
# Error log
sudo grep -i "error" /var/log/nginx/error.log

# Access log - find 500 errors
sudo grep " 500 " /var/log/nginx/access.log

# Access log - find slow requests (>1 second)
sudo awk '$NF > 1' /var/log/nginx/access.log

# Access log - count requests by IP
sudo grep -o "^[0-9]*\.[0-9]*\.[0-9]*\.[0-9]*" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### System Logs
```bash
# Authentication failures
sudo grep "Failed password" /var/log/auth.log

# Out of memory events
sudo grep -i "out of memory" /var/log/kern.log

# Service start/stop events
sudo grep -i "start\|stop" /var/log/syslog | grep "myapp"
```

### Application Logs
```bash
# Find stack traces (often multiple lines)
sudo grep -A 20 "Exception:" /var/log/myapp.log

# Count error types
sudo grep -o "ERROR.*" /var/log/myapp.log | sort | uniq -c | sort -nr

# Find patterns in JSON logs
sudo grep '"level":"error"' /var/log/myapp.log | jq '.message'
```

---

## 🔧 Essential One-Liner Commands for Production

### Quick Health Checks
```bash
# Check if Nginx is running
systemctl is-active nginx && echo "OK" || echo "DOWN"

# Check if port is responding
nc -zv localhost 3000 && echo "Port open" || echo "Port closed"

# Check disk space alert
[ $(df -h / | awk 'NR==2 {print $5}' | sed 's/%//') -gt 85 ] && echo "ALERT: Disk >85%"

# Check memory usage
free -h | awk 'NR==2{printf "Memory Usage: %.2f%%\n", $3*100/$2}'

# Check load average
uptime | awk -F 'load average:' '{print "Load:", $2}'
```

### Performance Investigation
```bash
# Find top 10 CPU-consuming processes
ps aux --sort=-%cpu | head -11

# Find top 10 memory-consuming processes
ps aux --sort=-%mem | head -11

# Show I/O waiting processes
iotop -b -n 1 | grep -v "DISK"

# Show open file limits
lsof | wc -l
ulimit -n
```

### Log Analysis Patterns
```bash
# Extract unique error messages
sudo grep -i error /var/log/nginx/error.log | cut -d: -f4- | sort | uniq -c | sort -nr

# Show errors per hour
sudo grep -i error /var/log/nginx/error.log | cut -d'[' -f2 | cut -d']' -f1 | cut -d' ' -f1 | sort | uniq -c

# Find IPs making too many requests
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10
```

---

## 🚨 Quick Diagnostic Workflow

When something breaks, follow this sequence:

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

# 7. Check for permission issues
ls -la /var/www/myapp/
sudo -u www-data test -w /var/log/myapp/

# 8. Monitor in real-time during issue
sudo journalctl -u myapp -f &
curl http://localhost:3000/trigger-error
fg  # bring journalctl back to foreground
```

## 📝 Pro Tips

1. **Always use `sudo` with logs** - Many log files aren't readable by regular users
2. **Combine commands with pipes** - `grep` + `awk` + `sort` + `uniq` = powerful analysis
3. **Save output for comparison** - `sudo journalctl -u myapp > before.log` then compare with `after.log`
4. **Use aliases** for common commands:
   ```bash
   alias logs='sudo journalctl -u myapp -f'
   alias nginx-logs='sudo tail -f /var/log/nginx/error.log'
   alias check-port='sudo ss -tulpn | grep'
   ```
5. **Use `watch` for real-time monitoring**:
   ```bash
   watch -n 1 'systemctl status myapp | grep Active'
   watch -n 5 'ss -tulpn | grep :3000'
   ```

This toolkit covers 99% of production troubleshooting scenarios. Master these tools and you'll debug most issues in minutes rather than hours.
