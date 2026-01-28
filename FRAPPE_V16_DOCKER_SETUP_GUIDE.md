# Frappe v16 Docker Setup Guide for Devcontainer

## Context & Problem

Frappe Framework v16 **requires**:
- Python 3.14.2 (not 3.11.6)
- Node.js 24.13.0 (not 20.19.2)

The official `frappe/frappe_docker` repository only contains images for v15 with:
- Python 3.11.6
- Node.js 20.19.2

**Problem:** These versions are compiled into the Docker base image (via pyenv/nvm) and CANNOT be changed retroactively in a running container.

---

## Solution: Parallel Images Strategy

If your project is a **fork of frappe/frappe_docker**:

### Strategy: Parallel Images Instead of Modifying Official Images

```
images/bench/          → Python 3.11.6, Node 20.19.2 (official, UNCHANGED)
images/bench-adomio/   → Python 3.14.2, Node 24.13.0  (v16 custom, NEW)
```

**Advantage:** Official files remain untouched for upstream updates!

---

## Step 1: Create New Base Image (bench-adomio)

### 1.1 Create Directory and Copy Dockerfile

```bash
mkdir -p images/bench-adomio
cp images/bench/Dockerfile images/bench-adomio/Dockerfile
```

### 1.2 Adapt Dockerfile for v16

**File:** `images/bench-adomio/Dockerfile`

#### Change 1: Update Labels

```dockerfile
# FROM:
LABEL author=frappé

# TO:
LABEL author=adomio
LABEL description="Frappe Bench v16 Base Image with Python 3.14.2 and Node.js 24.13.0"
```

#### Change 2: Update Python Versions

```dockerfile
# FROM:
ENV PYTHON_VERSION_V14=3.10.13
ENV PYTHON_VERSION=3.11.6

# TO:
ENV PYTHON_VERSION_V15=3.11.6
ENV PYTHON_VERSION=3.14.2
```

#### Change 3: Adapt pyenv Installation (CRITICAL!)

```dockerfile
# FROM:
RUN git clone --depth 1 https://github.com/pyenv/pyenv.git .pyenv \
    && pyenv install $PYTHON_VERSION_V14 \
    && pyenv install $PYTHON_VERSION \
    && PYENV_VERSION=$PYTHON_VERSION_V14 pip install --no-cache-dir virtualenv \
    && PYENV_VERSION=$PYTHON_VERSION pip install --no-cache-dir virtualenv \
    && pyenv global $PYTHON_VERSION $PYTHON_VERSION_V14 \

# TO:
RUN git clone https://github.com/pyenv/pyenv.git .pyenv \
    && cd .pyenv && git pull origin master && cd ~ \
    && pyenv install $PYTHON_VERSION_V15 \
    && pyenv install $PYTHON_VERSION \
    && PYENV_VERSION=$PYTHON_VERSION_V15 pip install --no-cache-dir virtualenv \
    && PYENV_VERSION=$PYTHON_VERSION pip install --no-cache-dir virtualenv \
    && pyenv global $PYTHON_VERSION $PYTHON_VERSION_V15 \
```

**IMPORTANT:** `--depth 1` must be removed + `git pull origin master` added, because pyenv only supports Python 3.14.2 in the latest commits!

#### Change 4: Update Node.js Versions

```dockerfile
# FROM:
ENV NODE_VERSION_14=16.20.2
ENV NODE_VERSION=20.19.2

# TO:
ENV NODE_VERSION_V15=20.19.2
ENV NODE_VERSION=24.13.0
```

#### Change 5: Update nvm Version

```dockerfile
# FROM:
RUN wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash \
    && . ${NVM_DIR}/nvm.sh \
    && nvm install ${NODE_VERSION_14} \
    && nvm use v${NODE_VERSION_14} \
    && npm install -g yarn \
    && nvm install ${NODE_VERSION} \
    && nvm use v${NODE_VERSION} \
    && npm install -g yarn \

# TO:
RUN wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash \
    && . ${NVM_DIR}/nvm.sh \
    && nvm install ${NODE_VERSION_V15} \
    && nvm use v${NODE_VERSION_V15} \
    && npm install -g yarn \
    && nvm install ${NODE_VERSION} \
    && nvm use v${NODE_VERSION} \
    && npm install -g yarn \
```

### 1.3 Build bench-adomio Image

```bash
cd /path/to/your/frappe_docker_project
DOCKER_BUILDKIT=1 docker build -t bench-adomio:latest -f images/bench-adomio/Dockerfile images/bench-adomio/
```

**Duration:** ~20-25 minutes (Python is compiled from source)

---

## Step 2: Adapt Development Image (bench-claude)

### 2.1 Completely Rewrite Dockerfile

**File:** `images/bench-claude/Dockerfile`

```dockerfile
FROM bench-adomio:latest

LABEL author=adomio
LABEL description="Frappe Bench v16 with Claude Code integration"

USER frappe
WORKDIR /home/frappe

# Install Claude Code and related tools
RUN . ${NVM_DIR}/nvm.sh \
    && nvm use 24 \
    && npm install -g @anthropic-ai/claude-code \
    && npm install -g claude-yolo \
    && npm install -g frappe-mcp-server

# Verify installations
RUN . ${NVM_DIR}/nvm.sh \
    && nvm use 24 \
    && node --version \
    && npm --version \
    && yarn --version \
    && bench --help

# Verify Python and Node versions
RUN bash -c "python --version | grep '3.14'" \
    && bash -c "node --version | grep 'v24'"

EXPOSE 8000-8005 9000-9005 6787

CMD ["/bin/bash"]
```

