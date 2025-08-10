# Travel Memory

This README walks you through everything needed to **deploy the TravelMemory application** on an AWS EC2 Ubuntu instance, configure a MongoDB Atlas cluster, and expose the frontend via Nginx reverse proxy. Each step includes commands and a clear explanation so you understand *why* you're doing it.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [EC2: Update & Install Node.js 22](#ec2-update-and-install-nodejs-22)
3. [MongoDB Atlas: Create Cluster & Database](#mongodb-atlas-create-cluster--database)
4. [Deploy Backend (Node.js / Express)](#deploy-backend-nodejs--express)
5. [Deploy Frontend (React)](#deploy-frontend-react)
6. [Nginx: Configure Reverse Proxy](#nginx-configure-reverse-proxy)
7. [Run Backend/Frontend as Services (recommended)](#run-backendfrontend-as-services-recommended)
8. [Troubleshooting & Common Issues](#troubleshooting--common-issues)
9. [Security & Best Practices](#security--best-practices)
10. [Next Steps / Enhancements](#next-steps--enhancements)

---

# Prerequisites

* An **AWS account** and permissions to create EC2 instances and security groups.
* A key pair (`.pem`) to SSH into EC2.
* Basic familiarity with SSH, terminals, and editing files (nano/vi).
* Git installed locally or on the EC2 instance for cloning repositories.
* A MongoDB Atlas account for managed database (free tier is fine).

> Note: Replace example placeholders (e.g., `EC2_PUBLIC_IP`, `your-bucket-name`, and `ENTER_YOUR_URL`) with actual values for your environment.

---

# EC2 — Update and install Node.js 22

## 1. SSH into your EC2 instance

```bash
ssh -i /path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
```

**Why:** This opens a secure shell (SSH) session to run administrative commands on your server.

## 2. Update package lists & upgrade installed packages

```bash
sudo apt update && sudo apt upgrade -y
```

**Explanation:**

* `sudo apt update` refreshes the list of available packages and versions.
* `sudo apt upgrade -y` upgrades installed packages to their latest versions. `-y` auto-confirms.

## 3. Install Node.js 22 and npm (via NodeSource)

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

**Explanation:**

* The curl command downloads and runs the NodeSource installer script which configures apt repositories for Node 22.
* `sudo apt install nodejs` installs Node.js and the bundled npm.

## 4. Verify installations

```bash
node -v
npm -v
```

**Expected:** You should see Node 22.x and a corresponding `npm` version.

---

# MongoDB Atlas — Create cluster & database

This section uses MongoDB Atlas (managed DB). You will create a cluster, user, whitelist IP or allow access, and create the `travelmemory` database using Compass or the Atlas UI.

## 1. Create a MongoDB Atlas cluster

1. Sign in or create an account at [https://cloud.mongodb.com](https://cloud.mongodb.com)
2. Click **Build a Cluster** → choose **AWS** and a region close to your EC2 instance.
3. Choose the **free tier (M0)** for testing, then **Create Cluster**.

**Why:** Atlas provides a managed, production-ready MongoDB instance without self-hosting.

## 2. Add a Database User

* In Atlas: **Database Access** → **Add New Database User**
* Create credentials (username & password). Store these securely.
* For learning, choose **Read and write to any database**; in production scope permissions properly.

## 3. Network Access — Allow IPs

* In Atlas: **Network Access** → **Add IP Address**
* You can add your current IP or select **Allow access from anywhere (0.0.0.0/0)** for testing only.

**Security note:** Allowing `0.0.0.0/0` is insecure for production — instead add only the EC2 public IP or VPC peering.

## 4. Connect (get connection URI)

* Click **Connect** → choose **Connect with MongoDB Compass** or **Connect your application**
* Use the connection string (URI) format:

```
mongodb+srv://<username>:<password>@cluster0.mongodb.net/travelmemory?retryWrites=true&w=majority
```

* Replace `<username>` and `<password>` with your user credentials.

## 5. Create `travelmemory` database using Compass

* Open MongoDB Compass and paste the URI.
* After connecting, click **Create Database** → name it `travelmemory` and create a collection (e.g., `trips`).
* Add documents with fields like `location`, `dateVisited`, and `notes`.

**Why:** The application uses this DB for storing travel entries.

---

# Deploy Backend (Node.js / Express)

This section clones the repository, installs dependencies, configures environment variables, and runs the backend.

## 1. Clone the repository on EC2

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/backend
```

**Explanation:** `git clone` downloads the source; `cd` moves into the backend source folder where the backend server code lives.

## 2. Create `.env` file with MongoDB URI and PORT

```bash
nano .env
```

Add:

```
MONGO_URI='ENTER_YOUR_URL'
PORT=3001
```

* Replace `ENTER_YOUR_URL` with the Atlas connection URI (URL-encode or wrap in quotes if it contains special characters).

**Why:** The backend reads `MONGO_URI` to connect to MongoDB and `PORT` to know which port to listen on.

## 3. Install dependencies

```bash
npm install
```

**Why:** Installs libraries listed in `package.json` (Express, Mongoose, etc.).

## 4. Start the backend server

```bash
node index.js
```

**Explanation:** This launches the backend. By default it will listen on port `3001` as per `.env`.

**Tip (production):** Use `pm2` or systemd to keep the process running in the background. Example with pm2:

```bash
sudo npm install -g pm2
pm2 start index.js --name travelmemory-backend
pm2 save
pm2 startup
```

## 5. Test the API

From a browser or using `curl`:

```bash
curl http://EC2_PUBLIC_IP:3001/hello
```

You should see `Hello World` (or the API's test response).

> If the request times out, check: security group inbound rules allow port `3001` from your IP (or 0.0.0.0/0 for testing).

---

# Deploy Frontend (React)

This section shows how to run the React frontend in development mode or build it for production.

## 1. SSH into EC2 (if not already)

```bash
ssh -i /path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
```

## 2. Go to frontend folder

```bash
cd TravelMemory/frontend
```

## 3. Configure environment variable for backend URL

Create `.env` with the backend URL so React can call the API:

```bash
echo "REACT_APP_BACKEND_URL=http://EC2_PUBLIC_IP:3001" > .env
```

**Why:** React reads `REACT_APP_` prefixed env vars at build time and uses them in the client app.

## 4. Install frontend packages

```bash
npm install
```

## 5. Run in development mode (optional)

```bash
npm start
```

This starts a dev server on port `3000`. Access at `http://EC2_PUBLIC_IP:3000`.

## 6. Production build and serve with Nginx (recommended)

To deploy as static assets:

```bash
npm run build
sudo mv build /var/www/travelmemory
```

Then configure Nginx (see next section) to serve `/var/www/travelmemory`.

**Why production build:** React dev server is for development only; production uses optimized static files.

---

# Nginx — Configure Reverse Proxy (serve frontend & proxy API)

Using Nginx lets you expose the frontend on port 80 (no `:3000`) and route API calls to the backend.

## 1. Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

## 2. Example site config (save as `/etc/nginx/sites-available/travelmemory`)

```nginx
server {
    listen 80;
    server_name frontend.aviralpaliwal.ink;  # replace with your domain or EC2 public IP

    root /var/www/travelmemory;              # React build directory
    index index.html;

    location /api/ {
        proxy_pass http://localhost:3001/;   # forward API requests to backend
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Fallback to index.html for SPA routing
    location / {
        try_files $uri /index.html;
    }
}
```

**Explanation:**

* Requests to `/api/*` are proxied to the backend on `localhost:3001`.
* All other requests serve the static React files and support client-side routing via `try_files`.

## 3. Enable the site and reload Nginx

```bash
sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 4. Allow HTTP in firewall

If UFW is enabled:

```bash
sudo ufw allow 'Nginx Full'
```

## 5. (Optional) Enable HTTPS with Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d frontend.aviralpaliwal.ink
```

**Why SSL:** Protects traffic, required for many modern browsers and security best practices.

---

# Run Backend & Frontend as Services (recommended)

For production, run the backend as a service with `pm2` or `systemd` and serve frontend with Nginx.

### Example using systemd for backend

Create `/etc/systemd/system/travelmemory-backend.service`:

```ini
[Unit]
Description=TravelMemory Backend
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/TravelMemory/backend
ExecStart=/usr/bin/node index.js
Restart=on-failure
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable travelmemory-backend
sudo systemctl start travelmemory-backend
```

---

# Troubleshooting & Common Issues

### 1. Backend not reachable (timeout)

* Check EC2 Security Group — allow inbound traffic on port `3001` (or 80 if proxied via Nginx).
* Confirm backend is running: `ps aux | grep node` or `pm2 list`.

### 2. CORS errors in browser

If your frontend is served from a different origin than backend, enable CORS in Express:

```js
// example in Express (Node.js)
const cors = require('cors');
app.use(cors({ origin: '*' })); // Use specific origin in production
```

### 3. React env var not picked up

* Remember React reads `REACT_APP_*` variables at **build time**.
* After editing `.env`, rebuild the frontend with `npm run build`.

### 4. `app.py` or file missing in Docker builds

* Ensure `COPY` paths in Dockerfile are correct and that build context (the `.`) is used.

### 5. Permissions & file ownership

* If files under `/var/www` are owned by root, allow nginx user to read them:

```bash
sudo chown -R www-data:www-data /var/www/travelmemory
sudo chmod -R 755 /var/www/travelmemory
```

---

# Security & Best Practices

* **Do not** use `0.0.0.0/0` in production for MongoDB Atlas; instead whitelist your EC2 IP or use VPC peering.
* Use **least-privilege IAM roles** for EC2 if using AWS services programmatically.
* Store secrets (Mongo URI, API keys) in **AWS Systems Manager Parameter Store**, **Secrets Manager**, or environment variables securely (avoid committing `.env` to Git).
* Enable HTTPS (TLS) via Certbot/LetsEncrypt.
* Use monitoring (CloudWatch, PM2 dashboard) and logging for production readiness.

---
