# Managing Chat Application Agent Bundles

## Overview

Each dataset in the FireIQ Chat Application is powered by a customized agent bundle. These bundles are created using ff-cli and based on the FireIQ agent bundle library.

## Agent Bundle Architecture

A FireIQ chat agent bundle includes:
- **Agent Bundle Library Import**: Base functionality from FireIQ
- **Data Dictionary**: Schema and structure of your dataset
- **Ontology**: Business concepts and relationships
- **Tool Calls**: Dataset-specific operations and queries
- **Custom Logic**: Domain-specific behavior

## Creating Agent Bundles

See the [Developer Guide](../developer-guide/customizing-bundles.md) for detailed instructions on creating FireIQ-based agent bundles using ff-cli.

Quick overview:
```bash
ff-cli create bundle --type fireiq --name my-dataset-bundle
```

## Managing Bundles

_[To be documented]_

### Deployment

Once created and configured, FireIQ agent bundles are deployed and managed like any other agent bundle:

```bash
ff-cli deploy bundle my-dataset-bundle --environment production
```

See [Updating Agent Bundles](../../firefoundry/local-development/updating-agent-bundles.md) for details.

### Versioning

_[To be documented]_

- Bundle versioning strategy
- Rollback procedures
- Testing new versions

### Monitoring

_[To be documented]_

- Bundle performance metrics
- Usage patterns
- Error tracking

## Multiple Datasets

_[To be documented]_

- Managing multiple agent bundles
- Dataset switching in the UI
- Shared resources and optimization

## Best Practices

_[To be documented]_

- Bundle organization
- Configuration management
- Testing strategies

