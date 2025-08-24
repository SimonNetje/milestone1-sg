# Milestone 1 – Simon Gielen

## 📌 Overview
This project sets up a **web stack** using Docker Compose that:
- Serves a webpage from an **Ubuntu 24.04 Nginx** container.
- Fetches the student’s name live from **MongoDB**.
- Keeps data **persistent** on the host (survives container restarts).
- Supports **TLS encryption** with a self-signed certificate (bonus).

---


**Services:**
- **web**: Ubuntu 24.04 + Nginx, serves static HTML + JS, proxies `/api` to Flask API.
- **api**: Flask app that queries MongoDB and returns JSON.
- **mongo**: MongoDB 7.x, seeded with `Simon Gielen` on first run.

---

## 📁 Project Structure
```
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
```

---
## ✅ Requirements Mapping
| Requirement | Implementation |
|-------------|----------------|
| Serve a webpage from a container | `web` service (Ubuntu 24.04 base, installs Nginx) serving `index.html` |
| Display student name from database | Frontend JavaScript `fetch('/api/name')` → Nginx proxy → Flask API (`api`) → MongoDB |
| Database persistence | Named volume `mongo-data:/data/db` ensures Mongo data survives container restarts |
| Reverse proxy from Nginx to API | `web/nginx.conf` routes `/api/` → `api:5000` via `proxy_pass` |
| HTTPS support | Self-signed certificate generated in `web/entrypoint.sh`, served by Nginx on port 443 (`8085:443` mapping) |
| Works after restart | On refresh, frontend fetches latest value from Mongo; DB state persists due to named volume |
| Container naming rule | `contnginx2-m1-SG` (web), `contmongo-m1-SG` (mongo), `contapi-m1-SG` (api) |
| Config files documented | All configs included and explained: `docker-compose.yml`, `web/Dockerfile`, `nginx.conf`, `entrypoint.sh`, `api/Dockerfile`, `app.py`, `mongo-init/init.js` |
| Generative AI usage (mandatory) | Prompts and ChatGPT responses included in appendix, plus reflection on how AI helped debug and improve the stack |

---
## YOUTUBE DEMONSTRATION
- https://youtu.be/w-54Y0As6Fs

## 🔒 Security Notes
- **HTTPS/TLS**: All browser traffic is encrypted with a self-signed certificate generated at container startup.
- **Reverse Proxy**: Nginx acts as a reverse proxy, isolating the API from direct exposure.

## Data persistance explained
The web page updates automatically because the frontend fetches the name from the API every time it loads, so any change in MongoDB is shown after a refresh.
The data persists because MongoDB stores its files in a named Docker volume (mongo-data), which lives on the host and is not deleted when containers are removed or restarted.

---

## Install docker on VM
   1. sudo apt update
   2. sudo apt install apt-transport-https curl
   3. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --
   dearmor -o /etc/apt/keyrings/docker.gpg
   4. echo "deb [arch=$(dpkg --print-architecture) signed-
   by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu
   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee
   /etc/apt/sources.list.d/docker.list > /dev/null
   5. sudo apt update
### prevent the usage of sudo as much as possible
   1. sudo usermod -aG docker <user>
   2. newgrp docker # load the new group permissions in
restart the VM after these changes

## Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  config.vm.network "forwarded_port", guest: 8085, host: 8085
  config.vm.network "forwarded_port", guest: 443, host: 443

  config.vm.network "private_network", ip: "192.168.56.5"
