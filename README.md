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
```
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
```

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

---
# AI Prompts
## prompt 1
I need to create a Docker Compose setup with three services: A web frontend (served by Nginx on Ubuntu) it should say "*name retrieved from database* has reached milestone1!!!" make the website simple no buttons only the message stated above and it should remain persistent through restarts and i should be able to change the name and after a refresh it to appear on the frontend site i also want for the container id to be below my name A Flask API that talks to MongoDB A MongoDB database with persistent storage and seeded data I also want to set specific container names: contmongo-m1-SG for MongoDB contnginx-m1-SG for nginx contapi-m1-SG for the API Can you outline the directory structure and main config files I’ll need so I can start building it? additionally I want to enable HTTPS with a self-signed certificate in Nginx for the web service. you should give me the steps and minimal config changes needed to do that, keeping my existing container names? i also want you to give me the updated directory structure and necessary commands to make this as smooth as possible
```
You want a dead-simple, three-service stack that actually works:

web: Nginx serves a tiny static page over HTTPS and proxies /api/ to the Flask API.

api: Flask talks to MongoDB, returns the current name + its container ID, and lets you update the name at runtime.

mongo: persistent volume + seeded with an initial name.

I’m giving you:

directory tree,

minimal configs,

exact commands (with sudo) to create the files,

how to run it,

how to flip the name live,

the minimal HTTPS changes (keeps your container names).

1) Directory structure
milestone-stack/
├─ docker-compose.yml
├─ web/
│  ├─ Dockerfile
│  ├─ nginx.conf
│  ├─ entrypoint.sh
│  └─ html/
│     └─ index.html
├─ api/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  └─ app.py
└─ mongo-init/
   └─ init.js

2) Core files
docker-compose.yml
services:
  mongo:
    image: mongo:7
    container_name: contmongo-m1-SG
    restart: unless-stopped
    volumes:
      - mongo-data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 10s
      timeout: 5s
      retries: 10

  api:
    build: ./api
    container_name: contapi-m1-SG
    restart: unless-stopped
    environment:
      - MONGO_URL=mongodb://mongo:27017
      - MONGO_DB=milestone
      - MONGO_COLL=settings
    depends_on:
      mongo:
        condition: service_healthy
    expose:
      - "5000"

  web:
    build: ./web
    container_name: contnginx-m1-SG
    restart: unless-stopped
    ports:
      # Host 8085 -> container 443 (HTTPS)
      - "8085:443"
    depends_on:
      - api
    healthcheck:
      test: ["CMD", "wget", "-qO-", "https://localhost/healthz", "--no-check-certificate"]
      interval: 10s
      timeout: 5s
      retries: 10
    # If you’re on older Compose, add: networks: [default]

volumes:
  mongo-data:

web/Dockerfile
FROM nginx:1.27-alpine

RUN apk add --no-cache openssl bash

# Copy nginx config and static site
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

# Entrypoint will create a self-signed cert on first boot if missing
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 443
HEALTHCHECK CMD wget -qO- https://localhost/healthz --no-check-certificate || exit 1
ENTRYPOINT ["/entrypoint.sh"]

web/nginx.conf
worker_processes auto;

events {}

http {
  server_tokens off;
  include       mime.types;
  default_type  application/octet-stream;
  sendfile      on;

  # Redirect HTTP->HTTPS (optional: only if you also expose 80)
  # server {
  #   listen 80;
  #   return 301 https://$host$request_uri;
  # }

  upstream api_upstream {
    server api:5000;
  }

  server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    # Health
    location = /healthz { return 200 "ok"; add_header Content-Type text/plain; }

    # Static
    root /usr/share/nginx/html;
    index index.html;

    location / {
      try_files $uri /index.html;
    }

    # API passthrough to Flask
    location /api/ {
      proxy_pass http://api_upstream;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}

web/entrypoint.sh
#!/usr/bin/env bash
set -euo pipefail

SSL_DIR=/etc/nginx/ssl
mkdir -p "$SSL_DIR"

# Generate self-signed cert if not present
if [ ! -f "$SSL_DIR/selfsigned.key" ] || [ ! -f "$SSL_DIR/selfsigned.crt" ]; then
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout "$SSL_DIR/selfsigned.key" \
    -out "$SSL_DIR/selfsigned.crt" \
    -subj "/CN=localhost"
fi

exec nginx -g 'daemon off;'

web/html/index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Milestone</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    html,body{height:100%;margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,"Helvetica Neue",Arial}
    .wrap{min-height:100%;display:flex;align-items:center;justify-content:center;background:#0b1020;color:#eef2ff}
    .card{padding:32px;border-radius:16px;background:#121a33;box-shadow:0 10px 30px rgba(0,0,0,.4);text-align:center}
    h1{margin:0 0 12px;font-size:28px;letter-spacing:.5px}
    p{margin:6px 0 0;opacity:.8}
    .cid{font-size:12px;opacity:.7;margin-top:6px}
  </style>
</head>
<body>
<div class="wrap"><div class="card">
  <h1 id="msg">Loading…</h1>
  <p class="cid" id="cid"></p>
</div></div>
<script>
(async () => {
  try {
    const r = await fetch('/api/name', {cache:'no-store'});
    if (!r.ok) throw new Error('API error');
    const { name, container_id } = await r.json();
    document.getElementById('msg').textContent = `${name} has reached milestone1!!!`;
    document.getElementById('cid').textContent = `Container: ${container_id}`;
  } catch (e) {
    document.getElementById('msg').textContent = 'Backend unavailable';
  }
})();
</script>
</body>
</html>

api/Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
ENV PYTHONUNBUFFERED=1
EXPOSE 5000
CMD ["python", "app.py"]

api/requirements.txt
Flask==3.0.3
pymongo==4.8.0

api/app.py
import os, json, socket
from flask import Flask, jsonify, request
from pymongo import MongoClient

MONGO_URL = os.getenv("MONGO_URL", "mongodb://mongo:27017")
DB_NAME   = os.getenv("MONGO_DB", "milestone")
COLL_NAME = os.getenv("MONGO_COLL", "settings")

client = MongoClient(MONGO_URL)
db = client[DB_NAME]
coll = db[COLL_NAME]

app = Flask(__name__)

def get_container_id():
    # Docker container hostname is the container ID by default
    cid = socket.gethostname()
    return cid[:12]

@app.get("/api/healthz")
def healthz():
    return "ok", 200

@app.get("/api/name")
def get_name():
    doc = coll.find_one({"_id": "display_name"}) or {"value": "Unknown"}
    return jsonify({"name": doc.get("value", "Unknown"), "container_id": get_container_id()})

@app.put("/api/name")
def set_name():
    data = request.get_json(force=True, silent=True) or {}
    name = data.get("name", "").strip()
    if not name:
        return jsonify({"error": "name required"}), 400
    coll.update_one({"_id": "display_name"}, {"$set": {"value": name}}, upsert=True)
    doc = coll.find_one({"_id": "display_name"})
    return jsonify({"name": doc["value"], "container_id": get_container_id()})

mongo-init/init.js
// Seed once at first container start
const db = db.getSiblingDB('milestone');
db.settings.updateOne(
  { _id: "display_name" },
  { $setOnInsert: { value: "Simon Gielen" } },
  { upsert: true }
);

3) One-shot commands to create everything (with sudo)

Run these in an empty target folder (e.g., ~/milestone-stack):

# Create dirs
sudo mkdir -p milestone-stack/{web/html,api,mongo-init}
cd milestone-stack

# docker-compose.yml
sudo tee docker-compose.yml > /dev/null <<'YML'
<PASTE THE docker-compose.yml FROM ABOVE>
YML

# --- WEB FILES ---
sudo tee web/Dockerfile > /dev/null <<'DOCKER'
<PASTE THE web/Dockerfile FROM ABOVE>
DOCKER

sudo tee web/nginx.conf > /dev/null <<'NGINX'
<PASTE THE web/nginx.conf FROM ABOVE>
NGINX

sudo tee web/entrypoint.sh > /dev/null <<'SH'
<PASTE THE web/entrypoint.sh FROM ABOVE>
SH
sudo chmod +x web/entrypoint.sh

sudo tee web/html/index.html > /dev/null <<'HTML'
<PASTE THE web/html/index.html FROM ABOVE>
HTML

# --- API FILES ---
sudo tee api/Dockerfile > /dev/null <<'DOCKER'
<PASTE THE api/Dockerfile FROM ABOVE>
DOCKER

sudo tee api/requirements.txt > /dev/null <<'REQ'
<PASTE THE api/requirements.txt FROM ABOVE>
REQ

sudo tee api/app.py > /dev/null <<'PY'
<PASTE THE api/app.py FROM ABOVE>
PY

# --- MONGO SEED ---
sudo tee mongo-init/init.js > /dev/null <<'JS'
<PASTE THE mongo-init/init.js FROM ABOVE>
JS

4) Build and run
# From milestone-stack/
sudo docker compose build
sudo docker compose up -d
sudo docker compose ps


Open the site (self-signed):

https://<your-host-ip>:8085/
Use -k (insecure) with curl while testing:

curl -k https://127.0.0.1:8085/healthz
curl -k https://127.0.0.1:8085/api/name


You insisted on IP, not localhost: use your VM/host IP directly, e.g. https://192.168.56.5:8085/.

If a browser “forces” HTTPS or blocks it, that’s expected for self-signed. Proceed to the site anyway or add a local trust if you want the lock icon. Functionality is unaffected.

5) Change the name live (no rebuild)

Update while everything is running, then refresh the page:

# Example change to "Milestone Boss"
curl -k -X PUT https://<HOST-IP>:8085/api/name \
  -H 'Content-Type: application/json' \
  -d '{"name":"Milestone Boss"}'

# Verify
curl -k https://<HOST-IP>:8085/api/name


The homepage will show:

Milestone Boss has reached milestone1!!!
Container: <12-char-container-id-of-web>


(Actually the API returns its own container ID. The page shows that value under the name, per your request.)

6) Minimal HTTPS steps (keeps your container names)

You asked for the smallest possible change:

Add OpenSSL and an entrypoint to the web image (web/Dockerfile, web/entrypoint.sh).

Add SSL stanza and listen 443 ssl; in web/nginx.conf.

Map host port 8085:443 in Compose.

That’s it. No external cert tooling, no ACME.

If you ever want HTTP→HTTPS redirect, expose 80 and uncomment the redirect server block.

7) Why this works (and avoids classic traps)

Browser JS can’t resolve api container DNS from your host. So Nginx proxies /api/ internally to api:5000. Clean, single origin, no CORS drama.

Mongo seeds exactly once via docker-entrypoint-initdb.d, and data persists in mongo-data.

Name is mutable at runtime: PUT /api/name, frontend fetches on page load.

Container names are exactly what you demanded:

contmongo-m1-SG, contapi-m1-SG, contnginx-m1-SG.

8) Quick troubleshooting (no sugar-coating)

Page says “Backend unavailable”: your web can’t reach api. Run sudo docker compose logs web api. Usually a typo in proxy_pass or API build failed.

curl -k https://IP:8085/api/name hangs or 502: API not healthy. Check contapi-m1-SG logs.

Browser blocks self-signed: that’s not a bug. Accept the risk or import the cert to your trust store.

You only see localhost working: you’re testing from the VM/host mismatch. Hit the actual IP you mapped (e.g., 192.168.56.5:8085), not loopback.

If you want me to convert the heredoc placeholders into a single copy-paste blob with all content inlined, say so and I’ll drop the full ready-to-paste script.
```
## Reflection on prompt
*This first prompt made it work first try so no further tweaking was needed now i am adding load balancing*
---
## Prompt 2
vagrant@vagrant:~/milestone-stack$ sudo docker compose ps
NAME              IMAGE                 COMMAND                  SERVICE   CREATED         STATUS                          PORTS
contapi-m1-SG     milestone-stack-api   "python app.py"          api       2 minutes ago   Restarting (0) 25 seconds ago
contmongo-m1-SG   mongo:7               "docker-entrypoint.s…"   mongo     2 minutes ago   Up 2 minutes (healthy)          27017/tcp
contnginx-m1-SG   milestone-stack-web   "/entrypoint.sh"         web       2 minutes ago   Up 2 minutes (unhealthy)        80/tcp, 0.0.0.0:8085->443/tcp, [::]:8085->443/tcp
vagrant@vagrant:~/milestone-stack$ curl -k https://192.168.56.5:8085/healthz
okvagrant@vagrant:~/milestone-stackcurl -k https://192.168.56.5:8085/api/nameme
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx</center>
</body>
</html>
## Response 2

