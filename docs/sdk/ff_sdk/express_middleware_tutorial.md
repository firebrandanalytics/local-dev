# Express.js Middleware for News Analysis with WebSocket Streaming

## Introduction

This tutorial demonstrates how to build an Express.js middleware layer between your consumers and the FireFoundry agent bundle. The middleware provides:

- **REST API**: Proxy endpoints for article analysis
- **WebSocket Streaming**: Real-time progress updates during analysis
- **Connection Management**: Handle multiple concurrent clients
- **Error Handling**: Robust error handling and recovery
- **CORS Support**: Cross-origin resource sharing for web clients

This pattern is ideal when you need to:
- Add authentication/authorization before reaching the agent bundle
- Transform or enrich requests/responses
- Broadcast analysis progress to multiple clients
- Implement rate limiting or caching
- Provide a unified API for multiple agent bundles

**Prerequisites:**
- News Analysis Agent Bundle deployed and accessible
- Node.js and TypeScript environment
- Basic understanding of Express.js and WebSockets

---

## Part 1: Basic Express Middleware

### Installation

```bash
npm install express ws cors
npm install @firebrandanalytics/ff-sdk
npm install --save-dev @types/express @types/ws @types/cors
```

### Project Structure

```
src/
├── server.ts           # Main server setup
├── routes/
│   └── analysis.ts     # REST API routes
├── websocket/
│   ├── manager.ts      # WebSocket connection manager
│   └── handlers.ts     # WebSocket message handlers
├── middleware/
│   ├── auth.ts         # Authentication middleware
│   └── errorHandler.ts # Error handling middleware
├── services/
│   └── analysisService.ts # Agent bundle interaction
└── types/
    └── index.ts        # Shared types
```

### Configuration (`src/config.ts`)

```typescript
export interface AppConfig {
  port: number;
  agentBundleUrl: string;
  agentBundleApiKey?: string;
  collectionId: string;
  corsOrigins: string[];
}

export function loadConfig(): AppConfig {
  return {
    port: parseInt(process.env.PORT || '3000'),
    agentBundleUrl: process.env.AGENT_BUNDLE_URL || 'http://localhost:3001',
    agentBundleApiKey: process.env.AGENT_BUNDLE_API_KEY,
    collectionId: process.env.NEWS_COLLECTION_ID || 'c0000000-0000-0000-0001-000000000001',
    corsOrigins: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000']
  };
}
```

### Analysis Service (`src/services/analysisService.ts`)

Encapsulate agent bundle communication:

```typescript
import { RemoteAgentBundleClient, FFError } from '@firebrandanalytics/ff-sdk';
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';
import { loadConfig } from '../config.js';

export class AnalysisService {
  private client: RemoteAgentBundleClient;
  private collectionId: string;

  constructor() {
    const config = loadConfig();
    
    this.client = new RemoteAgentBundleClient(config.agentBundleUrl, {
      api_key: config.agentBundleApiKey,
      timeout: 120000 // 2 minutes for long-running analyses
    });
    
    this.collectionId = config.collectionId;
  }

  /**
   * Analyze a single article (synchronous)
   */
  async analyzeArticle(
    articleText: string,
    metadata?: { source_url?: string; published_date?: string }
  ): Promise<IMPACT_ANALYSIS_OUTPUT> {
    try {
      const result = await this.client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
        this.collectionId,
        'analyze_article',
        articleText,
        metadata
      );
      
      return result;
    } catch (error) {
      if (error instanceof FFError) {
        throw new Error(`Analysis failed: ${error.message}`);
      }
      throw error;
    }
  }

  /**
   * Analyze article with streaming (using iterator)
   * This is a future enhancement when the agent bundle supports streaming
   */
  async analyzeArticleStreaming(
    articleText: string,
    metadata?: { source_url?: string; published_date?: string },
    progressCallback?: (progress: any) => void
  ): Promise<IMPACT_ANALYSIS_OUTPUT> {
    try {
      // For now, simulate streaming with intermediate progress updates
      progressCallback?.({ type: 'status', message: 'Starting analysis...' });
      
      const result = await this.client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
        this.collectionId,
        'analyze_article',
        articleText,
        metadata
      );
      
      progressCallback?.({ type: 'complete', result });
      return result;
    } catch (error) {
      progressCallback?.({ type: 'error', error: error instanceof Error ? error.message : 'Unknown error' });
      throw error;
    }
  }

  /**
   * Get analysis history
   */
  async getHistory(limit?: number): Promise<any[]> {
    try {
      const articles = await this.client.invoke_entity_method<any[]>(
        this.collectionId,
        'get_all_articles'
      );
      
      return limit ? articles.slice(0, limit) : articles;
    } catch (error) {
      if (error instanceof FFError) {
        throw new Error(`Failed to fetch history: ${error.message}`);
      }
      throw error;
    }
  }

  /**
   * Health check
   */
  async healthCheck(): Promise<{ healthy: boolean; ready: boolean; info?: any }> {
    try {
      const healthy = await this.client.health_check();
      const ready = await this.client.is_ready();
      const info = healthy && ready ? await this.client.get_info() : undefined;
      
      return { healthy, ready, info };
    } catch (error) {
      return { healthy: false, ready: false };
    }
  }
}
```

