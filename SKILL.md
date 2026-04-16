---
name: tenants
description: Manage resources on the Tenants PaaS platform — deploy full-stack projects or perform CRUD operations on databases, servers, sites, and Docker images via API.
allowed-tools: Bash(curl *) Bash(mkdir *) Bash(rm -rf /tmp/tenants-deploy*) Bash(docker *) Bash(sudo docker *) Bash(sudo chmod *) Bash(ls *) Bash(cp *) Bash(grep *) Bash(head *) Bash(sed *) Bash(tr *) Bash(cut *) mcp__docker__build_image mcp__docker__list_images Read Write
---

# Tenants PaaS

Manage resources on the Tenants multi-tenant hosting platform. Supports full deployment pipelines and CRUD operations on all resource types.

## Arguments

Required:
- `username` — Keycloak username
- `password` — Keycloak password

Optional:
- `base_url` — Platform URL (default: `tenants.zhefuz.link`)

Deployment-specific:
- `project_name` — Resource name (default: ask the user)
- `needs_db` — Whether the project needs a managed database (default: ask the user; skip for static sites)
- `db_type` — `postgres` | `mysql` | `redis` (default: `postgres`; only relevant if `needs_db` is true)

## Determine intent

Understand what the user wants before acting:
- **Deploy** → full pipeline (see [Deploy workflow](#deploy-workflow))
- **List** → list all resources of a given type
- **Get** → get details of a specific resource
- **Delete** → delete a resource (confirm with user first)
- **Bind/Unbind** → attach or detach a server from a site

All operations require authentication first.

## Authentication

Get a session cookie via OIDC. See [references/auth.md](references/auth.md) for the full curl flow.

All subsequent API calls use: `-b /tmp/tenants-deploy/cookies`

## CRUD Operations

### List resources

```bash
# Databases
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/databases"

# Docker images
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/docker-images"

# Servers
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/servers"

# Sites
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/sites"
```

All list responses return `{"success": true, "data": [...]}` with an array of resource objects.

### Get resource

```bash
# Database by name
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/databases/NAME"

# Docker image by ID
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/docker-images/IMAGE_ID"

# Server by name
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/servers/NAME"

# Site by name
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/sites/NAME"
```

### Download Docker image

Download a previously uploaded image as a tar file (suitable for `docker load`). Only available for images with `running` status.

```bash
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/docker-images/IMAGE_ID/download" \
  -o downloaded-image.tar
```

Response is a binary `application/x-tar` stream, not JSON. The suggested filename is `{imageName}-{version}.tar`.

### Delete resource

> ⚠️ Always confirm with the user before deleting. Deletions are irreversible.

```bash
# Database by name
curl -s -b /tmp/tenants-deploy/cookies -X DELETE "https://BASE_URL/api/v1/databases/NAME"

# Docker image by ID
curl -s -b /tmp/tenants-deploy/cookies -X DELETE "https://BASE_URL/api/v1/docker-images/IMAGE_ID"

# Server by name
curl -s -b /tmp/tenants-deploy/cookies -X DELETE "https://BASE_URL/api/v1/servers/NAME"

# Site by name
curl -s -b /tmp/tenants-deploy/cookies -X DELETE "https://BASE_URL/api/v1/sites/NAME"
```

### Site: bind / unbind server

```bash
# Bind a server to a site
curl -s -b /tmp/tenants-deploy/cookies -X PUT "https://BASE_URL/api/v1/sites/SITE_NAME/server" \
  -H "Content-Type: application/json" -d '{"serverName":"SERVER_NAME"}'

# Unbind the server from a site
curl -s -b /tmp/tenants-deploy/cookies -X DELETE "https://BASE_URL/api/v1/sites/SITE_NAME/server"
```

## Create resources (standalone)

To create a resource without a full deploy:

```bash
# Create database
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/databases" \
  -H "Content-Type: application/json" -d '{"name":"NAME","dbType":"DB_TYPE"}'

# Create server (requires an uploaded docker image ID)
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/servers" \
  -H "Content-Type: application/json" -d '{"name":"NAME","dockerImageId":"IMAGE_ID"}'

# Create site
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/sites" \
  -H "Content-Type: application/json" -d '{"name":"NAME"}'
```

For uploading a Docker image, see step 4 of the deploy workflow below.

## Deploy workflow

First, determine whether the project needs a database:
- **With database** (backends, full-stack apps): `Auth → Create DB → Build image → Upload → Create server → Create site → Bind → Verify`
- **Without database** (static sites, no-DB backends): `Auth → Build image → Upload → Create server → Create site → Bind → Verify`

If unclear, ask the user whether their project requires a database before proceeding.

### 1. Authenticate

Get a session cookie via OIDC. See [references/auth.md](references/auth.md).

All subsequent API calls use: `-b /tmp/tenants-deploy/cookies`

### 2. Create managed database _(optional)_

> Skip this step if the project does not need a database (e.g. static sites, purely front-end apps).

```bash
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/databases" \
  -H "Content-Type: application/json" -d '{"name":"NAME","dbType":"DB_TYPE"}'
```

Save `host`, `port`, `databaseName`, `dbUsername`, `password` from the response — they must be baked into the Docker image (the platform does not inject env vars).

**Poll until ready:** After creating the database, poll `GET /databases/NAME` until `status` is `ready`. If `status` becomes `failed`, stop and report the error. Do not proceed to the next step until the database is `ready`.

### 3. Build Docker image

Write app files to `/tmp/tenants-deploy/app/` using the **Write** tool. See [references/app-templates.md](references/app-templates.md) for starter templates.

> ⚠️ **The cluster runs linux/amd64.** Always build with `--platform=linux/amd64` and pin base images to amd64 in the Dockerfile (`FROM --platform=linux/amd64 …`). Building on an ARM host (Apple Silicon, etc.) without this flag produces an arm64 image that fails with `exec format error` on the cluster. See [references/docker.md](references/docker.md).

Build with the Docker CLI (preferred — guarantees the platform flag is honored):

```bash
sudo docker build --platform=linux/amd64 -t APP_NAME:VERSION /tmp/tenants-deploy/app/
```

**Always use `sudo`** — the user is typically not in the `docker` group.

Then export as tar for upload:
```bash
sudo docker save APP_NAME:VERSION -o /tmp/tenants-deploy/image.tar
sudo chmod 644 /tmp/tenants-deploy/image.tar
```

### 4. Upload image

```bash
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/docker-images" \
  -F "file=@/tmp/tenants-deploy/image.tar"
```

Save the `id` from the response.

### 5. Create server

```bash
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/servers" \
  -H "Content-Type: application/json" -d '{"name":"NAME","dockerImageId":"IMAGE_ID"}'
```

Port is auto-detected from the Docker image `EXPOSE` directive (default: 8080).

**Poll until ready:** After creating the server, poll `GET /servers/NAME` until `status` is `ready`. If `status` becomes `failed`, check logs with `GET /servers/NAME/logs` and report. Do not proceed to bind until the server is `ready`.

### 6. Create site & bind

```bash
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/sites" \
  -H "Content-Type: application/json" -d '{"name":"NAME"}'
```

```bash
curl -s -b /tmp/tenants-deploy/cookies -X PUT "https://BASE_URL/api/v1/sites/NAME/server" \
  -H "Content-Type: application/json" -d '{"serverName":"NAME"}'
```

Live at: `https://NAME-USERNAME.zhefuz.link/`

### 7. Verify & cleanup

```bash
curl -s "https://NAME-USERNAME.zhefuz.link/"
rm -rf /tmp/tenants-deploy
```

## Response format

Success:

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

Error:

```json
{
  "success": false,
  "data": null,
  "error": "message"
}
```

## References

* **OIDC authentication flow** [references/auth.md](references/auth.md)
* **API endpoint reference** [references/api.md](references/api.md)
* **Example app templates** [references/app-templates.md](references/app-templates.md)
* **Docker image building** [references/docker.md](references/docker.md)

## Notes

- Site URL pattern: `https://{site}-{username}.zhefuz.link/`
- Container resources: 128Mi/100m requests, 512Mi/500m limits
- Limits: 5 sites, 3 databases per user
- Port auto-detected from Docker image `EXPOSE` directive (default: 8080)
- Resource names must follow RFC 1123: start with a letter, lowercase letters/digits/hyphens only, max 63 chars
- Always confirm with user before deleting resources
