# Docker Images Repository

This repository contains Docker images for personal use.

## Images

### DevContainer

A development container based on Ubuntu 22.04 with common development tools.

**Image:** `ghcr.io/fx/docker/devcontainer:latest`

**Features:**
- Base: Ubuntu 22.04
- Docker-in-Docker support
- mise (formerly rtx) for runtime management
- Node.js LTS (via mise)
- GitHub CLI
- Common development utilities

**Usage:**

```bash
docker pull ghcr.io/fx/docker/devcontainer:latest
```

## Building Images

Images are automatically built and published to GitHub Container Registry when changes are pushed to the main branch.

## License

This repository is for personal use.