# Next.js EC2 Deployment Guide (Docker + Nginx + SSL)

A comprehensive guide for deploying a Next.js application to AWS EC2 using Docker, Nginx, and Let's Encrypt SSL. Based on a real deployment migrating from AWS Amplify to EC2 to solve AI chat streaming issues.

---

## Table of Contents

1. [Why EC2 Instead of Amplify](#why-ec2-instead-of-amplify)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [EC2 Instance Launch](#ec2-instance-launch)
5. [Server Setup & Deployment](#server-setup--deployment)
6. [SSL Setup](#ssl-setup)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [Maintenance](#maintenance)

---

## Why EC2 Instead of Amplify

**Problem:** AWS Amplify's CDN and Lambda@Edge infrastructure **buffers SSE (Server-Sent Events) responses**, breaking real-time AI streaming (e.g., Vercel AI SDK + OpenAI).

**Solution:** EC2 runs a real Node.js server where streaming works natively. Nginx configured with `proxy_buffering off` passes SSE responses through immediately.

| Platform | Streaming | Architecture |
|----------|-----------|-------------|
| Amplify | Broken (buffered) | Client → CloudFront → Lambda@Edge (buffers SSE) |
| EC2 | Works | Client → Nginx (no buffering) → Node.js (native streaming) |

> **Note:** CloudFront was also considered but rejected — it caches/buffers responses, causing the same streaming issue.

---

## Prerequisites

- AWS account with EC2 access
- GitHub repository with your Next.js app
- GitHub Personal Access Token (PAT) for cloning
- Terminal with SSH
- A domain or subdomain (free options available)

---

## Project Setup

### 1. Enable Standalone Output

Add `output: "standalone"` to `next.config.ts` — required for Docker:

```typescript
const nextConfig: NextConfig = {
  output: "standalone",
  // ... rest of config
};
```

### 2. Create Dockerfile (Multi-stage)

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

# Stage 2: Build the application
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# NEXT_PUBLIC_ vars must be present at build time (inlined into client bundle)
ARG NEXT_PUBLIC_SUPABASE_URL
ARG NEXT_PUBLIC_SUPABASE_ANON_KEY
ENV NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL
ENV NEXT_PUBLIC_SUPABASE_ANON_KEY=$NEXT_PUBLIC_SUPABASE_ANON_KEY

RUN npm run build

# Stage 3: Production runner (~200MB image)
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

> **Key:** `NEXT_PUBLIC_*` vars are declared as `ARG` + `ENV` in the builder stage because Next.js inlines them into the client JS bundle at build time. Runtime `env_file` alone won't work for these.

### 3. Create docker-compose.yml

```yaml
services:
  app:
    build:
      context: .
      args:
        - NEXT_PUBLIC_SUPABASE_URL=${NEXT_PUBLIC_SUPABASE_URL}
        - NEXT_PUBLIC_SUPABASE_ANON_KEY=${NEXT_PUBLIC_SUPABASE_ANON_KEY}
    restart: always
    env_file:
      - .env.production
    networks:
      - webnet

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot
    depends_on:
      - app
    networks:
      - webnet

networks:
  webnet:
```

### 4. Create Nginx Config

Create `nginx/default.conf`:

```nginx
upstream nextjs {
    server app:3000;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://nextjs;
        proxy_http_version 1.1;

        # === CRITICAL FOR STREAMING ===
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding on;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Extended timeouts for streaming
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

> **The streaming fix is here:** `proxy_buffering off`, `proxy_cache off`, and `chunked_transfer_encoding on` ensure Nginx passes SSE chunks immediately.

### 5. Create .dockerignore

```
node_modules
.next
.git
.gitignore
.env*
*.md
.DS_Store
```

### 6. Create .env.example

```bash
# Domain
DOMAIN=yourdomain.com

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# OpenAI
OPENAI_API_KEY=

# Email (SMTP)
EMAIL_HOST=
EMAIL_PORT=
EMAIL_USER=
EMAIL_PASS=
EMAIL_FROM=
```

### 7. Create deploy.sh

```bash
#!/bin/bash
set -e

DOMAIN=""
if [ -f .env.production ]; then
    DOMAIN=$(grep -E "^DOMAIN=" .env.production | cut -d '=' -f2)
fi

case "$1" in

setup)
    echo ">>> Installing Docker..."
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo usermod -aG docker $USER
    echo ">>> Docker installed. Log out and back in, then run: ./deploy.sh deploy"
    ;;

deploy)
    echo ">>> Pulling latest code..."
    git pull origin main

    echo ">>> Loading env vars..."
    set -a && source .env.production && set +a

    echo ">>> Building and starting containers..."
    docker compose up --build -d

    echo ">>> Cleaning up old images..."
    docker image prune -f

    echo ">>> Deployed! App is running."
    docker compose ps
    ;;

ssl)
    if [ -z "$DOMAIN" ]; then
        echo "Error: Set DOMAIN in .env.production first"
        exit 1
    fi
    echo ">>> Stopping containers to free port 80..."
    docker compose down
    echo ">>> Obtaining SSL certificate for $DOMAIN..."
    sudo certbot certonly --standalone -d $DOMAIN --agree-tos --no-eff-email --email admin@$DOMAIN
    echo ">>> Starting containers..."
    set -a && source .env.production && set +a
    docker compose up --build -d
    echo ">>> SSL configured for https://$DOMAIN"
    ;;

