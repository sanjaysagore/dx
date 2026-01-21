# 🚀 FastAPI + Angular Deployment on AWS EC2  
**Docker • AWS ECR • Traefik • HTTPS • Custom Domain**

---

## 📌 Overview

This repository documents a **production-grade deployment** of:

- 🐍 **FastAPI backend**
- 🎨 **Angular frontend**
- 🐳 **Docker & Docker Compose**
- ☁️ **AWS EC2 + AWS ECR**
- 🔐 **HTTPS (Let’s Encrypt via Traefik)**
- 🌍 **Custom domain (GoDaddy)**

The setup uses **IAM Roles (no access keys on EC2)** and supports **multiple domains over HTTPS**.

---

## 🧱 Architecture

```
User Browser
   │
   │  https://jnvlink.com
   │
   ▼
Traefik (HTTPS + Let's Encrypt)
   │
   ├── Angular Frontend
   │
   └── FastAPI Backend
```

---

## 📁 Project Structure

```
project-root/
├── backend/
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/
│   ├── dist/                 # Angular production build
│   └── Dockerfile
│
├── traefik/
│   ├── traefik.yml
│   └── letsencrypt/
│       └── acme.json
│
├── docker-compose.yml
└── README.md
```

---

## 🐍 Backend – FastAPI

### backend/main.py

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/api/health")
def health():
    return {"status": "ok"}
```

---

### backend/requirements.txt

```text
fastapi
uvicorn
```

---

### backend/Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 🎨 Frontend – Angular

### Build Angular locally

```bash
cd frontend
ng build --configuration production
```

---

### frontend/Dockerfile

```dockerfile
FROM nginx:alpine
COPY dist /usr/share/nginx/html
EXPOSE 80
```

---

## 🐳 Local Development

```bash
docker-compose up --build
```

Local:
- UI: http://localhost:8080
- API: http://localhost:8080/api/health

---

## ☁️ AWS ECR

### Configure AWS CLI (local)

```bash
aws configure
```

### Create repositories

```bash
aws ecr create-repository --repository-name dx-backend
aws ecr create-repository --repository-name dx-frontend
```

### Login to ECR

```bash
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin 262101605315.dkr.ecr.ap-south-1.amazonaws.com
```

### Tag & push images

```bash
docker tag dx-backend:1.0 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-backend:1.0
docker tag dx-frontend:1.0 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-frontend:1.0

docker push 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-backend:1.0
docker push 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-frontend:1.0
```

---

## 🖥️ EC2 Setup

```bash
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

Logout and login again.

---

## 🔐 IAM Role

Attach policy:

```
AmazonEC2ContainerRegistryReadOnly
```

Verify:

```bash
aws sts get-caller-identity
```

---

## 🌍 Domain (GoDaddy)

DNS records:

| Type | Host | Value |
|----|----|----|
| A | @ | EC2_PUBLIC_IP |
| A | www | EC2_PUBLIC_IP |
| A | api | EC2_PUBLIC_IP |

---

## 🔐 HTTPS with Traefik

### traefik/traefik.yml

```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@gmail.com
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web
```

Create cert storage:

```bash
mkdir -p traefik/letsencrypt
touch traefik/letsencrypt/acme.json
chmod 600 traefik/letsencrypt/acme.json
```

---

## 🧩 docker-compose.yml (Production)

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik/traefik.yml:/traefik.yml
      - ./traefik/letsencrypt:/letsencrypt
    restart: always

  backend:
    image: 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-backend:1.0
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`api.jnvlink.com`)"
      - "traefik.http.routers.backend.entrypoints=websecure"
      - "traefik.http.routers.backend.tls.certresolver=letsencrypt"
      - "traefik.http.services.backend.loadbalancer.server.port=8000"

  frontend:
    image: 262101605315.dkr.ecr.ap-south-1.amazonaws.com/dx-frontend:1.0
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`jnvlink.com`,`www.jnvlink.com`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls.certresolver=letsencrypt"
      - "traefik.http.services.frontend.loadbalancer.server.port=80"
```

---

## 🚀 Run on EC2

```bash
docker-compose pull
docker-compose up -d
```

---

## ✅ Verify

- https://jnvlink.com
- https://www.jnvlink.com
- https://api.jnvlink.com/api/health

---

## 🏁 Conclusion

This setup represents a **real-world enterprise deployment** of **FastAPI + Angular** on AWS using Docker, ECR, and Traefik with HTTPS.
