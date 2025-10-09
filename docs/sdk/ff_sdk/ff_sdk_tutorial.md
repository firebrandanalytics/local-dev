# FireFoundry SDK (FF SDK) Tutorial

## Introduction

The **FireFoundry SDK** (ff-sdk) is a lightweight consumption SDK for interacting with AI agent bundles deployed on the FireFoundry platform. It provides a simple, type-safe interface for invoking entity methods, streaming results, and calling custom API endpoints.

This tutorial assumes you have:
- An agent bundle deployed and running (e.g., locally in Minikube)
- Access to the base URL where your agent bundle is exposed
- An API key for authentication (if required by your deployment)
- A TypeScript development environment with Node.js installed

## Installation

```bash
npm install @firebrandanalytics/ff-sdk
```

## Core Concepts

### Agent Bundles and Entities

In FireFoundry:
- **Agent Bundles** are collections of AI agents and workflows deployed as microservices
- **Entities** are persistent business objects that can have behavior (methods you can invoke)
- Each entity has a unique **Entity ID** (UUID)

### Communication Patterns

The FF SDK supports three main patterns:
1. **Direct Entity Method Invocation** - Call methods on specific entities
2. **Custom API Endpoints** - Call bundle-level endpoints that don't require an entity ID
3. **Streaming** - Receive progress updates via async iteration

---

## Part 1: Basic Usage

### Setting Up the Client

First, create a client instance pointing to your deployed agent bundle:

```typescript
import { RemoteAgentBundleClient } from '@firebrandanalytics/ff-sdk';

// For local Minikube deployment
const BASE_URL = 'http://localhost:3001';

// Create client with optional API key
const client = new RemoteAgentBundleClient(BASE_URL, {
  api_key: 'your-api-key-here', // Optional - include if your deployment requires auth
  timeout: 60000 // Optional - request timeout in milliseconds (default: 200000)
});
```

**Environment-Specific Configuration:**
- **Local Minikube**: `http://localhost:3001` (assuming port-forward to your agent bundle)
- **Staging**: `https://your-staging-url.example.com`
- **Production**: `https://your-production-url.example.com`

### Health Checks

Before invoking methods, verify the service is healthy:

```typescript
async function checkServiceHealth() {
  const isHealthy = await client.health_check();
  console.log('Service healthy:', isHealthy);

  const isReady = await client.is_ready();
  console.log('Service ready:', isReady);

  if (isHealthy && isReady) {
    const info = await client.get_info();
    console.log('Service info:', info);
    // Outputs: { app_id, app_name, app_description, server }
  }
}
```

### Invoking Entity Methods

The most common pattern is invoking methods on specific entities:

```typescript
async function invokeEntityMethod() {
  const entityId = 'your-entity-uuid-here';
  
  try {
    const result = await client.invoke_entity_method(
      entityId,
      'method_name',
      arg1,
      arg2,
      arg3
    );
    
    console.log('Result:', result);
    return result;
  } catch (error) {
    console.error('Method invocation failed:', error);
    throw error;
  }
}
```

**Type Safety with Shared Types:**

In a monorepo setup, you can import shared types for full type safety:

```typescript
import type { MyMethodOutput } from '@my-workspace/shared-types';

const result = await client.invoke_entity_method<MyMethodOutput>(
  entityId,
  'my_method',
  'arg1',
  'arg2'
);

// TypeScript now knows the shape of `result`
console.log(result.someProperty); // ✅ Type-safe
```

---

## Part 2: Intermediate Usage

### Custom API Endpoints

Some agent bundles expose custom API endpoints that don't require an entity ID:

```typescript
import type { WorkflowStatus } from '@my-workspace/shared-types';

// GET endpoint with query parameters
async function getActiveWorkflows() {
  const workflows = await client.call_api_endpoint<WorkflowStatus[]>('workflows', {
    query: { status: 'active', limit: '10' }
  });
  
  return workflows;
}

// POST endpoint with request body
async function createEntity() {
  const result = await client.call_api_endpoint<{ id: string }>('entities', {
    method: 'POST',
    body: { 
      name: 'Test Entity', 
      type: 'sample' 
    },
    query: { validate: 'true' } // Optional query params with POST
  });
  
  return result.id;
}
```

