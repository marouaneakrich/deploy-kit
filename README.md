# 🚀 Ultimate Production Deployment Guide: VPS & CI/CD

Welcome to your comprehensive guide for deploying a Node.js application to a VPS (Ubuntu). This guide focuses on **security**, **scalability**, and **automation**. 

> **📦 Included Resources**: This guide accompanies the `VPS Deployment with CI_CD Guide.pdf` and `deploy-kit.mp4` files found in this directory. Refer to them for visual walkthroughs!

---

## 📋 Table of Contents
1. [Server Access & One-Time Setup](#1-server-access--one-time-setup)
2. [Security Hardening](#2-security-hardening)
3. [Environment Setup (Docker & Nginx)](#3-environment-setup-docker--nginx)
4. [Reverse Proxy Configuration](#4-reverse-proxy-configuration)
5. [SSL Certificates (HTTPS)](#5-ssl-certificates-https)
6. [CI/CD Pipeline (GitHub Actions)](#6-cicd-pipeline-github-actions)

---

## 1. Server Access & One-Time Setup

First, we need to log in to your server and prepare the system.

### 🔑 Login
Open your terminal (CMD/PowerShell on Windows, Terminal on Mac/Linux) and log in as the `root` user provided by your VPS provider.

```bash
ssh root@YOUR_SERVER_IP
# Enter password when prompted
```

### 🔄 Update System
Always ensure your server is up to date before installing anything.

```bash
sudo apt update && sudo apt upgrade -y
```

### 👤 Create a Deployer User
**Never** use `root` for day-to-day operations or deployment. Let's create a dedicated user named `deployer`.

```bash
# Create the user
adduser deployer

# Grant superuser (sudo) privileges
usermod -aG sudo deployer

# Switch to the new user
su - deployer
```

---

## 2. Security Hardening

Now we make the server secure.

### 🛡️ SSH Key Setup (Local Machine)
**Do this step on your computer, NOT the server.** This creates a secure key for logging in without a password (needed for GitHub Actions too).

```bash
ssh-keygen -t ed25519 -C "deployer"
# Press ENTER to accept default location
# Press ENTER for no passphrase (for automation purposes)
```

Display the public key content to copy it:
```bash
# Windows (PowerShell)
cat ~/.ssh/id_ed25519.pub

# Mac/Linux
cat ~/.ssh/id_ed25519.pub
```
*Copy this key. You will need it mainly for **GitHub Secrets** later.*

> **Tip**: To enable password-less login from your computer to the server, you can manually add this public key to the server's `~/.ssh/authorized_keys` file for the `deployer` user.

### 🔒 Secure SSH Configuration
Back on the **server**, edit the SSH config to disable password login and root login.

```bash
sudo nano /etc/ssh/sshd_config
```

Find and change these lines (remove `#` if present to uncomment):
```ssh
PermitRootLogin no
PasswordAuthentication no
```
*Press `Ctrl+X`, then `Y`, then `Enter` to save and exit.*

**Restart SSH to apply changes:**
```bash
sudo systemctl restart ssh
```

### 🧱 Firewall (UFW)
Enable the firewall and allow only necessary ports.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw enable
# Type 'y' to confirm
```

### 🚫 Install Fail2Ban
Protect against brute-force attacks.
```bash
sudo apt install fail2ban -y
```

---

## 3. Environment Setup (Docker & Nginx)

### 🐳 Install Docker
We'll use Docker to run the application containers.

```bash
# Download and install Docker automatically
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Add deployer user to docker group (avoids using sudo for docker commands)
sudo usermod -aG docker deployer
```
*Note: You might need to log out and log back in for the group change to take effect.*

### 🌐 Install Nginx
Nginx will act as the reverse proxy (the traffic cop) for your app.

```bash
sudo apt install nginx -y
```

---

## 4. Reverse Proxy Configuration

Configure Nginx to route traffic from the internet to your Docker container running on port 3000.

Create a site configuration:
```bash
sudo nano /etc/nginx/sites-available/myapp
# Replace 'myapp' with your project name
```

**Paste the following configuration:**
```nginx
server {
    server_name api.yourdomain.com; # ⚠️ REPLACE with your actual domain

    location / {
        proxy_pass http://localhost:3000; # Points to your Docker container
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Activate the site:**
```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Test configuration for syntax errors
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

---

## 5. SSL Certificates (HTTPS)

Secure your site with free Let's Encrypt certificates.

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Generate Certificate
sudo certbot --nginx -d api.yourdomain.com
```
Follow the prompts (enter email for renewal alerts). Certbot will automatically update your Nginx config.

---

## 6. CI/CD Pipeline (GitHub Actions)

Automate deployment so your app updates whenever you push to `main`.

### ⚙️ GitHub Repository Settings
1. Go to your GitHub Repository.
2. Navigate to **Settings** > **Secrets and variables** > **Actions**.
3. Click **New repository secret**.
4. Add the following secrets:

| Secret Name | Value |
| :--- | :--- |
| `VPS_IP` | The public IP address of your server. |
| `SSH_PRIVATE_KEY` | The **private** key generated earlier (content of `id_ed25519` file, NOT `.pub`). |

### 📄 Workflow File
Create a file in your project at `.github/workflows/deploy.yml`:

```yaml
name: Deploy to VPS

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          push: true
          # Replace with your Docker Hub username and repo
          tags: youruser/myapp:latest 

      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_IP }}
          username: deployer
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull youruser/myapp:latest
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp -p 3000:3000 --restart unless-stopped --env-file .env youruser/myapp:latest
```

---

**🎉 Done!** Now, every time you push code to the `main` branch:
1. GitHub builds your Docker image.
2. It pushes the image to Docker Hub.
3. It logs into your VPS via SSH.
4. It pulls the new image and restarts your application container.