*)
    echo "Usage: ./deploy.sh {setup|deploy|ssl}"
    ;;
esac
```

```bash
chmod +x deploy.sh
```

---

## EC2 Instance Launch

### Step-by-Step in AWS Console

1. **Go to EC2 → Launch Instance**

2. **Name:** `your-app-name`

3. **AMI:** Ubuntu Server 24.04 LTS (HVM), 64-bit (x86)

4. **Instance Type:** `t3.small` (2 vCPU, 2GB RAM)
   - t3.micro may run out of memory during `npm run build`

5. **Key Pair:**
   - Click "Create new key pair"
   - Name: `your-app-key`
   - Type: RSA, Format: .pem
   - Download and secure it:
     ```bash
     mv ~/Downloads/your-app-key.pem ~/.ssh/
     chmod 400 ~/.ssh/your-app-key.pem
     ```

6. **Security Group** — Create with 3 inbound rules:

   | Type  | Port | Source        | Why              |
   |-------|------|---------------|------------------|
   | SSH   | 22   | My IP         | Your access only |
   | HTTP  | 80   | 0.0.0.0/0     | Web traffic      |
   | HTTPS | 443  | 0.0.0.0/0     | SSL traffic      |

7. **Storage:** Change from 8GB to **20GB** gp3 (Docker + node_modules need space)

8. **Launch** and wait ~30 seconds

### Elastic IP (Recommended)

Without an Elastic IP, your public IP changes on every stop/start.

1. EC2 → Elastic IPs → Allocate Elastic IP address
2. Select → Actions → Associate with your instance

### Domain Setup

**Option A: Existing domain** — Add A record pointing to your Elastic IP

**Option B: Free subdomain** — Use [FreeDNS](https://freedns.afraid.org):
1. Sign up (free)
2. Subdomains → Add
3. Type: A, Subdomain: your-choice, pick a domain, Destination: your EC2 IP
4. Save

---

## Server Setup & Deployment

### 1. SSH into EC2

```bash
ssh -i ~/.ssh/your-app-key.pem ubuntu@YOUR_EC2_IP
```

> **Important:** Make sure your terminal prompt shows `ubuntu@ip-xxx-xxx-xxx-xxx`, NOT your local machine name. Running deploy.sh locally will fail (`apt-get: command not found`).

### 2. Clone Repository

GitHub requires a Personal Access Token (passwords no longer work):

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token, check `repo` scope
3. Clone:

```bash
git clone https://USERNAME:TOKEN@github.com/ORG/REPO.git
cd REPO
```

> **Fine-grained tokens:** Must have Repository permissions → Contents → Read-only enabled, otherwise you'll get a 403.

### 3. Install Docker

```bash
./deploy.sh setup
exit  # Log out
```

SSH back in (required for docker group):
```bash
ssh -i ~/.ssh/your-app-key.pem ubuntu@YOUR_EC2_IP
cd REPO
```

### 4. Configure Environment

```bash
cp .env.example .env.production
nano .env.production   # Fill in your values
```

**Important:** Values with spaces must be quoted:
```bash
# CORRECT
EMAIL_PASS="wiog sboo pefg cmnw"