### REST API Routes (`src/routes/analysis.ts`)

```typescript
import express, { Request, Response, NextFunction } from 'express';
import { AnalysisService } from '../services/analysisService.js';

const router = express.Router();
const analysisService = new AnalysisService();

/**
 * POST /api/analyze
 * Analyze a single article
 */
router.post('/analyze', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { article, metadata } = req.body;

    if (!article || typeof article !== 'string') {
      return res.status(400).json({
        error: 'Missing or invalid article text'
      });
    }

    console.log(`Analyzing article (${article.length} chars)...`);

    const result = await analysisService.analyzeArticle(article, metadata);

    res.json({
      success: true,
      data: result
    });
  } catch (error) {
    next(error);
  }
});

/**
 * POST /api/batch
 * Analyze multiple articles
 */
router.post('/batch', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { articles } = req.body;

    if (!Array.isArray(articles)) {
      return res.status(400).json({
        error: 'articles must be an array'
      });
    }

    console.log(`Batch analyzing ${articles.length} articles...`);

    const results = await Promise.allSettled(
      articles.map(item => 
        analysisService.analyzeArticle(
          item.article,
          item.metadata
        )
      )
    );

    const successful = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');

    res.json({
      success: true,
      data: {
        total: articles.length,
        successful: successful.length,
        failed: failed.length,
        results: results.map(r => 
          r.status === 'fulfilled' 
            ? { success: true, data: r.value }
            : { success: false, error: (r.reason as Error).message }
        )
      }
    });
  } catch (error) {
    next(error);
  }
});

/**
 * GET /api/history
 * Get analysis history
 */
router.get('/history', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const limit = req.query.limit ? parseInt(req.query.limit as string) : undefined;

    const history = await analysisService.getHistory(limit);

    res.json({
      success: true,
      data: {
        total: history.length,
        articles: history
      }
    });
  } catch (error) {
    next(error);
  }
});

/**
 * GET /api/health
 * Health check endpoint
 */
router.get('/health', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const health = await analysisService.healthCheck();

    res.status(health.healthy && health.ready ? 200 : 503).json({
      success: health.healthy && health.ready,
      data: health
    });
  } catch (error) {
    next(error);
  }
});

export default router;
```

### Error Handling Middleware (`src/middleware/errorHandler.ts`)

```typescript
import { Request, Response, NextFunction } from 'express';

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error('Error:', error);

  // Don't send error response if headers already sent
  if (res.headersSent) {
    return next(error);
  }

  res.status(500).json({
    success: false,
    error: error.message || 'Internal server error'
  });
}

export function notFoundHandler(
  req: Request,
  res: Response
) {
  res.status(404).json({
    success: false,
    error: 'Endpoint not found'
  });
}
```

### Main Server (`src/server.ts`)

```typescript
import express from 'express';
import cors from 'cors';
import { loadConfig } from './config.js';
import analysisRoutes from './routes/analysis.js';
import { errorHandler, notFoundHandler } from './middleware/errorHandler.js';

const config = loadConfig();
const app = express();

// Middleware
app.use(cors({
  origin: config.corsOrigins,
  credentials: true
}));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Request logging
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  next();
});

// Routes
app.use('/api', analysisRoutes);

// Error handling
app.use(notFoundHandler);
app.use(errorHandler);

// Start server
app.listen(config.port, () => {
  console.log(`\n🚀 News Analysis Middleware running on port ${config.port}`);
  console.log(`   Agent Bundle URL: ${config.agentBundleUrl}`);
  console.log(`   Collection ID: ${config.collectionId}`);
  console.log(`\n📡 API Endpoints:`);
  console.log(`   POST   http://localhost:${config.port}/api/analyze`);
  console.log(`   POST   http://localhost:${config.port}/api/batch`);
  console.log(`   GET    http://localhost:${config.port}/api/history`);
  console.log(`   GET    http://localhost:${config.port}/api/health`);
  console.log(`\n🔌 WebSocket: ws://localhost:${config.port}\n`);
});

