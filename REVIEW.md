# PR Review

## Task Cross-Reference

This repository intentionally has no `docs/` tree — it holds Dockerfiles, not a spec-driven project. **If** `docs/changes/` or `docs/tasks.md` are ever added, cross-reference every PR against those task lists: when a PR completes work tracked there, the task checkboxes MUST be updated in the same PR, and changes should be requested if they are missing. While those paths do not exist, there is nothing to cross-reference — do not send reviewers chasing them.

---

## Repository Context

This repo contains only Dockerfiles and the GitHub Actions workflows that publish them to GHCR. The images form a chain, each built FROM the one above it: `devcontainer` → `coder` (headless workspace) → `coder-desktop` (adds the X11/VNC stack). All are published multi-arch for `linux/amd64` and `linux/arm64`.

## This Repository Is Public

Flag any content sourced from private infrastructure: internal hostnames or IP addresses, container registry addresses, Kubernetes cluster or namespace names, secret names, auth keys or tokens of any kind, and client- or organization-specific configuration. Request changes on every occurrence — this repository is public and its images are published to a public registry.

No credential, token, or auth key may be baked into an image, including via a build arg. Build args are recorded in image history and are readable by anyone who pulls it.

What this rule is **not**: `.gitignore` entries that keep secrets out of the repo are safety controls, not disclosures — do not ask for them to be removed. `.tailscale/` in particular reveals nothing the public `coder/Dockerfile` does not already state by installing Tailscale, and an attacker who can read that path already has code execution in the workspace, where `tailscale status` and `/var/lib/tailscale` are right there. Removing the ignore would only make committing a key more likely. Comments describing *private infrastructure behaviour* are a fair thing to flag; the ignore rule itself is not.

## Dockerfile Review Checks

- **Both architectures.** Anything installed must work on `linux/amd64` and `linux/arm64`, or fail non-fatally with an explicit comment saying why. A `curl | sh` installer with no arm64 build that aborts the layer breaks half the matrix.
- **Caches die in the layer that made them.** `rm -rf ~/.cache/...` in a later `RUN` does not shrink the image. Same for `rm -rf /var/lib/apt/lists/*` — it belongs in the same `RUN` as the `apt-get install`.
  - Do **not** claim a leftover npm/bun cache in the mise layer without checking the built image first. mise's cache is `~/.cache/mise` and is already removed; there is no `~/.npm`, `~/.bun` or `_cacache` in these images (verified). What *is* large under `~/.local/share/mise/installs` is the tool payload itself — the Codex CLI vendors a ~294 MB binary — and deleting it would remove the CLI, not a cache. This has been reported as a finding before and was wrong both on the cache's existence and on what the large directory is.
- **`--no-install-recommends`** on every `apt-get install`.
- **Base pinning.** Every non-root image must take its base via the `BASE_IMAGE` build arg so CI can point it at the digest built in the same run. A hard-coded `:latest` base means a PR is never actually testing its own changes to the layer below.
- **Removals are breaking.** These images are consumed by machines already running, both as Coder workspace images and as `.devcontainer` bases. Deleting a package or binary that was previously present needs a stated reason, not silence.
- **Right layer.** `devcontainer/` stays generic — nothing Coder-specific. `coder/` holds what every workspace needs. `coder-desktop/` holds the graphical stack and nothing else. The most common mistake is putting a universally-useful tool in `coder-desktop/` (headless workspaces silently lose it) or putting a graphical package in `coder/` (every headless workspace pays to pull it). Check the layer on every added package.
- **`github.event_name` inside `build-image.yml` is the CALLER's event, not `workflow_call`.** A reusable workflow inherits the `github` context from the caller, so `github.event_name == 'schedule'` and `type=ref,event=pr` both work there. Do not "fix" either as broken — verified from a live run: `metadata-action` inside the reusable workflow emitted `pr-1` for all three images, and `ghcr.io/fx/docker/*:pr-1` exist. This has been reported as a finding twice and was wrong both times.
- **New variants need a selector.** A fourth image is only justified if the workspace template has something to switch on. Reject variants that duplicate a runtime decision — GPU support in particular cannot be baked, because `nvidia-utils` must match the host driver version.
- **Baked vs. runtime.** The `coder` image exists to eliminate per-boot installs. Anything static and universal should be baked; anything that is a secret, a per-workspace identity, or a long-running daemon must not be. Question additions that violate either direction.
- **Alphabetized package lists**, so duplicates and conflicts are visible.
- **No `sudo` in a Dockerfile.** The arm64 leg builds under QEMU user-mode emulation, which ignores the setuid bit, so `sudo` fails there while passing a local amd64 build. Require `USER root` / `USER vscode` around the privileged step instead.

## Verification Expectations

There is no test suite; the image build is the only check, and it does not run on every PR. A PR that changes a Dockerfile should say that the **whole chain** was built locally and what was verified — at minimum that new tools resolve in a **non-interactive, non-login** shell (`docker run --rm <img> bash -c 'command -v ...'`), not just under `bash -l`. Ask for that evidence when it is missing.

Building only the edited layer is not enough, and neither is building on top of a stale local tag from an earlier failed run — that verifies an image the Dockerfiles no longer describe. Reported image sizes are part of the evidence: if a change was supposed to move weight between layers, the sizes should show it.
