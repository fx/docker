# AGENTS.md

Conventions for working in `fx/docker`.

## What this repository is

A **public** repository holding the Dockerfiles for personally-maintained container images, published to GitHub Container Registry by GitHub Actions. There is no application code, no test suite, and no package manager — the Dockerfiles and the workflow that builds them are the whole product.

```
devcontainer/Dockerfile     # base image: Ubuntu 24.04 + Docker CE + mise + Node LTS
coder/Dockerfile            # headless Coder workspace, FROM devcontainer
coder-desktop/Dockerfile    # + X11/VNC desktop stack, FROM coder
.github/workflows/build-image.yml    # reusable single-image build
.github/workflows/build-images.yml   # wires the chain together
```

### The image chain

Each image is built **from** the one above it, as chained jobs in one workflow:

```
devcontainer                  generic dev base
  └── coder                   headless Coder workspace
        └── coder-desktop     + X11/VNC desktop stack
```

Every link consumes the **digest** the previous job just pushed, via the `BASE_IMAGE` build arg. Never hard-code `:latest` as a base in a Dockerfile's `FROM` — the `ARG BASE_IMAGE=…:latest` default exists for local builds only. If CI resolved a base by tag, a PR touching two Dockerfiles would build the lower one against whatever `main` last published, and the change could never be reviewed as a whole.

All three are built for `linux/amd64` **and** `linux/arm64`. Anything you add must work on both, or be explicitly tolerated when it doesn't (see the CodeRabbit CLI install in `coder/Dockerfile` for the pattern).

Published tags: `ghcr.io/fx/docker/{devcontainer,coder,coder-desktop}:latest`.

### Which layer does a change belong in?

- **`devcontainer/`** — generic. It must stay useful as a plain `.devcontainer` base for any repo. Nothing about Coder, Tailscale, VNC, or agent CLIs belongs here.
- **`coder/`** — what **every** Coder workspace needs: Tailscale, code-server, the mise toolchain and agent CLIs.
- **`coder-desktop/`** — **only** the graphical stack. Nothing else may go here; a tool that belongs to all workspaces goes in `coder/` so headless workspaces get it too.

The split exists because most workspaces run headless, and the desktop stack is ~650 MB they would otherwise pull for nothing. Adding a package to the wrong layer quietly gives that back — check where it belongs before adding it.

Resist adding a fourth image without a real selector driving it. In particular **there is no `coder-gpu`**: the GPU-specific piece is `nvidia-utils`, which must match the host's driver version and therefore cannot be resolved at build time. `coder-desktop` carries both the Mesa and the bare GL loader packages so it serves GPU and software rendering alike, and the driver-matched package stays a runtime install.

## The point of the `coder` images: bake, don't install on boot

The Coder workspace templates that consume this image used to `curl | sh` Tailscale, code-server, the desktop packages, and every CLI on **every workspace start**, costing minutes per boot. The `coder` image exists to make those steps no-ops.

When you touch `coder/Dockerfile` or `coder-desktop/Dockerfile`, hold that line:

- **Bake** anything that is the same for every workspace and every user: packages, binaries, static config.
- **Do not bake** anything that is a secret, a per-workspace identity, or a running daemon. Auth keys, SSH keys, Tailscale node state, and `tailscaled`/`dockerd`/`Xvfb` processes are the startup script's job, and this image is public — a baked credential is a published credential.
- Version-volatile tools go in via **mise**, not a pinned tarball. The workspace still runs `mise use -g <tool>@latest` at start; the baked copy just makes the common case resolve instantly instead of cold-downloading. That is why `claude-code` and `codex` are baked despite shipping new versions constantly.

A weekly `schedule:` rebuild keeps the baked tools from drifting far behind upstream between Dockerfile edits. If you add something whose freshness matters, rely on that rather than pinning a version you will forget to bump.

## Dockerfile conventions

- Alphabetize package lists. They get edited by many hands and a sorted list makes conflicts and duplicates obvious.
- Use `--no-install-recommends` on every `apt-get install`, and `rm -rf /var/lib/apt/lists/*` in the same `RUN`.
- Delete download caches in the same layer that creates them (`~/.cache/mise`, `/root/.cache/code-server`). A cache removed in a *later* `RUN` still ships in the image.
- Comment the *why*, not the *what*. `RUN apt-get install xvfb` needs no comment; the reason both Mesa and bare GL loader packages are installed does.
- Group related installs into one `RUN` so the layer count stays sane, but keep independently-cacheable, slow steps (mise tools, code-server) separate so an edit to one doesn't rebuild the other.
- **Never call `sudo`.** The arm64 leg of the multi-arch build runs under QEMU user-mode emulation, which does not honour the setuid bit, so `sudo` fails there while working fine locally on amd64 — a failure you will only see in CI. Switch `USER root`, do the work, switch back.

## Verifying a change

There is no CI test job — the build **is** the test, and it only runs after merge to `main` unless the PR touches a watched path. Verify locally before pushing:

```bash
docker build -f devcontainer/Dockerfile -t fx-devcontainer:test .
docker build -f coder/Dockerfile --build-arg BASE_IMAGE=fx-devcontainer:test -t fx-coder:test .
docker build -f coder-desktop/Dockerfile --build-arg BASE_IMAGE=fx-coder:test -t fx-coder-desktop:test .
```

Build the whole chain, not just the layer you edited — and delete the old tags first. A stale `fx-coder:test` left over from a failed build will happily serve as a base for the next one, and you will verify an image that no longer matches the Dockerfiles.

Then check the things that actually break:

```bash
# Tools must resolve in a NON-interactive, non-login shell — that is how ssh
# one-liners, git hooks, and agent subprocesses invoke them. A tool that only
# works under `bash -l` is a broken tool.
docker run --rm fx-coder:test bash -c 'command -v gh kubectl claude codex bun code-server'

# The split is only real if the headless image stays headless.
docker run --rm fx-coder:test bash -c 'command -v Xvfb && echo LEAKED-INTO-BASE'
```

For the desktop stack, start it and confirm noVNC answers rather than trusting that the packages installed:

```bash
docker run --rm fx-coder-desktop:test bash -c '
  Xvfb :1 -screen 0 1280x720x24 +extension GLX +render -noreset & sleep 3
  DISPLAY=:1 glxinfo -B | grep "OpenGL renderer"'
```

Confirm the image sizes did not jump unexpectedly (`docker images`); a sudden increase usually means a cache was left behind, or something landed in a lower layer than intended.

## Consumers

Changing these images changes machines that are already running. Before altering or removing something in `devcontainer/`, remember it is consumed both as a Coder workspace fallback image and as a `.devcontainer` base elsewhere. Removing a package that was previously present is a breaking change even when nothing here references it.

The Coder workspace templates that pair with these images live in a **separate, private** infrastructure repository; the template is what picks `coder` vs `coder-desktop`. When a change here lets a template drop a startup step, or adds a variant for it to select, that template change is a separate PR in that repo — and **no detail of that private infrastructure may land in this repository**, which is public. Keep image documentation generic: describe what is baked and why, never internal hostnames, registry addresses, cluster or namespace names, secret names, or client/organization-specific configuration.

## Task Tracking

**You MUST load the `/project-management` skill before creating, modifying, or completing any task.** It owns all task-tracking rules and knows where tasks belong. Do not manage tasks without it.

## Code Review Rules

Read `REVIEW.md` at the repository root and apply it in full as the review rules for this repo. It is the canonical review-conventions file.
