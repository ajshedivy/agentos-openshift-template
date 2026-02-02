# OpenShift Deployment

This directory contains Kubernetes/OpenShift manifests for deploying IBM i MCP Server and Agents using Kustomize.

## Overview

The applications are deployed on OpenShift using a source-to-image (S2I) build strategy. With this strategy, the application can be built from source in a remote repository or local development environment. Deployments are configured with an ImageStreamTag trigger; any new image build will trigger the application re-deployment.

## Prerequisites

1. OpenShift cluster access
2. `oc` CLI tool installed
3. Kustomize installed
4. A `.env` file in the `ibmi-mcp-server` and `ibmi-agent-infra/agent-os-api` directories with required environment variables
5. Enable Red Hat OpenShift internal image registry following the instructions in this [link](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/setting-up-and-configuring-the-registry)

## Deployment Instructions

1. **Copy required files**

   Run the commands below from the **project root directory**.

   ```bash
   # Copy tools directory into the MCP server deployment directory
   cp -r tools deployment/openshift/ibmi-mcp-server/

   # Create .env files from examples (edit with your actual values)
   cp deployment/openshift/ibmi-mcp-server/.env.example deployment/openshift/ibmi-mcp-server/.env
   cp deployment/openshift/ibmi-agent-infra/agent-os-api/.env.example deployment/openshift/ibmi-agent-infra/agent-os-api/.env
   ```

   Edit both `.env` files with your IBM i connection details and API keys.

2. **Set your namespace**

   Replace `<NAMESPACE_PLACEHOLDER>` in the root `kustomization.yaml` with your actual OpenShift namespace.

   Switch to your OpenShift namespace by running command:

   ```bash
   oc project <your_namespace>
   ```

3. **Deploy using Kustomize**:

   ```bash
   kustomize build . | oc apply -f -
   ```

4. **Monitor the image build**:

   ```bash
   oc logs -f bc/ibmi-mcp-server
   oc logs -f bc/agent-os-api
   oc logs -f bc/pgvector
   ```

5. **Check deployment status**:

   ```bash
   oc get pods
   ```

6. **Get the URL for each application**

   ```bash
   echo "https://$(oc get route ibmi-mcp-server -o jsonpath='{.spec.host}')"
   echo "https://$(oc get route agent-ui -o jsonpath='{.spec.host}')"
   ```

6. **Trigger the build manually**:

   ```bash
   # Trigger a new build using the source from remote repo
   oc start-build ibmi-mcp-server
   oc start-build agent-os-api
   oc start-build pgvector
   # Trigger a new build using the source from local
   oc start-build ibmi-mcp-server --from-dir=.
   oc start-build agent-os-api --from-dir=.
   oc start-build pgvector --from-dir=.
   ```

## Troubleshooting

- **Build failures**: Check build logs with `oc logs -f bc/ibmi-mcp-server`
- **Pod crashes**: Check pod logs with `oc logs <pod-name>`
- **Storage issues**: Verify PVC is bound with `oc get pvc`
- **Access issues**: Verify route with `oc get route ibmi-mcp-server`
