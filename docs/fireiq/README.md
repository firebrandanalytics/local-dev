# FireIQ Suite

The FireIQ Suite is a collection of AI-powered applications built on top of the FireFoundry platform. It provides enterprise-grade tools for data analysis, preparation, and conversational data access.

## Prerequisites

FireIQ requires a functioning FireFoundry cluster. Please ensure you have completed the [FireFoundry installation](../firefoundry/getting-started/prerequisites.md) before proceeding.

## Applications

### Chat Application
Natural language interface for querying and analyzing your data. Users can toggle between different datasets, each powered by customized agent bundles.

- [Chat Application Documentation](./chat-application/README.md)

### Data Analysis Application
AI-assisted tools for data ingestion, cleansing, and preparation. This application helps you build the data dictionaries, ontologies, and tool configurations needed for effective chat interfaces.

- [Data Analysis Application Documentation](./data-analysis-application/README.md)

## Getting Started

1. [Prerequisites](./getting-started/prerequisites.md) - Requirements and FireFoundry setup
2. [Installation](./getting-started/installation.md) - Installing FireIQ applications using ff-cli
3. [Developer Guide](./developer-guide/README.md) - Building customized agent bundles

## Architecture Overview

FireIQ extends FireFoundry by providing:
- **Agent Bundle Library**: Pre-built components for data analysis and conversational AI
- **Web Applications**: User interfaces for chat and data preparation
- **ff-cli Integration**: Streamlined deployment and agent bundle creation

Each data set you want to make accessible through the chat interface requires its own customized agent bundle, created using ff-cli and based on the FireIQ agent bundle library.

