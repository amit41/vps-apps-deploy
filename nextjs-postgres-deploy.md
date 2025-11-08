# Next.js + PostgreSQL Deployment Guide (Next.js, Nginx, PM2, Certbot)

This file contains step-by-step commands and configuration snippets to deploy a Next.js app with PostgreSQL on a Debian/Ubuntu server. Replace the placeholder tokens shown in angle brackets (for example: `<username>`, `<password>`, `<database_name>`, `<your-server-ip>`, `your-domain.com`) with your real values before running commands.

> Notes:

- Run these commands on your Debian/Ubuntu VPS (use sudo where shown). Some steps require root privileges.

## 1) Install Node.js (Node 22 x)

Download NodeSource setup script and install Node.js:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x -o nodesource_setup.sh
sudo -E bash nodesource_setup.sh
sudo apt-get install -y nodejs
```

Check versions:

```bash
node -v
npm -v
```

## 2) Install Git

```bash
sudo apt install -y git
git -v
```

## 3) Install PostgreSQL

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql.service
sudo service postgresql status
```

## 4) Create PostgreSQL database and user

Open the Postgres prompt as the `postgres` system user:

```bash
sudo -u postgres psql
```

Inside psql run (replace placeholders):

```sql
CREATE DATABASE <database_name>;
CREATE USER <username> WITH ENCRYPTED PASSWORD '<password>';
ALTER ROLE <username> SET client_encoding TO 'utf8';
ALTER ROLE <username> SET default_transaction_isolation TO 'read committed';
ALTER ROLE <username> SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE <database_name> TO <username>;
\c <database_name> postgres
GRANT ALL PRIVILEGES ON SCHEMA public TO <username>;
\q
```

Tip: Use strong passwords and consider managing DB credentials via environment variables or a secret manager.

## 5) Prepare project folder

Create `apps` folder and clone the sample project (or clone your own):

```bash
mkdir -p ~/apps
cd ~/apps
git clone <your-git-repo>
cd <your-git-repo>
```

Create the `.env` file (example using `nano`):

```bash
sudo nano .env
```

Add these lines (replace placeholders):

```
NEXT_PUBLIC_APP_TITLE="My App Title"
DATABASE_URL="postgresql://<username>:<password>@localhost:5432/<database_name>?schema=public"
```

Save and exit. Then install packages:

```bash
npm install
```

If your project uses Prisma or another ORM, follow its setup (below shows Prisma push step).

## 6) Install Nginx

```bash
sudo apt install -y nginx
```

Create an Nginx site config (example path: `/etc/nginx/sites-available/nextjs.conf`). Replace `<your-server-ip>` or use your domain name.

```nginx
server {
 listen 80;
 server_name <your-server-ip>;

 location / {
  proxy_pass http://localhost:3000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_set_header Host $host;
  proxy_cache_bypass $http_upgrade;
  proxy_buffering off;
  proxy_set_header X-Accel-Buffering no;
 }
}
```

Save the file then enable it and remove the default site:

```bash
sudo ln -s /etc/nginx/sites-available/nextjs.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo service nginx restart
```

If you're using a domain, update `server_name` later to: `your-domain.com www.your-domain.com` and restart Nginx.

## 7) Push DB schema (if using Prisma)

If your project uses Prisma and you need to create DB schema:

```bash
npx prisma db push
```

Adjust for other ORMs (TypeORM, Sequelize, etc.) accordingly.

## 8) Build & start the app (local, foreground)

Build and run (the project may use different scripts; this is the typical flow):

```bash
npm run build
npm start
```

Visit: http://<your-server-ip> or http://your-domain.com (if DNS points to the server and Nginx/proxy is configured).

Note: Running in an interactive shell will stop the app when you close the terminal.

## 9) Install PM2 and run app in background

Install PM2 globally and launch your Next.js app with the npm script:

```bash
sudo npm install -g pm2
pm2 start npm --name "nextjs" -- start
```

Useful PM2 commands:

```bash
pm2 status
pm2 logs        # Ctrl+C to exit logs
pm2 flush       # clear logs
pm2 restart nextjs
pm2 stop nextjs
pm2 delete nextjs
```

Make PM2 persistent across reboots (run the printed command as sudo):

```bash
pm2 startup
# follow the printed instruction, usually copy+paste the command it outputs (run with sudo)
pm2.save
```

Now PM2 will resurrect apps after a reboot.

## 10) Enable HTTPS with Certbot (Let's Encrypt)

Install certbot via snap and configure for Nginx:

```bash
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

Follow interactive prompts to request certificates for your domain(s).

Test automatic renewal (dry-run):

```bash
sudo certbot renew --dry-run
```

## 11) Final notes & verification

- Ensure DNS A records point your domain to `<your-server-ip>`.
- If you change environment variables, update PM2 process (`pm2 restart nextjs`) or re-deploy.
- Keep PostgreSQL secured: consider firewall rules (ufw), pg_hba.conf tuning, and regular backups.
- Monitor logs: `pm2 logs nextjs` and check `sudo journalctl -u postgresql` and `/var/log/nginx`.

## Example placeholders to replace

- `<database_name>` — name of your Postgres DB
- `<username>` — DB user you created
- `<password>` — password for the DB user
- `<your-server-ip>` — public IP of your VPS
- `your-domain.com` — your purchased domain
- `<your-git-repo>` — you project github repository

## Post-deploy checklist

1. Confirm Node and NPM installed: `node -v && npm -v`
2. Confirm Git installed: `git -v`
3. Confirm Postgres running: `sudo service postgresql status`
4. Confirm Nginx running and proxying: `sudo nginx -t` and check `curl -I http://localhost` on the server
5. Confirm PM2 process up: `pm2 status`
6. Confirm HTTPS: access `https://your-domain.com` and test renewal with `sudo certbot renew --dry-run`

---
