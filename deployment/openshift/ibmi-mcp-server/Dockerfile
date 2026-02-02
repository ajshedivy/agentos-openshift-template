# IBM i MCP Server Dockerfile
# Runs @ibm/ibmi-mcp-server from npm using npx

FROM node:20-slim

LABEL name="ibmi-mcp-server" \
      description="IBM i Model Context Protocol Server" \
      maintainer="IBM i MCP Team" \
      version="latest"

# Set production environment
ENV NODE_ENV=production

# Run as non-root user for security
USER node

# Set working directory
WORKDIR /app

# MCP server configuration
ENV MCP_HTTP_PORT=3010
ENV MCP_TRANSPORT_TYPE=http

# Expose MCP server port
EXPOSE 3010

# Health check to verify server is running
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3010/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) }).on('error', () => process.exit(1))"

# Run MCP server using npx (fetches latest from npm)
CMD ["npx", "-y", "@ibm/ibmi-mcp-server@latest"]
