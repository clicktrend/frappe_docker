FROM docker.io/frappe/bench:latest

# Claude Code installieren
RUN npm install -g @anthropic-ai/claude-code
RUN npm install -g claude-yolo
RUN npm install -g frappe-mcp-server

RUN node --version \
    && npm --version \
    && yarn --version \
    && bench --help

RUN bash -c "node --version" \
    && bash -c "npm --version" \
    && bash -c "yarn --version" \
    && bash -c "bench --help"