**Note:** The `/api/` prefix is automatically added - just provide the route name.

### Binary Downloads

For endpoints that return binary data (PDFs, images, CSV files):

```typescript
import fs from 'fs';

async function downloadReport() {
  // Download a PDF report
  const pdfBuffer = await client.call_api_endpoint_binary('generate-report', {
    query: { 
      format: 'A4', 
      startDate: '2024-01-01' 
    }
  });
  
  fs.writeFileSync('report.pdf', pdfBuffer);
  console.log('Report saved to report.pdf');
}

async function exportData() {
  // Generate and download a chart
  const chartBuffer = await client.call_api_endpoint_binary('render-chart', {
    method: 'POST',
    body: { 
      type: 'bar', 
      data: [10, 20, 30, 40, 50] 
    }
  });
  
  fs.writeFileSync('chart.png', chartBuffer);
}
```

### Error Handling

The FF SDK wraps errors in `FFError` for consistent error handling:

```typescript
import { FFError } from '@firebrandanalytics/ff-sdk';

async function robustInvocation(entityId: string) {
  try {
    const result = await client.invoke_entity_method(
      entityId,
      'process_data',
      inputData
    );
    return result;
  } catch (error) {
    if (error instanceof FFError) {
      console.error('FF Error:', error.message);
      // Handle specific error cases
      if (error.message.includes('not found')) {
        console.error('Entity does not exist');
      } else if (error.message.includes('timeout')) {
        console.error('Request timed out - try increasing timeout');
      }
    } else {
      console.error('Unexpected error:', error);
    }
    throw error;
  }
}
```

### Binary Blob Uploads

For methods that accept binary files (like image processing or document analysis):

```typescript
import fs from 'fs';

async function processDocument(entityId: string) {
  const documentBuffer = fs.readFileSync('document.pdf');
  
  // The method signature must accept Buffer parameters
  // e.g., async process_document(buffer: Buffer, filename: string)
  const result = await client.invoke_entity_method_with_blobs(
    entityId,
    'process_document',
    [{ $blob: 0 }, 'document.pdf'],  // Args with blob placeholder
    [documentBuffer]                 // Binary files
  );
  
  return result;
}

async function analyzeImage(entityId: string) {
  const imageBuffer = fs.readFileSync('image.jpg');
  
  const analysis = await client.invoke_entity_method_with_blobs(
    entityId,
    'analyze_image',
    [{ $blob: 0 }, { format: 'jpg', quality: 'high' }],
    [imageBuffer]
  );
  
  return analysis;
}
```

---

## Part 3: Advanced Usage

### Streaming with Iterators

For long-running operations, use iterators to receive progress updates:

```typescript
async function streamingAnalysis(entityId: string) {
  console.log('Starting analysis with streaming...');
  
  // Start the iterator
  const iterator = await client.start_iterator(
    entityId,
    'streaming_method',
    arg1,
    arg2
  );
  
  // Consume progress updates
  for await (const progress of iterator) {
    console.log('Progress:', progress.type, progress.data);
    
    // Handle different progress types
    switch (progress.type) {
      case 'status':
        console.log(`Status: ${progress.data.message}`);
        break;
      case 'partial_result':
        console.log('Partial result:', progress.data);
        break;
      case 'complete':
        console.log('Analysis complete!');
        return progress.data.result;
    }
  }
  
  // Iterator automatically cleans up on completion
}
```

**Manual Iterator Management:**

```typescript
async function manualIteratorControl(entityId: string) {
  const iterator = await client.start_iterator(entityId, 'process');
  
  try {
    let done = false;
    while (!done) {
      const result = await iterator.next();
      done = result.done;
      
      if (!result.done) {
        console.log('Value:', result.value);
      }
    }
  } finally {
    // Manual cleanup if needed
    await iterator.cleanup();
  }
}
```

