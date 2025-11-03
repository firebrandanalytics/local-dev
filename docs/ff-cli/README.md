# ff-cli Documentation

The `ff-cli` tool provides command-line interfaces for managing FireFoundry projects, including Docker image builds, Kubernetes deployments, and configuration profiles.

## Documentation

- **[Profile Management](profiles.md)** - Manage Docker registry authentication profiles for different environments
- **[Operations Commands](ops.md)** - Build, install, and upgrade agent bundles using Docker and Helm

## Quick Start

### Profiles

Profiles store Docker registry authentication settings, allowing you to switch between different environments (local minikube, GCP, Azure, etc.) without manually specifying credentials each time.

```bash
# Create a profile
ff-cli profile create my-profile

# List all profiles
ff-cli profile list

# Set current profile
ff-cli profile select my-profile
```

### Operations

Build and deploy agent bundles:

```bash
# Build a Docker image (uses current profile)
ff-cli ops build my-bundle --tag 1.0.0

# Install to Kubernetes
ff-cli ops install my-bundle

# Upgrade an existing deployment
ff-cli ops upgrade my-bundle
```

## Integration

Profiles and operations work together seamlessly. When you run `ff-cli ops build`, it will automatically use your current profile for registry authentication. See [profiles.md](profiles.md) and [ops.md](ops.md) for detailed usage.

