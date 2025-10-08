# Deploying the Talespring Example

This guide walks you through deploying and testing the **talespring** example agent bundle - a creative storytelling AI that demonstrates FireFoundry's Entity-Bot-Prompt architecture.

## Prerequisites

Before starting, ensure you've completed:

1. **[Prerequisites](01-prerequisites.md)** - Core tools installed
2. **[Environment Setup](03-environment-setup.md)** - minikube cluster running
3. **[Deploy Services](04-deployment.md)** - Control plane and core services deployed
4. **[FF CLI Setup](05-ff-cli-setup.md)** - FireFoundry CLI installed

## Step 1: Create Project with Talespring Example

First, list available examples to verify talespring is accessible:

```bash
# List available examples
ff-cli examples list
```

You should see talespring along with other examples. Now create a new project:

```bash
# Create project with talespring example
cd /tmp  # Or your preferred workspace directory
ff-cli project create talespring-demo --with-example=talespring
cd talespring-demo
```

**What you get**:
- Complete monorepo structure with Turborepo and pnpm
- Working talespring agent in `apps/talespring/`
- Pre-configured Dockerfile for containerization
- Template configuration files for deployment

## Step 2: Install Dependencies and Build

```bash
# Install all dependencies
pnpm install

# Build the project
pnpm run build
```

This compiles TypeScript and prepares the agent bundle for deployment.

## Step 3: Build Docker Image

Build the Docker image using minikube's Docker daemon (so Kubernetes can access it):

```bash
# Switch to minikube's Docker environment
eval $(minikube -p ff-local-dev docker-env)

# Build the Docker image from project root
docker build \
  --build-arg GITHUB_TOKEN=$GITHUB_TOKEN \
  -t talespring:latest \
  -f apps/talespring/Dockerfile \
  .
```

**Note**: The `GITHUB_TOKEN` build arg is required to access private FireFoundry packages during the build. This uses the token you configured in [FF CLI Setup](05-ff-cli-setup.md).

Verify the image was built:

```bash
docker images | grep talespring
```

## Step 4: Prepare Configuration Files

Navigate to the talespring directory and prepare configuration:

```bash
cd apps/talespring

# Copy the secrets template
cp secrets.yaml.template secrets.yaml
```

**For local development**, the default secrets template works as-is since we're connecting to shared Firebrand services. For production deployments, you would edit `secrets.yaml` with actual credentials.

Now create the local values file. The talespring example includes a `values.local.yaml.template` file. Copy and customize it:

```bash
# Copy the values template
cp values.local.yaml.template values.local.yaml
```

Edit `values.local.yaml` to configure for local minikube deployment. Update these key sections:

```yaml
# Local minikube values for talespring
bundleName: "talespring"  # IMPORTANT: Must match your service name

# Image configuration
image:
  repository: talespring
  tag: "latest"
  pullPolicy: Never  # Use local Docker image

# Service configuration
service:
  type: ClusterIP
  http:
    enabled: true
    port: 3001
    targetPort: 3001

# ... rest of config remains the same
```

**Critical**: The `bundleName` field determines the Kong route path (`/agents/ff-dev/talespring`). If omitted, it defaults to `my-bundle` which will create the wrong route.

## Step 5: Deploy to Kubernetes

Deploy talespring using the FireFoundry agent-bundle Helm chart:

```bash
# Deploy from the apps/talespring directory
helm install talespring firebrandanalytics/agent-bundle \
  -f values.local.yaml \
  -f secrets.yaml \
  --namespace ff-dev
```

**Verify deployment**:

```bash
# Check pod status
kubectl get pods -n ff-dev | grep talespring

# Watch pod startup (wait until STATUS shows Running)
kubectl get pods -n ff-dev -w
```

The pod should transition to `Running` status within 30-60 seconds.

## Step 6: Verify Kong Route Registration

The agent bundle controller automatically discovers your service and creates a Kong route. Verify the route was created:

```bash
# Port-forward Kong Admin API (if not already running)
kubectl port-forward -n ff-control-plane svc/firefoundry-control-kong-admin 8001:8001 &

# Check Kong routes
curl -s http://localhost:8001/routes | jq '.data[] | {name, paths}'
```

You should see a route named `ff-agent-ff-dev-talespring-route` with path `/agents/ff-dev/talespring`.