end
```

## ⚙ Configuration Files

### `docker-compose.yaml`
```
services:                         # Top-level key: defines the different containers (services) in the stack
  mongo:                          # First service: MongoDB database
    image: mongo:7                # Use the official MongoDB v7 image from Docker Hub
    container_name: contmongo-m1-SG  # Explicit container name, required by the assignment
    restart: unless-stopped       # Restart the container unless it’s manually stopped
    volumes:                      # Mount host paths/volumes inside the container
      - mongo-data:/data/db       # Named volume → persists MongoDB data outside the container
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
                                   # Bind-mount local folder with init.js → seeds DB at first start
    healthcheck:                  # Define how Docker checks MongoDB health
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
                                   # Run `mongosh` ping command → returns 1 if Mongo is alive
      interval: 10s               # Run the healthcheck every 10 seconds
      timeout: 5s                 # Fail if no response within 5 seconds
      retries: 10                 # Mark unhealthy after 10 consecutive failures

  api:                            # Second service: Flask API
    build: ./api                  # Build image from ./api/Dockerfile
    container_name: contapi-m1-SG # Explicit container name (assignment requirement)
    restart: unless-stopped       # Same restart policy as others
    environment:                  # Environment variables passed to the API container
      - MONGO_URL=mongodb://mongo:27017   # Connection string for Mongo (service DNS = "mongo")
      - MONGO_DB=milestone                # Database name
      - MONGO_COLL=settings               # Collection name
    depends_on:                   # Wait until Mongo is healthy before starting API
      mongo:
        condition: service_healthy
    expose:
      - "5000"                    # Expose port 5000 on the internal Docker network (not host)

  web:                            # Third service: Nginx frontend
    build: ./web                  # Build from ./web/Dockerfile
    container_name: contnginx2-m1-SG # Explicit container name (assignment requirement)
    restart: unless-stopped
    ports:
      - "8085:443"                # Map host port 8085 → container port 443 (HTTPS)
    depends_on:
      - api                       # Web starts only after API service is up
    volumes:
      - ./web/html:/usr/share/nginx/html:ro
                                   # Mount host HTML files → ensures they live outside the container
    healthcheck:                   # Health check for Nginx
      test: ["CMD", "curl", "-ks", "https://localhost/healthz"]
                                   # Use curl to hit the Nginx health endpoint over HTTPS (-k ignores self-signed cert)
      interval: 10s                # Run check every 10 seconds
      timeout: 5s                  # Fail if it takes longer than 5 seconds
      retries: 10                  # Mark container unhealthy after 10 failed checks

volumes:                           # Declare named volumes (managed by Docker)
  mongo-data:                      # Used for MongoDB data persistence
```

### `web/Dockerfile`
```
FROM ubuntu:24.04                                   # Base image: Ubuntu 24.04, as required
ENV DEBIAN_FRONTEND=noninteractive                  # Prevent apt-get prompts during build

RUN apt-get update \                                # Update package index
 && apt-get install -y --no-install-recommends \    # Install minimal set of packages without extras
      nginx openssl curl bash \                     # Nginx (web server), OpenSSL (for self-signed certs),
                                                    # curl (used by healthcheck), bash (for scripts)
 && rm -rf /var/lib/apt/lists/*                     # Clean package lists → smaller image

COPY nginx.conf /etc/nginx/nginx.conf               # Copy custom Nginx configuration into container

# HTML files are no longer copied in (they are mounted at runtime from the host)
# COPY html/ /usr/share/nginx/html/

COPY entrypoint.sh /entrypoint.sh                   # Copy entrypoint script (generates cert + starts nginx)
RUN chmod +x /entrypoint.sh                         # Make entrypoint script executable

EXPOSE 443                                          # Document that container listens on port 443 (HTTPS)
HEALTHCHECK CMD curl -ks https://localhost/healthz || exit 1
                                                    # Define healthcheck: run curl against /healthz, ignore cert
ENTRYPOINT ["/entrypoint.sh"]                       # Run entrypoint script as PID 1

```
### `web/nginx.conf`
```
worker_processes auto;                               # Scale workers to CPU cores
events {}                                            # Use defaults (epoll/kqueue chosen automatically)

http {
  server_tokens off;                                 # Don’t leak Nginx version in headers
  include       mime.types;                          # Map file extensions to MIME types
  default_type  application/octet-stream;            # Fallback MIME
  sendfile      on;                                  # Kernel zero-copy for static files

  # (Optional HTTP→HTTPS redirect block omitted; you only expose 443)

  upstream api_upstream {                            # Name the API backend as an upstream
    server api:5000;                                 # Resolve "api" via Docker DNS, port 5000
  }

  server {
    listen 443 ssl;                                  # HTTPS listener
    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;  # Self-signed cert generated by entrypoint
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    location = /healthz { return 200 "ok"; add_header Content-Type text/plain; }
                                                      # Simple health endpoint for Docker healthcheck

    root /usr/share/nginx/html;                      # Serve your static site from here
    index index.html;

    location / {                                     # Frontend route
      try_files $uri /index.html;                    # Serve file if exists, else SPA fallback
    }

    location /api/ {                                 # Everything under /api hits Flask
      proxy_pass http://api_upstream;                # Forward path as-is to the upstream
      proxy_set_header Host $host;                   # Preserve Host header
      proxy_set_header X-Real-IP $remote_addr;       # Client IP for logging
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Standard proxy chain
    }
  }
}

```
### `web/entrypoint.sh`
```
#!/usr/bin/env bash
set -euo pipefail                                   # Fail fast; catch unset vars; strong shell hygiene

SSL_DIR=/etc/nginx/ssl
mkdir -p "$SSL_DIR"

# Generate self-signed cert if not present (first boot)
if [ ! -f "$SSL_DIR/selfsigned.key" ] || [ ! -f "$SSL_DIR/selfsigned.crt" ]; then
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout "$SSL_DIR/selfsigned.key" \
    -out "$SSL_DIR/selfsigned.crt" \
    -subj "/CN=localhost"                            # CN=localhost; acceptable for your IP test with -k
fi

exec nginx -g 'daemon off;'                          # Hand off to nginx in the foreground (PID 1)
```

### `web/html/index.html`
```
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">                               <!-- Standards-compliant encoding -->
  <title>Milestone</title>
  <meta name="viewport" content="width=device-width, initial-scale=1"> <!-- Responsive -->
  <style>
    /* Minimal, centered card UI */
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
  <h1 id="msg">Loading…</h1>                          <!-- Placeholder until API returns -->
  <p class="cid" id="cid"></p>                         <!-- API container ID goes here -->
