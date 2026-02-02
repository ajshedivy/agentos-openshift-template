# IBM i MCP Server Component

OpenShift deployment component for the IBM i MCP Server, providing SQL tools and IBM i database access through the Model Context Protocol.

## Files

This directory contains the Kubernetes manifests for deploying IBM i MCP Server to OpenShift:

| File | Purpose |
|------|---------|
| **kustomization.yaml** | Kustomize configuration that orchestrates all resources, generates ConfigMaps and Secrets |
| **ibmi-mcp-server-imagestream.yaml** | ImageStream for tracking built container images |
| **ibmi-mcp-server-buildconfig.yaml** | BuildConfig for S2I (source-to-image) builds from Git repository |
| **ibmi-mcp-server-deployment.yaml** | Deployment manifest with container spec, health checks, and volume mounts |
| **ibmi-mcp-server-service.yaml** | Service exposing the MCP server internally on port 3010 |
| **ibmi-mcp-server-route.yaml** | Route for external HTTPS access to the MCP server |
| **Dockerfile** | Container image definition for building the IBM i MCP Server |
| **.env** | Environment variables for IBM i connection and authentication (create from .env.example) |
| **tools/** | Directory containing YAML tool configurations mounted as ConfigMap |
| **secrets/** | Directory containing RSA keypair for IBM i authentication (create before deployment) |

## Setup Instructions

### 1. Prepare Environment Configuration

Create the `.env` file with your IBM i connection details:

```bash
# Get the example file
wget -O .env https://raw.githubusercontent.com/IBM/ibmi-mcp-server/refs/heads/main/.env.example

# Edit with your values
vim .env
```

Required variables:

```ini
# IBM i Connection
DB2i_HOST=your-ibmi-hostname
DB2i_USER=your-username
DB2i_PASS=your-password

# MCP Server Configuration
MCP_HTTP_PORT=3010
MCP_AUTH_MODE=ibmi

# IBM i HTTP Authentication
IBMI_AUTH_KEY_ID=production
IBMI_AUTH_PRIVATE_KEY_PATH=secrets/private.pem
IBMI_AUTH_PUBLIC_KEY_PATH=secrets/public.pem
IBMI_HTTP_AUTH_ENABLED=true
IBMI_AUTH_ALLOW_HTTP=false  # Must be false in production
```

### 2. Copy Tools Directory

The server needs tool configuration files from the template root:

```bash
# From template root directory
cp -r tools deployment/openshift/ibmi-mcp-server/
```

This copies all YAML tool definitions that will be mounted as a ConfigMap.

### 3. Generate Authentication Keys

Create RSA keypair for IBM i HTTP authentication:

```bash
mkdir -p secrets
ssh-keygen -t rsa -b 4096 -m PEM -f secrets/private.pem -N ""
openssl rsa -in secrets/private.pem -pubout -outform PEM -out secrets/public.pem
```

The keys are mounted as a Secret with restricted permissions (mode 0400).

### 4. Update Namespace

Edit `kustomization.yaml` if deploying to a namespace other than the current context:

```yaml
namespace: your-namespace
```

Kustomize will automatically replace `NAMESPACE_PLACEHOLDER` in the manifests.

### 5. Deploy

From this directory:

```bash
kustomize build . | oc apply -f -
```

Or deploy the entire stack from the parent directory:

```bash
cd ../
kustomize build . | oc apply -f -
```

## Configuration Variables

Key environment variables configured in `.env`:

| Variable | Purpose | Example |
|----------|---------|---------|
| **DB2i_HOST** | IBM i hostname or IP | `my-ibmi.example.com` |
| **DB2i_USER** | IBM i user profile | `MCPUSER` |
| **DB2i_PASS** | IBM i password | `MySecurePassword` |
| **MCP_HTTP_PORT** | HTTP server port | `3010` |
| **MCP_AUTH_MODE** | Authentication mode | `ibmi`, `jwt`, or `none` |
| **IBMI_AUTH_KEY_ID** | Key identifier for JWT | `production` |
| **IBMI_AUTH_PRIVATE_KEY_PATH** | Path to private key | `secrets/private.pem` |
| **IBMI_AUTH_PUBLIC_KEY_PATH** | Path to public key | `secrets/public.pem` |
| **IBMI_HTTP_AUTH_ENABLED** | Enable HTTP auth endpoints | `true` or `false` |
| **IBMI_AUTH_ALLOW_HTTP** | Allow non-HTTPS requests | `false` (production) |

See [IBM i MCP Server documentation](https://github.com/IBM/ibmi-mcp-server) for complete configuration reference.

## Build and Verification

### Trigger Build

```bash
# Build from Git repository
oc start-build ibmi-mcp-server

# Build from local directory (for testing)
oc start-build ibmi-mcp-server --from-dir=.
```

### Monitor Build

```bash
# Follow build logs
oc logs -f bc/ibmi-mcp-server

# Check build status
oc get builds | grep ibmi-mcp-server
```

### Verify Deployment

```bash
# Check pod status
oc get pods -l app=ibmi-mcp-server

# View pod logs
oc logs -f deployment/ibmi-mcp-server

# Test health endpoint
oc exec deployment/ibmi-mcp-server -- curl http://localhost:3010/healthz
```

### Get Route URL

```bash
echo "https://$(oc get route ibmi-mcp-server -o jsonpath='{.spec.host}')"
```

## Architecture

The deployment creates these resources:

```
┌─────────────────────────────────────────────────┐
│ Route (HTTPS)                                   │
│ https://ibmi-mcp-server-{namespace}.apps...     │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ Service: ibmi-mcp-server                        │
│ Port: 3010                                      │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ Deployment: ibmi-mcp-server                     │
│ • Image: Built via S2I from Git                 │
│ • ConfigMap: Environment variables from .env    │
│ • ConfigMap: Tool definitions from tools/       │
│ • Secret: RSA keypair from secrets/             │
│ • Health checks: /healthz on port 3010          │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ ImageStream: ibmi-mcp-server                    │
│ • Triggers automatic redeployment on new build  │
└─────────────────────────────────────────────────┘
```

## Troubleshooting

### Build fails

```bash
# Check build logs for errors
oc logs -f bc/ibmi-mcp-server

# Common issues:
# - Git repository not accessible
# - Dockerfile syntax errors
# - Missing base image
```

### Pod fails to start

```bash
# Check pod events
oc describe pod -l app=ibmi-mcp-server

# Check pod logs
oc logs -l app=ibmi-mcp-server

# Common issues:
# - Missing .env ConfigMap (create .env file first)
# - Missing secrets (generate RSA keypair first)
# - Missing tools ConfigMap (copy tools/ directory first)
```

### Health check failures

```bash
# Check readiness probe
oc get pods -l app=ibmi-mcp-server -o json | grep -A 5 readinessProbe

# Test health endpoint from inside pod
oc exec deployment/ibmi-mcp-server -- curl -v http://localhost:3010/healthz

# Common issues:
# - Port 3010 not accessible
# - Application crashed during startup
# - Missing required environment variables
```

### Connection issues to IBM i

```bash
# Verify environment variables are set
oc exec deployment/ibmi-mcp-server -- env | grep DB2i

# Test network connectivity to IBM i
oc exec deployment/ibmi-mcp-server -- nc -zv $DB2i_HOST 8076

# Check application logs for connection errors
oc logs -f deployment/ibmi-mcp-server | grep -i error

# Common issues:
# - IBM i hostname not resolvable
# - Mapepire service not running on IBM i
# - Firewall blocking port 8076
# - Invalid credentials in .env
```

## Security Considerations

### Production Checklist

- [ ] Set `IBMI_AUTH_ALLOW_HTTP=false` in `.env`
- [ ] Use strong passwords for `DB2i_PASS`
- [ ] Rotate RSA keypair regularly
- [ ] Restrict Route to specific domains if possible
- [ ] Enable OpenShift network policies
- [ ] Use OpenShift Secrets instead of ConfigMap for sensitive data
- [ ] Set resource limits in deployment.yaml
- [ ] Enable audit logging

### Secrets Management

The deployment uses two mechanisms for sensitive data:

1. **ConfigMap** (ibmi-mcp-server-env): Contains `.env` variables
   - Visible in OpenShift console
   - Consider using Sealed Secrets or External Secrets Operator

2. **Secret** (ibmi-mcp-secrets): Contains RSA keypair
   - Mounted read-only with mode 0400
   - Not visible in logs or console

For enhanced security, migrate all sensitive variables from ConfigMap to Secret.

## Related Documentation

- **Parent OpenShift README:** [../README.md](../README.md)
- **Main Deployment Guide:** [../../README.md](../../README.md)
- **IBM i MCP Server:** https://github.com/IBM/ibmi-mcp-server
- **MCP Context Forge:** https://github.com/IBM/mcp-context-forge
- **Kustomize:** https://kustomize.io/