**If the route shows wrong path** (like `/agents/ff-dev/my-bundle`):
1. You forgot to set `bundleName` in values.local.yaml
2. Delete the wrong Kong route: `curl -X DELETE http://localhost:8001/routes/<wrong-route-name>`
3. Update values.local.yaml with `bundleName: "talespring"`
4. Upgrade the Helm release: `helm upgrade talespring firebrandanalytics/agent-bundle -f values.local.yaml -f secrets.yaml --namespace ff-dev`
5. Restart the agent controller: `kubectl delete pod -n ff-control-plane -l app.kubernetes.io/component=agent-bundle-controller`

## Step 7: Test Talespring Through Kong

Set up port-forwarding to access Kong's proxy (the API gateway):

```bash
# Port-forward Kong proxy to localhost (run in separate terminal or background)
kubectl port-forward -n ff-control-plane svc/firefoundry-control-kong-proxy 8080:80
```

**Note for macOS users**: Direct NodePort access doesn't work with Docker Desktop's minikube. You must use port-forwarding to access services.

Now test talespring endpoints:

```bash
# Health check
curl http://localhost:8080/agents/ff-dev/talespring/health/ready

# Expected: {"status":"healthy","timestamp":"..."}

# Service info
curl http://localhost:8080/agents/ff-dev/talespring/info

# Expected: {"app_id":"...","app_name":"ChildrensStories",...}
```

## Step 8: Test with Postman

Use Postman to test the talespring agent's story generation capabilities:

**Base URL**: `http://localhost:8080/agents/ff-dev/talespring`

**Available Endpoints**:

1. **GET** `/health/ready` - Health check
2. **GET** `/info` - Service metadata
3. **POST** `/invoke` - Execute agent methods

**Example Request - Generate Story**:

```http
POST http://localhost:8080/agents/ff-dev/talespring/invoke
Content-Type: application/json

{
  "entityName": "EntityTaleSpring",
  "method": "generateStory",
  "args": {
    "theme": "adventure",
    "ageGroup": "8-10"
  }
}
```

The agent will generate a creative story based on the theme and age group, demonstrating FireFoundry's LLM integration and prompt engineering patterns.

## Troubleshooting

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n ff-dev <pod-name>

# Check logs
kubectl logs -n ff-dev <pod-name>
```

Common issues:
- **ImagePullBackOff**: Image not found - rebuild with `eval $(minikube docker-env)` first
- **CrashLoopBackOff**: Configuration error - check secrets.yaml and values.local.yaml
- **Pending**: Insufficient resources - check `minikube status` and resource allocation

### Route Not Created

```bash
# Check agent controller logs
kubectl logs -n ff-control-plane -l app.kubernetes.io/component=agent-bundle-controller --tail=100
```

Common causes:
- Service doesn't have required labels (chart should add these automatically)
- Agent controller not running - check `kubectl get pods -n ff-control-plane`
- Service discovery lag - wait 10-20 seconds after deployment

### Authentication Errors

If you see `"No API key found in request"`:

1. Authentication is enabled in your control plane configuration
2. For local development, authentication should be disabled
3. Check `~/dev/ff-configs/environments/dev/control-plane-values.yaml`:
   ```yaml
   agentBundleController:
     authentication:
       enabled: false  # Should be false for local dev
   ```
4. If changed, upgrade control plane: `helm upgrade firefoundry-control ...`
5. Restart agent controller to apply changes

## What You've Accomplished

You've successfully:

- Created a FireFoundry project with the talespring example
- Built and containerized an agent bundle
- Deployed to Kubernetes using Helm
- Configured automatic Kong route registration
- Tested the agent through the API gateway

**Next Steps**:
- **[Update Agent Bundles](07-updating-agent-bundles.md)** - Make changes and redeploy
- Explore the talespring source code in `apps/talespring/src/` to understand Entity-Bot-Prompt patterns
- Review entity definitions, bot implementations, and prompt composition
- Try creating your own agent bundle: `ff-cli agent-bundle create my-service`
- Learn about monitoring and operations: **[Operations Guide](08-operations.md)**

## Additional Resources

- **Entity-Bot-Prompt Architecture**: See `~/dev/CLAUDE.md` for framework overview
- **Agent SDK Documentation**: Available in generated project's README
- **Example Agents**: Use `ff-cli examples list` to discover more examples
- **Troubleshooting Guide**: **[Troubleshooting](08-troubleshooting.md)** for common issues