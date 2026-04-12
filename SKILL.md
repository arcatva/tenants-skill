---
name: tenants-deploy
description: Deploy a full-stack project (frontend + backend + managed database) to the Tenants PaaS platform via API.
allowed-tools: Bash(curl *) Bash(mkdir *) Bash(rm -rf /tmp/tenants-deploy*) Bash(docker *) Bash(sudo docker *) Bash(sudo chmod *) Bash(ls *) Bash(cp *) Bash(grep *) Bash(head *) mcp__docker__build_image mcp__docker__list_images Read Write
---

# Tenants PaaS Deployment

Deploy a complete project to the Tenants multi-tenant hosting platform via API.

## Arguments

Required:
- `username` — Keycloak username
- `password` — Keycloak password

Optional:
- `base_url` — Platform URL (default: `tenants.zhefuz.link`)
- `project_name` — Resource name (default: ask the user)
- `db_type` — `postgres` | `mysql` | `redis` (default: `postgres`)

## Quick workflow

```
Auth → Create DB → Build image → Upload → Create server → Create site → Bind → Verify
```

### 1. Authenticate

Get a session cookie via OIDC. See [references/auth.md](references/auth.md) for the full curl flow.

All subsequent API calls use: `-b /tmp/tenants-deploy/cookies`

### 2. Create managed database

```bash
curl -s -b /tmp/tenants-deploy/cookies -X POST "https://BASE_URL/api/v1/databases" \
  -H "Content-Type: application/json" -d '{"name":"NAME","dbType":"postgres"}'
```

Save `host`, `port`, `databaseName`, `dbUsername`, `password` from the response — they must be baked into the Docker image (the platform does not inject env vars).

### 3. Build Docker image

Write app files to `/tmp/tenants-deploy/app/` using the **Write** tool. See [references/app-templates.md](references/app-templates.md) for starter templates.

Build with Docker MCP or CLI (see [references/docker.md](references/docker.md)):
- Prefer `mcp__docker__build_image` tool if available
- Otherwise use `sudo docker build` — **always use `sudo`**, the user is typically not in the `docker` group

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
  -H "Content-Type: application/json" -d '{"name":"NAME","dockerImageId":"IMAGE_ID","port":8080}'
```

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
```

```bash
rm -rf /tmp/tenants-deploy
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
- Images must listen on port **8080**
