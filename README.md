# Docker Images

Container images for personal use, published to GitHub Container Registry.

## Images

### `devcontainer`

`ghcr.io/fx/docker/devcontainer:latest`

A generic development container. Use it as a `.devcontainer` base for any project.

- Ubuntu 24.04 (`mcr.microsoft.com/devcontainers/base`)
- Docker CE with Buildx and Compose (docker-in-docker; start the daemon yourself)
- [mise](https://mise.jdx.dev/) with Node.js LTS
- GitHub CLI, git, tmux, jq, Python 3 and the usual shell utilities

### `coder`

`ghcr.io/fx/docker/coder:latest`

Built **from** the `devcontainer` image, for use as a headless [Coder](https://coder.com/) workspace image. It bakes in everything a workspace would otherwise download on every start, so boot does auth and daemons only:

- **Tailscale** — client and daemon installed; the workspace still runs `tailscale up` with its own auth key
- **code-server** — at `/usr/local/bin/code-server`
- **Tooling via mise** — Bun, chezmoi, `gh`, `kubectl`, plus the Claude Code and Codex CLIs
- **CodeRabbit CLI** — `coderabbit` / `cr`
- mise shims on `PATH` for non-interactive shells, telemetry opt-outs, and a UTF-8 locale

Version-volatile tools are installed through mise rather than pinned, so a workspace can pull a newer release with `mise upgrade` without waiting for an image rebuild.

### `coder-desktop`

`ghcr.io/fx/docker/coder-desktop:latest`

Built **from** the `coder` image, adding a graphical desktop: Xvfb, x11vnc, noVNC, Fluxbox, PulseAudio, and both Mesa software rendering and the bare GL loaders used with a passed-through GPU.

It is a separate image rather than part of `coder` because most workspaces are headless and the stack is ~650 MB. The workspace template picks between the two from its desktop option. There is no GPU variant — `nvidia-utils` has to match the host's driver version, so it stays a runtime install.

## Usage

```bash
docker pull ghcr.io/fx/docker/devcontainer:latest
docker pull ghcr.io/fx/docker/coder:latest
docker pull ghcr.io/fx/docker/coder-desktop:latest
```

## Building

The images form a chain — `devcontainer` → `coder` → `coder-desktop` — each built from the exact digest the previous job just pushed. All are built for `linux/amd64` and `linux/arm64` by [`.github/workflows/build-images.yml`](.github/workflows/build-images.yml) on push to `main`, on pull requests touching a Dockerfile, weekly, and on manual dispatch.

Pull requests **from a fork** are not built at all: publishing needs a `packages: write` token, which GitHub withholds from fork PRs, so the jobs are skipped rather than left to fail on the push. A fork PR therefore produces no images and no `pr-N` tags — expect the build to run once a maintainer lands or re-pushes the branch in this repository.

Locally:

```bash
docker build -f devcontainer/Dockerfile -t fx-devcontainer:test .
docker build -f coder/Dockerfile --build-arg BASE_IMAGE=fx-devcontainer:test -t fx-coder:test .
docker build -f coder-desktop/Dockerfile --build-arg BASE_IMAGE=fx-coder:test -t fx-coder-desktop:test .
```

See [AGENTS.md](AGENTS.md) for conventions and what to verify before pushing.

## License

This repository is for personal use.
