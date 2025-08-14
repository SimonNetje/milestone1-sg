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
I need to create a Docker Compose setup with three services: A web frontend (served by Nginx on Ubuntu) it should say "*name retrieved from database* has reached milestone1!!!" make the website simple no buttons only the message stated above and it should remain persistent through restarts and i should be able to change the name and after a refresh it to appear on the fontend site i also want for the container id to be below my name A Flask API that talks to MongoDB A MongoDB database with persistent storage and seeded data I also want to set specific container names: contmongo-m1-SG for MongoDB contnginx-m1-SG for nginx contapi-m1-SG for the API Can you outline the directory structure and main config files I’ll need so I can start building it?
## response 1
```
Milestone1 Docker Stack (Nginx on Ubuntu + Flask API + MongoDB)

This gives you a minimal, button‑free frontend that shows:

<name from MongoDB> has reached milestone1!!!
<nginx container id>

Name persists in Mongo (volume). You can change it via the API (e.g., with curl) and a browser refresh will show the update.

1) Directory layout

milestone1/
├─ docker-compose.yml
├─ api/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  └─ app.py
├─ nginx/
│  ├─ Dockerfile
│  ├─ nginx.conf
│  └─ html/
│     └─ index.html
└─ mongo/
   └─ init/
      └─ seed.js

Container names (as requested):

MongoDB: contmongo-m1-SG

Nginx: contnginx-m1-SG

API: contapi-m1-SG

2) docker-compose.yml

version: "3.9"

services:
  mongo:
    image: mongo:7
    container_name: contmongo-m1-SG
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpass
      MONGO_INITDB_DATABASE: milestone
    volumes:
      - mongodata:/data/db
      - ./mongo/init:/docker-entrypoint-initdb.d:ro
    networks:
      - m1net

  api:
    build: ./api
    container_name: contapi-m1-SG
    restart: unless-stopped
    environment:
      MONGO_URL: mongodb://root:rootpass@mongo:27017/milestone?authSource=admin
    depends_on:
      - mongo
    networks:
      - m1net

  nginx:
    build: ./nginx
    container_name: contnginx-m1-SG
    restart: unless-stopped
    ports:
      - "8085:80"
    depends_on:
      - api
    networks:
      - m1net

volumes:
  mongodata:

networks:
  m1net:
    driver: bridge

3) API service

api/Dockerfile

FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]

api/requirements.txt

flask==3.0.3
pymongo==4.8.0

api/app.py

from flask import Flask, jsonify, request
from pymongo import MongoClient
import os

app = Flask(__name__)

MONGO_URL = os.environ.get("MONGO_URL", "mongodb://root:rootpass@mongo:27017/milestone?authSource=admin")
client = MongoClient(MONGO_URL)
db = client.get_database()

COLL = db["status"]
DOC_ID = "singleton"  # single-document collection for simplicity

# Ensure doc exists
COLL.update_one({"_id": DOC_ID}, {"$setOnInsert": {"name": "Simon Gielen"}}, upsert=True)

@app.get("/api/name")
def get_name():
    doc = COLL.find_one({"_id": DOC_ID}) or {"name": "(unset)"}
    return jsonify({"name": doc.get("name", "(unset)")})

@app.post("/api/name")
def set_name():
    data = request.get_json(silent=True) or {}
    name = data.get("name", "").strip()
    if not name:
        return jsonify({"error": "Provide non-empty 'name'"}), 400
    COLL.update_one({"_id": DOC_ID}, {"$set": {"name": name}}, upsert=True)
    return jsonify({"ok": True, "name": name})

# Simple health endpoint
@app.get("/healthz")
def health():
    return {"ok": True}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

4) Mongo seed

mongo/init/seed.js

// Runs automatically on first database init
const dbName = 'milestone'
const coll = 'status'

// Create the collection and seed one document if missing
const docId = 'singleton'
db = db.getSiblingDB(dbName)

if (!db[coll].findOne({_id: docId})) {
  db[coll].insertOne({_id: docId, name: 'Simon Gielen'})
}

This executes once on first startup (new volume). Data persists in mongodata.

5) Nginx service (Ubuntu base)

nginx/Dockerfile

FROM ubuntu:24.04

# Install nginx and tidy up
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends nginx ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

# Remove default site and add ours
RUN rm -f /etc/nginx/sites-enabled/default
COPY nginx.conf /etc/nginx/nginx.conf

# Static files
COPY html/ /usr/share/nginx/html/

EXPOSE 80
STOPSIGNAL SIGQUIT
CMD ["nginx", "-g", "daemon off;"]

nginx/nginx.conf

user  www-data;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events { worker_connections 1024; }

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;
  sendfile      on;
  keepalive_timeout  65;
  server_tokens off;

  # gzip etc. omitted

  server {
    listen 80;

    # Make $hostname available to SSI via a simpler variable name
    set $h $hostname;

    # Serve the SPA-like static page with SSI enabled
    location / {
      root   /usr/share/nginx/html;
      index  index.html;
      ssi on;                  # enable Server Side Includes
      ssi_types text/html;     # process only HTML
    }

    # Reverse proxy to the API for CORS simplicity (optional)
    location /api/ {
      proxy_pass http://api:5000/api/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_http_version 1.1;
    }

    location /healthz { return 200 'ok\n'; add_header Content-Type text/plain; }
  }
}

nginx/html/index.html

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Milestone 1</title>
  <style>
    html,body{height:100%;margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Inter,Arial}
    .wrap{min-height:100%;display:flex;align-items:center;justify-content:center}
    .card{padding:32px 40px;border:1px solid #ddd;border-radius:16px;box-shadow:0 6px 30px rgba(0,0,0,.06);text-align:center}
    h1{font-size:28px;margin:0 0 8px}
    p{margin:0;color:#666}
    .cid{margin-top:12px;font-family:ui-monospace,Consolas,monospace;font-size:14px;color:#444}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1 id="msg">Loading…</h1>
      <p class="cid">container: <!--# echo var="h" --></p>
    </div>
  </div>
  <script>
    async function load() {
      try {
        const r = await fetch('/api/name');
        if (!r.ok) throw new Error('bad status '+r.status);
        const data = await r.json();
        const name = (data && data.name) ? data.name : '(unset)';
        document.getElementById('msg').textContent = `${name} has reached milestone1!!!`;
      } catch (e) {
        document.getElementById('msg').textContent = 'API unavailable';
        console.error(e);
      }
    }
    load();
  </script>
</body>
</html>

6) Usage

From the milestone1/ directory:

# 1) Build and start
docker compose up -d --build

# 2) Verify
curl http://127.0.0.1:8085/healthz
curl http://127.0.0.1:8085/api/name

# 3) Open in browser (served on host port 8085)
# http://localhost:8085

# 4) Change the name while containers are running
curl -s -X POST http://localhost:8085/api/name \
  -H 'Content-Type: application/json' \
  -d '{"name":"Simon Gielen"}' | jq

# Refresh the page to see the update.

Persistence: MongoDB data is stored in the named volume mongodata. Stopping/starting won’t lose the name.

Container ID rendering: The line under the message shows Nginx’s container hostname (by default the Docker container’s short ID). We use Nginx SSI to echo it into the HTML at request time, so it’s always current without JS tricks.

7) Common pitfalls (so you don’t waste time)

Browser caching: Hard refresh if you don’t see updates (Ctrl+F5).

Port already in use: If 0.0.0.0:8085 is occupied, either stop the other process or change the compose mapping to e.g. 9090:80.

CORS noise: We proxy /api/ via Nginx, so no CORS preflight issues.

Seed runs once: If you want to re-seed, delete the mongodata volume: docker volume rm milestone1_mongodata and docker compose up -d again.

8) Minimal test matrix



You can copy this entire tree into a fresh folder and run it as-is. Adjust credentials as needed.

```
## Reflection on prompt
This first prompt made it work first try so no further tweaking was needed now i am adding load balancing
---

## AI Assistance Reflection:
  I used generative AI to:
  Design the stack layout (web → API → DB).
  Write docker-compose.yml and config files.
  Debug proxy path issues (/api/name mapping).
  Add scaling & TLS support.
  Document steps and explain commands.
  This allowed me to focus on understanding the architecture rather than memorizing syntax.


