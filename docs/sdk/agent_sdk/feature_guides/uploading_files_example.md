# Working with Binary Files and Working Memory

## Overview

Working Memory in FireFoundry allows entities to store and retrieve files, data, and artifacts that persist across entity executions. This tutorial shows how to:

1. Upload binary files (PDFs, Word docs, etc.) to working memory using the **Context Service Client**
2. Create entities and invoke them using the **Agent Bundle SDK** 
3. Access and read those files from within a runnable entity's `run_impl()` method
4. Process the files and store results back to working memory

We'll build a **Document Processor** entity that extracts text from uploaded documents.

## Prerequisites and Setup

You'll need to install the Context Service Client separately from your backend application:

```bash
npm install @firebrandanalytics/cs-client
npm install @firebrandanalytics/ff-agent-sdk
```

The Context Service Client handles working memory operations, while the Agent Bundle SDK handles entity invocation and AI processing.

## Understanding Working Memory

Working Memory provides these key operations:
- **Store files**: `insert_code_memory()` - Save files with metadata
- **List files**: `get_working_memory_manifest_rpc()` - Get all files for an entity
- **Read text content**: `get_memory_content_rpc()` - Retrieve text content as string
- **Read binary content**: `get_binary_file_rpc()` - Retrieve binary files as Buffer
- **Organize by paths**: Files are organized with path-like names for easy retrieval

## Creating the Document Processor Entity

First, let's create an entity that can accept file uploads and process them:

