# syntax=docker/dockerfile:1
#
# Coder workspace image — headless.
#
# Middle link of the chain: devcontainer -> coder -> coder-desktop. Bakes in what
# EVERY Coder workspace previously curl-installed on each start (Tailscale,
# code-server, the mise-managed tool + agent-CLI set), and nothing that only some
# workspaces want. The graphical desktop stack is the one genuinely optional
# piece, so it lives in ../coder-desktop/Dockerfile rather than here — most
# workspaces run headless and should not pay to pull X11, VNC and Mesa.
#
# Workspace startup scripts should treat everything here as present and only do
# what genuinely cannot be baked (auth, daemons, mounted secrets).
#
# Anything version-volatile is still installed via mise, so a workspace can pull
# a newer release with `mise upgrade` without waiting on an image rebuild — the
# baked copy just makes the common case a no-op instead of a cold download.
ARG BASE_IMAGE=ghcr.io/fx/docker/devcontainer:latest
FROM ${BASE_IMAGE}

LABEL org.opencontainers.image.source="https://github.com/fx/docker"
LABEL org.opencontainers.image.description="Headless Coder workspace image: devcontainer base + Tailscale, code-server and agent CLIs"

USER root
ENV DEBIAN_FRONTEND=noninteractive

# Tailscale. Workspaces still run `tailscale up` at start (the auth key is a
# runtime secret), but the ~30s install is gone.
RUN curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$(lsb_release -cs).noarmor.gpg" \
        -o /usr/share/keyrings/tailscale-archive-keyring.gpg \
    && curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$(lsb_release -cs).tailscale-keyring.list" \
        -o /etc/apt/sources.list.d/tailscale.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends tailscale \
    && rm -rf /var/lib/apt/lists/*

# code-server, installed standalone so it is owned by the image rather than
# re-downloaded into /tmp on every workspace start.
RUN curl -fsSL https://code-server.dev/install.sh \
        | sh -s -- --method=standalone --prefix=/usr/local/lib/code-server \
    && ln -sf /usr/local/lib/code-server/bin/code-server /usr/local/bin/code-server \
    && code-server --version \
    && rm -rf /root/.cache/code-server

# mise shims must win in *every* context, not just interactive shells: non-login
# callers (ssh one-liners, VS Code tasks, git hooks, MCP subprocesses) never run
# `mise activate` and would otherwise get the apt-provided gh/kubectl. /usr/local/bin
# precedes /usr/bin everywhere, and each shim re-resolves its version through mise,
# so this keeps tracking `mise use` / `mise upgrade` at runtime.
ENV MISE_SHIMS=/home/vscode/.local/share/mise/shims
ENV PATH="${MISE_SHIMS}:${PATH}"

# Telemetry opt-outs and a UTF-8 locale (tmux infers UTF-8 from LC_*/LANG; without
# it, sessions come up in a non-UTF-8 mode and need `tmux -u`). C.UTF-8 is built
# into glibc, so no locale-gen is required.
ENV DO_NOT_TRACK=1 \
    RAILWAY_NO_TELEMETRY=1 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

USER vscode
WORKDIR /home/vscode

# Toolchain + agent CLIs via mise.
#   bun          — required by the workspace session-sync CLI installer
#   chezmoi      — dotfiles bootstrap
#   gh, kubectl  — newer than the apt/distro versions, and shimmed above
#   claude-code, codex — agent CLIs; dotfiles re-pin these to @latest at start,
#                        which is a fast no-op when the baked build is current
RUN mise use -g node@lts \
    && mise use -g bun@latest \
    && mise use -g chezmoi@latest \
    && mise use -g gh@latest \
    && mise use -g kubectl@latest \
    && mise use -g claude-code@latest \
    && mise use -g "npm:@openai/codex@latest" \
    && mise install \
    && mise reshim \
    && rm -rf ~/.cache/mise

# gh and kubectl are the two tools most often invoked non-interactively, so pin
# explicit /usr/local/bin symlinks ahead of the apt copies.
RUN sudo ln -sf "${MISE_SHIMS}/gh" /usr/local/bin/gh \
    && sudo ln -sf "${MISE_SHIMS}/kubectl" /usr/local/bin/kubectl

# CodeRabbit CLI (installs to ~/.local/bin as `coderabbit` + `cr`). CI=1 skips
# the interactive browser sign-in; workspaces authenticate at start with an API
# key. Not fatal if the installer has no build for this architecture.
RUN curl -fsSL https://cli.coderabbit.ai/install.sh | CI=1 sh \
    || echo "WARNING: CodeRabbit CLI install failed on $(dpkg --print-architecture); continuing"

ENV PATH="/home/vscode/.local/bin:${PATH}"
ENV DEBIAN_FRONTEND=