export default app;
```

---

## Part 2: WebSocket Streaming

### WebSocket Connection Manager (`src/websocket/manager.ts`)

```typescript
import WebSocket from 'ws';
import { IncomingMessage } from 'http';
import { v4 as uuidv4 } from 'uuid';

export interface ClientConnection {
  id: string;
  ws: WebSocket;
  isAlive: boolean;
  metadata?: Record<string, any>;
}

export class WebSocketManager {
  private clients: Map<string, ClientConnection> = new Map();
  private heartbeatInterval: NodeJS.Timeout | null = null;

  constructor(private wss: WebSocket.Server) {
    this.setupHeartbeat();
    this.setupConnection();
  }

  /**
   * Setup new WebSocket connections
   */
  private setupConnection() {
    this.wss.on('connection', (ws: WebSocket, req: IncomingMessage) => {
      const clientId = uuidv4();
      const client: ClientConnection = {
        id: clientId,
        ws,
        isAlive: true
      };

      this.clients.set(clientId, client);
      console.log(`WebSocket client connected: ${clientId} (Total: ${this.clients.size})`);

      // Send welcome message
      this.sendToClient(clientId, {
        type: 'connected',
        clientId,
        timestamp: new Date().toISOString()
      });

      // Setup pong response
      ws.on('pong', () => {
        const client = this.clients.get(clientId);
        if (client) {
          client.isAlive = true;
        }
      });

      // Handle client disconnection
      ws.on('close', () => {
        console.log(`WebSocket client disconnected: ${clientId}`);
        this.clients.delete(clientId);
      });

      // Handle errors
      ws.on('error', (error) => {
        console.error(`WebSocket error for client ${clientId}:`, error);
        this.clients.delete(clientId);
      });
    });
  }

  /**
   * Setup heartbeat to detect stale connections
   */
  private setupHeartbeat() {
    this.heartbeatInterval = setInterval(() => {
      this.clients.forEach((client, clientId) => {
        if (!client.isAlive) {
          console.log(`Terminating stale connection: ${clientId}`);
          client.ws.terminate();
          this.clients.delete(clientId);
          return;
        }

        client.isAlive = false;
        client.ws.ping();
      });
    }, 30000); // 30 seconds
  }

  /**
   * Send message to specific client
   */
  sendToClient(clientId: string, message: any): boolean {
    const client = this.clients.get(clientId);
    if (!client || client.ws.readyState !== WebSocket.OPEN) {
      return false;
    }

    try {
      client.ws.send(JSON.stringify(message));
      return true;
    } catch (error) {
      console.error(`Failed to send to client ${clientId}:`, error);
      return false;
    }
  }

  /**
   * Broadcast message to all connected clients
   */
  broadcast(message: any, excludeClientId?: string) {
    const payload = JSON.stringify(message);
    let sent = 0;

    this.clients.forEach((client, clientId) => {
      if (clientId === excludeClientId) return;
      if (client.ws.readyState !== WebSocket.OPEN) return;

      try {
        client.ws.send(payload);
        sent++;
      } catch (error) {
        console.error(`Failed to broadcast to client ${clientId}:`, error);
      }
    });

    return sent;
  }

  /**
   * Get client by ID
   */
  getClient(clientId: string): ClientConnection | undefined {
    return this.clients.get(clientId);
  }

  /**
   * Get all connected clients
   */
  getConnectedClients(): ClientConnection[] {
    return Array.from(this.clients.values());
  }

  /**
   * Shutdown manager
   */
  shutdown() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }

    this.clients.forEach(client => {
      client.ws.close(1000, 'Server shutting down');
    });

    this.clients.clear();
  }
}
```

### WebSocket Message Handlers (`src/websocket/handlers.ts`)

```typescript
import WebSocket from 'ws';
import { WebSocketManager } from './manager.js';
import { AnalysisService } from '../services/analysisService.js';

