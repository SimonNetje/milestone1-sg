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
├── docker-compose.yml
├── web/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── html/
│   │   └── index.html
│   └── certs/
├── api/
│   ├── app.py
│   └── requirements.txt
├── mongo/
│   ├── init/
│   │   └── init.js
│   └── data/
└── lb/
    └── nginx.conf

---
## ✅ Requirements Mapping
| Requirement | Implementation |
|-------------|----------------|
| Serve a webpage from a container | `web` service (Ubuntu 24.04 + Nginx) serving `index.html` |
| Display student name from database | JavaScript fetch to `/api/name` → Flask → MongoDB |
| Database persistence | MongoDB volume `./mongo/data:/data/db` |
| Reverse proxy from Nginx to API | `web/nginx.conf` proxies `/api` to Flask |
| Load balancing | `m1-lb` container with upstream to `web` replicas |
| HTTPS support | Self-signed cert in `/web/certs` mounted into Nginx |
| Horizontal scaling | `docker compose up -d --scale web=3` |
| Works after restart | Data remains via bound volumes |
| Bonus: Container hostname shown | `___HOST___` replaced with `$hostname` in Nginx sub_filter |
---
## 🔒 Security Notes
- **HTTPS/TLS**: Self-signed certificate encrypts traffic between browser and LB.
- **Reverse Proxy**: LB hides backend service IPs, reduces direct attack surface.
- **Internal Network**: Services communicate on isolated Docker network `m1net`.
---

## ⚙ Configuration Files


---

## ⚙ Configuration Files

docker-compose.yaml
```
version: "3.9"
services:
  web:
    build:
      context: ./web
    depends_on:
      - api
    volumes:
      - ./web/html:/var/www/html:ro
      - ./web/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./web/certs:/etc/nginx/certs:ro
    networks: [ m1net ]

  api:
    image: python:3.12-slim
    container_name: contapi-m1-SG
    working_dir: /app
    command: sh -c "pip install --no-cache-dir -r requirements.txt && python app.py"
    volumes:
      - ./api:/app:ro
    environment:
      - MONGO_URI=mongodb://contmongo-m1-SG:27017
      - MONGO_DB=milestone
      - MONGO_COLL=students
    depends_on:
      - mongo
    networks: [ m1net ]

  mongo:
    image: mongo:7
    container_name: contmongo-m1-SG
    volumes:
      - ./mongo/data:/data/db
      - ./mongo/init:/docker-entrypoint-initdb.d:ro
    networks: [ m1net ]

  lb:
    image: nginx:alpine
    container_name: m1-lb
    depends_on:
      - web
    volumes:
      - ./lb/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./web/certs:/etc/nginx/certs:ro
    ports:
      - "8085:80"
      - "8443:443"
    networks: [ m1net ]

networks:
  m1net: {}
```

### `web/Dockerfile`
```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends nginx ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

# Copy nginx config from host (mounted at runtime), fallback kept here so image can start without mount.
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

    # Replace placeholder in HTML with container hostname to prove load-balancing
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

  # Optional TLS server. Enable by mounting certs and mapping 443 in compose.
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
      proxy_pass http://api:5000/;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

### web/html/index.html
```
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Milestone 1</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; padding: 2rem; }
    h1 { font-size: 2rem; }
    .muted { color: #666; }
    .box { margin-top: 1rem; padding: 0.75rem 1rem; border: 1px solid #ddd; border-radius: 8px; }
  </style>
</head>
<body>
  <h1 id="msg">Loading…</h1>
  <div class="box">
    <div class="muted">Served by container:</div>
    <div id="host">___HOST___</div>
  </div>

  <script>
    fetch('/api/name')
      .then(r => r.json())
      .then(d => {
        const name = d.fullname || "Unknown";
        document.getElementById('msg').textContent = name + " has reached Milestone 1!!!";
      })
      .catch(e => {
        document.getElementById('msg').textContent = 'API error: ' + e;
      });
  </script>
</body>
</html>
```
### mongo/init
```
db = db.getSiblingDB('milestone');
db.createCollection('students');
db.students.insertOne({ fullname: "Simon Gielen" });
```
### api/app.py
```
from flask import Flask, jsonify
from pymongo import MongoClient
import os, socket

app = Flask(__name__)

MONGO_URI = os.environ.get("MONGO_URI", "mongodb://contmongo-m1-SG:27017")
DB_NAME = os.environ.get("MONGO_DB", "milestone")
COLL_NAME = os.environ.get("MONGO_COLL", "students")

client = MongoClient(MONGO_URI)
db = client[DB_NAME]
coll = db[COLL_NAME]

@app.get('/api/name')
def get_name():
    doc = coll.find_one({})
    fullname = (doc or {}).get('fullname', 'Unknown')
    return jsonify({"fullname": fullname})

@app.get('/api/meta')
def meta():
    return jsonify({"hostname": socket.gethostname()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
### api/requirements.txt
```
flask==3.0.3
pymongo==4.8.0
```
### lb/nginx.conf
```
worker_processes auto;
events { worker_connections 1024; }
http {
  resolver 127.0.0.11 valid=10s ipv6=off;

  upstream web_backend {
    zone web_backend 64k;
    server web:80 resolve;
  }

  server {
    listen 80;
    location / {
      proxy_pass http://web_backend;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }

  server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/certs/milestone.crt;
    ssl_certificate_key /etc/nginx/certs/milestone.key;
    location / {
      proxy_pass http://web_backend;
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
# POC
<img width="815" height="347" alt="image" src="https://github.com/user-attachments/assets/8e335773-7fab-4997-83d7-7c86b3de2a4f" />

## AI Assistance Reflection:
  I used generative AI to:
  Design the stack layout (web → API → DB).
  Write docker-compose.yml and config files.
  Debug proxy path issues (/api/name mapping).
  Add scaling & TLS support.
  Document steps and explain commands.
  This allowed me to focus on understanding the architecture rather than memorizing syntax.


