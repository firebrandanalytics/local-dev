# FireFoundry File Upload Patterns - Developer Guide

A comprehensive guide to implementing binary file upload, storage, and retrieval in FireFoundry Agent Bundles using the Entity-Bot-Prompt pattern.

## Table of Contents

1. [Overview](#overview)
2. [Architecture Components](#architecture-components)
3. [Server-Side Implementation](#server-side-implementation)
4. [Client-Side Implementation](#client-side-implementation)
5. [Working Memory Integration](#working-memory-integration)
6. [Common Patterns](#common-patterns)
7. [Error Handling](#error-handling)
8. [Troubleshooting](#troubleshooting)
9. [Complete Example](#complete-example)

## Overview

FireFoundry provides a robust file upload system that enables binary data handling across HTTP APIs while integrating with the working memory system for persistent storage. The system consists of:

- **Agent Bundle Server**: Handles multipart uploads and manages entity operations
- **Client SDK**: Provides RemoteAgentBundleClient for HTTP-based file operations
- **Working Memory**: Persistent storage backend via Context Service
- **Buffer Serialization**: JSON-safe transport of binary data

## Architecture Components

### Core Classes

```
DocumentProcessorEntity (Agent SDK)
├── process_document() - Main file upload handler
├── working_memory_provider - Context service interface
└── upload_to_working_memory() - Storage abstraction

WorkingMemoryProvider (Agent SDK)
├── add_memory_from_buffer() - Store binary data
├── get_binary_file_with_metadata_rpc() - Retrieve with metadata
└── get_memory_content_rpc() - Retrieve text content

RemoteAgentBundleClient (FF-SDK)
├── invoke_entity_method_with_blobs() - Upload files via HTTP
├── call_api_endpoint() - REST API calls
└── health_check() - Service connectivity

ExpressTransport (Agent SDK)
├── /invoke/multipart - Multipart upload endpoint
├── inject_blobs_into_args() - Placeholder replacement
└── Multer integration - File parsing middleware
```

### Request Flow

```
Client Application
    ↓ (HTTP multipart/form-data)
ExpressTransport (/invoke/multipart)
    ↓ (Multer parsing + blob injection)
AppRunnerHarness
    ↓ (Method invocation)
Entity.process_document(buffer, filename, metadata)
    ↓ (Working memory storage)
Context Service (persistent storage)
```

## Server-Side Implementation

### 1. Entity Implementation

Extend `DocumentProcessorEntity` to handle file uploads:

```typescript
import { 
  DocumentProcessorEntity,
  EntityDecorator,
  EntityFactory,
  logger,
} from "@firebrandanalytics/ff-agent-sdk";

@EntityDecorator({
  generalType: "Instance",
  specificType: "FileUploadEntity",
})
export class FileUploadEntity extends DocumentProcessorEntity {
  constructor(factory: EntityFactory<any>, idOrDto: UUID | any) {
    super(factory, idOrDto);
  }

  /**
   * Main file upload handler - override for custom behavior
   */
  public async process_document(
    document_buffer: Buffer,
    filename: string,
    metadata?: Record<string, unknown>
  ) {
    logger.info(`Processing upload: ${filename}`, {
      entity_id: this.id,
      buffer_size: document_buffer.length,
    });

    // Call parent implementation for working memory storage
    const result = await super.process_document(document_buffer, filename, {
      ...metadata,
      uploaded_via: "FileUploadEntity",
      timestamp: new Date().toISOString(),
    });

    // Store reference in entity data for tracking
    const dto = await this.get_dto();
    const uploads = Array.isArray(dto.data.uploads) ? dto.data.uploads : [];
    uploads.push({
      filename,
      working_memory_id: result.working_memory_id,
      size: result.file_info.size,
      uploaded_at: new Date().toISOString(),
    });
    
    await this.update_data({
      ...dto.data,
      uploads,
    });

    return result;
  }

  /**
   * Retrieve uploaded file from working memory
   */
  public async retrieve_file(working_memory_id: string) {
    try {
      // Use working_memory_provider from parent class
      const result = await this.working_memory_provider.get_binary_file_with_metadata_rpc(
        working_memory_id
      );

      return {
        working_memory_id,
        content: result.buffer,
        metadata: result.metadata,
        success: true,
      };
    } catch (error) {
      const error_message = error instanceof Error ? error.message : "Unknown error";
      throw new Error(`Failed to retrieve file: ${error_message}`);
    }
  }

  /**
   * List all uploaded files for this entity
   */
  public async list_uploads() {
    const dto = await this.get_dto();
    return {
      entity_id: this.id,
      uploads: dto.data.uploads || [],
    };
  }
}
```

### 2. Agent Bundle Setup

```typescript
import { FFAgentBundle } from "@firebrandanalytics/ff-agent-sdk";

export class FileUploadAgentBundle extends FFAgentBundle<any> {
  constructor() {
    super(
      {
        id: "your-agent-bundle-id",
        name: "file-upload",
        version: "1.0.0",
      },
      new FileUploadConstructors()
    );
  }

  // Optional: Add custom API endpoints
  protected setup_api_endpoints(): void {
    this.add_api_endpoint({
      method: "GET",
      path: "/api/test-entity",
      handler: async (req, res) => {
        // Create or get a test entity
        const entity = await this.entity_factory.get_or_create_entity(
          FileUploadEntity,
          "test-entity-id"
        );
        
        res.json({
          entity_id: entity.id,
          message: "Test entity ready for file uploads",
        });
      },
    });
  }
}
```

### 3. Working Memory Configuration

The `DocumentProcessorEntity` automatically integrates with working memory through the `working_memory_provider`. Ensure your agent bundle is configured with proper context service connectivity:

```typescript
// Environment variables required
CONTEXT_SERVICE_ADDRESS=http://context-service:50051
CONTEXT_SERVICE_API_KEY=your-api-key
```

## Client-Side Implementation

### 1. RemoteAgentBundleClient Setup

```typescript
import { RemoteAgentBundleClient } from "@firebrandanalytics/ff-sdk";
import { readFileSync } from "fs";

// Create client with authentication
const client = new RemoteAgentBundleClient("http://localhost:3000", {
  api_key: process.env.API_KEY, // Optional
});
```

### 2. File Upload Pattern

```typescript
async function uploadFile(filePath: string, entityId: string) {
  // Read file as Buffer
  const fileBuffer = readFileSync(filePath);
  const filename = path.basename(filePath);
  
  // Upload using blob placeholder pattern
  const uploadResult = await client.invoke_entity_method_with_blobs(
    entityId,
    "process_document",
    [
      { $blob: 0 },           // Blob placeholder pointing to files[0]
      filename,               // Original filename
      {                       // Custom metadata
        source: "client_upload",
        category: "document",
        uploaded_at: new Date().toISOString(),
      }
    ],
    [fileBuffer]             // Array of Buffer objects
  );
  
  console.log("Upload result:", uploadResult);
  return uploadResult;
}
```

### 3. File Retrieval Pattern

```typescript
async function downloadFile(entityId: string, workingMemoryId: string) {
  // Retrieve file via entity method
  const retrievedFile = await client.invoke_entity_method(
    entityId,
    "retrieve_file",
    [workingMemoryId]
  );
  
  // Handle Buffer reconstruction from HTTP JSON serialization
  let fileContent: Buffer | null = null;
  
  if (retrievedFile?.content && typeof retrievedFile.content === 'object' && 
      retrievedFile.content.type === 'Buffer' && Array.isArray(retrievedFile.content.data)) {
    // Buffer was serialized as {type: 'Buffer', data: [bytes]} - reconstruct it
    fileContent = Buffer.from(retrievedFile.content.data);
  } else if (Buffer.isBuffer(retrievedFile?.content)) {
    // Already a proper Buffer
    fileContent = retrievedFile.content;
  }
  
  return {
    content: fileContent,
    metadata: retrievedFile?.metadata,
    size: fileContent?.length || 0,
  };
}
```

### 4. Multiple File Upload

```typescript
async function uploadMultipleFiles(filePaths: string[], entityId: string) {
  // Read all files as Buffers
  const fileBuffers = filePaths.map(filePath => readFileSync(filePath));
  const filenames = filePaths.map(path => path.basename(path));
  
  // Create args with multiple blob placeholders
  const args = [
    { $blob: 0 }, filenames[0], { source: "batch_upload", index: 0 },
    { $blob: 1 }, filenames[1], { source: "batch_upload", index: 1 },
    // ... additional files
  ];
  
  const uploadResult = await client.invoke_entity_method_with_blobs(
    entityId,
    "process_multiple_documents", // Custom entity method
    args,
    fileBuffers
  );
  
  return uploadResult;
}
```

## Working Memory Integration

### Storage Patterns

The `DocumentProcessorEntity.process_document` method automatically:

1. **Validates input**: Checks buffer and filename
2. **Uploads to working memory**: Via `WorkingMemoryProvider.add_memory_from_buffer`
3. **Returns metadata**: Including `working_memory_id` for retrieval
4. **Handles errors**: With proper error messages and logging

### Custom Working Memory Operations

For advanced use cases, access the working memory provider directly:

```typescript
export class AdvancedFileEntity extends DocumentProcessorEntity {
  /**
   * Custom working memory operation
   */
  public async store_processed_data(processedBuffer: Buffer, originalId: string) {
    const result = await this.working_memory_provider.add_memory_from_buffer({
      buffer: processedBuffer,
      metadata: {
        type: "processed_document",
        original_working_memory_id: originalId,
        processed_at: new Date().toISOString(),
      },
      memory_type: "file",
    });
    
    return result.workingMemoryId;
  }
  
  /**
   * Retrieve with custom metadata handling
   */
  public async get_file_with_history(working_memory_id: string) {
    const result = await this.working_memory_provider.get_binary_file_with_metadata_rpc(
      working_memory_id
    );
    
    return {
      buffer: result.buffer,
      metadata: result.metadata,
      processing_history: result.metadata.processing_history || [],
    };
  }
}
```

## Common Patterns

### 1. Entity with File Tracking

```typescript
public async process_document(
  document_buffer: Buffer,
  filename: string,
  metadata?: Record<string, unknown>
) {
  // Process with parent
  const result = await super.process_document(document_buffer, filename, metadata);
  
  // Track in entity data
  await this.track_upload(result.working_memory_id, filename, result.file_info);
  
  return result;
}

private async track_upload(workingMemoryId: string, filename: string, fileInfo: any) {
  const dto = await this.get_dto();
  const uploads = dto.data.uploads || [];
  
  uploads.push({
    id: workingMemoryId,
    filename,
    size: fileInfo.size,
    uploaded_at: new Date().toISOString(),
  });
  
  await this.update_data({ ...dto.data, uploads });
}
```

### 2. File Processing Pipeline

```typescript
public async process_with_pipeline(
  document_buffer: Buffer,
  filename: string,
  pipeline: string[]
) {
  // Store original
  const originalResult = await this.process_document(document_buffer, filename, {
    stage: "original",
    pipeline_id: crypto.randomUUID(),
  });
  
  let currentBuffer = document_buffer;
  let currentId = originalResult.working_memory_id;
  
  // Apply pipeline stages
  for (const stage of pipeline) {
    currentBuffer = await this.apply_processing_stage(currentBuffer, stage);
    
    // Store intermediate result
    const stageResult = await this.working_memory_provider.add_memory_from_buffer({
      buffer: currentBuffer,
      metadata: {
        stage,
        pipeline_id: originalResult.working_memory_id,
        previous_stage: currentId,
      },
    });
    
    currentId = stageResult.workingMemoryId;
  }
  
  return {
    original_id: originalResult.working_memory_id,
    final_id: currentId,
    pipeline_stages: pipeline,
  };
}
```

### 3. File Validation

```typescript
public async process_document(
  document_buffer: Buffer,
  filename: string,
  metadata?: Record<string, unknown>
) {
  // Validate file type
  const allowedTypes = ['.pdf', '.txt', '.docx', '.png', '.jpg'];
  const ext = path.extname(filename).toLowerCase();
  
  if (!allowedTypes.includes(ext)) {
    throw new Error(`File type not allowed: ${ext}`);
  }
  
  // Validate file size (10MB limit)
  if (document_buffer.length > 10 * 1024 * 1024) {
    throw new Error(`File too large: ${document_buffer.length} bytes`);
  }
  
  // Content validation (example: check for PDF header)
  if (ext === '.pdf' && !document_buffer.subarray(0, 4).equals(Buffer.from('%PDF'))) {
    throw new Error('Invalid PDF file');
  }
  
  return super.process_document(document_buffer, filename, {
    ...metadata,
    validated: true,
    file_type: ext,
  });
}
```

## Error Handling

### Server-Side Error Patterns

```typescript
public async process_document(
  document_buffer: Buffer,
  filename: string,
  metadata?: Record<string, unknown>
) {
  try {
    // Validation
    if (!Buffer.isBuffer(document_buffer)) {
      throw new Error("Invalid buffer provided");
    }
    
    if (!filename || typeof filename !== 'string') {
      throw new Error("Valid filename required");
    }
    
    // Process
    const result = await super.process_document(document_buffer, filename, metadata);
    
    logger.info("File upload successful", {
      entity_id: this.id,
      working_memory_id: result.working_memory_id,
      filename,
      size: document_buffer.length,
    });
    
    return result;
    
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : "Unknown error";
    
    logger.error("File upload failed", {
      entity_id: this.id,
      filename,
      error: errorMessage,
      buffer_size: document_buffer?.length || 0,
    });
    
    throw new Error(`File upload failed: ${errorMessage}`);
  }
}
```

### Client-Side Error Handling

```typescript
async function uploadWithRetry(filePath: string, entityId: string, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await uploadFile(filePath, entityId);
      return result;
      
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      
      // Handle specific error types
      if (errorMessage.includes("Cannot convert undefined to a BigInt")) {
        throw new Error("Server configuration error - form-data library issue");
      }
      
      if (errorMessage.includes("ECONNREFUSED")) {
        throw new Error("Server not accessible - check connection and port forwarding");
      }
      
      if (errorMessage.includes("File too large")) {
        throw new Error("File exceeds size limit - compress or split the file");
      }
      
      // Retry on temporary errors
      if (attempt < maxRetries && (
        errorMessage.includes("timeout") ||
        errorMessage.includes("ECONNRESET")
      )) {
        console.log(`Upload attempt ${attempt} failed, retrying...`);
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
        continue;
      }
      
      throw error;
    }
  }
}
```

## Troubleshooting

### Common Issues and Solutions

#### 1. "Cannot convert undefined to a BigInt"

**Cause**: Server using global `FormData` instead of Node.js `form-data` library
**Solution**: 
```typescript
// Client: Ensure using form-data library
import FormData from "form-data"; // Correct

// Server: Ensure ExpressTransport uses compatible multipart parsing
// This is handled automatically by the Agent SDK
```

#### 2. "Buffer serialization issues" 

**Cause**: Improper Buffer reconstruction on client side
**Solution**:
```typescript
// Always check Buffer format when retrieving
if (retrievedFile?.content?.type === 'Buffer' && Array.isArray(retrievedFile.content.data)) {
  const buffer = Buffer.from(retrievedFile.content.data);
} else if (Buffer.isBuffer(retrievedFile?.content)) {
  const buffer = retrievedFile.content;
} else {
  throw new Error("Invalid buffer format received");
}
```

#### 3. "[internal] Invalid URL" Context Service Error

**Cause**: Missing `http://` protocol in CONTEXT_SERVICE_ADDRESS
**Solution**:
```bash
# Wrong
CONTEXT_SERVICE_ADDRESS=localhost:50051

# Correct  
CONTEXT_SERVICE_ADDRESS=http://localhost:50051
```

#### 4. File Upload Size Limits

**Cause**: Default multer size limits exceeded
**Solution**: Configure limits in ExpressTransport (handled by Agent SDK):
```typescript
// The Agent SDK handles this automatically, but can be configured via:
// Environment variables or Agent Bundle configuration
```

#### 5. Working Memory Connection Issues

**Symptoms**: 
- Files upload but cannot be retrieved
- "working_memory_provider not accessible" errors

**Solution**:
```typescript
// Ensure DocumentProcessorEntity is properly initialized
// Check context service connectivity
// Verify CONTEXT_SERVICE_API_KEY is set
```

### Debug Logging

Enable detailed logging for troubleshooting:

```bash
# Environment variable
CONSOLE_LOG_LEVEL=debug

# Application logging
logger.debug("File upload debug info", {
  entity_id: this.id,
  filename,
  buffer_size: document_buffer.length,
  working_memory_provider_available: !!this.working_memory_provider,
});
```

## Complete Example

### Server Implementation (Complete Entity)

```typescript
import { 
  DocumentProcessorEntity,
  EntityDecorator,
  EntityFactory,
  logger,
} from "@firebrandanalytics/ff-agent-sdk";
import { UUID } from "@firebrandanalytics/shared-types";
import path from "path";

@EntityDecorator({
  generalType: "Instance",
  specificType: "CompleteFileUploadEntity",
})
export class CompleteFileUploadEntity extends DocumentProcessorEntity {
  constructor(factory: EntityFactory<any>, idOrDto: UUID | any) {
    super(factory, idOrDto);
  }

  public async process_document(
    document_buffer: Buffer,
    filename: string,
    metadata?: Record<string, unknown>
  ) {
    try {
      // Validation
      this.validateUpload(document_buffer, filename);
      
      logger.info("Processing file upload", {
        entity_id: this.id,
        filename,
        size: document_buffer.length,
      });

      // Process with parent
      const result = await super.process_document(document_buffer, filename, {
        ...metadata,
        entity_type: "CompleteFileUploadEntity",
        validated: true,
        file_extension: path.extname(filename),
      });

      // Track upload
      await this.trackUpload(result.working_memory_id, filename, result.file_info);

      return result;
      
    } catch (error) {
      logger.error("File upload failed", {
        entity_id: this.id,
        filename,
        error: error instanceof Error ? error.message : "Unknown error",
      });
      throw error;
    }
  }

  public async retrieve_file(working_memory_id: string) {
    try {
      const result = await this.working_memory_provider.get_binary_file_with_metadata_rpc(
        working_memory_id
      );

      return {
        working_memory_id,
        content: result.buffer,
        metadata: result.metadata,
        success: true,
        retrieved_at: new Date().toISOString(),
      };
    } catch (error) {
      throw new Error(`Failed to retrieve file: ${error instanceof Error ? error.message : "Unknown error"}`);
    }
  }

  public async list_uploads() {
    const dto = await this.get_dto();
    return {
      entity_id: this.id,
      uploads: dto.data.uploads || [],
      total_uploads: (dto.data.uploads || []).length,
    };
  }

  public async delete_upload(working_memory_id: string) {
    // Remove from working memory (if supported)
    // Update entity tracking
    const dto = await this.get_dto();
    const uploads = (dto.data.uploads || []).filter(
      (upload: any) => upload.working_memory_id !== working_memory_id
    );
    
    await this.update_data({
      ...dto.data,
      uploads,
    });
    
    return { success: true, working_memory_id };
  }

  public async test_upload_echo(message: string) {
    return {
      entity_id: this.id,
      message: `Echo: ${message}`,
      timestamp: new Date().toISOString(),
      upload_count: ((await this.get_dto()).data.uploads || []).length,
    };
  }

  private validateUpload(buffer: Buffer, filename: string) {
    if (!Buffer.isBuffer(buffer)) {
      throw new Error("Invalid buffer provided");
    }
    
    if (!filename || typeof filename !== 'string') {
      throw new Error("Valid filename required");
    }
    
    if (buffer.length === 0) {
      throw new Error("Empty file not allowed");
    }
    
    if (buffer.length > 50 * 1024 * 1024) { // 50MB limit
      throw new Error(`File too large: ${buffer.length} bytes`);
    }
  }

  private async trackUpload(workingMemoryId: string, filename: string, fileInfo: any) {
    const dto = await this.get_dto();
    const uploads = Array.isArray(dto.data.uploads) ? dto.data.uploads : [];
    
    uploads.push({
      working_memory_id: workingMemoryId,
      filename,
      size: fileInfo.size,
      uploaded_at: new Date().toISOString(),
      file_type: path.extname(filename),
    });
    
    await this.update_data({
      ...dto.data,
      uploads,
      last_upload_at: new Date().toISOString(),
    });
  }
}
```

### Client Implementation (Complete Example)

```typescript
import { RemoteAgentBundleClient } from "@firebrandanalytics/ff-sdk";
import { readFileSync, existsSync } from "fs";
import path from "path";

export class FileUploadClient {
  private client: RemoteAgentBundleClient;

  constructor(serverUrl: string, apiKey?: string) {
    this.client = new RemoteAgentBundleClient(serverUrl, {
      api_key: apiKey,
    });
  }

  async connect(): Promise<boolean> {
    try {
      const health = await this.client.health_check();
      const ready = await this.client.is_ready();
      return health && ready;
    } catch {
      return false;
    }
  }

  async getTestEntity(): Promise<string> {
    const response = await this.client.call_api_endpoint<{
      entity_id: string;
      message: string;
    }>("test-entity", { method: "GET" });

    if (!response?.entity_id) {
      throw new Error("No test entity available");
    }

    return response.entity_id;
  }

  async uploadFile(
    filePath: string, 
    entityId: string,
    metadata?: Record<string, unknown>
  ): Promise<any> {
    if (!existsSync(filePath)) {
      throw new Error(`File not found: ${filePath}`);
    }

    const fileBuffer = readFileSync(filePath);
    const filename = path.basename(filePath);

    console.log(`Uploading ${filename} (${fileBuffer.length} bytes)...`);

    const result = await this.client.invoke_entity_method_with_blobs(
      entityId,
      "process_document",
      [
        { $blob: 0 },
        filename,
        {
          source: "file_upload_client",
          original_path: filePath,
          ...metadata,
        }
      ],
      [fileBuffer]
    );

    console.log(`Upload complete: ${result.working_memory_id}`);
    return result;
  }

  async downloadFile(entityId: string, workingMemoryId: string): Promise<{
    content: Buffer | null;
    metadata: any;
    size: number;
  }> {
    const retrievedFile = await this.client.invoke_entity_method(
      entityId,
      "retrieve_file",
      [workingMemoryId]
    );

    // Handle Buffer reconstruction
    let fileContent: Buffer | null = null;
    
    if (retrievedFile?.content && typeof retrievedFile.content === 'object' && 
        retrievedFile.content.type === 'Buffer' && Array.isArray(retrievedFile.content.data)) {
      fileContent = Buffer.from(retrievedFile.content.data);
    } else if (Buffer.isBuffer(retrievedFile?.content)) {
      fileContent = retrievedFile.content;
    }

    return {
      content: fileContent,
      metadata: retrievedFile?.metadata,
      size: fileContent?.length || 0,
    };
  }

  async listUploads(entityId: string): Promise<any> {
    return await this.client.invoke_entity_method(
      entityId,
      "list_uploads",
      []
    );
  }

  async testConnection(entityId: string): Promise<any> {
    return await this.client.invoke_entity_method(
      entityId,
      "test_upload_echo",
      ["Connection test from client"]
    );
  }
}

// Usage Example
async function main() {
  const client = new FileUploadClient("http://localhost:3000", process.env.API_KEY);
  
  // Connect and verify
  if (!(await client.connect())) {
    throw new Error("Cannot connect to server");
  }
  
  // Get entity
  const entityId = await client.getTestEntity();
  console.log("Using entity:", entityId);
  
  // Test connection
  const echoResult = await client.testConnection(entityId);
  console.log("Echo test:", echoResult);
  
  // Upload file
  const uploadResult = await client.uploadFile("./test-file.txt", entityId, {
    category: "test",
    priority: "high",
  });
  
  // List uploads
  const uploads = await client.listUploads(entityId);
  console.log("All uploads:", uploads);
  
  // Download file
  const downloadResult = await client.downloadFile(
    entityId, 
    uploadResult.working_memory_id
  );
  console.log("Downloaded:", downloadResult.size, "bytes");
}

// Run example
main().catch(console.error);
```

---

This guide provides comprehensive coverage of FireFoundry file upload patterns, from basic implementations to advanced use cases. The patterns shown here enable robust binary file handling in distributed agent bundle architectures while maintaining proper error handling and integration with the working memory system.

For additional examples and troubleshooting, refer to the complete file upload example in the [FireFoundry Agent Bundle Examples repository](https://github.com/firebrandanalytics/ff-agent-bundle-examples).