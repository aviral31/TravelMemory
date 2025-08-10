# Travel Memory

## Architecture Diagram: https://drive.google.com/file/d/1-pgxESV0xnaB3wE1CPYxvhF_vftqii3G/view?usp=sharing

---

## Prerequisites

* AWS account with permissions for EC2, Load Balancers, and Security Groups.
* Key pair (.pem) for SSH.
* MongoDB Atlas account.
* Basic Git, SSH, and Linux terminal knowledge.

---

## Step 1 — Provision EC2 Instances

* Launch **4 Ubuntu EC2 instances** (2 for frontend, 2 for backend).
* Configure Security Groups to allow:

  * Port 22 (SSH)
  * Port 80 (HTTP)
  * Port 3000 (Frontend)
  * Port 3001 (Backend)

---

## Step 2 — Install Node.js and npm

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:

```bash
node -v
npm -v
```

---

## Step 3 — Configure MongoDB Atlas

1. Create a **free cluster** in Atlas.
2. Add a DB user with read/write access.
3. Whitelist EC2 public IPs or use `0.0.0.0/0` (testing only).
4. Get the connection URI and set it in `.env` for the backend:

```
MONGO_URI='your_connection_uri'
PORT=3001
```

---

## Step 4 — Deploy Backend

* SSH into backend instances.
* Clone repo:

```bash
git clone https://github.com/aviral31/TravelMemory.git
cd TravelMemory/backend
npm install
node index.js
```

* Use PM2 for process management.

---

## Step 5 — Deploy Frontend

* SSH into frontend instances.
* Go to `frontend` folder:

```bash
cd TravelMemory/frontend
```

* Set API endpoint in `.env`:

```
REACT_APP_BACKEND_URL=http://<backend-instance-ip>:3001
```

* Build and move files:

```bash
npm install
npm start
```

---

## Step 6 — Configure Nginx Reverse Proxy

```bash
sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/travelmemory
```

Example config:

```nginx
server {
    listen 80;
    server_name frontend.aviralpaliwal.ink;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

---

## Step 7 — Create Application Load Balancer

1. Go to **EC2 → Load Balancers → Create Load Balancer → Application Load Balancer**.
2. Scheme: Internet-facing.
3. Add **listeners**:

   * HTTP:80 → Target Group (frontend, port 80)
   * HTTP:3000 → Target Group (frontend, port 3000)
4. Register frontend instances in the Target Groups.

---

## Step 8 — Configure Cloudflare CNAME

* Add CNAME record:

  * Name: `frontend`
  * Target: `<ALB-DNS-Name>`
  * Proxy Status: DNS only.

---

## Step 9 — Test Deployment

* Access app via: `http://frontend.aviralpaliwal.ink`
* Verify frontend loads and backend API is functional.

---

