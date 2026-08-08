# syntax=docker/dockerfile:1
#
# Coder workspace image — with graphical desktop.
#
# Last link of the chain: devcontainer -> coder -> coder-desktop. This adds ONLY
# the X11/VNC desktop stack on top of the headless `coder` image, so workspaces
# that never open a desktop are not made to pull it. Everything else — Tailscale,
# code-server, the toolchain, the agent CLIs — comes from the base and must not
# be duplicated here.
#
# The workspace template selects between `coder` and `coder-desktop` from its
# desktop option; nothing at runtime switches between them.
ARG BASE_IMAGE=ghcr.io/fx/docker/coder:latest
FROM ${BASE_IMAGE}

LABEL org.opencontainers.image.source="https://github.com/fx/docker"
LABEL org.opencontainers.image.description="Coder workspace image with graphical desktop: X11, VNC, noVNC, Fluxbox, audio and OpenGL"

USER root
ENV DEBIAN_FRONTEND=noninteractive

# X11 + VNC + noVNC + Fluxbox + audio + OpenGL.
#
# Both the Mesa (software rendering) and the bare GL loader packages are
# installed: on GPU workspaces the vendor GLX/EGL libraries are bind-mounted
# from the host and only the loaders are needed, while everything else falls
# back to llvmpipe. Installing both means one image serves either case.
#
# The GPU-specific part deliberately stays at runtime: `nvidia-utils` has to
# match the host's driver version, which is not knowable at build time, so
# there is no useful `coder-gpu` variant to bake — only this image plus a
# driver-matched package the startup script installs when it sees /dev/nvidia0.
RUN apt-get update && apt-get install -y --no-install-recommends \
    dbus-x11 \
    fluxbox \
    imagemagick \
    libegl-mesa0 \
    libegl1 \
    libgl1 \
    libgl1-mesa-dri \
    libglu1-mesa \
    libglx-mesa0 \
    libglx0 \
    mesa-utils \
    novnc \
    pulseaudio \
    scrot \
    websockify \
    x11-utils \
    x11-xserver-utils \
    x11vnc \
    xterm \
    xvfb \
    && rm -rf /var/lib/apt/lists/*

# noVNC ships vnc.html, not index.html, and the Coder app proxies the bare "/".
RUN [ -f /usr/share/novnc/vnc.html ] && ln -sf vnc.html /usr/share/novnc/index.html || true

USER vscode
WORKDIR /home/vscode

ENV DEBIAN_FRONTEND=
