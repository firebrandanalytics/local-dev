# FireFoundry Prerequisites

This document outlines the prerequisites for installing and running FireFoundry.

## For Cloud Deployment

_[To be documented]_

### Infrastructure

- Kubernetes cluster (version requirements TBD)
- Cloud provider account (Azure, AWS, or GCP)
- Terraform (for infrastructure provisioning)

### Tools

- kubectl
- Helm
- ff-cli

### Resources

- Cluster sizing requirements
- Storage requirements
- Network configuration

## For Local Development

See the comprehensive [Local Development Prerequisites](../local-development/environment-setup.md) for detailed requirements.

### Required Software

- Docker Desktop or similar
- Minikube
- Node.js (version 23.x)
- ff-cli ([releases](https://github.com/firebrandanalytics/ff-cli-releases))

### System Requirements

- RAM: Minimum 16GB recommended
- CPU: Multi-core processor recommended
- Disk: Sufficient space for Docker images and data

## Access & Credentials

_[To be documented]_

- Cloud provider credentials
- Container registry access
- Database access

## Next Steps

Once prerequisites are met:

- **Cloud Deployment**: See [Deployment Guide](../platform/deployment.md)
- **Local Development**: See [Environment Setup](../local-development/environment-setup.md)
