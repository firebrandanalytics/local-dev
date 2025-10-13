# FireIQ Developer Guide

This guide covers how to build customized agent bundles for the FireIQ Chat Application.

## Overview

FireIQ agent bundles are built on the FireIQ agent bundle library and customized for specific datasets. The customization process involves:

1. Creating a new FireIQ-based agent bundle using ff-cli
2. Defining data dictionaries for your dataset
3. Creating ontologies that capture business concepts
4. Configuring tool calls for data access
5. Testing and deploying the bundle

## FireIQ Agent Bundle Library

The FireIQ agent bundle library provides:
- Base agent implementations for conversational data access
- Common data analysis patterns
- Integration points for customization
- Utilities for data dictionary and ontology handling

When you create a FireIQ agent bundle, your bundle imports this library and extends it with dataset-specific configuration and behavior.

## Creating FireIQ Agent Bundles

FireIQ agent bundles are created using ff-cli with the FireIQ bundle type:

```bash
ff-cli create bundle --type fireiq --name my-dataset-bundle
```

This creates a new agent bundle that:
- Imports the FireIQ agent bundle library
- Provides templates for data dictionaries, ontologies, and tool configuration
- Is ready for customization to your specific dataset

## Bundle Management

Once created and customized, FireIQ agent bundles are managed like any other FireFoundry agent bundle:

- Local development follows standard [agent development practices](../../firefoundry/local-development/agent-development.md)
- Deployment uses [standard ff-cli commands](../../firefoundry/local-development/updating-agent-bundles.md)
- Operations use FireFoundry's [standard tooling](../../firefoundry/platform/operations.md)

## Documentation

- [Agent Bundle Library](./agent-bundle-library.md) - Overview of the FireIQ library
- [Customizing Bundles](./customizing-bundles.md) - Creating FireIQ-based agent bundles
- [Data Dictionaries](./data-dictionaries.md) - Building data dictionaries
- [Ontologies](./ontologies.md) - Defining ontologies
- [Tool Configuration](./tool-configuration.md) - Configuring available operations

## Workflow

1. Use the [Data Analysis Application](../data-analysis-application/README.md) to prepare your dataset
2. Export configuration from the data analysis application
3. Create a new FireIQ agent bundle using ff-cli
4. Import the configuration into your bundle
5. Customize and extend as needed
6. Test locally
7. Deploy to your FireFoundry environment
8. Configure in the chat application

## Next Steps

- [Learn about the Agent Bundle Library](./agent-bundle-library.md)
- [Create your first FireIQ bundle](./customizing-bundles.md)

