# Deployment Guide

Deploy the IBM i Agent Platform (AgentOS) to OpenShift.

## Table of Contents

- [Directory Structure](#directory-structure)
- [Core Services](#core-services)
- [OpenShift Deployment](#openshift-deployment)
- [Quick Reference](#quick-reference)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)

---

## Directory Structure

| Directory | Purpose | Documentation |
|-----------|---------|---------------|
| **openshift/** | Production OpenShift deployment using Kustomize | [openshift/apps/openshift/README.md](openshift/apps/openshift/README.md) |

### OpenShift Components

The OpenShift deployment includes these application components:

| Component | Location | Purpose |
|-----------|----------|---------|
| **ibmi-mcp-server** | `openshift/apps/openshift/ibmi-mcp-server/` | IBM i MCP Server with SQL tools |
| **ibmi-agent-infra** | `openshift/apps/openshift/ibmi-agent-infra/` | AgentOS API, UI, and pgvector database |

Each component has its own README with detailed setup instructions.

---

## Core Services

This template deploys the following core services:

### AgentOS API
The backend service that orchestrates AI agents and manages conversations. Located in `ibmi-agent-infra/agent-os-api/`.

### AgentOS UI
The web interface for interacting with IBM i agents. Located in `ibmi-agent-infra/agent-os-ui/`.

### IBM i MCP Server
Provides SQL tools and database access to IBM i systems via the Model Context Protocol (MCP). Located in `ibmi-mcp-server/`.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      AgentOS UI                             │
│                   (Web Interface)                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      AgentOS API                            │
│              (Agent Orchestration Service)                  │
│   • Conversation management                                 │
│   • Agent execution                                         │
│   • Tool routing                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  IBM i MCP Server                           │
│                    (Port 3010)                              │
│   • SQL tools execution                                     │
│   • IBM i authentication                                    │
│   • YAML tool configuration                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                  ┌────────────┐
                  │ IBM i Db2  │
                  │ (Mapepire) │
                  └────────────┘
```

---

## OpenShift Deployment

Deploy using Kustomize with source-to-image (S2I) builds.

**Full OpenShift Guide:** [`openshift/apps/openshift/README.md`](openshift/apps/openshift/README.md)

### Quick Start

```bash
cd deployment/openshift/apps/openshift

# 1. Set your namespace in kustomization.yaml
#    Replace <NAMESPACE_PLACEHOLDER> with your OpenShift project name

# 2. Configure environment files
cp ibmi-mcp-server/.env.example ibmi-mcp-server/.env
# Edit .env with your IBM i connection details

# 3. Deploy all applications
kustomize build . | oc apply -f -

# 4. Monitor builds
oc logs -f bc/ibmi-mcp-server

# 5. Get routes
echo "IBM i MCP Server: https://$(oc get route ibmi-mcp-server -o jsonpath='{.spec.host}')"
echo "AgentOS API: https://$(oc get route agent-os-api -o jsonpath='{.spec.host}')"
echo "AgentOS UI: https://$(oc get route agent-os-ui -o jsonpath='{.spec.host}')"
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Deploy to OpenShift | `cd deployment/openshift/apps/openshift && kustomize build . \| oc apply -f -` |
| Check pod status | `oc get pods` |
| View MCP server logs | `oc logs -f deployment/ibmi-mcp-server` |
| View API logs | `oc logs -f deployment/agent-os-api` |
| Trigger rebuild | `oc start-build ibmi-mcp-server` |

---

## Troubleshooting

### Build Failures

```bash
# Check build logs
oc logs -f bc/ibmi-mcp-server

# Check build status
oc get builds

# Restart a failed build
oc start-build ibmi-mcp-server
```

### Pod Issues

```bash
# Check pod status
oc get pods

# View pod logs
oc logs deployment/ibmi-mcp-server

# Describe pod for events
oc describe pod -l app=ibmi-mcp-server
```

### Connection Issues

```bash
# Test health endpoint
curl https://$(oc get route ibmi-mcp-server -o jsonpath='{.spec.host}')/healthz

# Check IBM i connectivity from pod
oc exec deployment/ibmi-mcp-server -- nc -zv $DB2_HOST 8076
```

### Common Issues

| Issue | Solution |
|-------|----------|
| ImagePullBackOff | Check BuildConfig completed successfully |
| CrashLoopBackOff | Check logs for configuration errors, verify .env values |
| Connection refused | Verify IBM i credentials and Mapepire is running |
| 404 on route | Wait for pod to become ready, check service selectors |

See the [OpenShift README](openshift/apps/openshift/README.md#troubleshooting) for more details.

---

## Production Considerations

### Security

- **Use HTTPS:** OpenShift routes provide TLS termination by default
- **Secrets management:** Use OpenShift secrets for credentials
- **Network policies:** Restrict pod-to-pod communication
- **RBAC:** Configure appropriate service accounts and roles

### Performance

- **Resource limits:** Set appropriate CPU/memory limits in deployments
- **Horizontal scaling:** Configure replica counts based on load
- **Database connections:** Tune connection pool settings

### Monitoring

- **Health checks:** All services have built-in health endpoints
- **Log aggregation:** Use OpenShift logging or external solutions
- **Metrics:** Monitor pod resource usage via OpenShift console

---

## Resources

- **IBM i MCP Server:** https://github.com/IBM/ibmi-mcp-server
- **Agno Framework:** https://docs.agno.com
- [**Main README**](../README.md)
