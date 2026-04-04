# Complete VPS Deployment Guide (Docker + DuckDNS + SSL)

This guide takes your ScribbleHub scraper from a local Docker container to a secure, internet-facing application with full HTTPS encryption.

---

## Step 1: Prepare Your DuckDNS Domain
1. Log into [DuckDNS](https://www.duckdns.org/) using your Google/Reddit/GitHub account.
2. In the "Domains" section, choose an available sub-domain (e.g. `mynovelapp`). Click **Add Domain**.
3. It will now show up in your list like `mynovelapp.duckdns.org`.
4. Grab the **"Current IP"** field next to your domain, and **change it to your VPS's public IP Address**. Click the "Update IP" button. 

It takes roughly 5 to 10 minutes for internet DNS tables to recognize the change.

---

## Step 2: Configure Environment Security on VPS
When you upload your files to the VPS (via `git clone` or SFTP), it's crucial you correctly restrict the hosts inside `.env` so other web-scanners can't spam your API.

Create/update your `.env` file to look like this:
```env
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx

# SECURITY: Set the exact duckdns domain you registered (do not include http:// or ports)
ALLOWED_HOSTS=mynovelapp.duckdns.org
ALLOWED_ORIGINS=https://mynovelapp.duckdns.org
```

---

## Step 3: Start your Docker Container natively
Ensure you are inside the `scrapers/scribblehub/` folder on your VPS.

Run your container on an internal port mapping (You already set `8600` inside your `docker-compose.yml`, which is perfect!):
```bash
docker compose up -d --build
```
Your internal server is now listening on `<VPS-IP>:8600`. The outside world cannot hit this over standard web ports yet securely.

---

## Step 4: Install NGINX & Certbot
We are going to put NGINX in front of the docker container. It will intercept internet traffic (Port 80/443), apply HTTPS SSL security, and seamlessly forward it strictly into your Docker internal port (8600).

Run this on your VPS Ubuntu/Debian terminal:
```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
```

---

## Step 5: Configure NGINX Reverse Proxy
We need to create a server block configuration for NGINX. 

1. Create a config file for your domain:
   ```bash
   sudo nano /etc/nginx/sites-available/mynovelapp
   ```

2. Paste the following configuration, replacing `mynovelapp.duckdns.org` with your actual domain:
   ```nginx
   server {
       listen 80;
       server_name mynovelapp.duckdns.org;

       location / {
           proxy_pass http://127.0.0.1:8600;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
   *Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).*

3. Enable the newly created configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/mynovelapp /etc/nginx/sites-enabled/
   ```

4. Test your configuration and restart NGINX:
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

At this stage, you should be able to type `http://mynovelapp.duckdns.org` into your browser and see your Glassmorphic UI! But it is **not secure** yet. 

---

## Step 6: Acquire SSL Certificate (HTTPS)
Certbot makes this entirely automated. Simply type:

```bash
sudo certbot --nginx -d mynovelapp.duckdns.org
```

**Walkthrough setup:**
1. It will ask for your email address (used for renewal notices). Add your primary email.
2. Agree to the Terms of Service.
3. If it asks to "redirect all HTTP traffic to HTTPS", **Select YES (Option 2)**. This completely secures your API.

---

## Optional: Best Practice Hosting Suggestions 

1. **UFW Firewall:** Limit external traffic so strangers can't bypass your Nginx proxy and hit Docker directly on 8600!
   ```bash
   sudo ufw allow "Nginx Full"
   sudo ufw allow ssh
   sudo ufw deny 8600
   sudo ufw enable
   ```
2. **DuckDNS Cronjob (If your VPS uses Dynamic IP):**
   If you aren't using a fixed Elastic IP, create a small cron script provided directly by the DuckDNS website dashboard to constantly update your IP address on their service!
