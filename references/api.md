# API Reference

All endpoints require cookie authentication. Base path: `https://BASE_URL/api/v1`.

All resource names must follow RFC 1123: start with a lowercase letter, contain only lowercase letters, digits, and hyphens, max 63 characters.

## Authentication

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/login` | GET | Redirect to Keycloak |
| `/auth/logout` | GET | Logout |
| `/auth/me` | GET | Current user info |

## Databases

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/databases` | POST | `{"name","dbType"}` | Create database |
| `/databases` | GET | — | List user's databases |
| `/databases/{name}` | GET | — | Get database details |
| `/databases/{name}/logs` | GET | — | Get pod logs |
| `/databases/{name}/upgrade` | POST | — | Upgrade to latest major version |
| `/databases/{name}` | DELETE | — | Delete database |

`dbType` values: `postgres`, `mysql`, `redis`. Limit: 3 per user.

Response includes: `host`, `port`, `databaseName`, `dbUsername`, `password`, `majorVersion`, `latestVersion`.

**Upgrade**: `POST /databases/{name}/upgrade` triggers an async upgrade to the latest major version (Postgres 17, MySQL 8.4, Redis 8). For Postgres/MySQL the platform runs an automated dump→recreate→restore (expect 1-5 min downtime). Host/port stay stable; existing data is preserved. Poll status until `ready`.

## Docker Images

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/docker-images` | POST | `multipart: file` | Upload image tar |
| `/docker-images` | GET | — | List images |
| `/docker-images/{id}` | GET | — | Get image details |
| `/docker-images/{id}` | DELETE | — | Delete image |

Formats: `.tar`, `.tar.gz`, `.tgz`. Max: 5 GB. Name & version auto-extracted from manifest.

## Servers

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/servers` | POST | `{"name","dockerImageId"}` | Create server |
| `/servers` | GET | — | List servers |
| `/servers/{name}` | GET | — | Get server |
| `/servers/{name}/logs` | GET | — | Get pod logs |
| `/servers/{name}` | DELETE | — | Delete server |

Port auto-detected from image `EXPOSE` (default: 8080). Resources: 128Mi/100m → 512Mi/500m.

Statuses: `pending` → `provisioning` → `running` | `failed` (mirror K8s pod phases). Databases also have `upgrading` during major version upgrades. Use `/logs` endpoint to debug failures.

## Sites

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/sites` | POST | `{"name"}` | Create site |
| `/sites` | GET | — | List sites |
| `/sites/{name}` | GET | — | Get site |
| `/sites/{name}` | DELETE | — | Delete site |
| `/sites/{name}/server` | PUT | `{"serverName"}` | Bind server |
| `/sites/{name}/server` | DELETE | — | Unbind server |

Limit: 5 per user. URL pattern: `https://{site}-{user}.zhefuz.link/`
