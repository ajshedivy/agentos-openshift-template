# Local Deployment with Docker/Podman

Deploy the IBM i MCP Server stack locally using Docker Compose or Podman Compose for development and testing.

## Prerequisites

Choose one container platform:

### Docker
- **Docker Desktop** (macOS/Windows): [Download here](https://www.docker.com/products/docker-desktop/)
- **Docker Engine** (Linux): [Installation guide](https://docs.docker.com/engine/install/)

### Podman (Alternative)
- **Podman Desktop** (macOS/Windows): [Download here](https://podman-desktop.io/)
- **Podman CLI** (Linux): [Installation guide](https://podman.io/docs/installation)
- **podman-compose**: `pip install podman-compose`

## Quick Start

### 1. Build MCP Gateway Image

Clone and build the MCP Context Forge Gateway image:

```bash
git clone https://github.com/IBM/mcp-context-forge.git
cd mcp-context-forge

# Build with Docker
make docker-prod

# Or build with Podman
make podman-prod
```

This creates the local image `localhost/mcpgateway/mcpgateway` referenced in docker-compose.yml.

### 2. Configure Environment

Create a `.env` file in the template root (not in deployment/mcpgateway) with your IBM i connection details:

```bash
cd /path/to/agentos-openshift-template
cp .env.example .env
# Edit .env with your IBM i connection details
```

Required variables:

```ini
# IBM i connection details
DB2i_HOST="your_host"
DB2i_USER="your_user"
DB2i_PASS="your_pass"

# MCP Auth mode
MCP_AUTH_MODE=ibmi

# IBM i HTTP authentication settings
IBMI_AUTH_KEY_ID=development
IBMI_AUTH_PRIVATE_KEY_PATH=secrets/private.pem
IBMI_AUTH_PUBLIC_KEY_PATH=secrets/public.pem
IBMI_HTTP_AUTH_ENABLED=true
IBMI_AUTH_ALLOW_HTTP=true  # Development only - use false in production
```

**Note:** Generate RSA keypair if needed. See [IBM i MCP Server documentation](https://github.com/IBM/ibmi-mcp-server#ibm-i-http-authentication).

### 3. Start Services

**With Docker:**
```bash
docker-compose -f deployment/mcpgateway/docker-compose.yml up -d
```

**With Podman:**
```bash
podman-compose -f deployment/mcpgateway/docker-compose.yml up -d
```

### 4. Verify Services

```bash
# Docker
docker-compose -f deployment/mcpgateway/docker-compose.yml ps

# Podman
podman-compose -f deployment/mcpgateway/docker-compose.yml ps
```

## Files

| File | Purpose |
|------|---------|
| **docker-compose.yml** | Service definitions for gateway, MCP servers, databases, and admin UIs |

## Services

The compose stack includes:

| Service | Port | Description | Access URL |
|---------|------|-------------|------------|
| **gateway** | 4444 | MCP Context Forge main API and admin UI | http://localhost:4444 |
| **ibmi-mcp-server** | 3010 | IBM i SQL tools MCP server | http://localhost:3010 |
| **postgres** | - | PostgreSQL database (internal) | - |
| **redis** | 6379 | Cache service | redis://localhost:6379 |
| **pgadmin** | 5050 | Database admin UI | http://localhost:5050 |
| **redis_insight** | 5540 | Cache admin UI | http://localhost:5540 |

**Optional MCP servers** (uncomment in docker-compose.yml if needed):
- terraform-mcp-server (3020)
- fast-time-mcp-server (3030)

## Common Commands

### Service Management

```bash
# Start all services
docker-compose -f deployment/mcpgateway/docker-compose.yml up -d

# Start specific service
docker-compose -f deployment/mcpgateway/docker-compose.yml up -d gateway

# Stop all services
docker-compose -f deployment/mcpgateway/docker-compose.yml down

# Stop specific service
docker-compose -f deployment/mcpgateway/docker-compose.yml stop ibmi-mcp-server
```

### Logs

```bash
# Follow all logs
docker-compose -f deployment/mcpgateway/docker-compose.yml logs -f

# Follow specific service
docker-compose -f deployment/mcpgateway/docker-compose.yml logs -f gateway

# View last 100 lines
docker-compose -f deployment/mcpgateway/docker-compose.yml logs --tail=100 ibmi-mcp-server
```

### Rebuilds

```bash
# Rebuild specific service
docker-compose -f deployment/mcpgateway/docker-compose.yml build ibmi-mcp-server

# Rebuild and restart all
docker-compose -f deployment/mcpgateway/docker-compose.yml up --build -d
```

**Note:** Replace `docker-compose` with `podman-compose` if using Podman.

## Gateway Admin UI

Access the MCP Context Forge UI at **http://localhost:4444**

**Default Login:**
- User: `admin`
- Password: `changeme`

**Configure IBM i MCP Server:**
1. Navigate to **"Gateways/MCP Servers"** tab
2. Add server endpoint: `http://ibmi-mcp-server:3010`
3. Click "Connect" to register

Once connected, you can:
- View all IBM i SQL tools
- Create virtual servers grouping specific tools
- Configure tool-level authentication
- Monitor tool usage and performance
- Set up rate limiting per tool

## Troubleshooting

### Container won't start

```bash
# Check logs
docker logs ibmi-mcp-server

# Verify environment variables
docker exec ibmi-mcp-server env | grep DB2i

# Test connectivity
docker exec ibmi-mcp-server curl http://localhost:3010/healthz
```

### Port conflicts

Edit `docker-compose.yml` to change port mappings:

```yaml
services:
  gateway:
    ports:
      - "4445:4444"  # Change host port from 4444 to 4445
```

### Gateway can't connect to IBM i MCP Server

```bash
# Verify both containers on same network
docker network inspect mcpgateway_mcpnet

# Check IBM i MCP Server health
docker exec ibmi-mcp-server curl http://localhost:3010/healthz

# Review gateway logs
docker logs gateway
```

### Database connection issues

```bash
# Verify IBM i credentials in .env
cat .env | grep DB2i

# Check Mapepire service on IBM i
# SSH to IBM i and run:
sc status mapepire

# Test connectivity from container
docker exec ibmi-mcp-server nc -zv $DB2i_HOST 8076
```

## Next Steps

- **Production deployment:** See [../openshift/apps/openshift/README.md](../openshift/apps/openshift/README.md)
- **MCP Gateway documentation:** https://ibm.github.io/mcp-context-forge/
- **IBM i MCP Server:** https://github.com/IBM/ibmi-mcp-server
- **Main README:** [../../README.md](../../README.md)
