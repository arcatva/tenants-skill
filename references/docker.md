# Docker Image Building

Images are built via the **Docker MCP** server (`mcp-server-docker`) and exported via `docker save`.

## Prerequisites

Docker MCP must be configured in the project's `.mcp.json`:

```json
{
  "mcpServers": {
    "docker": {
      "command": "uvx",
      "args": ["mcp-server-docker"]
    }
  }
}
```

Docker daemon must be running (`sudo systemctl start docker`).

## Build with MCP

Use the `mcp__docker__build_image` tool if available:
- `path`: build context directory (e.g. `/tmp/tenants-deploy/app`)
- `tag`: image tag (e.g. `my-app:1.0.0`)
- `dockerfile`: path to Dockerfile (usually within the build context)

## Build with CLI (fallback)

If Docker MCP is not available, use the CLI directly. **Always try `sudo` first** — the current user is likely not in the `docker` group:

```bash
sudo docker build -t APP_NAME:VERSION /tmp/tenants-deploy/app/
```

## Export for upload

The platform accepts `.tar` files. After building, export and fix permissions (since `sudo` creates root-owned files):

```bash
sudo docker save APP_NAME:VERSION -o /tmp/tenants-deploy/image.tar
sudo chmod 644 /tmp/tenants-deploy/image.tar
```

## Image requirements

- Must listen on port **8080** (platform default)
- Database credentials must be hardcoded (no env var injection at runtime)
- Tag format `name:version` — auto-extracted from tar's `manifest.json` `RepoTags`
- Max upload size: 5 GB

## Verify build

Use `mcp__docker__list_images` to confirm the image was built successfully before exporting.
