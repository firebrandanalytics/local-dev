# Agent Bundle Development

Learn how to create, develop, and deploy AI agent bundles with FireFoundry.

## Why Agent Bundles?

Traditional AI applications are monolithic and difficult to maintain. Agent bundles provide:

- **Modularity**: Each agent handles specific domain responsibilities
- **Reusability**: Agent bundles can be composed into larger systems
- **Testability**: Individual agents can be tested in isolation
- **Maintainability**: Clear boundaries make debugging and updates easier

## Creating Your First Agent Bundle

### Option 1: Explore Examples First (Recommended)

```bash
# Discover available examples
ff-cli project create --list-examples

# Create project with a working example
ff-cli project create my-smart-agent --with-example=talespring
cd my-smart-agent

# Install dependencies and build
pnpm install && turbo build
```

### Option 2: Create Empty Project

```bash
# Create empty monorepo
ff-cli project create my-smart-agent
cd my-smart-agent

# Generate your first agent bundle
ff-cli agent-bundle create smart-service --description "My smart agent service" --port 3001

# Install dependencies and build
pnpm install && turbo build
```

### Option 3: Create with Initial Agent Bundle

```bash
# Create project with first empty agent bundle
ff-cli project create my-smart-agent smart-service
cd my-smart-agent

# Install dependencies and build
pnpm install && turbo build
```

## What You Get

- **Complete monorepo setup** with Turborepo, pnpm, and TypeScript
- **Working examples** (when using `--with-example`) demonstrating Entity-Bot-Prompt patterns
  - **talespring**: Creative writing and storytelling agent
  - **explain-analyze**: SQL query analysis and optimization agent
- **Docker configuration** for development and production builds
- **Agent SDK integration** with comprehensive documentation
- **Build system** with intelligent caching and dependency management
- **Cursor AI integration** for enhanced development experience

## Scaling with Additional Agent Bundles

Once your project is set up, you can easily add more agent bundles:

```bash
# Generate additional specialized agent bundles
ff-cli agent-bundle create analytics-service --description "Analytics and metrics service" --port 3002
ff-cli agent-bundle create notification-service --description "User notification service" --port 3003
ff-cli agent-bundle create auth-service --description "Authentication service" --port 3004

# Each agent bundle is:
# - Fully independent with its own port and configuration
# - Ready for Docker containerization
# - Kubernetes deployment ready
# - Integrated with the monorepo build system
```

## Deployment Process

Deploy your agent bundles to the local FireFoundry environment using the published Helm charts:

### Step 1: Build Your Agent Bundle

```bash
# Navigate to your agent project
cd my-ff-project

# Build the agent bundle Docker image
eval $(minikube docker-env)  # Use minikube's Docker daemon
docker build -t my-service:latest -f apps/my-service/Dockerfile .
```

### Step 2: Deploy Using Published Chart

```bash
# Deploy your agent bundle using the published chart
helm install my-service firebrandanalytics/agent-bundle \
  --set image.repository=my-service \
  --set image.tag=latest \
  --set service.port=3000 \
  --set env.BROKER_SERVICE_URL="http://ff-broker.ff-dev.svc.cluster.local:50061" \
  --set env.CONTEXT_SERVICE_URL="http://context-service.ff-dev.svc.cluster.local:50051" \
  --namespace ff-dev

# Verify deployment
kubectl -n ff-dev get pods | grep my-service
kubectl -n ff-dev logs deployment/my-service -f
```

### Step 3: Access Your Agent

```bash
# Set up port forwarding to access your agent
kubectl -n ff-dev port-forward svc/my-service 3001:3000 &

# Test your agent
curl http://localhost:3001/health
```

## Managing Multiple Agent Bundles

When developing multiple agent bundles, you can deploy them all to the same namespace:

```bash
# Deploy multiple agent bundles with different ports
helm install analytics-service firebrandanalytics/agent-bundle \
  --set image.repository=analytics-service \
  --set image.tag=latest \
  --set service.port=3000 \
  --namespace ff-dev

helm install notification-service firebrandanalytics/agent-bundle \
  --set image.repository=notification-service \
  --set image.tag=latest \
  --set service.port=3000 \
  --namespace ff-dev

# Set up port forwarding for each service
kubectl -n ff-dev port-forward svc/analytics-service 3002:3000 &
kubectl -n ff-dev port-forward svc/notification-service 3003:3000 &
```

### Useful Management Commands

```bash
# List all deployed agent bundles
helm list -n ff-dev

# Update an existing agent bundle
helm upgrade my-service firebrandanalytics/agent-bundle \
  --set image.repository=my-service \
  --set image.tag=v2.0 \
  --namespace ff-dev

# Remove an agent bundle
helm uninstall my-service -n ff-dev

# Check all running services
kubectl -n ff-dev get pods,svc
```

## Troubleshooting Agent Deployment

### Image Pull Issues

```bash
# If your image isn't found, ensure it's built in minikube's Docker
eval $(minikube docker-env)
docker images | grep my-service

# Rebuild if necessary
docker build -t my-service:latest -f apps/my-service/Dockerfile .
```

### Service Connection Issues

```bash
# Check if FireFoundry core services are running
kubectl -n ff-dev get pods | grep -E "(ff-broker|context-service|code-sandbox)"

# Test connectivity from your agent pod
kubectl -n ff-dev exec deployment/my-service -- curl -v http://ff-broker.ff-dev.svc.cluster.local:50061
```

## Example-Driven Development Workflow

The recommended development approach leverages working examples:

```bash
# 1. Explore available examples and their capabilities
ff-cli project create --list-examples

# 2. Create project with the most relevant example
ff-cli project create my-domain-app --with-example=talespring

# 3. Study the example implementation
cd my-domain-app
# - Review apps/talespring/ for Entity-Bot-Prompt patterns
# - Examine the entity definitions and bot implementations
# - Understand prompt composition techniques

# 4. Customize the example or create new agent bundles
ff-cli agent-bundle create my-custom-service --description "My domain-specific service" --port 3002

# 5. Develop and test
pnpm install && pnpm dev
```

## AI-Enhanced Development

FireFoundry includes AI pair programming support through Cursor integration:

```bash
# The .cursorrules file provides context to AI assistants about:
# - FireFoundry architecture patterns
# - Agent SDK usage examples
# - Common troubleshooting steps
# - Best practices for agent development
```

## Next Steps

Now that you can create agent bundles, learn about:

1. **[Operations & Maintenance](07-operations.md)** - Monitor and maintain your deployments
2. **[Troubleshooting](08-troubleshooting.md)** - Solve common issues