```typescript
import {
  AddInterface,
  EntityNode,
  EntityNodeTypeHelper,
  EntityFactory,
  RunnableEntityDecorator,
  RunnableEntityClass,
  RunnableEntityTypeHelper,
  RunnableEntityResponseIterator,
  IRunnableEntity
} from '@firebrandanalytics/ff-agent-sdk';
import {
  type UUID,
  type EntityInstanceNodeDTO,
  type JSONValue
} from '@firebrandanalytics/shared-types';

// Define the DTO structure
export interface DocumentProcessorDTOData {
  input_file_path?: string;  // Path to the uploaded file in working memory
  output_text_path?: string; // Path where extracted text will be stored
  file_type?: string;        // Type of document (pdf, docx, etc.)
  processing_status?: string;
  [key: string]: JSONValue;
}

export type DocumentProcessorDTO = EntityInstanceNodeDTO<DocumentProcessorDTOData> & {
  node_type: "DocumentProcessor";
};

// Define output type
interface ProcessingResult {
  extracted_text: string;
  file_info: {
    original_name: string;
    file_size: number;
    file_type: string;
  };
  processing_metadata: {
    processing_time_ms: number;
    character_count: number;
    word_count: number;
  };
}

// Define the runnable entity type helper
type DOC_PROCESSOR_RETH = RunnableEntityTypeHelper<
  EntityNodeTypeHelper<
    any,
    DocumentProcessorDTO,
    "DocumentProcessor",
    {},
    {}
  >,
  ProcessingResult,
  {}
>;

@RunnableEntityDecorator({
  generalType: "DocumentProcessor",
  specificType: "DocumentProcessor",
  allowedConnections: {}
})
export class DocumentProcessorEntity
  extends AddInterface<
    typeof RunnableEntityClass<DOC_PROCESSOR_RETH["enh"]>,
    IRunnableEntity<any, ProcessingResult>
  >(RunnableEntityClass<DOC_PROCESSOR_RETH["enh"]>)
{
  constructor(factory: EntityFactory<any>, idOrDto: UUID | DocumentProcessorDTO) {
    super(factory, idOrDto);
  }

  // Main processing implementation
  protected async *run_impl(): RunnableEntityResponseIterator<DOC_PROCESSOR_RETH["enh"], ProcessingResult> {
    const startTime = Date.now();
    
    yield await this.createStatusEnvelope('STARTED', 'Beginning document processing');

    try {
      // Step 1: Get the working memory provider
      const wmp = this.factory.entity_client.working_memory_provider;
      if (!wmp) {
        throw new Error('Working memory provider not available');
      }

      // Step 2: Get our entity DTO to find the input file path
      const dto = await this.get_dto();
      const inputFilePath = dto.data.input_file_path;
      
      if (!inputFilePath) {
        throw new Error('No input file path specified');
      }

      yield await this.createStatusEnvelope('PROCESSING', 'Loading input file from working memory');

      // Step 3: Get the working memory manifest to find our file
      const memory_manifest = await wmp.get_working_memory_manifest_rpc(this.id);
      
      // Step 4: Find the specific file in the manifest
      const fileEntry = memory_manifest.items.find(item => 
        item.path === inputFilePath || item.path.endsWith(inputFilePath)
      );
      
      if (!fileEntry) {
        throw new Error(`File not found in working memory: ${inputFilePath}`);
      }

      yield await this.createStatusEnvelope('PROCESSING', `Found file: ${fileEntry.path} (${fileEntry.size} bytes)`);

      // Step 5: Read the binary content
      const fileBuffer = await wmp.get_binary_file_rpc(fileEntry.id);
      
      yield await this.createStatusEnvelope('PROCESSING', 'Extracting text from document');

      // Step 6: Extract text (placeholder implementation)
      const extractedText = await this.extract_text(fileBuffer, dto.data.file_type || 'unknown');
      
      // Step 7: Prepare result data
      const processingTime = Date.now() - startTime;
      const wordCount = extractedText.split(/\s+/).filter(word => word.length > 0).length;
      
      const result: ProcessingResult = {
        extracted_text: extractedText,
        file_info: {
          original_name: fileEntry.path.split('/').pop() || 'unknown',
          file_size: fileEntry.size,
          file_type: dto.data.file_type || 'unknown'
        },
        processing_metadata: {
          processing_time_ms: processingTime,
          character_count: extractedText.length,
          word_count: wordCount
        }
      };

      yield await this.createStatusEnvelope('PROCESSING', 'Saving extracted text to working memory');

      // Step 8: Save the extracted text back to working memory
      const outputPath = dto.data.output_text_path || `extracted_text_${Date.now()}.txt`;
      
      const { workingMemoryId } = await wmp.add_memory_from_buffer({
        entityNodeId: this.id,
        name: outputPath,
        memoryType: 'text/plain',
        description: `Extracted text from ${fileEntry.path}`,
        contentType: 'file',
        buffer: Buffer.from(extractedText, 'utf-8'),
        metadata: {
          source_file: fileEntry.path,
          source_file_id: fileEntry.id,
          extraction_timestamp: new Date().toISOString(),
          processing_time_ms: processingTime
        }
      });

      // Update entity with output path
      await this.update_data({
        ...dto.data,
        output_text_path: outputPath,
        processing_status: 'completed'
      });

      yield await this.createStatusEnvelope('COMPLETED', `Text extraction complete. Saved to ${outputPath}`);

      return result;

    } catch (error) {
      yield await this.createStatusEnvelope('FAILED', `Processing failed: ${error.message}`);
      throw error;
    }
  }

  // Placeholder text extraction function
  private async extract_text(fileBuffer: Buffer, fileType: string): Promise<string> {
    // This is where you would implement actual text extraction
    // For different file types, you might use different libraries:
    
    switch (fileType.toLowerCase()) {
      case 'pdf':
        // return await extractPdfText(fileBuffer);
        return `[EXTRACTED PDF TEXT]\nThis would be the actual text content extracted from a PDF file.\nBuffer size: ${fileBuffer.length} bytes\nFile type: ${fileType}`;
      
      case 'docx':
      case 'doc':
        // return await extractWordText(fileBuffer);
        return `[EXTRACTED WORD TEXT]\nThis would be the actual text content extracted from a Word document.\nBuffer size: ${fileBuffer.length} bytes\nFile type: ${fileType}`;
      
      case 'txt':
        // For plain text, just convert buffer to string
        return fileBuffer.toString('utf-8');
      
      default:
        return `[UNSUPPORTED FILE TYPE: ${fileType}]\nBuffer size: ${fileBuffer.length} bytes\nRaw content preview: ${fileBuffer.toString('utf-8', 0, Math.min(200, fileBuffer.length))}...`;
    }
  }

  // Helper methods for working with the entity
  async set_input_file(filePath: string, fileType?: string): Promise<void> {
    await this.update_data({
      ...(this._dto?.data || {}),
      input_file_path: filePath,
      file_type: fileType,
      processing_status: 'pending'
    });
  }

  async get_processing_status(): Promise<string> {
    const dto = await this.get_dto();
    return dto.data.processing_status || 'not_started';
  }
}
```

