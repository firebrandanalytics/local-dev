# Firebrand Analytics Developer Documentation

Welcome to the Firebrand Analytics technical documentation. This documentation covers our enterprise AI platform offerings and SDKs.

## About Firebrand Analytics

Firebrand Analytics provides enterprise AI platforms and applications that help organizations build, deploy, and operate sophisticated GenAI solutions.

## Product Offerings

### FireFoundry

**An Agent-as-a-Service platform for building sophisticated GenAI applications**

FireFoundry is a batteries-included platform that combines opinionated runtime services, developer tooling, and a management console. It enables teams to ship faster by providing built-in persistence, observability, deployment, and streaming capabilities.

- [FireFoundry Documentation](./firefoundry/README.md)
- [Getting Started with FireFoundry](./firefoundry/getting-started/README.md)
- [AgentSDK](./firefoundry/sdk/agent-sdk/README.md)
- [FF SDK](./firefoundry/sdk/ff-sdk/README.md)

**Key Features:**
- Zero-code persistence via entity graph
- Built-in observability and monitoring
- Secure code execution sandbox
- Agent bundle deployment and lifecycle management
- Human-in-the-loop workflows (Waitables)
- Testing framework with LLM-as-validator

### FireIQ Suite

**AI-powered data analysis and conversational data access applications built on FireFoundry**

The FireIQ Suite provides enterprise tools for preparing datasets and enabling natural language data access. It consists of applications for data ingestion, cleansing, analysis, and a chat interface for conversational data queries.

- [FireIQ Documentation](./fireiq/README.md)
- [Getting Started with FireIQ](./fireiq/getting-started/README.md)
- [Chat Application](./fireiq/chat-application/README.md)
- [Data Analysis Application](./fireiq/data-analysis-application/README.md)

**Key Features:**
- AI-assisted data preparation and cleansing
- Automated data dictionary and ontology generation
- Chat-with-your-data interface
- Customizable agent bundles per dataset
- Multi-dataset support with easy switching

## Getting Started

### For FireFoundry

If you're building custom GenAI applications and need a complete platform:

1. [FireFoundry Prerequisites](./firefoundry/getting-started/prerequisites.md)
2. [Installation Guide](./firefoundry/getting-started/README.md)
3. [Agent Development](./firefoundry/local-development/agent-development.md)

### For FireIQ

If you want to enable conversational access to your data:

1. [Install FireFoundry](./firefoundry/getting-started/README.md) (required)
2. [FireIQ Prerequisites](./fireiq/getting-started/prerequisites.md)
3. [Install FireIQ](./fireiq/getting-started/installation.md)
4. [Prepare Your Data](./fireiq/data-analysis-application/data-preparation.md)

## Documentation Organization

```
docs/
├── firefoundry/          # FireFoundry platform documentation
│   ├── getting-started/  # Installation and quick start
│   ├── platform/         # Architecture and operations
│   ├── local-development/ # Local development setup
│   └── sdk/              # SDK documentation
│       ├── agent-sdk/    # Agent development SDK
│       ├── ff-sdk/       # Consumer integration SDK
│       └── utils/        # Utility libraries
│
└── fireiq/               # FireIQ Suite documentation
    ├── getting-started/  # Installation and setup
    ├── chat-application/ # Chat interface
    ├── data-analysis-application/ # Data preparation
    └── developer-guide/  # Custom bundle development
```

## Support & Resources

- [FireFoundry Troubleshooting](./firefoundry/local-development/troubleshooting.md)
- [FireFoundry Glossary](./firefoundry/README.md#appendices)

## Contributing

For information about contributing to this documentation, see CONTRIBUTING.md in the repository root.