### Configuration Management

For production deployments, externalize configuration:

```typescript
// config.ts
export interface AgentBundleConfig {
  baseUrl: string;
  apiKey?: string;
  timeout?: number;
}

export function loadConfig(): AgentBundleConfig {
  return {
    baseUrl: process.env.AGENT_BUNDLE_URL || 'http://localhost:3001',
    apiKey: process.env.AGENT_BUNDLE_API_KEY,
    timeout: parseInt(process.env.AGENT_BUNDLE_TIMEOUT || '60000')
  };
}

// client.ts
import { loadConfig } from './config.js';

const config = loadConfig();
const client = new RemoteAgentBundleClient(config.baseUrl, {
  api_key: config.apiKey,
  timeout: config.timeout
});
```

### Retry Logic

Implement retry logic for resilient applications:

```typescript
async function invokeWithRetry<T>(
  entityId: string,
  methodName: string,
  args: any[],
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await client.invoke_entity_method<T>(
        entityId,
        methodName,
        ...args
      );
    } catch (error) {
      lastError = error as Error;
      console.warn(`Attempt ${attempt} failed:`, error);
      
      if (attempt < maxRetries) {
        // Exponential backoff
        const delay = Math.pow(2, attempt) * 1000;
        console.log(`Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  throw new Error(`Failed after ${maxRetries} attempts: ${lastError!.message}`);
}
```

### Batch Processing

Process multiple items efficiently:

```typescript
async function batchProcess(entityId: string, items: any[]) {
  console.log(`Processing ${items.length} items...`);
  
  // Process in parallel with concurrency limit
  const concurrency = 5;
  const results = [];
  
  for (let i = 0; i < items.length; i += concurrency) {
    const batch = items.slice(i, i + concurrency);
    const batchResults = await Promise.all(
      batch.map(item => 
        client.invoke_entity_method(entityId, 'process_item', item)
          .catch(error => ({ error: error.message, item }))
      )
    );
    results.push(...batchResults);
    
    console.log(`Processed ${Math.min(i + concurrency, items.length)}/${items.length}`);
  }
  
  return results;
}
```

### Working with Collections

Many agent bundles expose collection entities as factories:

```typescript
async function createAndProcess() {
  // Get the root collection entity ID (from deployment/config)
  const collectionId = 'your-collection-entity-id';
  
  // Create a new entity via the collection
  const result = await client.invoke_entity_method<{ entity_id: string }>(
    collectionId,
    'create_and_process',
    { name: 'New Item', data: { /* ... */ } }
  );
  
  console.log('Created entity:', result.entity_id);
  
  // Now work with the newly created entity
  const status = await client.invoke_entity_method(
    result.entity_id,
    'get_status'
  );
  
  return status;
}

async function listAllItems(collectionId: string) {
  const items = await client.invoke_entity_method<any[]>(
    collectionId,
    'get_all_items'
  );
  
  console.log(`Found ${items.length} items`);
  return items;
}
```

---

## Complete Working Example

Here's a full command-line program demonstrating the FF SDK:

```typescript
#!/usr/bin/env node
import { RemoteAgentBundleClient, FFError } from '@firebrandanalytics/ff-sdk';

const BASE_URL = process.env.AGENT_BUNDLE_URL || 'http://localhost:3001';
const API_KEY = process.env.AGENT_BUNDLE_API_KEY;

