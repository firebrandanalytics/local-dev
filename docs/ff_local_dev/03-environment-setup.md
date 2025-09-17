# Local Development Environment Setup

Configure your local development environment to work with FireFoundry.

## Environment Configuration

**Before starting minikube, ensure Docker Desktop is running** (you should have verified this in the Prerequisites step):

```bash
# Verify Docker is still running from Prerequisites step
docker ps

# If Docker isn't running, start Docker Desktop:
# macOS: open -a Docker
# Windows: Launch Docker Desktop from Start menu

# Configure minikube with adequate resources
minikube start --memory=8192 --cpus=4 --disk-size=20g

# Enable essential addons
minikube addons enable ingress      # For HTTP routing to services
minikube addons enable metrics-server  # For resource monitoring and autoscaling

```

**Why these configurations?**

- **Memory/CPU allocation**: Agent services can be resource-intensive, especially during compilation and execution
- **Ingress addon**: Enables testing of HTTP-based agent interactions locally
- **Metrics server**: Allows you to monitor resource usage and debug performance issues

## Verification

Verify your environment is ready:

```bash
# Check minikube status
minikube status

# Verify addons are enabled
minikube addons list

# Test Docker is running
docker ps

# Verify kubectl can connect
kubectl get nodes
```

## Next Steps

With your environment configured, you're ready to:

1. **[Deploy FireFoundry Services](04-deployment.md)** - Get the infrastructure running
2. **[Start Agent Development](05-agent-development.md)** - Create your first agent bundle