**IMPORTANT:** FROM line changed from `docker.io/frappe/bench:latest` to `bench-adomio:latest`

### 2.2 Build bench-claude Image

```bash
docker build -t <your-project-name>:bench-claude-v16 -f images/bench-claude/Dockerfile .
```

**Duration:** ~5-10 minutes

---

## Step 3: Adapt Devcontainer Configuration

### 3.1 Update docker-compose.yml

**File:** `.devcontainer/docker-compose.yml`

```yaml
# FROM:
frappe:
  build:
    context: ../images/bench-claude
  image: <your-project>:bench-claude

# TO:
frappe:
  build:
    context: ..
    dockerfile: images/bench-claude/Dockerfile
  image: <your-project>:bench-claude-v16
```

**Changes:**
1. `context` from `../images/bench-claude` to `..` (parent directory)
2. `dockerfile` explicitly set to `images/bench-claude/Dockerfile`
3. `image` tag changed to `-v16`

---

## Step 4: Rebuild Devcontainer

### In VSCode:

1. Press: `Cmd/Ctrl + Shift + P`
2. Type: "Dev Containers: Rebuild Container"
3. Select: "Dev Containers: Rebuild Container"

### Or manually:

```bash
cd .devcontainer
docker-compose down
docker-compose up -d
```

---

## Step 5: Verification in New Container

After rebuild, execute in container:

```bash
python --version   # Should show: Python 3.14.2
node --version     # Should show: v24.13.0
bench --version    # Should show: 5.x.x (latest bench version)
```

---

## Step 6: Adapt Procfile (if present)

**File:** `development/frappe-bench/Procfile`

```bash
# Adjust SocketIO line:

# FROM:
socketio: /home/frappe/.nvm/versions/node/v20.19.2/bin/node apps/frappe/socketio.js

# TO:
socketio: /home/frappe/.nvm/versions/node/v24.13.0/bin/node apps/frappe/socketio.js
```

---

## Important Notes

### Docker Build Cache

After the first build (20-25 min), subsequent builds are very fast (~30 seconds) as long as the Dockerfile doesn't change!

### Why Not Inherit FROM frappe/bench:latest?

❌ **Doesn't work:**
```dockerfile
FROM docker.io/frappe/bench:latest
RUN pyenv install 3.14.2  # Takes just as long!
RUN nvm install 24.13.0   # Takes just as long!
```

**Problems:**
1. Build time almost identical (~20-25 min)
2. Image size doubles (both Python + Node versions)
3. Old versions remain in image (wasted space)

### Port Configuration (optional)

If you run multiple Frappe projects in parallel:

**File:** `.devcontainer/docker-compose.yml`

```yaml
frappe:
  ports:
    - "3010-3015:8000-8005"  # HTTP Ports (adjust as needed)
    - "4010:9000"            # SocketIO Port
  environment:
    - VIRTUAL_HOST=yourproject.local
    - VIRTUAL_PORT=8000
    - SELF_SIGNED_HOST=yourproject.local  # For HTTPS with nginx-proxy
```

**File:** `development/frappe-bench/Procfile`

```bash
web: bench serve --port 8000  # If changed, also adjust in common_site_config.json
```

**File:** `development/frappe-bench/sites/common_site_config.json`

```json
{
  "webserver_port": 8000,
  "socketio_port": 9000
}
```

---

## File Summary

### Newly created:
- `images/bench-adomio/Dockerfile` (copy of `images/bench/Dockerfile` with v16 adaptations)

### Modified:
- `images/bench-claude/Dockerfile` (FROM line + v16-specific changes)
- `.devcontainer/docker-compose.yml` (build context + image tag)
- `development/frappe-bench/Procfile` (Node.js path for socketio)

### Unchanged:
- `images/bench/Dockerfile` (original remains for upstream updates)

---

## Troubleshooting

### Build Fails at Python 3.14.2 Installation

**Problem:** pyenv doesn't have the latest version
**Solution:** Remove `git clone --depth 1` and add `git pull origin master`

### Container Doesn't Start After Rebuild

**Check:**
1. Is the image successfully built? `docker images | grep bench-claude-v16`
2. Is the old container still running? `docker ps -a`
3. Ports occupied? `netstat -tulpn | grep 8000`

### Bench Commands Don't Work

**Problem:** Python/Node not in PATH
**Solution:** Check in `.bashrc` or `.profile`:
```bash
export PYENV_ROOT="/home/frappe/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
```

---

## Quick Reference Commands

```bash
# Build images
docker build -t bench-adomio:latest -f images/bench-adomio/Dockerfile images/bench-adomio/
docker build -t yourproject:bench-claude-v16 -f images/bench-claude/Dockerfile .

# Verify image
docker run --rm bench-adomio:latest python --version
docker run --rm bench-adomio:latest node --version
docker run --rm bench-adomio:latest bench --version

# Rebuild container
cd .devcontainer
docker-compose down
docker-compose up -d

# Verify in container
python --version
node --version
bench --version
```

---

## Time Estimation

- **Step 1 (build bench-adomio):** 20-25 minutes
- **Step 2 (build bench-claude):** 5-10 minutes
- **Step 3 (adapt docker-compose.yml):** 2 minutes
- **Step 4 (devcontainer rebuild):** 3-5 minutes
- **Step 5+6 (verification + Procfile):** 5 minutes

**Total:** ~35-50 minutes for first complete setup

---

## Credits

This guide is based on the Frappe v16 migration for the ADOMIO ERP project.

Created: 2026-01-28
Frappe Version: v16 (16.2.1)
ERPNext Version: v16 (16.1.0)
