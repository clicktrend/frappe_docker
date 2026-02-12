# Build from ADOMIO bench image with v16 support
FROM bench-adomio:latest

LABEL author=adomio
LABEL description="Frappe Bench v16 with Claude Code integration"

USER frappe
WORKDIR /home/frappe

# Install Claude Code and related tools with Node 24
RUN . ${NVM_DIR}/nvm.sh \
    && nvm use 24 \
    && npm install -g @anthropic-ai/claude-code \
    && npm install -g claude-yolo \
    && npm install -g frappe-mcp-server

# Create .claude directory for configuration
RUN mkdir -p /home/frappe/.claude \
    && chown -R frappe:frappe /home/frappe/.claude

# Verify installations
RUN . ${NVM_DIR}/nvm.sh \
    && nvm use 24 \
    && node --version \
    && npm --version \
    && yarn --version \
    && bench --help

# Verify Python 3.14 and Node 24
RUN bash -c "python --version | grep '3.14'" \
    && bash -c "node --version | grep 'v24'"

EXPOSE 8000-8005 9000-9005 6787

CMD ["/bin/bash"]