</div></div>
<script>
(async () => {
  try {
    const r = await fetch('/api/name', {cache:'no-store'});   // Same-origin call via Nginx; no CORS hassle
    if (!r.ok) throw new Error('API error');
    const { name, container_id } = await r.json();            // Expect { name, container_id }
    document.getElementById('msg').textContent = `${name} has reached milestone1!!!`; // Assignment message
    document.getElementById('cid').textContent = `Container: ${container_id}`;        // Show API container ID
  } catch (e) {
    document.getElementById('msg').textContent = 'Backend unavailable';               // Clear error state
  }
})();
</script>
</body>
</html>

```
### `mongo-init/init.js`
```
// Seed once at first container start
const db = db.getSiblingDB('milestone');
db.settings.updateOne(
  { _id: "display_name" },
  { $setOnInsert: { value: "Simon Gielen" } },
  { upsert: true }
);
```
### `api/app.py`
```
import os, json, socket
from flask import Flask, jsonify, request
from pymongo import MongoClient

# ---- Config via env (set in docker-compose) ----
MONGO_URL = os.getenv("MONGO_URL", "mongodb://mongo:27017")
DB_NAME   = os.getenv("MONGO_DB", "milestone")
COLL_NAME = os.getenv("MONGO_COLL", "settings")

# ---- DB client and handles ----
client = MongoClient(MONGO_URL)               # Lazy-connect; actual I/O on first operation
db = client[DB_NAME]
coll = db[COLL_NAME]

app = Flask(__name__)

def get_container_id():
    cid = socket.gethostname()               # In Docker, hostname defaults to container ID
    return cid[:12]                          # Trim for readability

@app.get("/api/healthz")
def healthz():
    return "ok", 200                         # Simple liveness/readiness probe

@app.get("/api/name")
def get_name():
    doc = coll.find_one({"_id": "display_name"}) or {"value": "Unknown"}
    return jsonify({"name": doc.get("value", "Unknown"),
                    "container_id": get_container_id()})

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
    app.run(host="0.0.0.0", port=5000)       # Listen on all interfaces (required inside containers)

```
### `api/requirements.txt`
```
flask==3.0.3                                  # Web microframework
pymongo==4.8.0                                # MongoDB driver

```
### `api/Dockerfile`
```
FROM python:3.12-slim                        # Small Python base with 3.12 runtime
WORKDIR /app                                 # All paths relative to /app
COPY requirements.txt .                      # Copy dependency manifest
RUN pip install --no-cache-dir -r requirements.txt
                                             # Install Flask + PyMongo without leaving wheels/caches
COPY app.py .                                # Copy the application code
ENV PYTHONUNBUFFERED=1                       # Flush logs immediately (helpful in containers)
EXPOSE 5000                                  # Document API port (Compose uses this internally)
CMD ["python", "app.py"]                     # Start Flask dev server (sufficient for this assignment)