## Complete Usage Example

Now let's show the proper workflow where file upload happens **outside the Agent Bundle** using the **Context Service Client**, and the Agent Bundle reads the file during execution:

```typescript
import fs from 'fs';
import { ContextServiceClient } from '@firebrandanalytics/ff-sdk';
import { RemoteAppRunnerClient } from '@firebrandanalytics/ff-agent-sdk';

/**
 * External workflow showing the proper separation:
 * 1. External app creates entity via Agent Bundle API
 * 2. External app uploads file using Context Service Client
 * 3. External app invokes entity.run() via Agent Bundle API
 * 4. Agent Bundle reads file from working memory during run_impl
 */
async function processDocumentWorkflow(
  contextServiceUrl: string,
  agentBundleUrl: string,
  apiKey: string,
  filePath: string,
  fileName: string,
  fileType: string
): Promise<ProcessingResult> {
  
  // Step 1: Read the file from filesystem
  console.log(`📖 Reading file: ${filePath}`);
  const fileBuffer = fs.readFileSync(filePath);
  console.log(`📄 File loaded: ${fileName} (${fileBuffer.length} bytes)`);

  // Step 2: Initialize clients
  const contextClient = new ContextServiceClient({
    address: contextServiceUrl,
    apiKey: apiKey
  });

  const agentClient = new RemoteAppRunnerClient(agentBundleUrl, apiKey);

  // Step 3: Create DocumentProcessor entity via Agent Bundle
  console.log(`🏗️ Creating DocumentProcessor entity via Agent Bundle...`);
  
  // This would call a method on a collection entity to create the processor
  const createResult = await agentClient.invoke_entity_method(
    collectionEntityId, // Pre-existing collection entity
    'create_document_processor',
    { file_type: fileType }
  );
  
  const processorEntityId = createResult.entity_id;
  console.log(`✅ Created entity: ${processorEntityId}`);

  // Step 4: Upload file to working memory using Context Service Client
  console.log(`📤 Uploading file to working memory using Context Service Client...`);
  
  const inputFilePath = `uploads/${fileName}`;
  
  // Upload the blob first
  const blobResult = await contextClient.uploadBlob({
    content: fileBuffer,
    contentType: getContentTypeFromFileType(fileType),
    metadata: {
      original_filename: fileName,
      file_type: fileType,
      upload_timestamp: new Date().toISOString(),
      file_size: fileBuffer.length
    }
  });

  console.log(`✅ Blob uploaded: ${blobResult.blobId}`);

  // Create working memory record referencing the blob
  const wmResult = await contextClient.insertWMRecord({
    entityNodeId: processorEntityId,  // Associate with our processor entity
    name: inputFilePath,
    memoryType: getMemoryTypeFromFileType(fileType),
    description: `Uploaded file: ${fileName}`,
    blobId: blobResult.blobId,
    metadata: {
      original_filename: fileName,
      file_type: fileType,
      upload_timestamp: new Date().toISOString()
    }
  });

  console.log(`✅ Working memory record created: ${wmResult.workingMemoryId}`);

  // Step 5: Configure entity with file reference via Agent Bundle
  console.log(`🔗 Configuring entity with file reference...`);
  await agentClient.invoke_entity_method(
    processorEntityId,
    'set_input_file',
    inputFilePath,
    fileType
  );

  // Step 6: Run the entity via Agent Bundle (this will trigger run_impl)
  console.log(`🚀 Running document processor...`);
  const result = await agentClient.invoke_entity_method(
    processorEntityId,
    'run'
  );
  
  console.log('✅ Document processing complete');
  return result;
}

// Helper functions for MIME types (same as before)
function getMemoryTypeFromFileType(fileType: string): string {
  switch (fileType.toLowerCase()) {
    case 'pdf': return 'application/pdf';
    case 'docx': return 'application/vnd.openxmlformats-officedocument.wordprocessingml.document';
    case 'doc': return 'application/msword';
    case 'txt': return 'text/plain';
    default: return 'application/octet-stream';
  }
}

function getContentTypeFromFileType(fileType: string): string {
  return getMemoryTypeFromFileType(fileType);
}

/**
 * Alternative: Upload-then-process pattern
 * Upload first, then create and run entity in one call
 */
async function uploadThenProcess(
  contextServiceUrl: string,
  agentBundleUrl: string,
  apiKey: string,
  filePath: string,
  fileName: string,
  fileType: string
): Promise<ProcessingResult> {
  
  const fileBuffer = fs.readFileSync(filePath);
  const contextClient = new ContextServiceClient({
    address: contextServiceUrl,
    apiKey: apiKey
  });
  const agentClient = new RemoteAppRunnerClient(agentBundleUrl, apiKey);
  
  // 1. Upload file using Context Service Client
  const inputFilePath = `uploads/${fileName}`;
  
  // Upload blob
  const blobResult = await contextClient.uploadBlob({
    content: fileBuffer,
    contentType: getContentTypeFromFileType(fileType)
  });

  // Create working memory record for a temporary entity
  const tempEntityId = `temp-${Date.now()}`;
  await contextClient.insertWMRecord({
    entityNodeId: tempEntityId,
    name: inputFilePath,
    blobId: blobResult.blobId,
    metadata: { original_filename: fileName, file_type: fileType }
  });

  // 2. Create and run processor in one call, passing file reference
  return await agentClient.invoke_entity_method(
    collectionEntityId,
    'process_uploaded_file',
    inputFilePath,
    fileType,
    { temp_entity_id: tempEntityId } // To transfer working memory ownership
  );
}

/**
 * Simplified example for testing
 */
async function quickProcessTest(): Promise<void> {
  try {
    const result = await processDocumentWorkflow(
      'http://localhost:50051',   // Context Service URL
      'http://localhost:3001',    // Agent Bundle URL
      'your-api-key',             // API key for both services
      './test-documents/sample.pdf',
      'sample.pdf',
      'pdf'
    );
    
    console.log('Processing Results:');
    console.log(`- Text Length: ${result.extracted_text.length} characters`);
    console.log(`- Word Count: ${result.processing_metadata.word_count} words`);
    console.log(`- Processing Time: ${result.processing_metadata.processing_time_ms}ms`);
    console.log(`- File Size: ${result.file_info.file_size} bytes`);
    
  } catch (error) {
    console.error('Processing failed:', error);
  }
}

## Architectural Separation: cs-client vs ff-agent-sdk

It's important to understand the separation between the Context Service Client and Agent Bundle SDK:

### @firebrandanalytics/cs-client (Context Service Client)
- **Purpose**: Infrastructure operations (working memory, blobs, tools, RAG)
- **Used by**: External applications, backend services, upload handlers
- **Working Memory**: Read and write access via `insertWMRecord`, `fetchWMRecord`
- **Blob Storage**: Upload and download via `uploadBlob`, `getBlob`
- **Example use**: Upload files, manage working memory from external systems

### @firebrandanalytics/ff-agent-sdk (Agent Bundle SDK) 
- **Purpose**: AI agent logic, entity definitions, bot behaviors
- **Used by**: Agent Bundle implementations, entity run_impl methods
- **Working Memory**: Read-only access during entity execution
- **Example use**: Process files that were uploaded externally

### Typical Production Architecture

```
External Application (cs-client)
├── Upload blobs to Context Service
├── Create working memory records
├── Create entities via Agent Bundle API
└── Invoke entity methods

