# Deploying FireFoundry Services

Deploy the FireFoundry Control Plane for local development. Once the control plane is running, you'll use the FireFoundry CLI to create environments for your AI services.

## Overview

FireFoundry uses a **two-tier architecture**:

1. **Control Plane** (one-time setup) - Infrastructure services: Kong Gateway, Flux, Helm API, FF Console
2. **Environments** (managed via CLI) - Your AI services: FF Broker, Context Service, Code Sandbox

This guide covers deploying the Control Plane. Environment creation is handled by the `ff-cli` tool.

## Step 1: Clone the Configuration Repository

Clone the FireFoundry local development repository:

```bash
git clone https://github.com/firebrandanalytics/firefoundry-local.git
cd firefoundry-local
```

**Repository contents:**

- `control-plane/values.yaml` - Control plane configuration
- `control-plane/secrets.template.yaml` - Template for secrets
- `scripts/deploy-control-plane.sh` - Deployment script

## Step 2: Create Namespace and Registry Access

FireFoundry uses a private Azure Container Registry. You must authenticate to prove you have access.

```bash
# Create the control plane namespace
kubectl create namespace ff-control-plane

# Authenticate with Azure (requires Firebrand Azure account)
az login

# Switch to the correct subscription
az account set --subscription "Firebrand R&D"

# Retrieve the registry password
export ACR_PASSWORD=$(az acr credential show --name firebranddevet --query "passwords[0].value" -o tsv)

# Create the registry secret
kubectl create secret docker-registry myregistrycreds \
  --docker-server=firebranddevet.azurecr.io \
  --docker-username=firebranddevet \
  --docker-password="$ACR_PASSWORD" \
  --namespace=ff-control-plane
```

**Why this step?** This verifies you have authorized access to Firebrand's Azure resources before pulling container images.

## Step 3: Configure Secrets

Copy the secrets template and fill in the required values:

```bash
cp control-plane/secrets.template.yaml control-plane/secrets.yaml
```

Edit `control-plane/secrets.yaml` with the values provided by the FireFoundry platform team:

```yaml
ff-console:
  secret:
    data:
      PG_PASSWORD: "" # Database password
      OPENID_SECRET: "" # Azure AD client secret
      WORKING_MEMORY_STORAGE_KEY: "" # Azure Storage key
      APPLICATIONINSIGHTS_CONNECTION_STRING: "" # Optional
```

Contact Firebrand Support if you need these credentials.

## Step 4: Deploy the Control Plane

Run the deployment script:

```bash
./scripts/deploy-control-plane.sh
```

The script will:

- Install Flux CRDs (required for environment management)
- Add the FireFoundry Helm repository
- Deploy the control plane services

**Options:**

```bash
# Deploy specific chart version
./scripts/deploy-control-plane.sh -v 0.2.0

# Preview without deploying
./scripts/deploy-control-plane.sh --dry-run

# Skip CRD installation (if already installed)
./scripts/deploy-control-plane.sh --skip-crds
```

## Step 5: Verify Deployment

Wait 2-3 minutes for services to start, then verify:

```bash
# Check all pods are running
kubectl get pods -n ff-control-plane

# Expected pods:
# - ff-control-plane-postgresql-*
# - firefoundry-control-*-kong-*
# - firefoundry-control-*-flux-helm-controller-*
# - firefoundry-control-*-flux-source-controller-*
# - firefoundry-control-*-helm-api-*
# - firefoundry-control-*-ff-console-*
```

**Access Kong Gateway (optional):**

If you need to access services through the Kong Gateway, use port forwarding:

```bash
kubectl port-forward svc/firefoundry-control-firefoundry-control-plane-kong-proxy -n ff-control-plane 8000:8000
```

The gateway will be available at `http://localhost:8080`.

## Step 6: Install the FireFoundry CLI

Download and install the FireFoundry CLI from the releases page:

[FireFoundry CLI Releases](https://github.com/firebrandanalytics/ff-cli-releases)

**macOS/Linux:**

```bash
# Download the appropriate binary for your platform
# Make it executable and move to your PATH
chmod +x ff-cli
sudo mv ff-cli /usr/local/bin/
```

Verify installation:

```bash
ff-cli --version
```

## Step 7: Create a Profile

Create a CLI profile for your local minikube cluster:

```bash
ff-cli profile create local
```

When prompted:

1. **Configure registry settings?** Select **No** (or choose **Minikube** if you plan to build images)
2. **Configure kubectl context?** Select **Yes**, then choose **minikube**
3. **Set as current profile?** Select **Yes**

Verify your profile:

```bash
ff-cli profile list
```

## Step 8: Download the Internal Template

Firebrand employees can download the pre-configured internal template:

```bash
# You should already be logged in from Step 2, but verify:
az account show

# Download the internal template
curl -fsSL https://raw.githubusercontent.com/firebrandanalytics/firefoundry-local/main/scripts/setup-ff-template.sh | bash
```

This creates `~/.ff/environments/templates/internal.json` with database credentials, API keys, and service configuration for internal development.

## Step 9: Create Your First Environment

With the control plane running and template installed, create an environment for your AI services:

```bash
# Create a new environment using the internal template
ff-cli environment create --template internal --name my-env

# List environments
ff-cli environment list

# Check environment status
ff-cli environment status my-env
```

The environment will deploy:

- **FF Broker** - LLM orchestration service
- **Context Service** - Working memory management
- **Code Sandbox** - Secure code execution

## Troubleshooting

### Pods Stuck in ImagePullBackOff

Registry credentials are missing or incorrect:

```bash
# Verify the secret exists
kubectl get secret myregistrycreds -n ff-control-plane

# If missing, recreate it (see Step 2)
```

### Pods Stuck in Pending

Insufficient cluster resources:

```bash
# Check node resources
kubectl describe nodes

# For minikube, restart with more resources
minikube stop
minikube start --memory=8192 --cpus=4
```

### Deployment Script Fails

Check prerequisites:

```bash
# Verify kubectl is connected
kubectl cluster-info

# Verify Helm is installed
helm version

# Check script output for specific errors
./scripts/deploy-control-plane.sh --debug
```

### Environment Creation Fails

Verify the control plane is healthy:

```bash
# Check Helm API is running
kubectl get pods -n ff-control-plane | grep helm-api

# Check Flux controllers
kubectl get pods -n ff-control-plane | grep flux
```

## Next Steps

With FireFoundry deployed, you're ready to:

1. **[Start Agent Development](../local-development/agent-development.md)** - Create your first agent bundle
2. **[Learn Operations](./operations.md)** - Monitor and maintain your deployment
3. **[Troubleshooting Guide](../local-development/troubleshooting.md)** - Common issues and solutions
