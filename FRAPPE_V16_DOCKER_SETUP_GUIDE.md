# Frappe v16 Docker Setup Guide für Devcontainer

## Kontext & Problem

Frappe Framework v16 erfordert **zwingend**:
- Python 3.14.2 (nicht 3.11.6)
- Node.js 24.13.0 (nicht 20.19.2)

Das offizielle `frappe/frappe_docker` Repository enthält nur Images für v15 mit:
- Python 3.11.6
- Node.js 20.19.2

**Problem:** Diese Versionen sind im Docker Base-Image fest kompiliert (via pyenv/nvm) und können NICHT nachträglich im laufenden Container geändert werden.

---

## Lösung: Parallele Images Strategie

Wenn Ihr Projekt ein **Fork von frappe/frappe_docker** ist:

### Strategie: Parallele Images statt Modifikation des offiziellen Images

```
images/bench/          → Python 3.11.6, Node 20.19.2 (offiziell, UNVERÄNDERT)
images/bench-adomio/   → Python 3.14.2, Node 24.13.0  (v16 custom, NEU)
```

**Vorteil:** Offizielle Files bleiben untouched für upstream updates!

---

## Schritt 1: Neues Base-Image erstellen (bench-adomio)

### 1.1 Verzeichnis erstellen und Dockerfile kopieren

```bash
mkdir -p images/bench-adomio
cp images/bench/Dockerfile images/bench-adomio/Dockerfile
```

### 1.2 Dockerfile für v16 anpassen

**Datei:** `images/bench-adomio/Dockerfile`

#### Änderung 1: Labels aktualisieren

```dockerfile
# VON:
LABEL author=frappé

# ZU:
LABEL author=adomio
LABEL description="Frappe Bench v16 Base Image with Python 3.14.2 and Node.js 24.13.0"
```

#### Änderung 2: Python-Versionen aktualisieren

```dockerfile
# VON:
ENV PYTHON_VERSION_V14=3.10.13
ENV PYTHON_VERSION=3.11.6

# ZU:
ENV PYTHON_VERSION_V15=3.11.6
ENV PYTHON_VERSION=3.14.2
```

#### Änderung 3: pyenv Installation anpassen (KRITISCH!)

```dockerfile
# VON:
RUN git clone --depth 1 https://github.com/pyenv/pyenv.git .pyenv \
    && pyenv install $PYTHON_VERSION_V14 \
    && pyenv install $PYTHON_VERSION \
    && PYENV_VERSION=$PYTHON_VERSION_V14 pip install --no-cache-dir virtualenv \
    && PYENV_VERSION=$PYTHON_VERSION pip install --no-cache-dir virtualenv \
    && pyenv global $PYTHON_VERSION $PYTHON_VERSION_V14 \

# ZU:
RUN git clone https://github.com/pyenv/pyenv.git .pyenv \
    && cd .pyenv && git pull origin master && cd ~ \
    && pyenv install $PYTHON_VERSION_V15 \
    && pyenv install $PYTHON_VERSION \
    && PYENV_VERSION=$PYTHON_VERSION_V15 pip install --no-cache-dir virtualenv \
    && PYENV_VERSION=$PYTHON_VERSION pip install --no-cache-dir virtualenv \
    && pyenv global $PYTHON_VERSION $PYTHON_VERSION_V15 \
```

**WICHTIG:** `--depth 1` muss entfernt werden + `git pull origin master`, weil pyenv Python 3.14.2 nur in neuesten Commits unterstützt!

#### Änderung 4: Node.js-Versionen aktualisieren

```dockerfile
# VON:
ENV NODE_VERSION_14=16.20.2
ENV NODE_VERSION=20.19.2

# ZU:
ENV NODE_VERSION_V15=20.19.2
ENV NODE_VERSION=24.13.0
```

#### Änderung 5: nvm Version aktualisieren

```dockerfile
# VON:
RUN wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash \
    && . ${NVM_DIR}/nvm.sh \
    && nvm install ${NODE_VERSION_14} \
    && nvm use v${NODE_VERSION_14} \
    && npm install -g yarn \
    && nvm install ${NODE_VERSION} \
    && nvm use v${NODE_VERSION} \
    && npm install -g yarn \

# ZU:
RUN wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash \
    && . ${NVM_DIR}/nvm.sh \
    && nvm install ${NODE_VERSION_V15} \
    && nvm use v${NODE_VERSION_V15} \
    && npm install -g yarn \
    && nvm install ${NODE_VERSION} \
    && nvm use v${NODE_VERSION} \
    && npm install -g yarn \
```

### 1.3 bench-adomio Image bauen

```bash
cd /path/to/your/frappe_docker_project
DOCKER_BUILDKIT=1 docker build -t bench-adomio:latest -f images/bench-adomio/Dockerfile images/bench-adomio/
```

**Dauer:** ~20-25 Minuten (Python wird von Source kompiliert)

---

## Schritt 2: Entwickler-Image anpassen (bench-claude)

### 2.1 Dockerfile komplett neu schreiben

**Datei:** `images/bench-claude/Dockerfile`

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

**WICHTIG:** FROM-Zeile geändert von `docker.io/frappe/bench:latest` zu `bench-adomio:latest`

### 2.2 bench-claude Image bauen

```bash
docker build -t <your-project-name>:bench-claude-v16 -f images/bench-claude/Dockerfile .
```

**Dauer:** ~5-10 Minuten

---

## Schritt 3: Devcontainer-Konfiguration anpassen

### 3.1 docker-compose.yml aktualisieren

**Datei:** `.devcontainer/docker-compose.yml`

```yaml
# VON:
frappe:
  build:
    context: ../images/bench-claude
  image: <your-project>:bench-claude

# ZU:
frappe:
  build:
    context: ..
    dockerfile: images/bench-claude/Dockerfile
  image: <your-project>:bench-claude-v16
```

