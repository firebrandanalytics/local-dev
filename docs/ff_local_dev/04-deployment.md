# Deploying FireFoundry Services

Deploy the complete FireFoundry infrastructure for local development using published Helm charts.

## Step 0: Download Configuration Files

You'll receive a configuration package (`ff-dev-config.zip`) containing the necessary values and secrets files. Extract this package to your working directory:

```bash
# Extract the configuration package
unzip ff-dev-config.zip
cd ff-dev-config

# You should now have these files:
# - core-values.yaml          (Core services configuration)
# - control-plane-values.yaml (Control plane configuration)
# - secrets.yaml              (API keys and credentials)
```

**What's included:**

- **core-values.yaml**: Configuration for FF Broker, Context Service, and Code Sandbox
- **control-plane-values.yaml**: Configuration for PostgreSQL, Concourse, Harbor, and FF Console
- **secrets.yaml**: Pre-configured API keys, database credentials, and service tokens

**Important**: All deployment commands in this guide assume you're working from the directory containing these configuration files.

## Step 1: Container Registry Access

FireFoundry uses a private Azure Container Registry for security and control. Here's how to set it up:

```bash
# Authenticate with Azure
az login

# Switch to the correct subscription
az account set --subscription "Firebrand R&D"

# Retrieve the registry password
export ACR_PASSWORD=$(az acr credential show --name firebranddevet --query "passwords[0].value" -o tsv)

# Create Kubernetes secrets for both namespaces
kubectl create secret docker-registry myregistrycreds \
  --docker-server=firebranddevet.azurecr.io \
  --docker-username=firebranddevet \
  --docker-password="$ACR_PASSWORD" \
  --namespace=ff-dev

kubectl create secret docker-registry myregistrycreds \
  --docker-server=firebranddevet.azurecr.io \
  --docker-username=firebranddevet \
  --docker-password="$ACR_PASSWORD" \
  --namespace=ff-control-plane
```

**Why private registry?** Public registries pose security risks for enterprise AI applications. Private registries provide:

- **Access control**: Only authorized developers can pull/push images
- **Security scanning**: Automated vulnerability detection in container images
- **Compliance**: Meets enterprise security requirements for AI workloads

## Step 2: Add FireFoundry Helm Repository

Add the published FireFoundry charts to your Helm repositories:

```bash
# Add the FireFoundry Helm repository
helm repo add firebrandanalytics https://firebrandanalytics.github.io/ff_infra
helm repo update

# Verify the charts are available
helm search repo firebrandanalytics
```

**Available charts:**

- **firefoundry-control-plane**: Infrastructure services (PostgreSQL, Concourse, Harbor, FF Console)
- **firefoundry-core**: Core AI services (FF Broker, Context Service, Code Sandbox)

## Step 3: Create Namespaces

FireFoundry uses two separate namespaces for clear separation of concerns. You can create them manually or let Helm create them automatically:

**Option 1: Manual namespace creation (recommended for clarity)**

```bash
# Create the core runtime services namespace
kubectl create namespace ff-dev

# Create the control plane services namespace
kubectl create namespace ff-control-plane

# Verify namespaces were created
kubectl get namespaces | grep ff-
```

**Option 2: Let Helm create namespaces automatically**
The Helm commands below include `--create-namespace` flags that will create the namespaces if they don't exist.

## Step 4: Deploy FireFoundry Services

Deploy both the control plane and core services using the published charts. For local development, you'll need both to get the complete FireFoundry experience.

### Deploy Control Plane Services

First, deploy the infrastructure services (PostgreSQL, Concourse, Harbor, FF Console):

```bash
# Deploy control plane services
helm install firefoundry-control firebrandanalytics/firefoundry-control-plane \
  -f control-plane-values.yaml \
  --namespace ff-control-plane \
  --create-namespace
```

### Deploy Core Services

Next, deploy the core AI services (FF Broker, Context Service, Code Sandbox):

```bash
# Deploy core runtime services
helm install firefoundry-core firebrandanalytics/firefoundry-core \
  -f core-values.yaml \
  -f secrets.yaml \
  --namespace ff-dev \
  --create-namespace
```

**Why both?** Local development benefits from:

- **Core services**: Enable agent development, testing, and execution
- **Control plane**: Provides CI/CD pipelines, container registry, and monitoring tools
- **Full integration**: Test the complete agent lifecycle from development to deployment

## Step 5: Verification and Health Checks

Wait for all services to start up (this may take 2-3 minutes), then verify your deployment:

```bash
# Check that all pods are running
kubectl get pods -n ff-dev
kubectl get pods -n ff-control-plane

# Verify core services are healthy
kubectl logs -n ff-dev deployment/ff-broker
kubectl logs -n ff-dev deployment/context-service
kubectl logs -n ff-dev deployment/code-sandbox

# Test FF Broker connectivity (gRPC service)
kubectl -n ff-dev port-forward svc/ff-broker 50061:50061 &
curl -v localhost:50061  # Should connect to gRPC endpoint

# Access FF Console for monitoring (if control plane is deployed)
kubectl -n ff-control-plane port-forward svc/ff-console 3001:3001 &
# Open http://localhost:3001 in your browser
```

**Expected results:**

- All pods should show `Running` status
- FF Broker should accept gRPC connections on port 50061
- FF Console should be accessible at http://localhost:3001
- Service logs should show successful startup without critical errors

## Troubleshooting Common Issues

### Pods Stuck in ImagePullBackOff

If pods can't pull container images:

```bash
# Check registry credentials
kubectl get secret myregistrycreds -n ff-dev
kubectl get secret myregistrycreds -n ff-control-plane

# If missing, recreate the registry secrets (see Step 1)
```

### Pods Stuck in Pending

If pods can't be scheduled:

```bash
# Check available resources
kubectl describe nodes
kubectl top nodes

# Consider restarting minikube with more resources
minikube stop
minikube start --memory=8192 --cpus=4 --disk-size=20g
```

### Services Not Starting

If services fail to start:

```bash
# Check pod events and logs
kubectl describe pod <pod-name> -n ff-dev
kubectl logs <pod-name> -n ff-dev

# Common issues: missing secrets, database connectivity, resource limits
```

## Next Steps

With FireFoundry deployed, you're ready to:

1. **[FireFoundry CLI Setup](05-ff-cli-setup.md)** - Install the CLI for agent development
2. **[Start Agent Development](06-agent-development.md)** - Create your first agent bundle
3. **[Learn Operations](07-operations.md)** - Monitor and maintain your deployment
