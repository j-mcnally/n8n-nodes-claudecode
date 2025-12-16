# Docker Deployment Guide

This guide explains how to deploy n8n-nodes-claudecode in a Docker environment and resolve shell environment issues.

## The Issue

The error you're seeing occurs because the `claude-agent-sdk` requires:
1. A properly configured shell environment (bash/zsh/sh)
2. The `SHELL` environment variable to be set
3. Access to execute bash commands for tool operations

In many Docker containers, these aren't configured by default.

## Solutions

### Solution 1: Custom Dockerfile (Recommended)

Create a custom Dockerfile that extends the n8n image with proper shell configuration:

```dockerfile
FROM n8nio/n8n:latest

# Set shell environment
ENV SHELL=/bin/bash

# Install bash if not present (some minimal images don't have it)
USER root
RUN apk add --no-cache bash curl git

# Install the Claude Code community node
RUN cd /usr/local/lib/node_modules/n8n && \
    npm install @bilalmubarik/n8n-nodes-claudecode

# Switch back to node user
USER node

# Ensure bash is available
RUN bash --version
```

Build and run:
```bash
docker build -t n8n-claudecode .
docker run -it --rm \
  -p 5678:5678 \
  -e SHELL=/bin/bash \
  -v ~/.n8n:/home/node/.n8n \
  n8n-claudecode
```

### Solution 2: Docker Compose (Recommended for Production)

Create a `docker-compose.yml`:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    environment:
      - SHELL=/bin/bash
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./projects:/projects  # Mount your project directories
    command: >
      sh -c "
        apk add --no-cache bash curl git &&
        npm install -g @bilalmubarik/n8n-nodes-claudecode &&
        n8n start
      "

volumes:
  n8n_data:
```

Start with:
```bash
docker-compose up -d
```

### Solution 3: Environment Variables Only (Quick Fix)

If you're using an existing container, add these environment variables:

```bash
docker run -it --rm \
  -p 5678:5678 \
  -e SHELL=/bin/bash \
  -e N8N_COMMUNITY_NODE_PACKAGES=@bilalmubarik/n8n-nodes-claudecode \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

### Solution 4: Pre-built Image with Init Script

Create an entrypoint script `docker-entrypoint.sh`:

```bash
#!/bin/bash
set -e

# Ensure bash is available
if ! command -v bash &> /dev/null; then
    echo "Installing bash..."
    apk add --no-cache bash
fi

# Set shell environment
export SHELL=/bin/bash

# Install community node if not already installed
if ! npm list -g @bilalmubarik/n8n-nodes-claudecode &> /dev/null; then
    echo "Installing n8n-nodes-claudecode..."
    npm install -g @bilalmubarik/n8n-nodes-claudecode
fi

# Start n8n
exec n8n start
```

Then in your Dockerfile:

```dockerfile
FROM n8nio/n8n:latest

COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

ENV SHELL=/bin/bash

ENTRYPOINT ["/docker-entrypoint.sh"]
```

## Anthropic API Key Setup

The Claude Code SDK requires an Anthropic API key. In Docker, set it via environment variable:

```bash
docker run -it --rm \
  -p 5678:5678 \
  -e SHELL=/bin/bash \
  -e ANTHROPIC_API_KEY=your_api_key_here \
  -e N8N_COMMUNITY_NODE_PACKAGES=@bilalmubarik/n8n-nodes-claudecode \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Or in docker-compose.yml:

```yaml
services:
  n8n:
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
```

Then create a `.env` file:
```
ANTHROPIC_API_KEY=your_api_key_here
```

## Project Path Configuration

When using Docker, you need to mount your project directories:

```bash
docker run -it --rm \
  -p 5678:5678 \
  -e SHELL=/bin/bash \
  -e ANTHROPIC_API_KEY=your_api_key_here \
  -v ~/.n8n:/home/node/.n8n \
  -v /path/to/your/projects:/projects \
  n8nio/n8n
```

Then in your n8n workflow, set the **Project Path** parameter to:
```
/projects/your-project-name
```

## Troubleshooting

### Error: "shell environment isn't properly configured"

**Cause**: SHELL environment variable not set or bash not installed

**Fix**: Add `-e SHELL=/bin/bash` to your docker run command and ensure bash is installed

### Error: "Cannot execute bash commands"

**Cause**: Bash not available in the container

**Fix**: Install bash in your Dockerfile:
```dockerfile
RUN apk add --no-cache bash
```

### Error: "ANTHROPIC_API_KEY not found"

**Cause**: API key not configured

**Fix**: Set the environment variable:
```bash
-e ANTHROPIC_API_KEY=your_api_key_here
```

### Error: "Cannot access project path"

**Cause**: Project directory not mounted in Docker

**Fix**: Mount the directory:
```bash
-v /host/path:/container/path
```

## Testing Your Setup

After deploying, create a simple test workflow in n8n:

1. Add a **Manual Trigger** node
2. Add a **Claude Code** node with these settings:
   - **Operation**: Query
   - **Prompt**: "What shell am I running in? Run 'echo $SHELL' and show me the result"
   - **Model**: Sonnet 4.5
   - **Allowed Tools**: Select "Bash"
3. Execute the workflow

If successful, you should see the bash shell path in the output.

## Production Best Practices

1. **Use Docker Compose** for easier management
2. **Set ANTHROPIC_API_KEY** via environment variables or secrets
3. **Mount persistent volumes** for n8n data
4. **Mount project directories** that Claude Code needs to access
5. **Set resource limits** for the container
6. **Use health checks** to ensure n8n is running
7. **Configure logging** for debugging

Example production docker-compose.yml:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    environment:
      - SHELL=/bin/bash
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./projects:/projects
      - ./workflows:/workflows
    deploy:
      resources:
        limits:
          memory: 4G
        reservations:
          memory: 2G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
    command: >
      sh -c "
        apk add --no-cache bash curl git &&
        npm install -g @bilalmubarik/n8n-nodes-claudecode &&
        n8n start
      "

volumes:
  n8n_data:
    driver: local
```

## Questions?

If you continue to experience issues, please check:
- Container logs: `docker logs <container-id>`
- n8n execution logs in the UI
- Verify bash is installed: `docker exec <container-id> which bash`
- Verify SHELL variable: `docker exec <container-id> echo $SHELL`
