# Travel Memory

This README explains how to deploy the TravelMemory MERN application on AWS using EC2 instances, an Application Load Balancer (ALB), and Cloudflare DNS. It also covers MongoDB Atlas configuration and Nginx reverse proxy setup.

Prerequisites

AWS account with permissions for EC2, Load Balancers, and Security Groups.

Key pair (.pem) for SSH.

MongoDB Atlas account.

Basic Git, SSH, and Linux terminal knowledge.

Step 1 — Provision EC2 Instances

Launch 4 Ubuntu EC2 instances (2 for frontend, 2 for backend).

Configure Security Groups to allow:

Port 22 (SSH)

Port 80 (HTTP)

Port 3000 (Frontend)

Port 3001 (Backend)

Step 2 — Install Node.js and npm

sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

Verify:

node -v
npm -v

Step 3 — Configure MongoDB Atlas

Create a free cluster in Atlas.

Add a DB user with read/write access.

Whitelist EC2 public IPs or use 0.0.0.0/0 (testing only).

Get the connection URI and set it in .env for the backend:

MONGO_URI='your_connection_uri'
PORT=3001

Step 4 — Deploy Backend

SSH into backend instances.

Clone repo:

git clone https://github.com/aviral31/TravelMemory.git
cd TravelMemory/backend
npm install
node index.js

Use PM2 for process management.

Step 5 — Deploy Frontend

SSH into frontend instances.

Go to frontend folder:

cd TravelMemory/frontend

Set API endpoint in .env:

REACT_APP_BACKEND_URL=http://<backend-instance-ip>:3001

Build and move files:

npm install
npm run build
sudo mv build /var/www/travelmemory

Step 6 — Configure Nginx Reverse Proxy

sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/travelmemory

Example config:

server {
    listen 80;
    server_name frontend.aviralpaliwal.ink;

    root /var/www/travelmemory;
    index index.html;

    location /api/ {
        proxy_pass http://<backend-private-ip>:3001/;
    }

    location / {
        try_files $uri /index.html;
    }
}

Enable site:

sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/
sudo systemctl reload nginx

Step 7 — Create Application Load Balancer

Go to EC2 → Load Balancers → Create Load Balancer → Application Load Balancer.

Scheme: Internet-facing.

Add listeners:

HTTP:80 → Target Group (frontend, port 80)

HTTP:3000 → Target Group (frontend, port 3000)

Register frontend instances in the Target Groups.

Step 8 — Configure Cloudflare CNAME

Add CNAME record:

Name: frontend

Target: <ALB-DNS-Name>

Proxy Status: DNS only.

Step 9 — Test Deployment

Access app via: http://frontend.aviralpaliwal.ink

Verify frontend loads and backend API is functional.
