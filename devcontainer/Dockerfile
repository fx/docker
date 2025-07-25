FROM mcr.microsoft.com/devcontainers/base:ubuntu-22.04

# Install base dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    wget \
    gnupg \
    lsb-release \
    ca-certificates \
    apt-transport-https \
    software-properties-common \
    dirmngr \
    gh \
    xdg-utils \
    sudo \
    procps \
    htop \
    && rm -rf /var/lib/apt/lists/*

# Install Docker CE
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

# Set up vscode user with docker access
RUN usermod -aG docker vscode 2>/dev/null || true

# Install mise via APT repository
RUN ARCH="$(dpkg --print-architecture)" && \
    install -dm 755 /etc/apt/keyrings && \
    wget -qO - https://mise.jdx.dev/gpg-key.pub | gpg --dearmor | tee /etc/apt/keyrings/mise-archive-keyring.gpg 1> /dev/null && \
    echo "deb [signed-by=/etc/apt/keyrings/mise-archive-keyring.gpg arch=${ARCH}] https://mise.jdx.dev/deb stable main" | tee /etc/apt/sources.list.d/mise.list && \
    apt-get update && \
    apt-get install -y mise && \
    rm -rf /var/lib/apt/lists/*

# Switch to vscode user for user-specific installations
USER vscode
WORKDIR /home/vscode

# Configure mise for vscode user
RUN echo 'eval "$(mise activate bash)"' >> ~/.bashrc && \
    echo 'eval "$(mise activate zsh)"' >> ~/.zshrc && \
    mise --version

# Set up mise and install Node.js LTS
RUN mise install node@lts \
    && mise use -g node@lts

# Switch back to root for any final setup
USER root

# Enable Docker service (will be started by docker-in-docker feature)
RUN update-alternatives --set iptables /usr/sbin/iptables-legacy \
    && update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# Final user switch
USER vscode