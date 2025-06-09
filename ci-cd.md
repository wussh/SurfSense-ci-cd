# SurfSense CI/CD

This repository contains the CI/CD pipeline configuration for automatically deploying [SurfSense](https://github.com/MODSetter/SurfSense) to your server.

## Workflow

* On every `git push` to the `main` branch:

  * GitHub Actions connects to your server via **Cloudflare Tunnel + SSH**
  * The server runs:

    * `git pull` to update local source code
    * `systemctl restart` to restart backend and frontend services (SurfSense)
  * The whole process is automated 🚀

---

## How it works

### Components

| Component         | Description                                     |
| ----------------- | ----------------------------------------------- |
| GitHub Actions    | Automation platform, triggers deploy on push    |
| Cloudflare Tunnel | Secure SSH access without public IP             |
| systemd           | Manages SurfSense backend and frontend services |
| Ubuntu 24.04      | Example target OS                               |

---

## Files

### `.github/workflows/deploy_vm.yaml`

This workflow automates the deploy process:

```yaml
name: Deploy SurfSense

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest

    steps:
      - name: Add Cloudflare GPG key
        run: |
          sudo mkdir -p --mode=0755 /usr/share/keyrings
          curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

      - name: Add Cloudflare repo to APT sources
        run: |
          echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

      - name: Install cloudflared
        run: sudo apt-get update && sudo apt-get install -y cloudflared

      - name: SSH via Cloudflare Tunnel
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh -o "StrictHostKeyChecking=no" \
              -o "ProxyCommand=cloudflared access ssh --hostname linux.wush.site" \
              softwaredev@linux.wush.site << 'EOF'
                set -e
                cd /tmp/SurfSense-ci-cd
                git pull origin main
                sudo /bin/systemctl restart surfsense-backend.service
                sudo /bin/systemctl restart surfsense-frontend.service
          EOF
```

---

## Setup Instructions

### 1️⃣ Systemd Services

Create these services:

#### `/etc/systemd/system/surfsense-backend.service`

```ini
[Unit]
Description=SurfSense Backend Service
After=network.target

[Service]
User=softwaredev
WorkingDirectory=/tmp/SurfSense-ci-cd/surfsense_backend
ExecStart=/home/linuxbrew/.linuxbrew/bin/uv run main.py
Restart=always

[Install]
WantedBy=multi-user.target
```

#### `/etc/systemd/system/surfsense-frontend.service`

```ini
[Unit]
Description=SurfSense Frontend Service
After=network.target

[Service]
User=softwaredev
WorkingDirectory=/tmp/SurfSense-ci-cd/surfsense_web
ExecStart=/usr/local/bin/pnpm run dev
Restart=always

[Install]
WantedBy=multi-user.target
```

Then enable both:

```bash
sudo systemctl enable surfsense-backend.service
sudo systemctl enable surfsense-frontend.service
```

---

### 2️⃣ Sudo Permissions (important!)

To avoid `sudo` password issues in GitHub Actions:

```bash
sudo visudo
```

Add:

```bash
softwaredev ALL=NOPASSWD: /bin/systemctl restart surfsense-backend.service, /bin/systemctl restart surfsense-frontend.service
```

---

### 3️⃣ GitHub Secrets

Add this secret:

* `SSH_PRIVATE_KEY` → Your private key that has access to `softwaredev@linux.wush.site`

Example:

```bash
cat ~/.ssh/id_rsa
```

Copy the key (do not copy `id_rsa.pub`).

---

## Result

✅ Auto deploy SurfSense to your server in seconds whenever you push to `main`.

✅ Full logs in GitHub Actions for every deploy.

✅ Safe, fast, automated.

---

## Future Improvements

* Add deploy to multiple servers
* Notify Slack/Discord on successful deploy
* Auto health check after systemctl restart
* Blue/green deploy

---
