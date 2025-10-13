# Customizing FireIQ Agent Bundles

## Overview

This guide covers creating and customizing FireIQ agent bundles for your specific datasets.

## Creating a FireIQ Bundle

### Using ff-cli

Create a new FireIQ agent bundle:

```bash
ff-cli create bundle --type fireiq --name my-dataset-bundle
```

This command:
- Creates a new agent bundle directory
- Imports the FireIQ agent bundle library
- Sets up templates for data dictionaries, ontologies, and tool configuration
- Initializes the project structure

_[To be documented: Detailed project structure]_

## Customization Process

### 1. Import Configuration from Data Analysis App

_[To be documented]_

If you've used the Data Analysis Application to prepare your dataset:

```bash
ff-cli fireiq import-config --source <export-path> --bundle my-dataset-bundle
```

### 2. Define Data Dictionary

_[To be documented]_

See [Data Dictionaries](./data-dictionaries.md) for detailed guidance.

### 3. Create Ontology

_[To be documented]_

See [Ontologies](./ontologies.md) for detailed guidance.

### 4. Configure Tool Calls

_[To be documented]_

See [Tool Configuration](./tool-configuration.md) for detailed guidance.

### 5. Add Custom Logic

_[To be documented]_

Extend the FireIQ library with domain-specific behavior:

```typescript
// Example customization
```

## Local Development

_[To be documented]_

FireIQ bundles are developed like standard FireFoundry agent bundles:

- See [Agent Development](../../firefoundry/local-development/agent-development.md)
- See [Local Development Setup](../../firefoundry/local-development/environment-setup.md)

## Testing

_[To be documented]_

### Unit Testing

_[To be documented]_

### Integration Testing

_[To be documented]_

### Testing with Chat Application

_[To be documented]_

## Deployment

Once customized and tested, deploy your FireIQ bundle:

```bash
ff-cli deploy bundle my-dataset-bundle --environment production
```

See [Updating Agent Bundles](../../firefoundry/local-development/updating-agent-bundles.md) for details.

## Configuration Management

_[To be documented]_

- Environment-specific configuration
- Secrets management
- Version control best practices

## Troubleshooting

_[To be documented]_

Common issues and solutions.

## Examples

_[To be documented]_

- Sample FireIQ bundle
- Common customization patterns
- Advanced use cases