const analysisService = new AnalysisService();

export interface WebSocketMessage {
  type: string;
  requestId?: string;
  [key: string]: any;
}

export function setupMessageHandlers(
  manager: WebSocketManager,
  wss: WebSocket.Server
) {
  wss.on('connection', (ws: WebSocket) => {
    ws.on('message', async (data: WebSocket.RawData) => {
      try {
        const message: WebSocketMessage = JSON.parse(data.toString());
        await handleMessage(ws, message, manager);
      } catch (error) {
        console.error('Error handling message:', error);
        ws.send(JSON.stringify({
          type: 'error',
          error: error instanceof Error ? error.message : 'Invalid message format'
        }));
      }
    });
  });
}

async function handleMessage(
  ws: WebSocket,
  message: WebSocketMessage,
  manager: WebSocketManager
) {
  const { type, requestId } = message;

  switch (type) {
    case 'ping':
      ws.send(JSON.stringify({ type: 'pong', requestId }));
      break;

    case 'analyze':
      await handleAnalyze(ws, message, manager);
      break;

    case 'subscribe':
      await handleSubscribe(ws, message, manager);
      break;

    case 'unsubscribe':
      await handleUnsubscribe(ws, message, manager);
      break;

    default:
      ws.send(JSON.stringify({
        type: 'error',
        requestId,
        error: `Unknown message type: ${type}`
      }));
  }
}

async function handleAnalyze(
  ws: WebSocket,
  message: WebSocketMessage,
  manager: WebSocketManager
) {
  const { requestId, article, metadata } = message;

  if (!article || typeof article !== 'string') {
    ws.send(JSON.stringify({
      type: 'error',
      requestId,
      error: 'Missing or invalid article text'
    }));
    return;
  }

  try {
    // Send initial status
    ws.send(JSON.stringify({
      type: 'progress',
      requestId,
      status: 'starting',
      message: 'Starting analysis...',
      timestamp: new Date().toISOString()
    }));

    // Perform analysis with progress updates
    const result = await analysisService.analyzeArticleStreaming(
      article,
      metadata,
      (progress) => {
        ws.send(JSON.stringify({
          type: 'progress',
          requestId,
          ...progress,
          timestamp: new Date().toISOString()
        }));
      }
    );

    // Send completion
    ws.send(JSON.stringify({
      type: 'complete',
      requestId,
      result,
      timestamp: new Date().toISOString()
    }));

    // Broadcast to subscribed clients (if any)
    manager.broadcast({
      type: 'analysis_complete',
      summary: result.article_summary,
      overall_significance: result.overall_significance,
      timestamp: new Date().toISOString()
    });

  } catch (error) {
    ws.send(JSON.stringify({
      type: 'error',
      requestId,
      error: error instanceof Error ? error.message : 'Analysis failed',
      timestamp: new Date().toISOString()
    }));
  }
}

async function handleSubscribe(
  ws: WebSocket,
  message: WebSocketMessage,
  manager: WebSocketManager
) {
  const { requestId, topics = [] } = message;

  // Find client and update metadata
  const clients = manager.getConnectedClients();
  const client = clients.find(c => c.ws === ws);

  if (client) {
    client.metadata = {
      ...client.metadata,
      subscriptions: topics
    };

    ws.send(JSON.stringify({
      type: 'subscribed',
      requestId,
      topics,
      timestamp: new Date().toISOString()
    }));

    console.log(`Client ${client.id} subscribed to:`, topics);
  }
}

async function handleUnsubscribe(
  ws: WebSocket,
  message: WebSocketMessage,
  manager: WebSocketManager
) {
  const { requestId } = message;

  const clients = manager.getConnectedClients();
  const client = clients.find(c => c.ws === ws);

  if (client && client.metadata) {
    delete client.metadata.subscriptions;

    ws.send(JSON.stringify({
      type: 'unsubscribed',
      requestId,
      timestamp: new Date().toISOString()
    }));

    console.log(`Client ${client.id} unsubscribed`);
  }
}
```

### Integrated Server with WebSocket (`src/server.ts` - Updated)

```typescript
import express from 'express';
import cors from 'cors';
import { createServer } from 'http';
import WebSocket from 'ws';
import { loadConfig } from './config.js';
import analysisRoutes from './routes/analysis.js';
import { errorHandler, notFoundHandler } from './middleware/errorHandler.js';
import { WebSocketManager } from './websocket/manager.js';
import { setupMessageHandlers } from './websocket/handlers.js';

