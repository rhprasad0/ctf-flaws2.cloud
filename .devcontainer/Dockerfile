# Hardened Minimal Kali Linux Devcontainer for flaws2.cloud CTF
# Security Focus: Prompt injection protection and minimal attack surface

FROM kalilinux/kali-rolling:latest

# Avoid prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Security: Non-root user configuration
ARG USER_NAME=ctf
ARG USER_UID=1000
ARG USER_GID=1000

# Security environment defaults
ENV HISTFILE=/dev/null \
    HISTSIZE=0 \
    HISTFILESIZE=0 \
    PATH=/opt/cloudmapper:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Install minimal CTF tools only
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Core utilities
    ca-certificates \
    curl \
    wget \
    git \
    vim-tiny \
    sudo \
    gnupg \
    # AWS tools for flaws2.cloud
    awscli \
    # Network tools
    dnsutils \
    netcat-openbsd \
    nmap \
    # Web/parsing tools
    jq \
    # Python for scripting
    python3-minimal \
    python3-pip \
    python3-venv \
    # CloudMapper dependencies
    python3-dev \
    libffi-dev \
    build-essential \
    autoconf \
    automake \
    libtool \
    # Node.js for Claude CLI
    nodejs \
    npm \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install Claude Code via npm
RUN npm install -g @anthropic-ai/claude-code@latest

# Install CloudMapper with Python 3.13 compatible dependencies
# Note: pyjq has Python 3.13 issues, using jq binary + python-jq alternative
RUN git clone --depth 1 https://github.com/duo-labs/cloudmapper.git /opt/cloudmapper \
    && cd /opt/cloudmapper \
    && pip3 install --break-system-packages \
        boto3 \
        botocore \
        netaddr \
        pyyaml \
        parliament \
        jinja2 \
        policyuniverse \
        requests \
    && chmod -R 755 /opt/cloudmapper

# Security: Remove setuid/setgid binaries where possible
RUN find / -type f \( -perm -4000 -o -perm -2000 \) \
    ! -path "/usr/bin/sudo" \
    ! -path "/usr/bin/nmap" \
    -exec chmod a-s {} \; 2>/dev/null || true

# Create non-root user
RUN groupadd --gid ${USER_GID} ${USER_NAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} -m -s /bin/bash ${USER_NAME} \
    && echo "${USER_NAME} ALL=(ALL) NOPASSWD: /usr/bin/nmap, /usr/bin/apt-get" >> /etc/sudoers.d/${USER_NAME} \
    && chmod 0440 /etc/sudoers.d/${USER_NAME}

# Security: Disable core dumps
RUN echo "* hard core 0" >> /etc/security/limits.conf \
    && echo "* soft core 0" >> /etc/security/limits.conf

# Setup user environment
USER ${USER_NAME}
WORKDIR /home/${USER_NAME}

# Install uv (includes uvx) for AWS MCP server
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Security: Harden bash profile
RUN echo 'umask 077' >> ~/.bashrc \
    && echo 'unset HISTFILE' >> ~/.bashrc \
    && echo 'export HISTSIZE=0' >> ~/.bashrc \
    && echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc \
    && mkdir -p ~/.aws

CMD ["/bin/bash"]