```
Your API is crash-looping and Nginx marks itself unhealthy because of a tiny tooling miss.

What’s wrong (no fluff)

contapi-m1-SG keeps restarting → your api/app.py never starts the Flask server. You defined routes but didn’t call app.run(...). So the process exits immediately → Docker restarts it → Nginx upstream is dead → 502.

contnginx-m1-SG shows “unhealthy” even though /healthz works → your healthcheck uses wget but the image doesn’t have it. Install wget (or switch to curl).

Fix both in under 2 minutes.

1) Patch the API (start the server)

Replace api/app.py with this (or add the last block exactly as shown):

sudo tee api/app.py > /dev/null <<'PY'
import os, json, socket
from flask import Flask, jsonify, request
from pymongo import MongoClient

MONGO_URL = os.getenv("MONGO_URL", "mongodb://mongo:27017")
DB_NAME   = os.getenv("MONGO_DB", "milestone")
COLL_NAME = os.getenv("MONGO_COLL", "settings")

client = MongoClient(MONGO_URL)
db = client[DB_NAME]
coll = db[COLL_NAME]

app = Flask(__name__)

def get_container_id():
    cid = socket.gethostname()
    return cid[:12]

@app.get("/api/healthz")
def healthz():
    return "ok", 200

@app.get("/api/name")
def get_name():
    doc = coll.find_one({"_id": "display_name"}) or {"value": "Unknown"}
    return jsonify({"name": doc.get("value", "Unknown"), "container_id": get_container_id()})

@app.put("/api/name")
def set_name():
    data = request.get_json(force=True, silent=True) or {}
    name = data.get("name", "").strip()
    if not name:
        return jsonify({"error": "name required"}), 400
    coll.update_one({"_id": "display_name"}, {"$set": {"value": name}}, upsert=True)
    doc = coll.find_one({"_id": "display_name"})
    return jsonify({"name": doc["value"], "container_id": get_container_id()})

if __name__ == "__main__":
    # Crucial: actually start the server
    app.run(host="0.0.0.0", port=5000)
PY


Rebuild + restart only the API:

sudo docker compose build api
sudo docker compose up -d api


Sanity check logs:

sudo docker logs -f contapi-m1-SG --tail=50


You should see Flask listening on 0.0.0.0:5000.

2) Patch Nginx healthcheck (install wget)

Either switch healthcheck to curl or just install wget. Quickest is to install wget in your web image.

Replace web/Dockerfile with this small change (add wget to the apk add line):

sudo tee web/Dockerfile > /dev/null <<'DOCKER'
FROM nginx:1.27-alpine

RUN apk add --no-cache openssl bash wget

# Copy nginx config and static site
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

# Entrypoint will create a self-signed cert on first boot if missing
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 443
HEALTHCHECK CMD wget -qO- https://localhost/healthz --no-check-certificate || exit 1
ENTRYPOINT ["/entrypoint.sh"]
DOCKER


Rebuild + restart only the web:

sudo docker compose build web
sudo docker compose up -d web

3) Verify end-to-end

From the VM:

# Health endpoints
curl -k https://192.168.56.5:8085/healthz
curl -k https://192.168.56.5:8085/api/healthz

# Data endpoint
curl -k https://192.168.56.5:8085/api/name


Expected now:

/healthz → ok

/api/healthz → ok

/api/name → {"name":"Simon Gielen","container_id":"<12chars>"}

Then open in your host browser:

https://192.168.56.5:8085/


(Bypass the self-signed warning if Brave nags; thisisunsafe works on the interstitial.)

4) Change the name live (prove persistence)
curl -k -X PUT https://192.168.56.5:8085/api/name \
  -H 'Content-Type: application/json' \
  -d '{"name":"Milestone Boss"}'

curl -k https://192.168.56.5:8085/api/name


Refresh the homepage; it must read:

Milestone Boss has reached milestone1!!!
Container: <contapi id>

If anything still fights you

502 again? Then sudo docker logs contapi-m1-SG --tail=200 and fix whatever stack trace it shows (but adding app.run(...) usually ends the loop).

Web still “unhealthy”? Run:

sudo docker exec contnginx-m1-SG sh -lc 'nginx -T | sed -n "1,140p"; which wget; ls -l /etc/nginx/ssl'


If wget missing, your image didn’t rebuild. Do a clean:

sudo docker compose build --no-cache web
sudo docker compose up -d web


You were one missing line (app.run) away. Now it’ll behave.
```

