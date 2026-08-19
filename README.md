# UCSH Alumni Network

Welcome to the official repository for the **UCSH Alumni Network**, a full-stack platform built to connect and support the alumni community of the University of Computer Studies, Hinthada.

* **Live Project:** [https://alumna.ucsh.edu.mm/](https://alumna.ucsh.edu.mm/)
* **Environment:** Ubuntu Linux Virtual Machine

---

##  Complete Server Installation Guide

The following guide outlines how to deploy the application from scratch on an Ubuntu VM environment.

### Step 1: Server Preparation

Ensure your Ubuntu Virtual Machine has all the foundational packages required for a modern Node.js application.

1. **Update the system and install core utilities:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nano ufw docker.io nginx

```


2. **Install Node.js and NPM (Node Package Manager):**
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

```


3. **Install PM2 globally:**
PM2 is a daemon process manager that will keep your Next.js application running in the background.
```bash
sudo npm install -g pm2

```



### Step 2: Database Setup (MongoDB via Docker)

Virtual Machine hypervisors often do not pass through AVX CPU instructions, causing standard native installations of MongoDB 5.0+ to crash immediately. Deploying MongoDB 4.4 using Docker is the most stable workaround for a VM environment.

1. **Enable and start the Docker service:**
```bash
sudo systemctl enable --now docker

```


2. **Deploy the MongoDB Container:**
This command downloads and runs MongoDB on the standard port (27017) and ensures it automatically restarts if the server reboots.
```bash
sudo docker run --name local-mongo -d -p 27017:27017 --restart always mongo:4.4

```



### Step 3: Application Deployment

1. **Clone the Repository:**
```bash
cd ~
git clone https://github.com/AungPyaeSoneUCS/AlumniNetworkUCS.git
cd AlumniNetworkUCS

```


2. **Install Dependencies:**
```bash
npm install

```


3. **Configure Environment Variables:**
```bash
nano .env

```


Ensure your database string points to the local Docker container. Include your `NEXTAUTH_SECRET`, `NEXTAUTH_URL`, and other necessary variables:
```env
MONGODB_URI="mongodb://127.0.0.1:27017/AlumniNetworkDB"

```


*(Save and exit nano: Ctrl+O, Enter, Ctrl+X)*

### Step 4: Folder Permissions

For user uploads (like profile pictures or logos) to work correctly, the web server needs explicit read and write access to your public directory.

Run this command from inside your project folder:

```bash
sudo chmod -R 777 public/

```

*(Note: If you ever delete and recreate the public folder, you must run this command again).*

### Step 5: Process Management (PM2)

To avoid "ghost processes" and `EADDRINUSE` port 3000 conflicts, it is best to run the Next.js binary directly rather than wrapping it in an npm script.

1. **Build the production application:**
```bash
npm run build

```


2. **Start the application with the direct Next.js binary:**
```bash
pm2 start ./node_modules/next/dist/bin/next --name "next-app" -- start

```


3. **Save the active process list:**
This ensures your application boots up automatically if the VM restarts.
```bash
pm2 save
pm2 startup

```



### Step 6: Web Server & Reverse Proxy (NGINX)

Configure NGINX to handle web traffic securely and serve uploaded files directly from the disk to prevent Next.js caching issues.

1. **Create the NGINX Configuration:**
```bash
sudo nano /etc/nginx/sites-available/alumna.ucsh.edu.mm

```


2. **Paste the Server Block:**
*(Be sure your SSL certificates are generated via Certbot or placed in the designated paths).*
```nginx
server {
    listen 443 ssl http2;
    server_name alumna.ucsh.edu.mm;

    # SSL Configuration 
    ssl_certificate /etc/ssl/certs/ucsh_fullchain.crt;
    ssl_certificate_key /etc/ssl/private/ucsh.key;

    # Main Proxy to Next.js
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Serve user uploads directly from the disk for instant rendering
    location /uploads/ {
        alias /home/hinthadauser/AlumniNetworkUCS/public/uploads/;
        access_log off;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }
}

# HTTP to HTTPS Redirect
server {
    listen 80;
    server_name alumna.ucsh.edu.mm;
    return 301 https://$host$request_uri;
}

```


3. **Enable and Restart NGINX:**
```bash
sudo ln -s /etc/nginx/sites-available/alumna.ucsh.edu.mm /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

```



---

## 🔄 Standard Update Workflow

Whenever new code is merged into the GitHub repository, follow this workflow to update the live server cleanly and prevent cache conflicts.

1. **Navigate to the project directory:**
```bash
cd ~/AlumniNetworkUCS

```


2. **Pull the latest code:**
```bash
git pull origin main

```


*Troubleshooting:* If you receive an error about local files being overwritten (e.g., a modified `logo.png`), force the update to mirror GitHub exactly by running:
```bash
git fetch --all
git reset --hard origin/main

```


3. **Clean the cache and rebuild:**
```bash
rm -rf .next
npm run build

```


4. **Restart the application:**
```bash
pm2 restart next-app

```
