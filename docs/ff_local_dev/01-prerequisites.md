# Prerequisites and Setup

Before you can start developing with FireFoundry, you'll need to install the required tools and set up authentication. **Docker must be installed and running first**, as it provides the container runtime that minikube depends on.

## Step 1: Install and Verify Docker

Docker Desktop provides the container runtime that minikube uses to run Kubernetes. It must be installed and running before proceeding with other tools.

### Install Docker Desktop

**For macOS:**

```bash
# Install Docker Desktop via Homebrew
brew install --cask docker

# Or download directly from: https://www.docker.com/products/docker-desktop/
```

**For Windows:**

- Download Docker Desktop from: https://www.docker.com/products/docker-desktop/
- Run the installer and follow the setup wizard
- Ensure WSL 2 is enabled if prompted

### Launch and Verify Docker

After installation, you must launch Docker Desktop and verify it's working:

**For macOS:**

```bash
# Launch Docker Desktop
open -a Docker

# Wait for Docker Desktop to start (you'll see the whale icon in your menu bar)
# The icon should be steady (not animated) when ready
```

**For Windows:**

- Launch Docker Desktop from the Start menu
- Wait for the Docker Desktop interface to show "Engine running" status
- You should see the Docker whale icon in your system tray

### Verify Docker is Working

Once Docker Desktop is running, verify it's working properly:

```bash
# Test Docker is running and responsive
docker --version

# Test Docker can run containers
docker run hello-world

# Verify Docker daemon is accessible
docker ps
```

**Expected output:**

- `docker --version` should show the Docker version
- `docker run hello-world` should download and run a test container successfully
- `docker ps` should show an empty list of running containers (or any existing ones)

**If Docker isn't working:**

- Ensure Docker Desktop is fully started (check the system tray/menu bar icon)
- On Windows, ensure WSL 2 is properly configured
- Try restarting Docker Desktop
- Check Docker Desktop's troubleshooting section in the app

## Step 2: Install Kubernetes and Development Tools

With Docker running, you can now install the remaining tools:

```bash
# Install Kubernetes and development tools
brew install minikube kubectl helm azure-cli

# Install global Node.js dependencies (required for monorepo management)
npm install -g pnpm turbo
```

**Why these tools?**

- **minikube**: Provides a local Kubernetes cluster using Docker as the container runtime
- **kubectl**: Essential for debugging and managing Kubernetes resources
- **helm**: Manages complex multi-service deployments with templating and versioning
- **azure-cli**: Needed for accessing the private container registry
- **pnpm**: Fast, disk space efficient package manager used by FireFoundry monorepos
- **turbo**: High-performance build system for managing monorepo builds and caching

### Additional Helpful Tools

#### For Windows Users: Git Bash (Recommended)

**Windows users should install Git Bash** for the best command-line experience with FireFoundry:

```bash
# Download and install Git for Windows (includes Git Bash)
# Visit: https://git-scm.com/download/win
```

**After installing Git Bash:**

- Use Git Bash terminal for all FireFoundry commands instead of Command Prompt or PowerShell
- The bash commands shown in this guide will work directly without modification

#### For All Platforms: Kubernetes Management Tools

For managing the multi-namespace setup more efficiently:

```bash
# Install kubectx for easy context management
brew install kubectx

# Install k9s for visual cluster management
brew install k9s
```

**kubectx**: Quickly switch between kubectl contexts (useful for switching between client, Firebrand, and local minikube clusters)

**k9s**: Visual terminal UI for managing pods, services, and logs

```bash
k9s -n ff-dev -n ff-control-plane  # Monitor both namespaces simultaneously
```

## Next Steps

With Docker and the core development tools installed, you're ready to set up your local FireFoundry environment:

1. **[Understand the Architecture](02-architecture.md)** - Learn about FireFoundry's design
2. **[Set up your Environment](03-environment-setup.md)** - Configure your local development environment
3. **[Deploy FireFoundry Services](04-deployment.md)** - Get the infrastructure running

Once you have the infrastructure running, you can set up the FireFoundry CLI for agent development:

4. **[FireFoundry CLI Setup](05-ff-cli-setup.md)** - Install the CLI for creating agent projects (requires GitHub token)
