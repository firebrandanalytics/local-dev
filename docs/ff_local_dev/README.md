# FireFoundry Local Development Guide

Welcome to the FireFoundry local development documentation! This guide will walk you through setting up the complete FireFoundry ecosystem and developing your first agent bundle.

## Quick Start

If you're new to FireFoundry, follow these guides in order:

1. **[Prerequisites](01-prerequisites.md)** - Install tools and set up GitHub token
2. **[Architecture Overview](02-architecture.md)** - Understand the system design
3. **[Environment Setup](03-environment-setup.md)** - Configure your local development environment
4. **[Deploy Services](04-deployment.md)** - Deploy FireFoundry infrastructure
5. **[FireFoundry CLI Setup](05-ff-cli-setup.md)** - Install the CLI for agent development (requires GitHub token)
6. **[Agent Development](06-agent-development.md)** - Create your first agent bundle

## Reference Guides

- **[Operations & Maintenance](07-operations.md)** - Monitoring, debugging, and updates
- **[Troubleshooting](08-troubleshooting.md)** - Common issues and solutions

## What is FireFoundry?

FireFoundry is a comprehensive platform for building and deploying AI agents that can interact with various services, manage context, and execute code safely. It uses a **multi-namespace architecture** designed for operational excellence and clear separation of concerns.

### Key Benefits

- **Modularity**: Each agent handles specific domain responsibilities
- **Reusability**: Agent bundles can be composed into larger systems
- **Testability**: Individual agents can be tested in isolation
- **Maintainability**: Clear boundaries make debugging and updates easier
- **Scalability**: Independent scaling of runtime and control plane services

## Getting Help

- **Documentation**: Check the Agent SDK docs included in generated projects
- **Logs**: Always check pod logs first when troubleshooting
- **Community**: Internal Teams channels and team knowledge sharing
- **Support**: Reach out to the FireFoundry platform team for infrastructure issues:
  - **Doug**: doug@firebrand.ai
  - **Augustus**: augustus@firebrand.ai
  - **Jose**: jose@firebrand.ai

---

This guide provides a comprehensive foundation for FireFoundry development. As you build more complex agents, you'll discover additional patterns and best practices specific to your use cases.
