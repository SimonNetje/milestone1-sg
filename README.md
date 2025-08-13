# Milestone 1 – Docker Web + MongoDB – Simon Gielen

## 📌 Overview
This project sets up a **web stack** using Docker Compose that:
- Serves a webpage from an **Ubuntu 24.04 Nginx** container.
- Fetches the student’s name live from **MongoDB**.
- Keeps data **persistent** on the host (survives container restarts).
- Supports **horizontal scaling** for load balancing (bonus).
- Supports **TLS encryption** with a self-signed certificate (bonus).

---

## 🗂 Architecture
Browser --> [LB: Nginx] --> [Web: Ubuntu 24.04 + Nginx]
--> /api -> [API: Flask] -> [MongoDB]


**Services:**
- **web**: Ubuntu 24.04 + Nginx, serves static HTML + JS, proxies `/api` to Flask API.
- **api**: Flask app that queries MongoDB and returns JSON.
- **mongo**: MongoDB 7.x, seeded with `Simon Gielen` on first run.
- **lb**: Nginx load balancer for scaling + TLS.

---

## 📁 Project Structure
milestone_sg/
├─ docker-compose.yml
├─ web/
│ ├─ Dockerfile
│ ├─ nginx.conf
│ ├─ html/
│ │ └─ index.html
│ └─ certs/
├─ api/
│ ├─ app.py
│ └─ requirements.txt
├─ mongo/
│ ├─ init/
│ │ └─ init.js
│ └─ data/
└─ lb/
└─ nginx.conf


---

## ⚙ Configuration Files


---

## ⚙ Configuration Files

### `web/Dockerfile`
```dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends nginx ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
```
### `web/nginx.conf`
```
worker_processes auto;
events { worker_connections 1024; }

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;
  sendfile      on;
  server_tokens off;

  server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.html;

    sub_filter_once off;
    sub_filter_types text/html;
    sub_filter "___HOST___" $hostname;

    location /api/ {
      proxy_pass http://api:5000;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }

  server {
    listen 443 ssl;
    server_name milestone.local;

    ssl_certificate     /etc/nginx/certs/milestone.crt;
    ssl_certificate_key /etc/nginx/certs/milestone.key;

    root /var/www/html;
    index index.html;

    sub_filter_once off;
    sub_filter_types text/html;
    sub_filter "___HOST___" $hostname;

    location /api/ {
      proxy_pass http://api:5000;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```
## How to run
### (Optional) create tls cert
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout web/certs/milestone.key -out web/certs/milestone.crt \
  -subj "/CN=milestone.local"
```

### Add to your host machine hosts file:
```
192.168.56.5 milestone.local
```

### Start services
```
docker compose up -d --build
```

## Open website
- HTTP: http://192.168.56.5:8085
- HTTPS (bonus): https://milestone.local:8443

### Dynamic content test
```
docker exec -it contmongo-m1-SG mongosh --eval \
'db.getSiblingDB("milestone").students.updateOne({}, {$set:{fullname:"Test Name"}})'
```

### Persistence test
```
docker compose down
docker compose up -d
```
### Horizontal Scaling (Bonus)
```
docker compose up -d --build --scale web=3
```
### TLS Encryption (Bonus)
```
curl -vk https://milestone.local:8443/api/name
```
## AI Assistance Reflection:
  I used generative AI to:
  Design the stack layout (web → API → DB).
  Write docker-compose.yml and config files.
  Debug proxy path issues (/api/name mapping).
  Add scaling & TLS support.
  Document steps and explain commands.
  This allowed me to focus on understanding the architecture rather than memorizing syntax.