```
## How to run

### Start services
```
docker compose up -d --build
```

## Open website
- https://192.168.56.5:8085/

### Change name
```
curl -k -X PUT https://192.168.56.5:8085/api/name \
  -H 'Content-Type: application/json' \
  -d '{"name":"New Name"}'
```

### Persistence test
```
docker compose down
docker compose up -d
```


# POC
<img width="775" height="703" alt="image" src="https://github.com/user-attachments/assets/45418d20-afff-455e-8372-02831e406f86" />
<img width="1056" height="1159" alt="image" src="https://github.com/user-attachments/assets/09d357c7-25c4-4539-8b35-60c5f6e08f03" />



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
## Reflection on output

*it was close to working in one prompt but it said bad gateway on the web page so i sent the error hopng that would fix it*
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
## Reflection on output

*after implementing these changes it fixed the bad gateway part but it then switched to backend unavailable*
---
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
## Reflection on output

*after this output the whole application worked and no further changes were needed but while inspecting the files i noticed that my web/Dockerfile used the wrong image so i needed this to be corrected*

## prompt 4

while checking my web/Dockerfile i noticed that its using nginx alpine image i need the ubuntu 24.04 one can u change this
## response 4
```
Yes — that’s the only change you need. Here’s your corrected web/Dockerfile using Ubuntu 24.04 as base, with nginx installed during build:

FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends nginx openssl curl bash \
 && rm -rf /var/lib/apt/lists/*

# Copy nginx config and static site
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

# Entrypoint will create a self-signed cert on first boot if missing
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 443
HEALTHCHECK CMD curl -ks https://localhost/healthz || exit 1
ENTRYPOINT ["/entrypoint.sh"]

Apply the change
sudo docker compose build web
sudo docker compose up -d web


This way your web container now runs on Ubuntu 24.04 and installs nginx during image creation.
```
## Reflection on input

*this was the final ouput i needed to finish this assingment, but while reviewing my Dockferfile, i noticed that my html files are being copied into the image.*

## Prompt 5

while reviewing my Dockferfile, i noticed that my html files are being copied into the image. my files must not be inside containers but must live on the host. update my docker-compose.yml and my Dockerfile so that my web files are mounted from the host instead of baked into the image
## response 5
```
Absolutely — here are the exact edits.

Updated docker-compose.yml (web service only)
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
    container_name: contnginx2-m1-SG
    restart: unless-stopped
    ports:
      - "8085:443"
    depends_on:
      - api
    volumes:
      - ./web/html:/usr/share/nginx/html:ro   # mount site from host (read-only)
    healthcheck:
      test: ["CMD", "curl", "-ks", "https://localhost/healthz"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mongo-data:

Updated web/Dockerfile (remove COPY of html)
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends nginx openssl curl bash \
 && rm -rf /var/lib/apt/lists/*

COPY nginx.conf /etc/nginx/nginx.conf
# html comes from a host bind-mount now:
# COPY html/ /usr/share/nginx/html/

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 443
HEALTHCHECK CMD curl -ks https://localhost/healthz || exit 1
ENTRYPOINT ["/entrypoint.sh"]

Run it
docker compose down
docker compose up -d --build


(If ./web/html doesn’t exist on the host, create it and put your index.html there.)
```
---
## reflection on input

*after this my assignment complied with what was asked of me so no further prompts were needed*
---
## AI Assistance Reflection:
Generative AI was a essential tool used during this assignment. i used it in the following ways:
- code generation and scaffolding: AI helped me to write the files, this gave me the starting baseline instead of starting from scratch
- Debugging: When i encountered errors like 502 bad gateway and backend unavailable. I used AI as my guide to fix these issues


While AI generated solution, I still did manual testing and adapted or rejected things AI gave me or didnt give me for example it didnt update that it used wget in the healthcheck so i replaced it with curl because of the image differences

## Conclusion
AI saved me a lot of time by identifying common mistakes and also clarifying docker concepts. However I still had to apply judgement: testing every suggestion and disgarding wrong ones, and where AI was not right I was able to change it myself because in trying this so many times I started understanding how things worked together. This process improved my understanding of container networking, persitence and encryption rather than just copying whatever AI gave me. By the end of the process, I can explain every line of configuration and why it was needed. That is a sign that AI support helped me to a real understanding.