const config = loadConfig();
const app = express();
const server = createServer(app);

// WebSocket server
const wss = new WebSocket.Server({ 
  server,
  path: '/ws'
});

// Initialize WebSocket manager
const wsManager = new WebSocketManager(wss);
setupMessageHandlers(wsManager, wss);

// Express middleware
app.use(cors({
  origin: config.corsOrigins,
  credentials: true
}));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Request logging
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  next();
});

// Routes
app.use('/api', analysisRoutes);

// WebSocket status endpoint
app.get('/api/ws/status', (req, res) => {
  const clients = wsManager.getConnectedClients();
  res.json({
    success: true,
    data: {
      connected_clients: clients.length,
      clients: clients.map(c => ({
        id: c.id,
        connected_at: c.metadata?.connectedAt,
        subscriptions: c.metadata?.subscriptions
      }))
    }
  });
});

// Error handling
app.use(notFoundHandler);
app.use(errorHandler);

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('\n🛑 SIGTERM received, shutting down gracefully...');
  wsManager.shutdown();
  server.close(() => {
    console.log('✅ Server closed');
    process.exit(0);
  });
});

// Start server
server.listen(config.port, () => {
  console.log(`\n🚀 News Analysis Middleware running on port ${config.port}`);
  console.log(`   Agent Bundle URL: ${config.agentBundleUrl}`);
  console.log(`   Collection ID: ${config.collectionId}`);
  console.log(`\n📡 REST API Endpoints:`);
  console.log(`   POST   http://localhost:${config.port}/api/analyze`);
  console.log(`   POST   http://localhost:${config.port}/api/batch`);
  console.log(`   GET    http://localhost:${config.port}/api/history`);
  console.log(`   GET    http://localhost:${config.port}/api/health`);
  console.log(`   GET    http://localhost:${config.port}/api/ws/status`);
  console.log(`\n🔌 WebSocket:`);
  console.log(`   ws://localhost:${config.port}/ws\n`);
});

export default app;
export { wsManager };
```

---

## Part 3: Client Usage Examples

### REST API Client Example

```typescript
// Simple REST client
async function analyzeViaREST(article: string) {
  const response = await fetch('http://localhost:3000/api/analyze', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      article,
      metadata: {
        source_url: 'https://example.com/article',
        published_date: new Date().toISOString()
      }
    })
  });

  const data = await response.json();
  
  if (data.success) {
    console.log('Analysis result:', data.data);
    return data.data;
  } else {
    throw new Error(data.error);
  }
}
```

### WebSocket Client Example (Node.js)

```typescript
import WebSocket from 'ws';

class AnalysisWebSocketClient {
  private ws: WebSocket;
  private requestCallbacks: Map<string, (data: any) => void> = new Map();

  constructor(url: string = 'ws://localhost:3000/ws') {
    this.ws = new WebSocket(url);
    this.setupHandlers();
  }

  private setupHandlers() {
    this.ws.on('open', () => {
      console.log('✅ Connected to WebSocket');
    });

    this.ws.on('message', (data: WebSocket.RawData) => {
      try {
        const message = JSON.parse(data.toString());
        this.handleMessage(message);
      } catch (error) {
        console.error('Failed to parse message:', error);
      }
    });

    this.ws.on('error', (error) => {
      console.error('WebSocket error:', error);
    });

    this.ws.on('close', () => {
      console.log('❌ Disconnected from WebSocket');
    });
  }

  private handleMessage(message: any) {
    const { type, requestId } = message;

    if (requestId && this.requestCallbacks.has(requestId)) {
      const callback = this.requestCallbacks.get(requestId)!;
      callback(message);

      // Remove callback on completion or error
      if (type === 'complete' || type === 'error') {
        this.requestCallbacks.delete(requestId);
      }
    } else {
      // Handle broadcast messages
      console.log('Broadcast message:', message);
    }
  }

