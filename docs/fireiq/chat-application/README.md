# FireIQ Chat Application

The FireIQ Chat Application provides a natural language interface for querying and analyzing your data. Users can interact with multiple datasets, each powered by its own customized agent bundle.

## Overview

The chat application consists of:
- **Web Application**: User interface for conversational data access
- **Agent Bundles**: Dataset-specific bundles built on the FireIQ agent bundle library
- **Data Set Switching**: Users can toggle between different datasets in the UI

## Architecture

Each dataset accessible through the chat interface requires:
- A customized agent bundle created using ff-cli
- Data dictionary defining the dataset structure
- Ontology defining business concepts and relationships
- Tool calls for data access and analysis

## Documentation

- [Deployment](./deployment.md) - Deploying the chat web application
- [Configuration](./configuration.md) - Admin configuration and settings
- [Agent Bundles](./agent-bundles.md) - Managing dataset-specific agent bundles

## Workflow

1. Deploy the chat web application using ff-cli
2. Create customized agent bundles for each dataset (see [Developer Guide](../developer-guide/README.md))
3. Configure data set availability in the application
4. Users access the chat interface and select datasets

## Next Steps

- [Deploy the Chat Application](./deployment.md)
- [Learn about creating custom agent bundles](../developer-guide/customizing-bundles.md)

