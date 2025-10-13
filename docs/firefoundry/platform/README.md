# FireFoundry Platform

Platform architecture, deployment, and operations documentation.

## Contents

- [Architecture](./architecture.md) - Platform architecture and design
- [Deployment](./deployment.md) - Production deployment guide
- [Operations](./operations.md) - Platform operations and maintenance

## Overview

The FireFoundry platform consists of:

- **Kubernetes Runtime**: Microservices hosting specialized AI services
- **Core Services**: Broker, Context Service, Code Sandbox
- **Infrastructure**: PostgreSQL, blob storage, Kong Gateway
- **Supporting Services**: Monitoring, logging, auto-scaling

## Platform Components

### Core Services

- **Broker Service**: Intelligent router for AI model interactions
- **Context Service**: Working memory and persistence API
- **Code Sandbox**: Secure execution of AI-generated code
- **Kong Gateway**: API management and security

### Infrastructure

- **PostgreSQL**: Entity graph and metadata storage
- **Blob Storage**: Binary artifacts and working memory data
- **Monitoring**: Application-aware monitoring and tracing

For detailed information, see the [FireFoundry Platform Overview](../README.md).

## Documentation

- [Architecture](./architecture.md) - Detailed architecture documentation
- [Deployment](./deployment.md) - Deployment procedures and best practices
- [Operations](./operations.md) - Day-to-day operations, monitoring, and troubleshooting

## Related Documentation

- [Local Development](../local-development/README.md) - Running platform locally
- [Getting Started](../getting-started/README.md) - Quick start guide

