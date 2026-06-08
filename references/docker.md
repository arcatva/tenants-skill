# Docker Image Building

Images are built via the `docker` CLI and exported via `docker save`.

## Prerequisites

Docker daemon must be running (`sudo systemctl start docker`).

## Architecture: must be linux/amd64

The Tenants cluster runs on **linux/amd64** nodes. Images built on ARM hosts (Apple Silicon Macs, ARM Linux, etc.) without an explicit platform flag will be arm64, and the container will crash-loop on the cluster with `exec format error`.

**Two things must be in place:**

1. **Pin the base image to amd64** in the Dockerfile:

   ```dockerfile
   FROM --platform=linux/amd64 node:20-alpine
   ```

2. **Pass `--platform=linux/amd64` to `docker build`**, even when the Dockerfile already pins the platform (defense in depth, and needed for multi-stage builds where intermediate layers might otherwise mismatch):

   ```bash
   sudo docker build --platform=linux/amd64 -t APP_NAME:VERSION /tmp/tenants-deploy/app/
   ```

On an amd64 host this is a no-op. On ARM it forces emulation via QEMU (slower, but produces a working image). If the host lacks `binfmt_misc` QEMU support, install it:

```bash
sudo docker run --privileged --rm tonistiigi/binfmt --install amd64
```

## Build with CLI

**Always try `sudo` first** — the current user is likely not in the `docker` group:

```bash
sudo docker build --platform=linux/amd64 -t APP_NAME:VERSION /tmp/tenants-deploy/app/
```

## Build with MCP (optional)

If `mcp__docker__build_image` is available it can be used, but only if the tool exposes a platform parameter and the Dockerfile pins `FROM --platform=linux/amd64`. When in doubt, prefer the CLI path above — the explicit flag is the only reliable guarantee.

## Export for upload

The platform accepts `.tar` files. After building, export and fix permissions (since `sudo` creates root-owned files):

```bash
sudo docker save APP_NAME:VERSION -o /tmp/tenants-deploy/image.tar
sudo chmod 644 /tmp/tenants-deploy/image.tar
```

## Image requirements

- **Platform must be `linux/amd64`** (see section above)
- **Must run as non-root** — the platform applies a hardened security profile (no privilege escalation, all Linux capabilities dropped, seccomp `RuntimeDefault`). End the Dockerfile with a numeric `USER` (e.g. `USER 1000`) and listen on a high port (8080). A root image is rejected at deploy. See the 🔒 note and per-language templates in [app-templates.md](app-templates.md).
- Use `EXPOSE` in Dockerfile to declare the listening port (default: 8080 if omitted)
- Database credentials must be hardcoded (no env var injection at runtime)
- Tag format `name:version` — auto-extracted from tar's `manifest.json` `RepoTags`
- Max upload size: 500 MB

## Verify the image is non-root

Before uploading, confirm the image does not run as root:

```bash
sudo docker inspect APP_NAME:VERSION --format 'USER=[{{.Config.User}}]'
# expected: a non-empty, non-zero numeric UID, e.g. USER=[1000]
# USER=[] or USER=[root] or USER=[0] → will be rejected; add `USER 1000` to the Dockerfile
```

## Import from ghcr.io

Instead of building and uploading a tar, you can import a pre-built image directly from GitHub Container Registry into your image library.

```bash
# Public image — no credentials needed
curl -s -H "Authorization: Bearer $TOKEN" -X POST "https://BASE_URL/api/v1/docker-images/import" \
  -H "Content-Type: application/json" \
  -d '{"sourceRef":"ghcr.io/owner/app:tag"}'
```

For a **private** image, include a GitHub Personal Access Token with the `read:packages` scope:

```bash
curl -s -H "Authorization: Bearer $TOKEN" -X POST "https://BASE_URL/api/v1/docker-images/import" \
  -H "Content-Type: application/json" \
  -d '{"sourceRef":"ghcr.io/owner/private-app:tag","username":"GITHUB_USERNAME","token":"GITHUB_PAT"}'
```

Key constraints:
- **Only `ghcr.io` is permitted** as a source in v1. Any other registry host returns 400.
- For multi-arch images, the `linux/amd64` variant is selected automatically — no `--platform` flag needed.
- Size limit is the same as upload (500 MB).
- The GitHub PAT is used only for the one-time pull and is **never stored** on the platform.
- The call is synchronous; on success it returns the same image object (`DockerImageDto`) as upload, with `status: running`.
- Save the returned `id` to create a server, exactly as you would after an upload.

## Download from platform

Previously uploaded images can be downloaded as tar files from the platform:

```bash
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/api/v1/docker-images/IMAGE_ID/download" \
  -o image.tar
```

The downloaded tar is compatible with `docker load`:

```bash
sudo docker load -i image.tar
```

Only images with `running` status can be downloaded.

## Verify the built image's architecture

Before uploading, confirm the image is amd64:

```bash
sudo docker inspect APP_NAME:VERSION --format '{{.Architecture}}/{{.Os}}'
# expected: amd64/linux
```

If this reports `arm64/linux`, rebuild with `--platform=linux/amd64` — uploading as-is will produce a crash-looping pod.

## Verify build

Use `mcp__docker__list_images` to confirm the image was built successfully before exporting.
