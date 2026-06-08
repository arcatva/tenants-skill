# App Templates

Starter templates for deploying to Tenants PaaS. Replace `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` with values from the database creation response.

> ⚠️ Every Dockerfile below pins `FROM --platform=linux/amd64`. The Tenants cluster only runs amd64 nodes — do not remove this flag, even when templating for an arm64 host. Also pass `--platform=linux/amd64` to `docker build`. See [docker.md](docker.md).

> 🔒 **Run as non-root.** The platform runs every container as **UID 65532** (it sets `runAsUser: 65532` + `runAsGroup: 0`, overriding the image's own `USER`), with a hardened profile (no privilege escalation, dropped Linux capabilities, seccomp). So: end each Dockerfile with `USER 65532` (numeric, matches the platform) and make the app's files owned by `65532:0` + group-writable (`chown -R 65532:0 … && chmod -R g=u …`) so they're writable under that UID (the group-0 ownership also keeps it working if the platform ever assigns a different UID). Listen on **8080** (a high port) so no privileged-port capability is needed. A root image is rejected at deploy time.

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
# Non-root: own /app by 65532:0 + group-writable (matches the platform's forced
# UID 65532 / GID 0; works under any assigned UID thanks to the group-0 ownership).
RUN chown -R 65532:0 /app && chmod -R g=u /app
EXPOSE 8080
USER 65532
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
# Non-root: own /app by 65532:0 + group-writable (no named user needed; a numeric
# UID with no /etc/passwd entry runs fine for gunicorn).
RUN chown -R 65532:0 /app && chmod -R g=u /app
EXPOSE 8080
USER 65532
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
# Static binary needs no writable dirs; just run as the platform's non-root UID.
USER 65532
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
# nginx-unprivileged ships USER 101 + listens on 8080 with group-0-writable dirs,
# so it runs fine under the platform's forced UID 65532 — nothing else to do.
```

> Migrating an existing stock-`nginx` image and you have no source? Rebase it without rebuilding:
> ```dockerfile
> FROM --platform=linux/amd64 nginxinc/nginx-unprivileged:stable
> COPY --from=<your-old-image> /usr/share/nginx/html /usr/share/nginx/html
> ```
> Then redeploy the server on **port 8080**.

## Key constraints

- **Platform**: `linux/amd64` — pin it in `FROM` and pass `--platform=linux/amd64` to `docker build`
- **Non-root**: end with `USER 65532` (the platform runs containers as that UID); root images are rejected at deploy. See the 🔒 note above.
- **Port**: Always 8080 (a high port — no privileged-port capability needed)
- **DB credentials**: Hardcoded in source (no env var injection)
- **Image tag**: `name:version` format required (e.g. `myapp:1.0.0`)
- **Max size**: 500 MB
