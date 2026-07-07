# Deployment Guide — Vaelo.pk Landing Page

Host this page at **vaelo.pk** on a remote Linux server running **Node.js + aaPanel**, using **GitHub → git clone** workflow.

---

## Part 1 — Publish to GitHub (Local Machine)

Run these commands once in your project folder on Windows (PowerShell or Git Bash):

```powershell
# 1. Initialise git (if not already done)
git init

# 2. Stage all files
git add .

# 3. First commit
git commit -m "feat: initial coming soon landing page"

# 4. Create the remote repo on GitHub
#    Go to https://github.com/new → name it "LandingPage_cosmetic" → Create repository
#    Then copy the SSH or HTTPS URL and run:

git remote add origin https://github.com/onenet786/Comming-Soon-Landing-Page-Cosmetics.git

# 5. Push
git branch -M main
git push -u origin main
```

> Repo: https://github.com/onenet786/Comming-Soon-Landing-Page-Cosmetics

---

## Part 2 — Server Setup (aaPanel + Node.js)

### 2.1 — SSH into your server

```bash
ssh root@YOUR_SERVER_IP
# or with a key:
ssh -i ~/.ssh/id_rsa root@YOUR_SERVER_IP
```

---

### 2.2 — Install Git on the server (if missing)

```bash
# Ubuntu / Debian
apt update && apt install -y git

# CentOS / AlmaLinux
yum install -y git
```

---

### 2.3 — Clone the repository

Choose a clean location outside the default web root so aaPanel does not conflict:

```bash
cd /www/wwwroot
git clone https://github.com/onenet786/Comming-Soon-Landing-Page-Cosmetics.git cosmeticerp
cd cosmeticerp
```

---

### 2.4 — Verify Node.js is available

```bash
node -v    # should print v14+ 
npm -v
```

If Node.js is not on PATH but installed via aaPanel:

```bash
# aaPanel installs Node.js here by default:
export PATH=/www/server/node/bin:$PATH
node -v
```

Add this export to `/root/.bashrc` to make it permanent:

```bash
echo 'export PATH=/www/server/node/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

### 2.5 — Start the server (quick test)

```bash
cd /www/wwwroot/cosmeticerp
node server.js
# → CosmeticERP landing page running at http://localhost:3000
```

Press `Ctrl+C` to stop after verifying it works.

---

### 2.6 — Run with PM2 (persistent, auto-restart)

PM2 keeps the app alive after logout and auto-restarts on crash.

```bash
# Install PM2 globally (once per server)
npm install -g pm2

# Start the app
pm2 start server.js --name cosmeticerp --env production

# Save the process list so it survives reboots
pm2 save

# Enable PM2 to start on system boot
pm2 startup
# → Copy and run the command it prints out
```

Useful PM2 commands:

```bash
pm2 list                  # see all running apps
pm2 logs cosmeticerp      # live logs
pm2 restart cosmeticerp   # restart
pm2 stop cosmeticerp      # stop
pm2 delete cosmeticerp    # remove from PM2
```

---

### 2.7 — Configure aaPanel Reverse Proxy (expose on port 80/443)

The Node.js app runs on port **3000**. aaPanel's Nginx will proxy public traffic to it.

1. **Log in to aaPanel** → `Websites` → `Add Site`
2. **Domain**: enter `vaelo.pk` (and optionally `www.vaelo.pk`)
3. **Root directory**: can be anything — Nginx will proxy, not serve files directly
4. Click **Submit**

Then configure the proxy:

1. Click **Settings** (gear icon) next to your site
2. Go to **Reverse Proxy** tab → **Add Proxy**
3. Fill in:
   - **Proxy name**: `cosmeticerp`
   - **Target URL**: `http://127.0.0.1:3000`
4. Click **Submit**

aaPanel will write the Nginx config automatically. Your site is now live on port 80.

---

### 2.8 — Enable HTTPS (free SSL via aaPanel)

1. In aaPanel → your site **Settings** → **SSL** tab
2. Choose **Let's Encrypt**
3. Select your domain → click **Apply**
4. Enable **Force HTTPS** toggle

---

### 2.9 — Set a custom PORT (optional)

If port 3000 conflicts with another app, set a different port:

```bash
pm2 delete cosmeticerp
PORT=4200 pm2 start server.js --name cosmeticerp
pm2 save
```

Update the aaPanel proxy target URL to `http://127.0.0.1:4200`.

---

## Part 3 — Deploy Updates (Git Pull Workflow)

Whenever you push changes to GitHub, update the server with:

```bash
cd /www/wwwroot/cosmeticerp
git pull origin main
pm2 restart cosmeticerp
```

That's it — no build step required.

---

## Part 4 — Optional: Auto-Deploy with a GitHub Webhook

For fully automatic deploys on every push, create this script on your server:

```bash
nano /www/wwwroot/cosmeticerp/deploy.sh
```

Paste:

```bash
#!/bin/bash
cd /www/wwwroot/cosmeticerp
git pull origin main
pm2 restart cosmeticerp
echo "Deploy complete: $(date)"
```

Make it executable:

```bash
chmod +x /www/wwwroot/cosmeticerp/deploy.sh
```

Then set up a simple webhook receiver (Node.js or a tool like [webhook](https://github.com/adnanh/webhook)) and call this script from your GitHub repository's **Settings → Webhooks**.

---

## Quick Reference

| Task | Command |
|---|---|
| Pull latest code | `git pull origin main` |
| Restart app | `pm2 restart cosmeticerp` |
| View live logs | `pm2 logs cosmeticerp` |
| Check app status | `pm2 list` |
| Change port | `PORT=XXXX pm2 start server.js --name cosmeticerp` |
| Stop app | `pm2 stop cosmeticerp` |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `node: command not found` | Add `/www/server/node/bin` to PATH (see §2.4) |
| Port 3000 already in use | Use `PORT=4200` or find the process with `lsof -i :3000` |
| Site shows aaPanel default page | Verify the reverse proxy is saved and Nginx reloaded |
| SSL certificate fails | Make sure DNS A record for `vaelo.pk` points to your server IP |
| Changes not showing | Run `git pull && pm2 restart cosmeticerp` |
