production-ready CI/CD setup for your React + Express + PostgreSQL + Prisma + Multer (TypeScript) project, deployed on a VPS using GitHub Actions.

This is the same structure used in real companies.


---

1️⃣ Branch Strategy (VERY IMPORTANT)

Use a simple & professional flow:

main        → production (LIVE SERVER)
develop     → staging / testing
feature/*   → new features
fix/*       → bug fixes

Rules

❌ Never push directly to main

✅ All work → feature/*

✅ Merge → develop

✅ After testing → merge develop → main

🚀 CI/CD deploys only from main


Example:

feature/auth-login
feature/file-upload
fix/payment-bug


---

2️⃣ Project Structure (Recommended)

root/
│
├── frontend/        # React app (TypeScript)
├── backend/         # Express + Prisma + Multer
│   ├── prisma/
│   ├── src/
│   └── package.json
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
└── docker-compose.yml (optional but recommended)


---

3️⃣ VPS Preparation (One-time)

🔹 Login to VPS

ssh root@YOUR_SERVER_IP

🔹 Install required software

apt update && apt upgrade -y
apt install -y git nginx nodejs npm

🔹 Install Node (better version)

curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

🔹 Install PM2 (Process Manager)

npm install -g pm2

🔹 Install PostgreSQL

apt install postgresql postgresql-contrib -y

Create DB:

sudo -u postgres psql
CREATE DATABASE myapp;
CREATE USER myuser WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE myapp TO myuser;
\q


---

4️⃣ Server Folder Structure

/var/www/
├── myapp/
│   ├── frontend/
│   └── backend/

mkdir -p /var/www/myapp
cd /var/www/myapp


---

5️⃣ Backend Production Setup

Backend start script (backend/package.json)

"scripts": {
  "build": "tsc",
  "start": "node dist/server.js",
  "dev": "ts-node-dev src/server.ts",
  "prisma:deploy": "prisma migrate deploy"
}

Prisma config

DATABASE_URL="postgresql://myuser:password@localhost:5432/myapp"


---

6️⃣ Frontend Production Build

React build command:

npm run build

Output:

frontend/dist  or  frontend/build

This will be served via Nginx.


---

7️⃣ GitHub Secrets (VERY IMPORTANT)

Go to:

GitHub Repo → Settings → Secrets → Actions

Add these:

Name	Value

VPS_HOST	your_server_ip
VPS_USER	root
VPS_KEY	PRIVATE_SSH_KEY
VPS_PATH	/var/www/myapp


Generate SSH Key (local)

ssh-keygen -t ed25519

Copy public key to VPS:

cat ~/.ssh/id_ed25519.pub

Paste into:

~/.ssh/authorized_keys


---

8️⃣ GitHub Action (deploy.yml)

📁 .github/workflows/deploy.yml

name: Deploy to VPS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.VPS_KEY }}

      - name: Deploy
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} << 'EOF'
            cd ${{ secrets.VPS_PATH }}

            if [ ! -d ".git" ]; then
              git clone YOUR_REPO_URL .
            fi

            git pull origin main

            # Backend
            cd backend
            npm install
            npm run build
            npx prisma migrate deploy
            pm2 restart backend || pm2 start dist/server.js --name backend

            # Frontend
            cd ../frontend
            npm install
            npm run build

          EOF


---

9️⃣ Nginx Setup (Frontend + API)

nano /etc/nginx/sites-available/myapp

server {
    server_name yourdomain.com;

    root /var/www/myapp/frontend/dist;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}

Enable:

ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled
nginx -t
systemctl restart nginx


---

🔟 Multer File Upload (IMPORTANT)

Create folder:

mkdir -p /var/www/myapp/backend/uploads
chmod -R 755 uploads

Serve uploads:

app.use("/uploads", express.static("uploads"));


---

✅ Final Workflow (How YOU work daily)

feature/*  → develop → main → auto deploy 🚀

1. Create feature branch


2. Push feature


3. Merge to develop


4. Test


5. Merge develop → main


6. GitHub Action auto deploys to VPS




---

🔐 Extra Production Tips

Use .env.production

Use PM2 ecosystem config

Enable HTTPS with Certbot

Setup DB backup cron

Add CI checks (lint, test)



explain branch control + merge process step-by-step, with real commands, rules, and examples exactly how teams do it in production.

I’ll assume this setup:

main     → Production (auto deploy)
develop  → Staging / testing
feature/* → New features
fix/*     → Bug fixes


---

1️⃣ Create Base Branches (One-time)

Create develop from main

git checkout main
git pull origin main
git checkout -b develop
git push origin develop

Now you have:

main
develop


---

2️⃣ Branch Rules (CONTROL SYSTEM)

🔒 main branch

❌ No direct push

✅ Only merge from develop

🚀 Auto deploy to VPS


🔒 develop branch

❌ No direct push

✅ Merge from feature/* or fix/*


🧑‍💻 feature / fix branches

✅ Direct push allowed

❌ Never merge directly to main



---

3️⃣ Working on a New Feature (Real Example)

Example feature: User Login

git checkout develop
git pull origin develop
git checkout -b feature/user-login

Now you are on:

feature/user-login

Make code changes…

git add .
git commit -m "feat: user login api + ui"
git push origin feature/user-login


---

4️⃣ Merge feature → develop (PROPER WAY)

❌ Wrong

git checkout develop
git merge feature/user-login   ❌

✅ Correct (GitHub Pull Request)

Steps in GitHub:

1. Go to GitHub Repo


2. Click Compare & Pull Request


3. Base: develop ← Compare: feature/user-login


4. Review code


5. Merge PR



This ensures:

Code review

No broken code

History is clean



---

5️⃣ Testing on develop Branch

After merge:

git checkout develop
git pull origin develop

Now test:

Frontend

API

Prisma migrations

Multer uploads


If bugs found → create fix/* branch


---

6️⃣ Bug Fix Flow (Example)

git checkout develop
git checkout -b fix/login-validation

Fix bug → commit → push:

git commit -m "fix: login validation error"
git push origin fix/login-validation

PR:

fix/login-validation → develop


---

7️⃣ Merge develop → main (RELEASE PROCESS)

When everything is stable 👇

git checkout develop
git pull origin develop
git checkout main
git pull origin main
git merge develop
git push origin main

OR (Recommended)

🔥 GitHub Pull Request

Base: main
Compare: develop

Click Merge

👉 This triggers GitHub Actions CI/CD
👉 Auto deploy to VPS 🚀


---

8️⃣ Handle Merge Conflict (REAL CASE)

Example conflict

backend/src/app.ts

Git says:

CONFLICT (content)

Fix it:

git status
git checkout develop
git pull origin develop
git checkout feature/user-login
git rebase develop

Resolve conflict → save → then:

git add .
git rebase --continue
git push -f origin feature/user-login

Then create PR again.


---

9️⃣ Protect Branches (IMPORTANT 🔐)

GitHub Settings

Repo → Settings → Branches → Add rule

Protect main

Require pull request

Require review

Block force push

Block direct push


Protect develop

Require pull request

Block direct push



---

🔟 Example Day-to-Day Workflow

Monday:
feature/file-upload → develop

Tuesday:
fix/upload-size → develop

Friday:
develop → main → auto deploy 🚀


---

1️⃣1️⃣ Commit Message Standard (Use this)

feat: add user authentication
fix: resolve multer file size issue
chore: update dependencies
refactor: clean prisma service


---

1️⃣2️⃣ Visual Flow (Easy Memory)

feature/*  → develop  → main
fix/*      → develop  → main


---

1️⃣3️⃣ Golden Rules (Remember This)

✅ Never push to main
✅ Never merge feature directly to main
✅ Always PR
✅ Test on develop
✅ Only main deploys
