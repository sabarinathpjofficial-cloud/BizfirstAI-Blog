# 🚀 Quick Start: Deploy .NET 8 Blog to Ubuntu

> Get your Blogifier app live at `blog.bizfirstai.com` in ~15 minutes.

---

## Step 1 — Prepare the Server

SSH into your server and run:

```bash
sudo apt update && sudo apt upgrade -y
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O pkg.deb
sudo dpkg -i pkg.deb && rm pkg.deb
sudo apt update && sudo apt install -y aspnetcore-runtime-8.0 nginx
```

---

## Step 2 — Set Up the App Directory

```bash
sudo mkdir -p /var/www/blog
sudo useradd -r -s /bin/false blogapp
sudo chown -R blogapp:blogapp /var/www/blog
```

---

## Step 3 — Upload Your App

Run this **from your local machine** (where your `publish/` folder is):

```bash
scp -r ./publish/* your-username@your-server-ip:/tmp/blog-upload/
```

Then back on the server:

```bash
sudo mv /tmp/blog-upload/* /var/www/blog/
sudo chown -R blogapp:blogapp /var/www/blog
```

---

## Step 4 — Create a systemd Service

```bash
sudo nano /etc/systemd/system/blog.service
```

Paste this in:

```ini
[Unit]
Description=Blogifier App
After=network.target

[Service]
WorkingDirectory=/var/www/blog
ExecStart=/usr/bin/dotnet /var/www/blog/Blogifier.dll
Restart=always
User=blogapp
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://localhost:5000

[Install]
WantedBy=multi-user.target
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable blog.service
sudo systemctl start blog.service
sudo systemctl status blog.service   # should say "active (running)"
```

---

## Step 5 — Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/blog.bizfirstai.com
```

Paste this:

```nginx
server {
    listen 80;
    server_name blog.bizfirstai.com www.blog.bizfirstai.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 10M;
}
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/blog.bizfirstai.com /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 6 — Add SSL (HTTPS)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d blog.bizfirstai.com -d www.blog.bizfirstai.com   //   sudo certbot --nginx -d blog.bizfirstai.com
```

Follow the prompts — choose **redirect HTTP to HTTPS** when asked.

---

## Step 7 — Open the Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

---

## ✅ Done!

Visit **https://blog.bizfirstai.com** in your browser.

---

## Useful Commands

| Task | Command |
|------|---------|
| Check app status | `sudo systemctl status blog.service` |
| Restart app | `sudo systemctl restart blog.service` |
| View live logs | `sudo journalctl -u blog.service -f` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Renew SSL cert | `sudo certbot renew --dry-run` |

---

## Updating the App Later

```bash
sudo systemctl stop blog.service
# Upload new files from local:
#   scp -r ./publish/* user@server-ip:/tmp/blog-update/
sudo rm -rf /var/www/blog/* && sudo mv /tmp/blog-update/* /var/www/blog/
sudo chown -R blogapp:blogapp /var/www/blog
sudo systemctl start blog.service
```
