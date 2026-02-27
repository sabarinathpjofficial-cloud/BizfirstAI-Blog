# Deploy .NET 8 Blog to Ubuntu Server - Complete Guide

This guide provides step-by-step instructions for deploying your Blogifier .NET 8 application to Ubuntu 22.04 with Kestrel, systemd, Nginx, and Let's Encrypt SSL.

## Prerequisites
- Ubuntu 22.04 server
- SSH access to the server
- Domain name (blog.bizfirstai.com) pointing to your server IP
- Published .NET 8 application ready locally

---

## Step 1: Server Setup and .NET Runtime Installation

### 1.1 Update System Packages
```bash
sudo apt update
sudo apt upgrade -y
```

### 1.2 Install .NET 8 Runtime
```bash
# Add Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# Install .NET 8 Runtime
sudo apt update
sudo apt install -y aspnetcore-runtime-8.0

# Verify installation
dotnet --list-runtimes
```

### 1.3 Install Nginx
```bash
sudo apt install -y nginx
```

---

## Step 2: Create Application Directory

```bash
# Create application directory
sudo mkdir -p /var/www/blog

# Create a user for running the application
sudo useradd -r -s /bin/false blogapp

# Set ownership
sudo chown -R blogapp:blogapp /var/www/blog
```

---

## Step 3: Upload Published Application

### Option A: Using SCP from your local machine
```bash
# From your local machine (where publish folder is)
scp -r ./publish/* your-username@your-server-ip:/tmp/blog-publish/

# Then on the server, move files
sudo mv /tmp/blog-publish/* /var/www/blog/
sudo chown -R blogapp:blogapp /var/www/blog
sudo chmod 755 /var/www/blog
```

### Option B: Using rsync
```bash
# From your local machine
rsync -avz ./publish/ your-username@your-server-ip:/tmp/blog-publish/

# Then on the server
sudo mv /tmp/blog-publish/* /var/www/blog/
sudo chown -R blogapp:blogapp /var/www/blog
sudo chmod 755 /var/www/blog
```

---

## Step 4: Create systemd Service

### 4.1 Create Service File
```bash
sudo nano /etc/systemd/system/blog.service
```

### 4.2 Service File Content
```ini
[Unit]
Description=Blogifier .NET 8 Web Application
After=network.target

[Service]
Type=notify
# Change WorkingDirectory to your actual publish folder
WorkingDirectory=/var/www/blog
# Change ExecStart to point to your DLL (replace YourAppName.dll with actual name)
ExecStart=/usr/bin/dotnet /var/www/blog/Blogifier.dll

# Restart service after 10 seconds if it crashes
Restart=always
RestartSec=10

# User and Group
User=blogapp
Group=blogapp

# Environment variables
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
Environment=ASPNETCORE_URLS=http://localhost:5000

# Security settings
NoNewPrivileges=true
PrivateTmp=true

# Logging
SyslogIdentifier=blog-app
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### 4.3 Enable and Start Service
```bash
# Reload systemd to recognize new service
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable blog.service

# Start the service
sudo systemctl start blog.service

# Check service status
sudo systemctl status blog.service
```

---

## Step 5: Configure Nginx Reverse Proxy

### 5.1 Create Nginx Configuration
```bash
sudo nano /etc/nginx/sites-available/blog.bizfirstai.com
```

### 5.2 Nginx Configuration Content (HTTP - Before SSL)
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name blog.bizfirstai.com www.blog.bizfirstai.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;

        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Client body size limit (adjust for file uploads)
    client_max_body_size 10M;
}
```

### 5.3 Enable Site and Test Configuration
```bash
# Create symbolic link to enable site
sudo ln -s /etc/nginx/sites-available/blog.bizfirstai.com /etc/nginx/sites-enabled/

# Remove default site if exists
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

---

## Step 6: Configure SSL with Let's Encrypt

### 6.1 Install Certbot
```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 6.2 Obtain SSL Certificate
```bash
# Get certificate and auto-configure Nginx
sudo certbot --nginx -d blog.bizfirstai.com -d www.blog.bizfirstai.com

# Follow the prompts:
# - Enter your email address
# - Agree to terms of service
# - Choose whether to redirect HTTP to HTTPS (recommended: yes)
```

### 6.3 Verify Auto-Renewal
```bash
# Test renewal process
sudo certbot renew --dry-run

# Certbot automatically sets up a cron job/systemd timer for renewal
```

### 6.4 Final Nginx Configuration (After Certbot)
Certbot will automatically update your Nginx config. It should look like this:

```nginx
server {
    server_name blog.bizfirstai.com www.blog.bizfirstai.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    client_max_body_size 10M;

    listen [::]:443 ssl ipv6only=on;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/blog.bizfirstai.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.bizfirstai.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = www.blog.bizfirstai.com) {
        return 301 https://$host$request_uri;
    }

    if ($host = blog.bizfirstai.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    listen [::]:80;
    server_name blog.bizfirstai.com www.blog.bizfirstai.com;
    return 404;
}
```

---

## Step 7: Configure Firewall (UFW)

```bash
# Enable UFW if not already enabled
sudo ufw status

# Allow SSH (IMPORTANT - do this first!)
sudo ufw allow OpenSSH

# Allow HTTP and HTTPS
sudo ufw allow 'Nginx Full'

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

---

## Step 8: Useful Commands for Management

### Application Service Commands
```bash
# Start the application
sudo systemctl start blog.service

# Stop the application
sudo systemctl stop blog.service

# Restart the application
sudo systemctl restart blog.service