Agent Bundle (ff-agent-sdk)  
├── Define entity and bot logic
├── Read files from working memory
└── Process and return results
```

### Authentication

Both clients support API key authentication:

```typescript
// Context Service Client with API key
const contextClient = new ContextServiceClient({
  address: 'http://localhost:50051',
  apiKey: 'your-api-key'
});

// Agent Bundle Client with API key
const agentClient = new RemoteAppRunnerClient(
  'http://localhost:3001',
  'your-api-key'
);
```

This separation ensures:
- **Clean boundaries**: File handling separated from business logic
- **Scalability**: Agent Bundles focus on AI processing, not file I/O
- **Security**: Upload validation happens outside the Agent Bundle
- **Flexibility**: Multiple applications can upload files to the same Agent Bundle

### Choosing the Right Method

When reading from working memory, choose the appropriate method based on your content type:

- **`get_binary_file_rpc(wm_id)`**: Use for binary files like PDFs, Word docs, images, Excel files, etc.
- **`get_memory_content_rpc(wm_id)`**: Use for text-based content like JSON, plain text, CSV, etc.
- **`get_binary_file_with_metadata_rpc(wm_id)`**: Use for binary files when you also need metadata
- **`get_memory_content_with_metadata_rpc(wm_id)`**: Use for text content when you also need metadata

/**
 * Simplified workflow for testing
 */
async function quickProcessTest(): Promise<void> {
  try {
    // Example usage
    const result = await processDocumentWorkflow(
      './test-documents/sample.pdf',
      'sample.pdf',
      'pdf'
    );
    
    console.log('Processing Results:');
    console.log(`- Text Length: ${result.extracted_text.length} characters`);
    console.log(`- Word Count: ${result.processing_metadata.word_count} words`);
    console.log(`- Processing Time: ${result.processing_metadata.processing_time_ms}ms`);
    console.log(`- File Size: ${result.file_info.file_size} bytes`);
    
  } catch (error) {
    console.error('Processing failed:', error);
  }
}
```