**Änderungen:**
1. `context` von `../images/bench-claude` zu `..` (Parent-Verzeichnis)
2. `dockerfile` explizit auf `images/bench-claude/Dockerfile` gesetzt
3. `image` Tag auf `-v16` geändert

---

## Schritt 4: Devcontainer neu bauen

### In VSCode:

1. Drücken Sie: `Cmd/Ctrl + Shift + P`
2. Geben Sie ein: "Dev Containers: Rebuild Container"
3. Wählen Sie: "Dev Containers: Rebuild Container"

### Oder manuell:

```bash
cd .devcontainer
docker-compose down
docker-compose up -d
```

---

## Schritt 5: Verifikation im neuen Container

Nach dem Rebuild im Container ausführen:

```bash
python --version   # Sollte: Python 3.14.2 zeigen
node --version     # Sollte: v24.13.0 zeigen
bench --version    # Sollte: 5.x.x zeigen (aktuellste bench Version)
```

---

## Schritt 6: Procfile anpassen (wenn vorhanden)

**Datei:** `development/frappe-bench/Procfile`

```bash
# SocketIO-Zeile anpassen:

# VON:
socketio: /home/frappe/.nvm/versions/node/v20.19.2/bin/node apps/frappe/socketio.js

# ZU:
socketio: /home/frappe/.nvm/versions/node/v24.13.0/bin/node apps/frappe/socketio.js
```

---

## Wichtige Hinweise

### Docker Build Cache

Nach dem ersten Build (20-25 Min) sind nachfolgende Builds sehr schnell (~30 Sekunden), solange sich das Dockerfile nicht ändert!

### Warum nicht FROM frappe/bench:latest erben?

❌ **Funktioniert nicht:**
```dockerfile
FROM docker.io/frappe/bench:latest
RUN pyenv install 3.14.2  # Dauert genauso lang!
RUN nvm install 24.13.0   # Dauert genauso lang!
```

**Probleme:**
1. Build-Zeit fast identisch (~20-25 Min)
2. Image-Größe verdoppelt sich (beide Python + Node Versionen)
3. Alte Versionen bleiben im Image (verschwendeter Speicher)

### Port-Konfiguration (optional)

Falls Sie mehrere Frappe-Projekte parallel betreiben:

**Datei:** `.devcontainer/docker-compose.yml`

```yaml
frappe:
  ports:
    - "3010-3015:8000-8005"  # HTTP Ports (anpassen nach Bedarf)
    - "4010:9000"            # SocketIO Port
  environment:
    - VIRTUAL_HOST=yourproject.local
    - VIRTUAL_PORT=8000
    - SELF_SIGNED_HOST=yourproject.local  # Für HTTPS mit nginx-proxy
```

**Datei:** `development/frappe-bench/Procfile`

```bash
web: bench serve --port 8000  # Falls geändert, auch in common_site_config.json anpassen
```

**Datei:** `development/frappe-bench/sites/common_site_config.json`

```json
{
  "webserver_port": 8000,
  "socketio_port": 9000
}
```

---

## Zusammenfassung der Dateien

### Neu erstellt:
- `images/bench-adomio/Dockerfile` (Kopie von `images/bench/Dockerfile` mit v16-Anpassungen)

### Modifiziert:
- `images/bench-claude/Dockerfile` (FROM-Zeile + v16-spezifische Änderungen)
- `.devcontainer/docker-compose.yml` (build context + image tag)
- `development/frappe-bench/Procfile` (Node.js Pfad für socketio)

### Unverändert:
- `images/bench/Dockerfile` (Original bleibt für upstream updates)

---

## Troubleshooting

### Build schlägt bei Python 3.14.2 Installation fehl

**Problem:** pyenv hat nicht die neueste Version
**Lösung:** `git clone --depth 1` entfernen und `git pull origin master` hinzufügen

### Container startet nicht nach Rebuild

**Prüfen:**
1. Ist das Image erfolgreich gebaut? `docker images | grep bench-claude-v16`
2. Läuft der alte Container noch? `docker ps -a`
3. Ports belegt? `netstat -tulpn | grep 8000`

### Bench-Befehle funktionieren nicht

**Problem:** Python/Node nicht im PATH
**Lösung:** In `.bashrc` oder `.profile` prüfen:
```bash
export PYENV_ROOT="/home/frappe/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
```

---

## Quick Reference Commands

```bash
# Images bauen
docker build -t bench-adomio:latest -f images/bench-adomio/Dockerfile images/bench-adomio/
docker build -t yourproject:bench-claude-v16 -f images/bench-claude/Dockerfile .

# Image verifizieren
docker run --rm bench-adomio:latest python --version
docker run --rm bench-adomio:latest node --version
docker run --rm bench-adomio:latest bench --version

# Container neu bauen
cd .devcontainer
docker-compose down
docker-compose up -d

# Im Container verifizieren
python --version
node --version
bench --version
```

---

## Zeitabschätzung

- **Schritt 1 (bench-adomio bauen):** 20-25 Minuten
- **Schritt 2 (bench-claude bauen):** 5-10 Minuten
- **Schritt 3 (docker-compose.yml anpassen):** 2 Minuten
- **Schritt 4 (Devcontainer rebuild):** 3-5 Minuten
- **Schritt 5+6 (Verifikation + Procfile):** 5 Minuten

**Gesamt:** ~35-50 Minuten für den ersten vollständigen Setup

---

## Credits

Diese Anleitung basiert auf der Frappe v16 Migration für das ADOMIO ERP-Projekt.

Erstellt: 2026-01-28
Frappe Version: v16 (16.2.1)
ERPNext Version: v16 (16.1.0)