# Check service status
sudo systemctl status blog.service

# Enable auto-start on boot
sudo systemctl enable blog.service

# Disable auto-start on boot
sudo systemctl disable blog.service
```

### View Logs
```bash
# View application logs (real-time)
sudo journalctl -u blog.service -f

# View last 100 lines of logs
sudo journalctl -u blog.service -n 100

# View logs since today
sudo journalctl -u blog.service --since today

# View logs from specific time
sudo journalctl -u blog.service --since "2024-01-01 00:00:00"

# View Nginx error logs
sudo tail -f /var/log/nginx/error.log

# View Nginx access logs
sudo tail -f /var/log/nginx/access.log
```

### Nginx Commands
```bash
# Test Nginx configuration
sudo nginx -t

# Reload Nginx (without downtime)
sudo systemctl reload nginx

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx
```

### SSL Certificate Commands
```bash
# List certificates
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Test renewal process
sudo certbot renew --dry-run
```

---

## Step 9: Verify Deployment

### 9.1 Check if Application is Running
```bash
# Check if .NET app is listening on port 5000
sudo netstat -tlnp | grep 5000

# Or using ss
sudo ss -tlnp | grep 5000

# Check if service is active
sudo systemctl is-active blog.service
```

### 9.2 Test HTTP/HTTPS Access
```bash
# Test local connection
curl http://localhost:5000

# Test through Nginx
curl http://blog.bizfirstai.com

# Test HTTPS
curl https://blog.bizfirstai.com
```

### 9.3 Browser Test
Visit `https://blog.bizfirstai.com` in your browser

---

## Step 10: Production Best Practices

### 10.1 Database Backup (if using SQLite/Local DB)
```bash
# Create backup script
sudo nano /usr/local/bin/backup-blog.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/blog"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup database (adjust path as needed)
cp /var/www/blog/app.db $BACKUP_DIR/app_$DATE.db

# Keep only last 7 days of backups
find $BACKUP_DIR -name "app_*.db" -mtime +7 -delete
```

```bash
# Make executable
sudo chmod +x /usr/local/bin/backup-blog.sh

# Add to crontab (daily at 2 AM)
sudo crontab -e
# Add this line:
0 2 * * * /usr/local/bin/backup-blog.sh
```

### 10.2 Monitor Disk Space
```bash
# Check disk usage
df -h

# Check specific directory
du -sh /var/www/blog
```

### 10.3 Security Headers in Nginx
Add these to your Nginx server block for better security:

```nginx
# Add inside the server block
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
```

### 10.4 Rate Limiting in Nginx
```nginx
# Add before server block
limit_req_zone $binary_remote_addr zone=blog_limit:10m rate=10r/s;

# Inside location block
location / {
    limit_req zone=blog_limit burst=20 nodelay;
    # ... rest of proxy settings
}
```

---

## Troubleshooting

### Application Won't Start
```bash
# Check service status
sudo systemctl status blog.service

# View detailed logs
sudo journalctl -u blog.service -n 50 --no-pager

# Check if port is already in use
sudo netstat -tlnp | grep 5000

# Verify file permissions
ls -la /var/www/blog

# Test running manually
cd /var/www/blog
sudo -u blogapp dotnet Blogifier.dll
```

### Nginx 502 Bad Gateway
```bash
# Check if .NET app is running
sudo systemctl status blog.service

# Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# Verify proxy_pass URL matches your app's listening address
cat /etc/nginx/sites-available/blog.bizfirstai.com
```

### SSL Certificate Issues
```bash
# Check certificate status
sudo certbot certificates

# Test certificate renewal
sudo certbot renew --dry-run

# Check Nginx SSL configuration
sudo nginx -t
```

### Permission Issues
```bash
# Fix ownership
sudo chown -R blogapp:blogapp /var/www/blog

# Fix permissions
sudo chmod 755 /var/www/blog
sudo chmod 644 /var/www/blog/*
sudo chmod 755 /var/www/blog/*.dll
```

---

## Quick Reference: Complete Deployment Checklist

- [ ] Update Ubuntu packages
- [ ] Install .NET 8 Runtime
- [ ] Install Nginx
- [ ] Create application directory `/var/www/blog`
- [ ] Create `blogapp` user
- [ ] Upload published application files
- [ ] Set correct permissions
- [ ] Create and configure systemd service
- [ ] Enable and start the service
- [ ] Create Nginx configuration
- [ ] Enable Nginx site
- [ ] Test Nginx configuration
- [ ] Install Certbot
- [ ] Obtain SSL certificate
- [ ] Configure firewall (UFW)
- [ ] Verify deployment
- [ ] Set up backups
- [ ] Test application in browser

---

## Post-Deployment Updates

When you need to update your application:

```bash
# 1. Stop the service
sudo systemctl stop blog.service

# 2. Backup current version
sudo cp -r /var/www/blog /var/www/blog.backup.$(date +%Y%m%d)

# 3. Upload new files (from local machine)
scp -r ./publish/* your-username@your-server-ip:/tmp/blog-update/

# 4. Move new files (on server)
sudo rm -rf /var/www/blog/*
sudo mv /tmp/blog-update/* /var/www/blog/
sudo chown -R blogapp:blogapp /var/www/blog

# 5. Start the service
sudo systemctl start blog.service

# 6. Check status
sudo systemctl status blog.service

# 7. Check logs for any errors
sudo journalctl -u blog.service -n 50 -f
```

---

This guide should have your .NET 8 Blog application running securely on Ubuntu with HTTPS. If you encounter any issues, check the troubleshooting section and logs for more details.