## Prompt 3
there is something wrong with my api bc it says "backend unavailable"

## Response 3
```
“Backend unavailable” means your frontend’s fetch('/api/name') got a non-2xx (likely 502) or no response. Fix it methodically—no guessing.

Step 1 — Check containers
sudo docker compose ps
sudo docker logs --tail=100 contapi-m1-SG
sudo docker logs --tail=100 contnginx-m1-SG


If contapi-m1-SG is Restarting/Exited, jump to Step 4.

If it’s Up, keep going.

Step 2 — Hit the API through Nginx
curl -ik https://192.168.56.5:8085/api/healthz
curl -ik https://192.168.56.5:8085/api/name


200 OK = frontend message should load.

502/504 = Nginx can’t reach API → go to Step 3.

Step 3 — From inside web container, talk to API by service name

This isolates proxy/DNS problems.

# DNS resolution for "api"
sudo docker exec -it contnginx-m1-SG sh -lc 'getent hosts api || cat /etc/resolv.conf'

# Direct API health (bypasses TLS—this is inside the Docker network)
sudo docker exec -it contnginx-m1-SG sh -lc 'wget -qO- http://api:5000/api/healthz || echo FAIL'
sudo docker exec -it contnginx-m1-SG sh -lc 'wget -qO- http://api:5000/api/name || echo FAIL'


If these FAIL, your API isn’t listening or is crashing → Step 4.

If these return ok/JSON, Nginx config is wrong. Ensure this exact stanza exists:

web/nginx.conf (must match):

upstream api_upstream { server api:5000; }

server {
  listen 443 ssl;
  ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
  ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

  location = /healthz { return 200 "ok"; add_header Content-Type text/plain; }

  root /usr/share/nginx/html;
  index index.html;

  location / { try_files $uri /index.html; }

  location /api/ {
    proxy_pass http://api_upstream/; # keep trailing slash
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}


Then:

sudo docker compose build web
sudo docker compose up -d web

Step 4 — Prove the API is actually listening

Inside the API container:

sudo docker exec -it contapi-m1-SG sh -lc 'ss -lntp | grep ":5000" || python -c "import socket as s,os;print(os.uname());"'


No LISTEN on :5000 → your app.py isn’t starting the server.

Your api/app.py must end with:

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)


If it doesn’t, fix it and rebuild:

sudo sed -n '1,200p' api/app.py
# If missing, edit and add the app.run block, then:
sudo docker compose build api
sudo docker compose up -d api

Step 5 — Kill the common crash: Mongo connection errors

If logs show ServerSelectionTimeoutError, your API is trying to query Mongo before it’s reachable or with the wrong URI.

Check envs:

sudo docker inspect contapi-m1-SG | grep -A2 -E '"MONGO_URL"|"MONGO_DB"|"MONGO_COLL"'


They must be:

MONGO_URL=mongodb://mongo:27017
MONGO_DB=milestone
MONGO_COLL=settings


MongoClient() doesn’t connect at import time; the first find_one does. If it times out, you’ll see 502 for ~5–10s after boot. Two fixes:

A) Keep it simple (recommended): you already have depends_on on mongo with healthcheck—leave it, the API will work once Mongo is up.

B) Defensive handler (optional): wrap the call so API returns 503 instead of crashing:

from pymongo.errors import PyMongoError
@app.get("/api/name")
def get_name():
    try:
        doc = coll.find_one({"_id": "display_name"}) or {"value": "Unknown"}
        return jsonify({"name": doc.get("value", "Unknown"), "container_id": get_container_id()})
    except PyMongoError as e:
        return jsonify({"error": "db_unavailable"}), 503


Rebuild:

sudo docker compose build api
sudo docker compose up -d api

Step 6 — Re-test end-to-end
curl -ik https://192.168.56.5:8085/healthz
curl -ik https://192.168.56.5:8085/api/healthz
curl -ik https://192.168.56.5:8085/api/name


If those are 200, reload the page. If the page still says “Backend unavailable”, your web/html/index.html JS is failing. Sanity check it:

Must be exactly this fetch (note the leading slash and no http/https in URL):

<script>
(async () => {
  try {
    const r = await fetch('/api/name', {cache:'no-store'});
    if (!r.ok) throw new Error('API error');
    const { name, container_id } = await r.json();
    document.getElementById('msg').textContent = `${name} has reached milestone1!!!`;
    document.getElementById('cid').textContent = `Container: ${container_id}`;
  } catch (e) {
    document.getElementById('msg').textContent = 'Backend unavailable';
  }
})();
</script>

Zero-BS summary

502 = Nginx → API broken path or API not listening.

Fix order: API listening on 0.0.0.0:5000, Nginx proxy /api/ → api:5000 with trailing slash, Mongo URI correct.

Don’t trust the browser page—curl the endpoints; the HTTP codes will tell you the truth.
```

## AI Assistance Reflection:
  I used generative AI to:
  Design the stack layout (web → API → DB).
  Write docker-compose.yml and config files.
  Debug proxy path issues (/api/name mapping).
  Add scaling & TLS support.
  Document steps and explain commands.
  This allowed me to focus on understanding the architecture rather than memorizing syntax.


