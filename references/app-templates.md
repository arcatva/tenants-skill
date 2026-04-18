# App Templates

Starter templates for deploying to Tenants PaaS. Replace `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` with values from the database creation response.

> ⚠️ Every Dockerfile below pins `FROM --platform=linux/amd64`. The Tenants cluster only runs amd64 nodes — do not remove this flag, even when templating for an arm64 host. Also pass `--platform=linux/amd64` to `docker build`. See [docker.md](docker.md).

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
EXPOSE 8080
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
EXPOSE 8080
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
CMD ["/server"]
```

## Static site (no database)

For apps that don't need a database, skip the database creation step.

### Dockerfile

```dockerfile
FROM --platform=linux/amd64 node:20-alpine
WORKDIR /app
COPY . .
RUN npm install --production
EXPOSE 8080
CMD ["node", "server.js"]
```

## Key constraints

- **Platform**: `linux/amd64` — pin it in `FROM` and pass `--platform=linux/amd64` to `docker build`
- **Port**: Always 8080
- **DB credentials**: Hardcoded in source (no env var injection)
- **Image tag**: `name:version` format required (e.g. `myapp:1.0.0`)
- **Max size**: 500 MB