async function main() {
  // Initialize client
  const client = new RemoteAgentBundleClient(BASE_URL, {
    api_key: API_KEY,
    timeout: 60000
  });

  try {
    // Health check
    console.log('🔍 Checking service health...');
    const isHealthy = await client.health_check();
    if (!isHealthy) {
      throw new Error('Service is not healthy');
    }
    console.log('✅ Service is healthy\n');

    // Get service info
    const info = await client.get_info();
    console.log('📋 Service Info:');
    console.log(`   Name: ${info.app_name}`);
    console.log(`   Description: ${info.app_description}`);
    console.log(`   ID: ${info.app_id}\n`);

    // Example: Invoke a method
    const entityId = process.argv[2];
    if (!entityId) {
      console.log('Usage: npm start <entity-id>');
      process.exit(1);
    }

    console.log(`🚀 Invoking method on entity ${entityId}...`);
    const result = await client.invoke_entity_method(
      entityId,
      'example_method',
      'example_arg'
    );

    console.log('✅ Result:', JSON.stringify(result, null, 2));

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ FF Error:', error.message);
    } else {
      console.error('❌ Unexpected error:', error);
    }
    process.exit(1);
  }
}

main();
```

**Running the example:**

```bash
# Set environment variables
export AGENT_BUNDLE_URL="http://localhost:3001"
export AGENT_BUNDLE_API_KEY="your-api-key"

# Run with entity ID
npm start "c0000000-0000-0000-0001-000000000001"
```

---

## Future Directions

As you become more familiar with the FF SDK, consider exploring:

### Integration Patterns
- **Express/Fastify APIs**: Wrap agent bundle calls in REST APIs for web applications
- **Next.js**: Server-side rendering with agent bundle data
- **React/Vue**: Client-side integration using API routes
- **Background Workers**: Queue-based processing with agent bundles

### Advanced Features
- **WebSocket Streaming**: Real-time progress updates (alternative to iterator pattern)
- **Correlation Tracking**: Use correlation headers for distributed tracing
- **Circuit Breakers**: Implement circuit breaker pattern for resilience
- **Caching**: Add response caching for frequently-called methods

### Monitoring & Observability
- **Metrics**: Track invocation latency, success rates, error types
- **Logging**: Structured logging with correlation IDs
- **Alerting**: Set up alerts for degraded performance or failures

### Enterprise Integration
- **Authentication**: OAuth/OIDC integration for user context
- **Multi-tenancy**: Routing requests to tenant-specific agent bundles
- **Data Governance**: Audit trails and data lineage tracking

---

## Best Practices

1. **Always handle errors gracefully** - Use try/catch and check for `FFError`
2. **Use type safety** - Import shared types from your monorepo for compile-time safety
3. **Externalize configuration** - Use environment variables for URLs and API keys
4. **Implement timeouts** - Set appropriate timeouts based on operation complexity
5. **Log correlation IDs** - For troubleshooting across distributed systems
6. **Test health before invoking** - Check health/readiness before critical operations
7. **Use streaming for long operations** - Don't block on synchronous calls for long-running tasks
8. **Batch when possible** - Process multiple items with controlled concurrency
9. **Monitor and measure** - Track performance metrics and error rates
10. **Document entity IDs** - Maintain clear documentation of available entities and their methods

---

## Troubleshooting

**Connection Refused**
- Verify the base URL is correct for your environment
- Check that the agent bundle is running and accessible
- For Minikube, ensure port-forwarding is active

**Authentication Errors**
- Verify your API key is correct and has necessary permissions
- Check that the API key is properly configured in the client

**Timeout Errors**
- Increase the timeout value in client configuration
- Consider using streaming for long-running operations
- Check network connectivity and latency

**404 Not Found**
- Verify the entity ID exists in the deployed bundle
- Check that the method name is spelled correctly
- Ensure the entity has the method you're trying to invoke

**Type Errors**
- Ensure shared types package is up to date in your monorepo
- Check that the agent bundle version matches your expected types
- Use explicit type annotations for debugging

---

## Additional Resources

- **Full API Documentation**: See README.md in the ff-sdk package
- **Agent Bundle Development**: Refer to AgentSDK Getting Started guide
- **Platform Overview**: Review the FireFoundry platform documentation
- **Example Applications**: Check the examples directory for complete working applications

---

**Generated by Claude**
