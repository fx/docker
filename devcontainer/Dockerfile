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
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

RUN usermod -aG docker vscode 2>/dev/null || true

# mise (runtime version manager) via its APT repository.
RUN ARCH="$(dpkg --print-architecture)" && \
    install -dm 755 /etc/apt/keyrings && \
    wget -qO - https://mise.jdx.dev/gpg-key.pub | gpg --dearmor | tee /etc/apt/keyrings/mise-archive-keyring.gpg 1> /dev/null && \
    echo "deb [signed-by=/etc/apt/keyrings/mise-archive-keyring.gpg arch=${ARCH}] https://mise.jdx.dev/deb stable main" | tee /etc/apt/sources.list.d/mise.list && \
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
RUN git config --system --add safe.directory '*'

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
