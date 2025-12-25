# EC2 Deployment Guide — Ubuntu Server + Next.js

This guide explains how to deploy a Next.js application to an Ubuntu server running on AWS EC2. It covers creating the instance, securing it, installing dependencies, configuring Nginx as a reverse proxy, obtaining SSL with Certbot, running the Next.js app in production, and an example GitHub Actions workflow for automated deployments.

This guide assumes:
- You have an AWS account and basic familiarity with EC2.
- Your Next.js app is in a Git repository (GitHub, GitLab, etc.) and can be built with `npm`/`yarn`/`pnpm`.
- You want to run Next.js on the server (i.e., `next build` + `next start`) behind Nginx. If you use serverless or Vercel, this guide is not needed.

Table of contents
- Prerequisites
- Overview / Architecture
- Step 1 — Create EC2 instance
- Step 2 — Configure security & DNS
- Step 3 — SSH into the server
- Step 4 — Server initial setup (Ubuntu)
- Step 5 — Install Node.js and PM2 (or systemd alternative)
- Step 6 — Configure Nginx as reverse proxy
- Step 7 — Obtain SSL with Certbot
- Step 8 — Deploy your Next.js application
- Step 9 — Automate deployments (GitHub Actions example)
- Optional: Docker deployment
- Troubleshooting & useful commands
- Security & maintenance notes

---

Prerequisites (local)
- AWS account and IAM user with permissions to create EC2 instances, manage security groups.
- Created SSH keypair (you'll supply when launching EC2).
- Domain (e.g., example.com) where you can create A/AAAA records pointing to the EC2 public IPv4/IPv6.

Overview / Architecture
- EC2 instance (Ubuntu LTS) running Node.js.
- Nginx acts as a reverse proxy, serves static assets and handles SSL termination (Let's Encrypt / Certbot).
- Next.js runs in production mode (either via PM2 or systemd).
- Optional CI/CD (GitHub Actions) builds & uploads the app to the server and restarts the service.

Step 1 — Create EC2 instance
1. Console: EC2 > Launch Instance
   - AMI: Ubuntu Server 22.04 LTS (or 24.04 LTS depending on availability)
   - Instance type: t3.micro / t3.small for small apps (choose based on traffic)
   - Key pair: use or create an SSH key pair (.pem)
   - Storage: default 8–30 GB (expand if needed)
   - Network: choose VPC/subnet; enable Auto-assign Public IP if you want direct access
2. Security Group: add inbound rules (details in next step)

Step 2 — Configure security & DNS
Minimum inbound rules (Security Group):
- SSH: TCP 22 — source: YOUR_IP (restrict to your IP)
- HTTP: TCP 80 — source: 0.0.0.0/0, ::/0
- HTTPS: TCP 443 — source: 0.0.0.0/0, ::/0

DNS:
- Create an A record pointing your domain (e.g., app.example.com) to the EC2 public IPv4.
- If you use IPv6, create AAAA record accordingly.

Step 3 — SSH into the server
From your machine:
```bash
chmod 400 path/to/your-key.pem
ssh -i path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
```
Replace `ubuntu` with the default user for the AMI if different.

Step 4 — Server initial setup (Ubuntu)
Run as ubuntu (or your server user):
```bash
# update & basic tools
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl ufw fail2ban
```

Create a non-root user (optional but recommended):
```bash
# replace `deploy` with your username
sudo adduser deploy
sudo usermod -aG sudo deploy
# enable SSH key authorization for deploy user
sudo mkdir -p /home/deploy/.ssh
sudo cp ~/.ssh/authorized_keys /home/deploy/.ssh/
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Firewall (ufw) example:
```bash
# allow OpenSSH temporarily if you're connected
sudo ufw allow OpenSSH
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
sudo ufw status
```

Step 5 — Install Node.js and PM2 (or systemd alternative)
Install Node.js (LTS) using NodeSource (example for Node 20 LTS; check Node version at time of use):
```bash
# NodeSource setup script (example Node 20)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
npm -v
```

Optional: install yarn or pnpm if you use them:
```bash
# yarn
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update && sudo apt install -y yarn

# pnpm
npm i -g pnpm
```

Install PM2:
```bash
sudo npm install -g pm2
# enable pm2 startup (systemd)
pm2 startup systemd
# follow printed instructions — as root or with sudo — to enable startup
```

Systemd alternative (recommended if you prefer systemd):
Create a systemd service file (example below in "Systemd service" section).

Step 6 — Configure Nginx as reverse proxy
Install Nginx:
```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Example Nginx server block for reverse proxying to Next.js running on port 3000 (replace server_name):
```nginx
# /etc/nginx/sites-available/nextapp
server {
    listen 80;
    listen [::]:80;

    server_name app.example.com; # replace with your domain

    # Optionally serve static files directly:
    root /var/www/nextapp/public; # if you copy static assets here

    location /_next/static/ {
        # cache static assets aggressively
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Optional: Let's Encrypt ACME challenge
    location ~ /.well-known/acme-challenge {
        root /var/www/letsencrypt;
    }
}
```

Enable site and test:
```bash
sudo ln -s /etc/nginx/sites-available/nextapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Systemd service (example) — using `next start`:
Create `/etc/systemd/system/nextapp.service`:
```ini
[Unit]
Description=Next.js app
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/nextapp
Environment=NODE_ENV=production
# load env from file (optional)
# EnvironmentFile=/home/deploy/nextapp/.env
ExecStart=/usr/bin/npm run start
Restart=on-failure
RestartSec=5
# Increase open files limit if needed:
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable nextapp
sudo systemctl start nextapp
sudo journalctl -u nextapp -f
```

PM2 example:
```bash
# from /home/deploy/nextapp
pm2 start "npm -- start" --name nextapp
pm2 save
# ensure startup
pm2 startup systemd
# follow printed command (copy and run it with sudo)
```

Step 7 — Obtain SSL with Certbot
Install Certbot and obtain a certificate:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d app.example.com
# follow interactive prompts (email, agree terms)
# test renewal
sudo certbot renew --dry-run
```

After Certbot runs, it will update the Nginx site to redirect HTTP -> HTTPS and install the certificate.

Step 8 — Deploy your Next.js application
Option A — Manual (git clone on server)
```bash
# login as deploy user
cd /home/deploy
git clone https://github.com/your-org/your-nextjs-repo.git nextapp
cd nextapp
# copy env file
cp .env.production .env
# install deps
npm ci            # or yarn install / pnpm install
# build
npm run build
# start
npm run start     # or pm2 start
```

Option B — Upload artifact (CI builds and transfers the .next and node_modules or builds on server)

Environment variables
- Use a `.env` file in the app root (not committed) and load with systemd EnvironmentFile or export in PM2 ecosystem.
- Never commit secrets; use AWS Secrets Manager or Parameter Store for production secrets if possible.

Example `.env` (do not commit)
```env
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgres://...
```

Step 9 — Automate deployments (GitHub Actions example)
This workflow builds the app, rsyncs files to the server and restarts the service. It uses SSH deploy key as a secret.

Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Archive build
        run: tar -czf build.tar.gz .next package.json public

      - name: Copy to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }} # ex: deploy
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "build.tar.gz"
          target: "/home/deploy/"

      - name: Deploy on server
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/deploy
            tar -xzf build.tar.gz -C nextapp
            cd nextapp
            npm ci --production
            pm2 restart nextapp || pm2 start "npm -- start" --name nextapp