## Key Working Memory Patterns

### External File Upload (using Context Service Client)

```typescript
// External applications upload files using Context Service Client
import { ContextServiceClient } from '@firebrandanalytics/ff-sdk';

const contextClient = new ContextServiceClient({
  address: 'http://localhost:50051',
  apiKey: 'your-api-key'
});

// Step 1: Upload blob
const blobResult = await contextClient.uploadBlob({
  content: fileBuffer,
  contentType: 'application/pdf',
  metadata: { /* blob metadata */ }
});

// Step 2: Create working memory record
const wmResult = await contextClient.insertWMRecord({
  entityNodeId: targetEntityId,
  name: 'uploads/document.pdf',
  blobId: blobResult.blobId,
  memoryType: 'application/pdf',
  metadata: { /* working memory metadata */ }
});
```

### Agent Bundle File Access (using ff-agent-sdk)

```typescript
// From within a runnable entity's run_impl() - READ ONLY
const wmp = this.factory.entity_client.working_memory_provider;

// Read file manifest
const manifest = await wmp.get_working_memory_manifest_rpc(this.id);

// Read file content - choose the appropriate method:
// For binary files (PDFs, Word docs, images, etc.)
const fileBuffer = await wmp.get_binary_file_rpc(fileId);

// For text content (JSON, plain text, etc.)
const textContent = await wmp.get_memory_content_rpc(fileId);

// With metadata (returns content + metadata object)
const { buffer, metadata } = await wmp.get_binary_file_with_metadata_rpc(fileId);
const { content, metadata } = await wmp.get_memory_content_with_metadata_rpc(fileId);
```

### Complete External Workflow Pattern

```typescript
// EXTERNAL APPLICATION using cs-client + ff-agent-sdk

// 1. Initialize clients
const contextClient = new ContextServiceClient({
  address: contextServiceUrl,
  apiKey: apiKey
});
const agentClient = new RemoteAppRunnerClient(agentBundleUrl, apiKey);

// 2. Upload file using Context Service Client
const blobResult = await contextClient.uploadBlob({
  content: fileBuffer,
  contentType: 'application/pdf'
});

const wmResult = await contextClient.insertWMRecord({
  entityNodeId: processorEntityId,
  name: 'uploads/document.pdf',
  blobId: blobResult.blobId
});

// 3. Create/configure entity using Agent Bundle API
const entityId = await agentClient.invoke_entity_method(
  collectionId, 'create_processor', { file_type: 'pdf' }
);

// 4. Configure entity with file reference
await agentClient.invoke_entity_method(
  entityId, 'set_input_file', 'uploads/document.pdf', 'pdf'
);

// 5. Run entity (triggers run_impl which reads from working memory)
const result = await agentClient.invoke_entity_method(entityId, 'run');
```

### Storing Files (using Context Service Client)

