# 📘 TravelMemory AWS Deployment Guide (EC2 + Nginx + ALB + Cloudflare)

This document explains the complete deployment of the TravelMemory MERN application on AWS using EC2, Nginx, PM2, Application Load Balancer (ALB), and Cloudflare DNS.

The goal of this deployment is to make the application:

* Highly available (multi-instance setup)
* Scalable (load-balanced architecture)
* Production-ready (Nginx + PM2 + proper routing)
* Secure (controlled access + environment variables)

---

# 1. Target Architecture
The final architecture of the system is:
 <img width="849" height="648" alt="image" src="https://github.com/user-attachments/assets/c2cc6bb8-474a-42e6-8c07-ba8759b03f68" />


### Flow Explanation:
* Users access the application via a custom domain (Cloudflare)
* Cloudflare routes traffic to AWS Application Load Balancer
* ALB distributes requests across multiple EC2 instances
* Each EC2 instance runs:

  * Nginx (serves frontend)
  * Node.js backend (port 3000 via PM2)
* Backend connects to MongoDB Atlas database
---
# 2. Prerequisites

Before starting deployment, ensure the following are available:

* AWS account with EC2, ALB access
* Cloudflare domain configured
* SSH `.pem` key for EC2 access
* MongoDB Atlas cluster
* GitHub repository access
---

# 3. AWS Security Configuration
Security Groups are configured as follows:

### Inbound Rules:
* HTTP (80) → `0.0.0.0/0`
* HTTPS (443) → `0.0.0.0/0`
* SSH (22) → `0.0.0.0/0`
 

 <img width="785" height="392" alt="image" src="https://github.com/user-attachments/assets/a0d9d019-bebb-402b-ae0e-6261b861a903" />

<img width="786" height="389" alt="image" src="https://github.com/user-attachments/assets/2c2e196a-393c-4f55-9c81-f42c2893bdd2" />


### Outbound Rules:

* All traffic allowed (default)

### Security Best Practices Applied:

* No credentials stored in code
* Environment variables used for secrets
* SSH restricted (recommended: only personal IP)
---
# 4. EC2 Instance Setup

Two EC2 instances are created for high availability:
<img width="798" height="371" alt="image" src="https://github.com/user-attachments/assets/3696a336-2edd-4c3c-bc5f-b4e10e246ec3" />
<img width="789" height="417" alt="image" src="https://github.com/user-attachments/assets/e30c4f9e-17e3-4138-b995-3e298b9470cf" />


### Configuration:

* Ubuntu 22.04 LTS
* t2.micro (free tier)
* Same key pair
* Same security group


### Instances Used:

* Instance 1: Frontend + Backend
* Instance 2: Frontend + Backend
---

# 5. EC2 Server Setup
 
SSH into both instances:
```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```
<img width="861" height="308" alt="image" src="https://github.com/user-attachments/assets/12130e5f-91ce-4055-b9bb-62df49133213" />