  analyzeArticle(
    article: string,
    onProgress: (message: any) => void
  ): Promise<any> {
    return new Promise((resolve, reject) => {
      const requestId = `req_${Date.now()}_${Math.random()}`;

      this.requestCallbacks.set(requestId, (message) => {
        switch (message.type) {
          case 'progress':
            onProgress(message);
            break;
          case 'complete':
            resolve(message.result);
            break;
          case 'error':
            reject(new Error(message.error));
            break;
        }
      });

      this.ws.send(JSON.stringify({
        type: 'analyze',
        requestId,
        article
      }));
    });
  }

  subscribe(topics: string[]) {
    this.ws.send(JSON.stringify({
      type: 'subscribe',
      requestId: `sub_${Date.now()}`,
      topics
    }));
  }

  close() {
    this.ws.close();
  }
}

// Usage
async function main() {
  const client = new AnalysisWebSocketClient();

  // Wait for connection
  await new Promise(resolve => setTimeout(resolve, 1000));

  const article = `
    Breaking: Major tech company announces quantum computing breakthrough...
  `;

  try {
    const result = await client.analyzeArticle(article, (progress) => {
      console.log('Progress:', progress.message || progress.status);
    });

    console.log('\n✅ Analysis complete:');
    console.log('Summary:', result.article_summary);
    console.log('Overall:', result.overall_significance);
  } catch (error) {
    console.error('Analysis failed:', error);
  } finally {
    client.close();
  }
}

main();
```

### WebSocket Client Example (Browser)

```typescript
class BrowserAnalysisClient {
  private ws: WebSocket;
  private callbacks: Map<string, (data: any) => void> = new Map();

  constructor(url: string = 'ws://localhost:3000/ws') {
    this.ws = new WebSocket(url);
    this.setupHandlers();
  }

