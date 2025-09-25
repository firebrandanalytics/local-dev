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
ff-cli examples list

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
ff-cli agent-bundle create smart-service

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

Deploy your agent bundles using a simplified 2-file configuration approach:

### Step 1: Prepare Configuration Files

Your generated agent bundle includes template configuration files:
- `values.local.yaml` - Pre-configured for local minikube deployment
- `secrets.yaml.template` - Template for sensitive values

```bash
# Navigate to your agent bundle directory
cd apps/my-service

# Copy and edit the secrets template
cp secrets.yaml.template secrets.yaml
# Edit secrets.yaml to add your database passwords and API keys
```

### Step 2: Build Your Agent Bundle

```bash
# Build the agent bundle Docker image in minikube's Docker daemon
eval $(minikube docker-env)  # Use minikube's Docker daemon
docker build --build-arg GITHUB_TOKEN=$GITHUB_TOKEN -t my-service:latest -f Dockerfile ../..
```

**Note**: The `--build-arg GITHUB_TOKEN=$GITHUB_TOKEN` is required to authenticate with the private FireFoundry npm packages during the build process. Make sure your GITHUB_TOKEN environment variable is set (see [FireFoundry CLI Setup](05-ff-cli-setup.md)).

### Step 3: Deploy Using Simplified Command

```bash
# Deploy your agent bundle with the 2-file configuration
helm install my-service firebrandanalytics/agent-bundle \
  -f values.local.yaml \
  -f secrets.yaml \
  --namespace ff-dev

# Verify deployment
kubectl -n ff-dev get pods | grep my-service
kubectl -n ff-dev logs deployment/my-service -f
```

### Step 4: Access Your Agent

```bash
# Set up port forwarding to access your agent
kubectl -n ff-dev port-forward svc/my-service 3001:3001 &

# Test your agent
curl http://localhost:3001/health
```

## Managing Multiple Agent Bundles

When developing multiple agent bundles, each should have its own configuration files:

```bash
# Each agent bundle has its own values and secrets
cd apps/analytics-service
helm install analytics-service firebrandanalytics/agent-bundle \
  -f values.local.yaml \
  -f secrets.yaml \
  --namespace ff-dev

cd ../notification-service
helm install notification-service firebrandanalytics/agent-bundle \
  -f values.local.yaml \
  -f secrets.yaml \
  --namespace ff-dev

# Set up port forwarding for each service (assuming default port 3001)
kubectl -n ff-dev port-forward svc/analytics-service 3002:3001 &
kubectl -n ff-dev port-forward svc/notification-service 3003:3001 &
```

### Useful Management Commands

```bash
# List all deployed agent bundles
helm list -n ff-dev

# Update an existing agent bundle after configuration changes
cd apps/my-service
helm upgrade my-service firebrandanalytics/agent-bundle \
  -f values.local.yaml \
  -f secrets.yaml \
  --namespace ff-dev

# Remove an agent bundle
helm uninstall my-service -n ff-dev

# Check all running services
kubectl -n ff-dev get pods,svc
```

## Configuration Guide

### Understanding the 2-File Approach

The simplified deployment uses two configuration files:

1. **values.local.yaml** - Non-sensitive configuration
   - Image settings (repository, tag, pull policy)
   - Service endpoints (FireFoundry services URLs)
   - Database connection info (server, database name)
   - Application settings (log levels, environment)

2. **secrets.yaml** - Sensitive values
   - Database passwords
   - API keys
   - Authentication tokens

### Adding Custom Environment Variables

If your agent needs additional environment variables:

```yaml
# In values.local.yaml - for non-sensitive values
configMap:
  data:
    MY_CUSTOM_VAR: "my-value"
    FEATURE_FLAG_X: "enabled"

# In secrets.yaml - for sensitive values
secret:
  data:
    MY_API_KEY: "secret-key-value"
    EXTERNAL_SERVICE_TOKEN: "token-value"
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
