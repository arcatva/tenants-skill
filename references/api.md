# API Reference

All endpoints require cookie authentication. Base path: `https://BASE_URL/api/v1`.

All resource names must follow RFC 1123 and be **≤ 20 characters**: start with a lowercase letter, contain only lowercase letters, digits, and hyphens. The platform appends `-<username>-<5-char-suffix>` when creating the underlying K8s resource, so the 20-char cap keeps the composed name under the 63-char DNS label limit.

## Authentication

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/login` | GET | Redirect to Keycloak |
| `/auth/logout` | GET | Logout |
| `/auth/me` | GET | Current user info |

## Databases

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/managed-databases` | POST | `{"name","dbType"}` | Create database |
| `/managed-databases` | GET | — | List user's databases |
| `/managed-databases/{name}` | GET | — | Get database details |
| `/managed-databases/{name}/logs` | GET | — | Get pod logs |
| `/managed-databases/{name}/metrics` | GET | `?range=1h` (also `6h`, `24h`) | CPU / memory / network / restart time-series |
| `/managed-databases/{name}/upgrade` | POST | — | Upgrade to latest major version |
| `/managed-databases/{name}` | DELETE | — | Delete database |

`dbType` values: `postgres`, `mysql`, `redis`. Limit: 3 per user.

Response includes: `host`, `port`, `databaseName`, `dbUsername`, `password`, `majorVersion`, `latestVersion`.

**Upgrade**: `POST /managed-databases/{name}/upgrade` triggers an async upgrade to the latest major version (Postgres 17, MySQL 8.4, Redis 8). For Postgres/MySQL the platform runs an automated dump→recreate→restore (expect 1-5 min downtime). Host/port stay stable; existing data is preserved. Poll status until `ready`.

## Docker Images

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/docker-images` | POST | `multipart: file` | Upload image tar |
| `/docker-images/import` | POST | `{"sourceRef","username","token"}` | Import image from ghcr.io |
| `/docker-images` | GET | — | List images |
| `/docker-images/{id}` | GET | — | Get image details |
| `/docker-images/{id}` | DELETE | — | Delete image |
| `/docker-images/{id}/download` | GET | — | Download image as tar |

Formats: `.tar`, `.tar.gz`, `.tgz`. Max: 500 MB. Name & version auto-extracted from manifest.
Download returns a binary `application/x-tar` file (not JSON). Only available for images with `running` status.

**Import from ghcr.io:** `sourceRef` is required (e.g. `ghcr.io/owner/image:tag`). Only `ghcr.io` sources are permitted in v1 — other registries return 400. `username` and `token` are optional and only needed for private images (GitHub PAT with `read:packages` scope); the token is used for a single pull and is never stored. For multi-arch images the `linux/amd64` variant is selected automatically. Size limit is the same as upload (500 MB). On success returns the same `DockerImageDto` object as upload, with `status: running`.

## Servers

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/managed-servers` | POST | `{"name","dockerImageId"}` | Create server |
| `/managed-servers` | GET | — | List servers |
| `/managed-servers/{name}` | GET | — | Get server |
| `/managed-servers/{name}/logs` | GET | — | Get pod logs |
| `/managed-servers/{name}/metrics` | GET | `?range=1h` (also `6h`, `24h`) | CPU / memory / network / restart time-series |
| `/managed-servers/{name}` | DELETE | — | Delete server |

Port auto-detected from image `EXPOSE` (default: 8080). Resources: 128Mi/100m → 512Mi/500m.

Statuses: `pending` → `running` | `failed` (mirror K8s pod phases). Databases also have `upgrading` during major version upgrades. Use `/logs` endpoint to debug failures.

## Sites

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/sites` | POST | `{"name"}` | Create site |
| `/sites` | GET | — | List sites |
| `/sites/{name}` | GET | — | Get site |
| `/sites/{name}` | DELETE | — | Delete site |
| `/sites/{name}/managed-server` | PUT | `{"managedServerName"}` | Bind server |
| `/sites/{name}/managed-server` | DELETE | — | Unbind server |

Limit: 5 per user. URL pattern: `https://{site}-{user}.zhefuz.link/`