  private setupHandlers() {
    this.ws.onopen = () => {
      console.log('Connected to analysis service');
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      if (message.requestId && this.callbacks.has(message.requestId)) {
        const callback = this.callbacks.get(message.requestId)!;
        callback(message);

        if (message.type === 'complete' || message.type === 'error') {
          this.callbacks.delete(message.requestId);
        }
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = () => {
      console.log('Disconnected from analysis service');
    };
  }

  analyzeArticle(article: string): Promise<any> {
    return new Promise((resolve, reject) => {
      const requestId = `req_${Date.now()}_${Math.random()}`;

      this.callbacks.set(requestId, (message) => {
        if (message.type === 'progress') {
          // Emit progress event
          window.dispatchEvent(new CustomEvent('analysis-progress', {
            detail: message
          }));
        } else if (message.type === 'complete') {
          resolve(message.result);
        } else if (message.type === 'error') {
          reject(new Error(message.error));
        }
      });

      this.ws.send(JSON.stringify({
        type: 'analyze',
        requestId,
        article
      }));
    });
  }

  close() {
    this.ws.close();
  }
}

// Usage in browser
const client = new BrowserAnalysisClient();

// Listen for progress updates
window.addEventListener('analysis-progress', (event: any) => {
  console.log('Progress:', event.detail.message);
  // Update UI with progress
});

// Analyze article
document.getElementById('analyzeBtn')?.addEventListener('click', async () => {
  const article = (document.getElementById('articleText') as HTMLTextAreaElement).value;
  
  try {
    const result = await client.analyzeArticle(article);
    console.log('Analysis complete:', result);
    // Display results in UI
  } catch (error) {
    console.error('Analysis failed:', error);
  }
});
```

---

## Part 4: Advanced Features

### Authentication Middleware

Add JWT authentication to protect your API:

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

export interface AuthRequest extends Request {
  user?: {
    id: string;
    email: string;
  };
}

export function authenticateToken(
  req: AuthRequest,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({
      success: false,
      error: 'Authentication required'
    });
  }

  try {
    const user = jwt.verify(token, JWT_SECRET) as any;
    req.user = user;
    next();
  } catch (error) {
    return res.status(403).json({
      success: false,
      error: 'Invalid or expired token'
    });
  }
}

// Apply to protected routes
router.post('/analyze', authenticateToken, async (req: AuthRequest, res) => {
  console.log(`User ${req.user?.email} analyzing article`);
  // ... rest of handler
});
```

### Rate Limiting

Implement rate limiting to prevent abuse:

```typescript
// src/middleware/rateLimiter.ts
import rateLimit from 'express-rate-limit';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: {
    success: false,
    error: 'Too many requests, please try again later'
  }
});

export const analyzeLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10, // Limit analysis requests to 10 per minute
  message: {
    success: false,
    error: 'Analysis rate limit exceeded'
  }
});

// Apply in server.ts
app.use('/api', apiLimiter);
app.use('/api/analyze', analyzeLimiter);
```

### Request Caching

Cache results to improve performance:

```typescript
// src/middleware/cache.ts
import NodeCache from 'node-cache';
import crypto from 'crypto';

const cache = new NodeCache({ stdTTL: 3600 }); // 1 hour TTL

export function cacheMiddleware(durationSeconds: number) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Generate cache key from request
    const key = crypto
      .createHash('md5')
      .update(JSON.stringify(req.body))
      .digest('hex');

    const cachedResponse = cache.get(key);

    if (cachedResponse) {
      console.log(`Cache hit: ${key}`);
      return res.json({
        success: true,
        data: cachedResponse,
        cached: true
      });
    }

    // Store original res.json
    const originalJson = res.json.bind(res);

    // Override res.json to cache the response
    res.json = (body: any) => {
      if (body.success && body.data) {
        cache.set(key, body.data, durationSeconds);
      }
      return originalJson(body);
    };

    next();
  };
}

// Usage
router.post('/analyze', cacheMiddleware(3600), async (req, res) => {
  // ... analysis logic
});
```

### Monitoring and Metrics

Track API usage and performance:

```typescript
// src/middleware/metrics.ts
import { Request, Response, NextFunction } from 'express';

interface Metrics {
  requests: number;
  errors: number;
  totalDuration: number;
  analysisCounts: Record<string, number>;
}

const metrics: Metrics = {
  requests: 0,
  errors: 0,
  totalDuration: 0,
  analysisCounts: {}
};

export function metricsMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const startTime = Date.now();
  metrics.requests++;

  // Track response
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    metrics.totalDuration += duration;

    if (res.statusCode >= 400) {
      metrics.errors++;
    }

    // Track analysis requests
    if (req.path.includes('/analyze')) {
      metrics.analysisCounts[req.path] = 
        (metrics.analysisCounts[req.path] || 0) + 1;
    }
  });

  next();
}

// Metrics endpoint
router.get('/metrics', (req, res) => {
  res.json({
    success: true,
    data: {
      ...metrics,
      averageDuration: metrics.requests > 0 
        ? metrics.totalDuration / metrics.requests 
        : 0,
      uptime: process.uptime()
    }
  });
});
```

---

## Testing

### Example Test Suite

```typescript
// tests/api.test.ts
import request from 'supertest';
import app from '../src/server';

describe('News Analysis API', () => {
  it('should analyze an article', async () => {
    const response = await request(app)
      .post('/api/analyze')
      .send({
        article: 'Test article about quantum computing...'
      })
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(response.body.data).toHaveProperty('article_summary');
    expect(response.body.data).toHaveProperty('healthcare');
  });

  it('should return 400 for missing article', async () => {
    const response = await request(app)
      .post('/api/analyze')
      .send({})
      .expect(400);

    expect(response.body.success).toBe(false);
  });

  it('should fetch analysis history', async () => {
    const response = await request(app)
      .get('/api/history')
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(Array.isArray(response.body.data.articles)).toBe(true);
  });
});
```

---

## Deployment

### Docker Support

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  middleware:
    build: .
    ports:
      - "3000:3000"
    environment:
      - AGENT_BUNDLE_URL=http://agent-bundle:3001
      - NEWS_COLLECTION_ID=c0000000-0000-0000-0001-000000000001
      - CORS_ORIGINS=http://localhost:3000,http://localhost:3001
    depends_on:
      - agent-bundle

  agent-bundle:
    image: your-agent-bundle:latest
    ports:
      - "3001:3001"
```

---

## Best Practices

1. **Error Handling**: Always wrap agent bundle calls in try-catch
2. **Timeouts**: Set appropriate timeouts for long-running analyses
3. **Connection Management**: Properly clean up WebSocket connections
4. **Rate Limiting**: Protect your middleware from abuse
5. **Caching**: Cache frequently-requested analyses
6. **Monitoring**: Track metrics and log errors
7. **Security**: Implement authentication and input validation
8. **CORS**: Configure CORS properly for your environment
9. **Graceful Shutdown**: Handle SIGTERM for clean shutdown
10. **Health Checks**: Expose health endpoints for orchestration

---

**Generated by Claude**
