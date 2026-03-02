FROM fedora:latest

LABEL maintainer="Security Copilot Agent Dev Container"
LABEL description="Development environment for Microsoft Security Copilot agent development including Sentinel MCP setup"

# ─── System packages ──────────────────────────────────────────────────────────
RUN microdnf update -y && \
    microdnf install -y \
    git \
    curl \
    wget \
    jq \
    unzip \
    zip \
    tar \
    sudo \
    less \
    vim \
    nano \
    python3 \
    python3-pip \
    nodejs \
    npm \
    openssh-clients \
    ca-certificates \
    gnupg2 \
    bind-utils \
    iputils \
    procps-ng \
    findutils \
    diffutils \
    gcc \
    && microdnf clean all && \
    rm -rf /var/cache/dnf

# ─── GitHub CLI ───────────────────────────────────────────────────────────────
RUN cat << 'EOF' > /etc/yum.repos.d/gh-cli.repo
[gh-cli]
name=packages.githubcli.com
baseurl=https://cli.github.com/packages/rpm
enabled=1
gpgkey=https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x23F3D4EA75716059
gpgcheck=1
EOF

RUN microdnf install -y gh && \
    microdnf clean all && \
    rm -rf /var/cache/dnf

# ─── Azure CLI ────────────────────────────────────────────────────────────────
RUN rpm --import https://packages.microsoft.com/keys/microsoft.asc && \
    cat << 'EOF' > /etc/yum.repos.d/azure-cli.repo
[azure-cli]
name="Azure CLI"
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

RUN microdnf install -y azure-cli && \
    microdnf clean all && \
    rm -rf /var/cache/dnf

# ─── PowerShell ───────────────────────────────────────────────────────────────
RUN curl -sSL https://packages.microsoft.com/config/rhel/9/prod.repo \
    -o /etc/yum.repos.d/microsoft-prod.repo && \
    microdnf install -y powershell && \
    microdnf clean all && \
    rm -rf /var/cache/dnf

# ─── Node.js global packages ──────────────────────────────────────────────────
RUN npm install -g \
    @azure/mcp \
    typescript \
    ts-node

# ─── Create non-root user: sec-copilot ────────────────────────────────────────
RUN useradd -m -s /bin/bash sec-copilot && \
    echo "sec-copilot ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/sec-copilot && \
    chmod 0440 /etc/sudoers.d/sec-copilot

USER sec-copilot

WORKDIR /home/sec-copilot

# Ensure the user's local bin is on PATH
ENV PATH="/home/sec-copilot/.local/bin:${PATH}"

# ─── Update bash promt to use bash-it ────────────────────────────────────
RUN git clone --depth=1 https://github.com/Bash-it/bash-it.git ~/.bash_it
RUN ~/.bash_it/install.sh --silent
RUN sed -i "s/^export BASH_IT_THEME=.*$/export BASH_IT_THEME='nwinkler'/1" ~/.bashrc

# ─── Python packages for agent development ────────────────────────────────────
RUN pip3 install --no-cache-dir --upgrade pip && \
    pip3 install --no-cache-dir \
    azure-identity \
    azure-mgmt-securityinsight \
    azure-mgmt-loganalytics \
    azure-monitor-query \
    msal \
    requests \
    pyyaml \
    python-dotenv \
    openai
# Fix installation of semantic-kernel to avoid pydantic downgrade (which breaks other dependencies)
# 1. Ensure compatible pydantic is present
RUN pip3 install --no-cache-dir 'pydantic>=2.12' 'pydantic-core>=2.41'

# 2. Install semantic-kernel without pulling deps (avoids pydantic downgrade)
RUN pip3 install --no-cache-dir semantic-kernel --no-deps

# 3. Install the rest of semantic-kernel's dependencies
RUN pip3 install --no-cache-dir \
    'aiohttp~=3.8' 'cloudevents~=1.0' 'pydantic-settings~=2.0' 'defusedxml~=0.7' \
    'numpy>=1.26.0' 'openai<2,>=1.98.0' 'openapi_core<0.20,>=0.18' \
    'websockets<16,>=13' 'aiortc>=1.9.0' 'opentelemetry-api~=1.24' \
    'opentelemetry-sdk~=1.24' 'prance<25.4.9,>=23.6.21' 'pybars4~=0.9' \
    'jinja2~=3.1' 'nest-asyncio~=1.6' 'scipy>=1.15.1' 'protobuf' \
    'mcp>=1.26.0' 'azure-ai-agents>=1.2.0b3'

# ─── PowerShell Azure & Sentinel modules ──────────────────────────────────────
# Installed to CurrentUser scope so the sec-copilot user can use them
RUN pwsh -NoLogo -NonInteractive -Command " \
    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted; \
    Install-Module -Name Az                     -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Az.SecurityInsights    -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Az.Accounts            -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Az.Resources           -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Az.Monitor             -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Az.OperationalInsights -Force -AllowClobber -Scope CurrentUser; \
    Install-Module -Name Microsoft.Graph        -Force -AllowClobber -Scope CurrentUser; \
    Write-Host 'PowerShell modules installed successfully'"

CMD ["/bin/bash"]
