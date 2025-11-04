# Reverse Proxy Setup for withdraw.notify.live → app.spacer.live

This document describes how to configure an **Nginx reverse proxy** on an Ubuntu VPS so that  
**https://withdraw.notify.live** serves content from **https://app.spacer.live**,  
while keeping the browser URL as `withdraw.notify.live`.

---

## 📦 Requirements

- Domain: `notify.live` managed via **Hostinger**
- Subdomain: `withdraw.notify.live`
- VPS: Ubuntu with **sudo** access
- Web server: **Nginx**
- SSL certificate: **Let’s Encrypt / Certbot**

---

## 🧭 Step 1 — DNS Configuration

On **Hostinger → DNS Zone Editor**, create the following record:

| Type | Name | Value | TTL |
|------|------|--------|-----|
| A | withdraw | `<your VPS public IP>` | Auto |

> **Tip:** Wait 5–10 minutes for DNS propagation.  
> Verify with:
> ```bash
> ping withdraw.notify.live
> ```
> It should return your VPS IP.

---

## ⚙️ Step 2 — Install Nginx

If Nginx is not installed yet:

```bash
sudo apt update
sudo apt install nginx -y

Confirm it’s running:

sudo systemctl status nginx


---

🧱 Step 3 — Create the Reverse Proxy Configuration

Create a new server block file:

sudo nano /etc/nginx/sites-available/withdraw.notify.live

Paste the following configuration:

server {
    listen 80;
    server_name withdraw.notify.live;

    location / {
        proxy_pass https://app.spacer.live;
        proxy_set_header Host app.spacer.live;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_server_name on;
    }
}

Save and exit (Ctrl + O, Enter, Ctrl + X).

Enable the site and reload Nginx:

sudo ln -s /etc/nginx/sites-available/withdraw.notify.live /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

At this point, http://withdraw.notify.live should load content from https://app.spacer.live.


---

🔒 Step 4 — Enable HTTPS (Let’s Encrypt)

Install Certbot:

sudo apt install certbot python3-certbot-nginx -y

Request a certificate:

sudo certbot --nginx -d withdraw.notify.live

Choose to redirect HTTP to HTTPS when prompted.

Certbot will automatically update your Nginx configuration.


Test the renewal process:

sudo certbot renew --dry-run


---

🧩 Step 5 — Final Nginx Configuration (after SSL)

Your file /etc/nginx/sites-available/withdraw.notify.live should now look like this:

server {
    listen 80;
    server_name withdraw.notify.live;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name withdraw.notify.live;

    ssl_certificate /etc/letsencrypt/live/withdraw.notify.live/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/withdraw.notify.live/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass https://app.spacer.live;
        proxy_set_header Host app.spacer.live;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_server_name on;
    }
}

Reload Nginx:

sudo systemctl reload nginx


---

✅ Step 6 — Verify

Visit:

https://withdraw.notify.live

You should see the content from:

https://app.spacer.live

…but the browser address bar should still show withdraw.notify.live.


---

🧠 Notes

Certbot auto-renewal is handled by a cron job, so certificates stay valid.

To manually renew SSL certificates:

sudo certbot renew

To view Nginx logs:

sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

To disable this proxy later:

sudo rm /etc/nginx/sites-enabled/withdraw.notify.live
sudo systemctl reload nginx



---

🛠️ Troubleshooting

1. Mixed content (HTTP assets not loading) If the proxied site serves some assets over http://, browsers may block them.
You can fix this by forcing HTTPS rewrites in your Nginx config:

proxy_redirect off;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header X-Forwarded-Proto https;

2. Infinite redirect loop If the origin (app.spacer.live) forces HTTPS or redirects requests differently,
try temporarily disabling return 301 on port 80 until you confirm behavior.

3. SSL handshake error If you see errors like 502 Bad Gateway, check:

sudo tail -f /var/log/nginx/error.log

Sometimes the origin host may not support SNI — ensure proxy_ssl_server_name on; is included.


---

🏁 Result

URL	Content Source	SSL	Notes

https://withdraw.notify.live	https://app.spacer.live	✅ Yes	Reverse proxied via Nginx
https://app.spacer.live	Original PHP host	✅ Yes	No change required



---

Author: DegenWTF
Purpose: Mirror app.spacer.live under withdraw.notify.live using Nginx reverse proxy.

---
