# App Templates

Starter templates for deploying to Tenants PaaS. Replace `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` with values from the database creation response.

> ⚠️ Every Dockerfile below pins `FROM --platform=linux/amd64`. The Tenants cluster only runs amd64 nodes — do not remove this flag, even when templating for an arm64 host. Also pass `--platform=linux/amd64` to `docker build`. See [docker.md](docker.md).

> 🔒 **Run as non-root.** The platform runs every container with a hardened security profile (no privilege escalation, dropped Linux capabilities, seccomp). Images **must not run as root** — end each Dockerfile with a numeric `USER` (not a name, so Kubernetes can verify it) and make the app's files group-0-owned + group-writable (`chown -R <uid>:0 … && chmod -R g=u …`) so they stay readable/writable whatever UID the platform assigns. Listen on **8080** (a high port) so no privileged-port capability is needed. A root image will be rejected at deploy time.

## Node.js + Express + PostgreSQL

### package.json

```json
{
  "name": "APP_NAME",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.12.0"
  }
}
```

### Dockerfile

```dockerfile
FROM --platform=linux/amd64 node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY . .
# Non-root: node:alpine ships a uid-1000 'node' user. Own /app by group 0 and
# make it group-writable so it works under any assigned UID.
RUN chown -R 1000:0 /app && chmod -R g=u /app
EXPOSE 8080
USER 1000
CMD ["node", "server.js"]
```

### server.js (minimal)

```js
const express = require("express");
const { Pool } = require("pg");
const app = express();
app.use(express.json());

const pool = new Pool({
  host: "DB_HOST",
  port: DB_PORT,
  database: "DB_NAME",
  user: "DB_USER",
  password: "DB_PASS",
});

app.get("/", (req, res) => res.send("Hello from Tenants PaaS!"));
app.listen(8080, () => console.log("Running on 8080"));
```

## Python + Flask + PostgreSQL

### requirements.txt

```
flask==3.0.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

### Dockerfile

```dockerfile
FROM --platform=linux/amd64 python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# Non-root: own /app by group 0 + group-writable (no named user needed; a numeric
# UID with no /etc/passwd entry runs fine for gunicorn).
RUN chown -R 1000:0 /app && chmod -R g=u /app
EXPOSE 8080
USER 1000
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### app.py (minimal)

```python
import os
import psycopg2
from flask import Flask, jsonify

app = Flask(__name__)

def get_db():
    return psycopg2.connect(
        host="DB_HOST", port=DB_PORT,
        dbname="DB_NAME", user="DB_USER", password="DB_PASS"
    )

@app.route("/")
def index():
    return "Hello from Tenants PaaS!"
```

## Go + net/http + PostgreSQL

### Dockerfile

```dockerfile
FROM --platform=linux/amd64 golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server .

FROM --platform=linux/amd64 alpine:3.19
COPY --from=build /app/server /server
EXPOSE 8080
# Static binary needs no writable dirs; just run as a non-root numeric UID.
USER 1000
CMD ["/server"]
```

## Static site (no database)

For apps that don't need a database, skip the database creation step. Serve static
files with **nginx-unprivileged** — it already runs as non-root on port 8080 with
writable cache/temp dirs sorted out (stock `nginx` runs as root on port 80 and will
be rejected by the platform's non-root policy).

### Dockerfile

```dockerfile
FROM --platform=linux/amd64 nginxinc/nginx-unprivileged:stable
# Copy your built static assets (e.g. a Vite/React `dist/`) into the web root.
COPY dist/ /usr/share/nginx/html/
EXPOSE 8080
# Base image already sets USER 101 and listens on 8080 — nothing else to do.
```

> Migrating an existing stock-`nginx` image and you have no source? Rebase it without rebuilding:
> ```dockerfile
> FROM --platform=linux/amd64 nginxinc/nginx-unprivileged:stable
> COPY --from=<your-old-image> /usr/share/nginx/html /usr/share/nginx/html
> ```
> Then redeploy the server on **port 8080**.

## Key constraints

- **Platform**: `linux/amd64` — pin it in `FROM` and pass `--platform=linux/amd64` to `docker build`
- **Non-root**: end with a numeric `USER` (e.g. `USER 1000`); root images are rejected at deploy. See the 🔒 note above.
- **Port**: Always 8080 (a high port — no privileged-port capability needed)
- **DB credentials**: Hardcoded in source (no env var injection)
- **Image tag**: `name:version` format required (e.g. `myapp:1.0.0`)
- **Max size**: 500 MB