```

Secrets to configure in GitHub repo:
- EC2_HOST — server public IP or domain
- EC2_USER — e.g., deploy
- EC2_SSH_KEY — private key (PEM) content (careful with secrets limit)
- Optionally, EC2_SSH_PORT if non-standard

Optional: Using rsync instead of scp for incremental updates is faster and more efficient.

Optional: Docker deployment
- Build a multi-stage Docker image (build + runtime) and run via docker-compose or deploy to ECS. This is recommended for consistent environments and easier scaling.

Example Dockerfile skeleton:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/package*.json ./
RUN npm ci --production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["npm","run","start"]
```

Troubleshooting & useful commands
- Check service logs (systemd):
```bash
sudo journalctl -u nextapp -n 200 --no-pager
```
- Check pm2 logs:
```bash
pm2 logs nextapp
pm2 monit
```
- Nginx config test & reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```
- Verify Node process listening port:
```bash
ss -tlnp | grep 3000
```
- Certbot renewal failures: check `/var/log/letsencrypt/letsencrypt.log` and `sudo certbot renew --dry-run`.

Scaling & performance tips
- Use caching headers for static assets served by Nginx.
- Offload large static content to S3 + CloudFront.
- Use a process manager (PM2) or container orchestration for multiple instances.
- Use a load balancer (ALB) and autoscaling group for high availability.
- Monitor memory/CPU and increase instance size where needed.

Security & maintenance notes
- Restrict SSH to specific IP ranges and use a strong key.
- Keep the system updated: `sudo apt update && sudo apt upgrade -y`.
- Regularly renew certificates (Certbot sets up renewal cron job).
- Limit access to secrets, rotate keys and credentials periodically.

Appendix — Example files & snippets

1) Nginx config (as shown earlier)
```nginx
# /etc/nginx/sites-available/nextapp
server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~ /.well-known/acme-challenge {
        root /var/www/letsencrypt;
    }
}
```

2) systemd service (again)
```ini
[Unit]
Description=Next.js app
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/nextapp
Environment=NODE_ENV=production
ExecStart=/usr/bin/npm run start
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

3) PM2 ecosystem (optional)
```js
module.exports = {
  apps: [
    {
      name: 'nextapp',
      script: 'npm',
      args: 'start',
      env_production: {
        NODE_ENV: 'production',
      },
      cwd: '/home/deploy/nextapp'
    }
  ]
}
```

Closing notes
- This guide favors a straightforward server-hosted approach (Next.js on Node.js + Nginx + Certbot). For heavy scaling or simplified maintenance consider hosting on managed platforms such as Vercel, Render, or container orchestration (ECS / Kubernetes).
- Replace example domain, usernames, and paths with values for your environment.
- If you want, I can:
  - Generate a ready-to-use systemd service & Nginx config customized for your domain and repo.
  - Create a GitHub Actions deploy workflow tailored to your project structure.
  - Provide a Dockerfile + docker-compose for your Next.js app.

License
This document is provided as-is. Use and modify freely.
