# syntax=docker/dockerfile:1
#
# Generic development container: Ubuntu 24.04 + Docker CE + mise + Node LTS.
#
# This is the *base* image. It intentionally knows nothing about Coder,
# Tailscale, or agent CLIs — that lives in ../coder/Dockerfile, which builds
# FROM this image.
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

LABEL org.opencontainers.image.source="https://github.com/fx/docker"
LABEL org.opencontainers.image.description="Ubuntu 24.04 devcontainer with Docker CE, mise and Node LTS"

ENV DEBIAN_FRONTEND=noninteractive

# Retry apt transfers. The arm64 leg pulls from ports.ubuntu.com, which has been
# observed serving a half-synced index ("Hash Sum mismatch") and failing the
# build for reasons that have nothing to do with this repo. Set in the base
# image so every downstream layer in the chain inherits it.
RUN printf 'Acquire::Retries "3";\n' > /etc/apt/apt.conf.d/99-retries

# Base dependencies.
#
# python3 / psmisc / socat are here because downstream tooling (workspace
# startup scripts, devcontainer features) assumes them; installing them once
# here is cheaper than an apt round-trip on every container start.
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    dirmngr \
    dnsutils \
    file \
    gh \
    git \
    gnupg \
    htop \
    iproute2 \
    iputils-ping \
    jq \
    less \
    lsb-release \
    openssh-client \
    procps \
    psmisc \
    python3 \
    python3-venv \
    rsync \
    socat \
    software-properties-common \
    sudo \
    tmux \
    unzip \
    vim \
    wget \
    xdg-utils \
    zip \
    && rm -rf /var/lib/apt/lists/*

# Docker CE (docker-in-docker; the daemon is started by the consumer).
# The key is fetched to a file and dearmored from it, never piped: a pipeline
# reports only the last command's status, so a failed download would hand gpg
# empty input and still leave a zero exit and an empty keyring. apt would then
# fail later with a signature error that points nowhere near the real cause.
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.asc \
    && gpg --batch --yes --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg /tmp/docker.asc \
    && rm -f /tmp/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

RUN usermod -aG docker vscode 2>/dev/null || true

# mise (runtime version manager) via its APT repository.
# Same rule as the Docker key above — fetch to a file, dearmor from it. The
# upstream-documented `wget | gpg | tee` form is worse still: `tee` succeeding
# masks a failure in either earlier stage and writes an empty keyring.
RUN ARCH="$(dpkg --print-architecture)" && \
    install -dm 755 /etc/apt/keyrings && \
    wget -qO /tmp/mise-gpg-key.pub https://mise.jdx.dev/gpg-key.pub && \
    gpg --batch --yes --dearmor -o /etc/apt/keyrings/mise-archive-keyring.gpg /tmp/mise-gpg-key.pub && \
    rm -f /tmp/mise-gpg-key.pub && \
    echo "deb [signed-by=/etc/apt/keyrings/mise-archive-keyring.gpg arch=${ARCH}] https://mise.jdx.dev/deb stable main" > /etc/apt/sources.list.d/mise.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends mise && \
    rm -rf /var/lib/apt/lists/*

# Ubuntu's stock root/skel .bashrc guards interactivity with `[ -z "$PS1" ]`,
# which explodes under `set -u` (unbound variable) — e.g. when a provisioning
# script sources the profile with strict mode on. Use the ${PS1:-} form instead.
RUN for rc in /root/.bashrc /etc/skel/.bashrc /home/vscode/.bashrc; do \
        [ -f "$rc" ] && sed -i 's/\[ -z "\$PS1" \]/[ -z "${PS1:-}" ]/' "$rc" || true; \
    done

# Repos are bind-mounted from hosts with unrelated UIDs; without this every git
# command in a mounted workspace fails with "detected dubious ownership".
#
# Scoped to the two conventional mount roots rather than '*': /workspaces/<repo>
# is where VS Code devcontainers mount, /workspace/<name> is where Coder
# workspaces do. A trailing /* matches recursively, so nested repos and
# submodules under those roots are covered too. A blanket '*' would disable the
# ownership check for every path in the image, including anything a container
# later mounts from an untrusted source.
#
# Mounting a repo somewhere else (e.g. /src) will produce the dubious-ownership
# error; widen it for that consumer with an explicit
# `git config --global --add safe.directory <path>` rather than reinstating '*'.
RUN git config --system --add safe.directory '/workspace/*' \
    && git config --system --add safe.directory '/workspaces/*'

# `git-remote-https` and friends live in git-core and are not on the default PATH.
ENV PATH="/usr/lib/git-core:${PATH}"

# Docker-in-Docker needs the legacy iptables backend.
RUN update-alternatives --set iptables /usr/sbin/iptables-legacy \
    && update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

USER vscode
WORKDIR /home/vscode

# Activate mise for interactive shells and install Node LTS.
RUN echo 'eval "$(mise activate bash)"' >> ~/.bashrc && \
    echo 'eval "$(mise activate zsh)"' >> ~/.zshrc && \
    mise use -g node@lts && \
    mise install && \
    rm -rf ~/.cache/mise

ENV DEBIAN_FRONTEND=
