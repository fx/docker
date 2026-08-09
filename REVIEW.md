# PR Review

## Task Cross-Reference

Cross-reference every PR against task lists in `docs/changes/` and `docs/tasks.md`. If the PR completes work tracked in those files, the task checkboxes MUST be updated in this same PR. Request changes if missing.

---

## Repository Context

This repo contains only Dockerfiles and the GitHub Actions workflows that publish them to GHCR. The images form a chain, each built FROM the one above it: `devcontainer` → `coder` (headless workspace) → `coder-desktop` (adds the X11/VNC stack). All are published multi-arch for `linux/amd64` and `linux/arm64`.

## This Repository Is Public

Flag any content sourced from private infrastructure: internal hostnames or IP addresses, container registry addresses, Kubernetes cluster or namespace names, secret names, auth keys or tokens of any kind, and client- or organization-specific configuration. Request changes on every occurrence — this repository is public and its images are published to a public registry.

No credential, token, or auth key may be baked into an image, including via a build arg. Build args are recorded in image history and are readable by anyone who pulls it.

## Dockerfile Review Checks

- **Both architectures.** Anything installed must work on `linux/amd64` and `linux/arm64`, or fail non-fatally with an explicit comment saying why. A `curl | sh` installer with no arm64 build that aborts the layer breaks half the matrix.
- **Caches die in the layer that made them.** `rm -rf ~/.cache/...` in a later `RUN` does not shrink the image. Same for `rm -rf /var/lib/apt/lists/*` — it belongs in the same `RUN` as the `apt-get install`.
- **`--no-install-recommends`** on every `apt-get install`.
- **Base pinning.** Every non-root image must take its base via the `BASE_IMAGE` build arg so CI can point it at the digest built in the same run. A hard-coded `:latest` base means a PR is never actually testing its own changes to the layer below.
- **Removals are breaking.** These images are consumed by machines already running, both as Coder workspace images and as `.devcontainer` bases. Deleting a package or binary that was previously present needs a stated reason, not silence.
- **Right layer.** `devcontainer/` stays generic — nothing Coder-specific. `coder/` holds what every workspace needs. `coder-desktop/` holds the graphical stack and nothing else. The most common mistake is putting a universally-useful tool in `coder-desktop/` (headless workspaces silently lose it) or putting a graphical package in `coder/` (every headless workspace pays to pull it). Check the layer on every added package.
- **New variants need a selector.** A fourth image is only justified if the workspace template has something to switch on. Reject variants that duplicate a runtime decision — GPU support in particular cannot be baked, because `nvidia-utils` must match the host driver version.
- **Baked vs. runtime.** The `coder` image exists to eliminate per-boot installs. Anything static and universal should be baked; anything that is a secret, a per-workspace identity, or a long-running daemon must not be. Question additions that violate either direction.
- **Alphabetized package lists**, so duplicates and conflicts are visible.
- **No `sudo` in a Dockerfile.** The arm64 leg builds under QEMU user-mode emulation, which ignores the setuid bit, so `sudo` fails there while passing a local amd64 build. Require `USER root` / `USER vscode` around the privileged step instead.

## Verification Expectations

There is no test suite; the image build is the only check, and it does not run on every PR. A PR that changes a Dockerfile should say that the **whole chain** was built locally and what was verified — at minimum that new tools resolve in a **non-interactive, non-login** shell (`docker run --rm <img> bash -c 'command -v ...'`), not just under `bash -l`. Ask for that evidence when it is missing.

Building only the edited layer is not enough, and neither is building on top of a stale local tag from an earlier failed run — that verifies an image the Dockerfiles no longer describe. Reported image sizes are part of the evidence: if a change was supposed to move weight between layers, the sizes should show it.