# WRONG — bash will try to execute "sboo" as a command
EMAIL_PASS=wiog sboo pefg cmnw
```

### 5. Deploy

```bash
./deploy.sh deploy
```

Your app is now live at `http://YOUR_EC2_IP`

---

## SSL Setup

### 1. Install Certbot & Get Certificate

```bash
# Install certbot on host (not in Docker — more reliable)
sudo apt-get install -y certbot

# Stop containers to free port 80
docker compose down

# Get certificate
sudo certbot certonly --standalone -d yourdomain.com --email admin@yourdomain.com --agree-tos --no-eff-email
```

### 2. Update Nginx Config

Replace `yourdomain.com` in `nginx/default.conf` with your actual domain.

### 3. Redeploy

```bash
set -a && source .env.production && set +a
docker compose up --build -d
```

Your app is now live at `https://yourdomain.com`

### Auto-Renewal

```bash
# Test
sudo certbot renew --dry-run

# Add cron job
sudo crontab -e
# Add this line:
0 0 1 * * certbot renew --quiet --pre-hook "docker compose -f /home/ubuntu/REPO/docker-compose.yml stop nginx" --post-hook "docker compose -f /home/ubuntu/REPO/docker-compose.yml start nginx"
```

---

## Troubleshooting Guide

### SSH Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| SSH timeout after reboot | EC2 IP changed | Check new IP in console, or use Elastic IP |
| SSH timeout (IP same) | Home IP changed (ISP) | Security Group → SSH rule → change to "My IP" |
| Permission denied | Wrong key permissions | `chmod 400 ~/.ssh/your-key.pem` |

### Git Clone Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| Authentication failed | GitHub doesn't accept passwords | Use Personal Access Token |
| 403 Write access denied | Token missing permissions | Fine-grained: add Contents → Read-only. Classic: check `repo` scope |
| URL rejected / port error | Missing `@github.com/...` in URL | Full format: `https://USER:TOKEN@github.com/ORG/REPO.git` |

### Environment Variable Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| NEXT_PUBLIC_ vars undefined in browser | Not available at build time | Use Docker ARG + ENV in builder stage |
| "command not found" when sourcing .env | Values with unquoted spaces | Wrap in quotes: `VAR="value with spaces"` |
| Warnings about blank NEXT_PUBLIC_ vars | Shell doesn't have the vars | Run `set -a && source .env.production && set +a` before `docker compose` |

### Runtime Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| `crypto.randomUUID is not a function` | Site accessed over HTTP (not secure context) | Set up HTTPS. Temp fix: SSH tunnel `ssh -L 3000:localhost:80` |
| Streaming still buffered | Nginx config missing | Ensure `proxy_buffering off` and `proxy_cache off` in nginx config |
| Certbot "No renewals attempted" | Docker certbot issues | Install certbot on host directly: `sudo apt-get install certbot` |

### Build Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| `apt-get: command not found` | Running on Mac instead of EC2 | SSH into EC2 first. Check prompt: `ubuntu@...` not your Mac name |
| Build out of memory | t3.micro too small | Use t3.small (2GB RAM) or larger |

---

## Maintenance

### Common Commands

```bash
# View logs
docker compose logs -f app
docker compose logs -f nginx

# Restart
docker compose restart

# Redeploy after code changes
./deploy.sh deploy

# Check container status
docker compose ps

# Monitor resources
docker stats

# Clean up disk space
docker image prune -a
docker system prune
```

### Updating the App

On your local machine, push code to GitHub. Then on EC2:

```bash
cd ~/REPO
./deploy.sh deploy
```

This pulls latest code, rebuilds the Docker image, and restarts containers.

---

## Quick Reference

```bash
# First-time setup (on EC2)
git clone https://USER:TOKEN@github.com/ORG/REPO.git
cd REPO
./deploy.sh setup          # Install Docker
exit                       # Log out
ssh ...                    # Log back in
cp .env.example .env.production
nano .env.production       # Fill values
./deploy.sh deploy         # Build & run

# SSL (on EC2)
sudo apt-get install -y certbot
docker compose down
sudo certbot certonly --standalone -d DOMAIN
docker compose up --build -d

# Redeploy (on EC2)
./deploy.sh deploy
```