### Install dependencies:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git curl ca-certificates gnupg
```
### Install Node.js and PM2:

```bash
sudo apt install -y nodejs
sudo npm install -g pm2
```
### Verify installation:

```bash
node -v
npm -v
pm2 -v
```
---
# 6. Application Setup

## 6.1 Clone Repository
```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory
```
---
## 6.2 Backend Setup
 
Navigate to backend:

```bash
cd backend
npm install
```

<img width="714" height="368" alt="image" src="https://github.com/user-attachments/assets/4993e74f-73f4-467f-a314-68e8b64e6643" />


### Create `.env` file:
```env
MONGO_URI=mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/travelmemory
PORT=3000
```
### Start backend using PM2:

```bash
pm2 start index.js --name travelmemory-backend
pm2 save
pm2 startup
```
### Backend role:

* Handles API requests (`/trip`, etc.)
* Runs on port 3000
* Managed by PM2 for uptime
---
## 6.3 Frontend Setup
Navigate to frontend:

```bash
cd ../frontend
npm install
```

### Create `.env`:
```env
REACT_APP_BACKEND_URL=/api
```
### Build frontend:

```bash
npm run build
```
<img width="856" height="431" alt="image" src="https://github.com/user-attachments/assets/a0fc524e-ad5b-44c5-addc-e989af5e9eee" />


### Frontend role:
* React UI served via Nginx
* Communicates with backend via `/api`
---

# 7. Nginx Configuration (Reverse Proxy)
Nginx is used to:
 
* Serve React build
* Route API requests to backend

<img width="746" height="311" alt="image" src="https://github.com/user-attachments/assets/0996c00b-22cc-4593-917e-13119f954e3e" />


### Setup config:
```bash
sudo cp ~/TravelMemory/automation/nginx-travelmemory.conf /etc/nginx/sites-available/travelmemory
```
### Enable site:
```bash
sudo ln -sf /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/travelmemory
sudo rm -f /etc/nginx/sites-enabled/default
```
### Validate and restart:
```bash
sudo nginx -t
sudo systemctl reload nginx
```
### Nginx responsibilities:
* Serves frontend (`build/index.html`)
* Proxies `/api` → `localhost:3000`
* Ensures single-domain architecture
---
# 8. Application Load Balancer (ALB)
 
 <img width="749" height="376" alt="image" src="https://github.com/user-attachments/assets/e99f2b78-1db7-4b7c-99c6-a5f579ad1a17" />
<img width="797" height="405" alt="image" src="https://github.com/user-attachments/assets/e04869d0-1798-4f87-8f17-a3ef543b0bf9" />
<img width="803" height="402" alt="image" src="https://github.com/user-attachments/assets/1941123d-01db-4986-af32-85d19145b088" />


 

### Steps:
1. Create Target Group (HTTP:80)
2. Register EC2 instances
3. Create Application Load Balancer
4. Attach Target Group
5. Set health check path `/`

### Purpose:
* Distributes traffic across EC2 instances
* Ensures high availability
* Enables failover if one instance goes down
---
# 9. Cloudflare DNS Configuration
<img width="700" height="367" alt="image" src="https://github.com/user-attachments/assets/cf95ac2e-be91-45bb-b80e-e73da6f55087" />
 
### DNS Records:

#### CNAME (Main domain)
```
travel.yourdomain.com → ALB DNS
```

#### A Record (Origin testing)
```
origin.yourdomain.com → EC2 Public IP
```

### Role of Cloudflare:
* DNS routing
* Optional SSL termination
* Traffic proxying
---




# 10. Verification Steps

 <img width="691" height="223" alt="image" src="https://github.com/user-attachments/assets/4af2da4b-33d1-41e8-aff9-ed34a4d5ae05" />
<img width="729" height="380" alt="image" src="https://github.com/user-attachments/assets/40a67d4a-b460-4f4e-bc26-2d2afb56d895" />
<img width="735" height="383" alt="image" src="https://github.com/user-attachments/assets/04d89bcc-fe9e-49f4-b0bc-604302d555e0" />
<img width="754" height="394" alt="image" src="https://github.com/user-attachments/assets/4ccbe9ee-5683-42f6-8b14-c37a6a18fdc9" />
<img width="756" height="377" alt="image" src="https://github.com/user-attachments/assets/c8177707-a03a-4f48-8e8f-85e5b0cb145e" />



 


 

 

 

After deployment, verify:
* Website loads via domain
* API calls working (`/api`)
* ALB shows healthy targets
* PM2 backend running
* Nginx active (`systemctl status nginx`)
---

# 11. Troubleshooting
### SSH Issues:
* Check security group port 22
* Ensure correct public IP
### Website not loading:
* Verify Nginx running
* Check port 80 open

### 502 Error:
* Backend not running
* PM2 process failed

### ALB unhealthy:
* Nginx not responding on `/`
* Security group blocking traffic
---
# 13. Architecture Diagram
 

<img width="554" height="629" alt="image" src="https://github.com/user-attachments/assets/9a1d6653-6bd4-4196-8eaf-11c688033313" />



Flow representation:
```
User
 ↓
Cloudflare
 ↓
AWS ALB
 ↓
EC2 Instance 1   EC2 Instance 2
 ↓                ↓
Nginx             Nginx
 ↓                ↓
Node Backend (PM2)
 ↓
MongoDB Atlas
```

# 14. Cost Optimization

To avoid unnecessary AWS charges:

### Stop EC2 Instances
aws ec2 stop-instances --instance-ids <id>

### Delete Load Balancer (Important)
ALB continues to incur cost even when idle.

### Release Elastic IPs
Unattached Elastic IPs are chargeable.

### Recommended Workflow:
1. Start resources before demo
2. Stop/delete after use
3. Recreate when needed