```typescript
// Step 1: Upload blob to Context Service
const blobResult = await contextClient.uploadBlob({
  content: fileBuffer,              // Binary data as Buffer
  contentType: 'application/pdf',   // MIME type
  metadata: {                       // Additional metadata
    original_filename: 'document.pdf',
    upload_timestamp: new Date().toISOString(),
    file_size: fileBuffer.length
  }
});

// Step 2: Create working memory record
const wmResult = await contextClient.insertWMRecord({
  entityNodeId: entityId,           // Entity that owns this file
  name: 'path/to/file.pdf',        // File path/name
  memoryType: 'application/pdf',   // MIME type
  description: 'User uploaded PDF', // Human description
  blobId: blobResult.blobId,       // Reference to uploaded blob
  metadata: {                      // Additional metadata
    category: 'user_upload',
    department: 'legal'
  }
});
```

### Reading Files

```typescript
// Get manifest of all files for an entity
const manifest = await wmp.get_working_memory_manifest_rpc(entityId);

// Find specific file
const fileEntry = manifest.items.find(item => 
  item.path.endsWith('target_file.pdf')
);

// Read file content
const fileBuffer = await wmp.get_binary_file_rpc(fileEntry.id);
```

### File Organization

Working memory supports path-like organization:
```
uploads/
  ├── document1.pdf
  ├── document2.docx
  └── images/
      ├── chart1.png
      └── chart2.png

outputs/
  ├── extracted_text_1.txt
  ├── extracted_text_2.txt
  └── summaries/
      └── summary_1.json
```

```

## Usage Examples

### Basic Document Processing

```typescript
// Process a local PDF file
const result = await processDocumentWorkflow(
  'http://localhost:50051',       // Context Service URL
  'http://localhost:3001',        // Agent Bundle URL
  'your-api-key',                 // API key
  './documents/quarterly-report.pdf',
  'quarterly-report.pdf', 
  'pdf'
);

console.log(`Extracted ${result.processing_metadata.word_count} words in ${result.processing_metadata.processing_time_ms}ms`);
```

### Batch Processing Multiple Files

```typescript
async function processBatchDocuments(
  contextServiceUrl: string,
  agentBundleUrl: string,
  apiKey: string,
  files: Array<{path: string, name: string, type: string}>
) {
  const results = [];
  
  for (const file of files) {
    try {
      const result = await processDocumentWorkflow(
        contextServiceUrl,
        agentBundleUrl,
        apiKey,
        file.path,
        file.name,
        file.type
      );
      results.push({ file: file.name, success: true, result });
    } catch (error) {
      results.push({ file: file.name, success: false, error: error.message });
    }
  }
  
  return results;
}

// Process multiple documents
const batchResults = await processBatchDocuments(
  'http://localhost:50051',       // Context Service URL
  'http://localhost:3001',        // Agent Bundle URL
  'your-api-key',                 // API key
  [
    { path: './docs/contract.pdf', name: 'contract.pdf', type: 'pdf' },
    { path: './docs/proposal.docx', name: 'proposal.docx', type: 'docx' },
    { path: './docs/notes.txt', name: 'notes.txt', type: 'txt' }
  ]
);
```

## Best Practices

**1. Architectural Separation**
- Use ff-sdk for file uploads from external applications
- Use ff-agent-sdk for reading files within Agent Bundle entities  
- Keep file I/O separate from AI/business logic
- Validate files before uploading, not during entity processing

**2. File Organization**
- Use consistent path patterns (`uploads/`, `processed/`, `outputs/`)
- Include timestamps in generated file names
- Use descriptive metadata for searchability
- Organize by entity or workflow type

**3. Error Handling and Validation**
- Validate file types, sizes, and content before upload (ff-sdk side)
- Handle missing files gracefully in run_impl (ff-agent-sdk side)
- Implement proper error propagation between external app and Agent Bundle
- Use working memory metadata to track processing status

**4. Security Considerations**
- Scan files for malicious content before upload
- Implement proper access controls in working memory provider
- Use secure file transfer protocols for sensitive documents
- Consider encryption for confidential files

**5. Performance and Scalability**
- Stream large files when possible during upload
- Use appropriate timeouts for file processing
- Consider file processing queues for high-volume scenarios
- Clean up working memory periodically to manage storage

This pattern enables robust document processing workflows that maintain clear architectural boundaries between file handling (ff-sdk) and AI processing (ff-agent-sdk), making systems more maintainable and scalable.