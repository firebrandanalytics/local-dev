# Consuming the News Analysis Agent Bundle

## Introduction

This tutorial demonstrates how to consume the **News Article Impact Analyzer** agent bundle using the FireFoundry SDK. This is the consumer-side complement to the [AgentSDK Getting Started Guide](../agent_sdk/agent_sdk_getting_started.md), which showed how to build the agent bundle.

**What we'll build:** A command-line tool that submits news articles for analysis and receives structured impact assessments across three business verticals (healthcare, shipping/logistics, and technology).

**Prerequisites:**
- The News Analysis Agent Bundle is deployed and running (e.g., locally in Minikube)
- Node.js and TypeScript development environment
- Access to the agent bundle's base URL
- API key for authentication (if your deployment requires it)
- Shared types package from your monorepo (for type safety)

## Architecture Overview

Our consumer application follows this flow:

```
CLI Tool → FF SDK → Agent Bundle → NewsCollection → ArticleEntity → ImpactAnalysisBot → LLM
                                                                                            ↓
CLI Tool ← FF SDK ← Agent Bundle ← NewsCollection ← ArticleEntity ← Structured Analysis ←┘
```

The key insight: **we don't need to know the internal implementation details** of the agent bundle. We just need to know:
1. The collection entity ID (entry point)
2. The methods available (`analyze_article`, `get_all_articles`)
3. The expected input/output types

---

## Part 1: Basic Article Analysis

### Setting Up the Consumer

First, install the FF SDK and create a client:

```typescript
import { RemoteAgentBundleClient, FFError } from '@firebrandanalytics/ff-sdk';

// Configuration (update based on your environment)
const BASE_URL = process.env.AGENT_BUNDLE_URL || 'http://localhost:3001';
const API_KEY = process.env.AGENT_BUNDLE_API_KEY; // Optional

// Create client
const client = new RemoteAgentBundleClient(BASE_URL, {
  api_key: API_KEY,
  timeout: 60000 // 60 second timeout for analysis operations
});
```

**Environment-Specific URLs:**
- **Local Minikube**: `http://localhost:3001` (with port-forward to agent bundle service)
- **Staging**: `https://news-analysis-staging.example.com`
- **Production**: `https://news-analysis.example.com`

### Importing Shared Types

In a monorepo setup, import the shared types for compile-time safety:

```typescript
// From your shared types package
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';

// The output type matches what the agent bundle produces:
// {
//   article_summary: string;
//   healthcare: VerticalImpact;
//   shipping_logistics: VerticalImpact;
//   technology: VerticalImpact;
//   overall_significance: 'low' | 'medium' | 'high';
// }
```

### Finding Your Collection Entity ID

The agent bundle creates a root collection entity on startup. To find it, you can:

1. **Check the agent bundle logs** for the printed entity ID
2. **Use the health check** to verify connectivity first:

```typescript
async function findCollectionId(): Promise<string> {
  // Option 1: Hardcode from deployment (recommended for production)
  const COLLECTION_ID = 'c0000000-0000-0000-0001-000000000001';
  return COLLECTION_ID;
  
  // Option 2: Store in environment variable
  // return process.env.NEWS_COLLECTION_ID!;
  
  // Option 3: Use a discovery endpoint (if your bundle exposes one)
  // const info = await client.call_api_endpoint<{ collection_id: string }>('get_root_collection');
  // return info.collection_id;
}
```

### Analyzing Your First Article

Now let's analyze a news article:

```typescript
async function analyzeArticle(
  collectionId: string, 
  articleText: string
): Promise<IMPACT_ANALYSIS_OUTPUT> {
  console.log('📰 Submitting article for analysis...');
  console.log(`   Length: ${articleText.length} characters\n`);

  try {
    const result = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
      collectionId,
      'analyze_article',
      articleText
    );

    console.log('✅ Analysis complete!\n');
    return result;

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ Analysis failed:', error.message);
    }
    throw error;
  }
}
```

### Displaying Results

Format the analysis results for human consumption:

```typescript
function displayAnalysis(analysis: IMPACT_ANALYSIS_OUTPUT): void {
  console.log('═══════════════════════════════════════════════════════');
  console.log('📊 IMPACT ANALYSIS RESULTS');
  console.log('═══════════════════════════════════════════════════════\n');

  console.log('📝 SUMMARY:');
  console.log(analysis.article_summary);
  console.log();

  console.log('🏥 HEALTHCARE IMPACT:');
  displayVerticalImpact(analysis.healthcare);

  console.log('🚢 SHIPPING & LOGISTICS IMPACT:');
  displayVerticalImpact(analysis.shipping_logistics);

  console.log('💻 TECHNOLOGY IMPACT:');
  displayVerticalImpact(analysis.technology);

  console.log('🎯 OVERALL SIGNIFICANCE:', analysis.overall_significance.toUpperCase());
  console.log('═══════════════════════════════════════════════════════');
}

function displayVerticalImpact(impact: VerticalImpact): void {
  const emoji = {
    'none': '⚪',
    'low': '🟢', 
    'medium': '🟡',
    'high': '🟠',
    'critical': '🔴'
  };

  console.log(`   Impact Level: ${emoji[impact.impact_level]} ${impact.impact_level.toUpperCase()}`);
  console.log(`   Confidence: ${(impact.confidence * 100).toFixed(0)}%`);
  console.log(`   Reasoning: ${impact.reasoning}`);
  console.log(`   Key Factors:`);
  impact.key_factors.forEach(factor => {
    console.log(`      • ${factor}`);
  });
  console.log();
}
```

### Complete Basic Example

Here's a complete working program:

```typescript
#!/usr/bin/env node
import { RemoteAgentBundleClient, FFError } from '@firebrandanalytics/ff-sdk';
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';

const BASE_URL = process.env.AGENT_BUNDLE_URL || 'http://localhost:3001';
const API_KEY = process.env.AGENT_BUNDLE_API_KEY;
const COLLECTION_ID = process.env.NEWS_COLLECTION_ID || 'c0000000-0000-0000-0001-000000000001';

async function main() {
  // Initialize client
  const client = new RemoteAgentBundleClient(BASE_URL, {
    api_key: API_KEY,
    timeout: 60000
  });

  // Verify service is healthy
  const isHealthy = await client.health_check();
  if (!isHealthy) {
    console.error('❌ Service is not healthy');
    process.exit(1);
  }

  // Sample article
  const article = `
    Breaking: Major tech company announces breakthrough in quantum computing 
    that could revolutionize data processing across industries. The new quantum 
    processor demonstrates unprecedented speed improvements in complex calculations, 
    potentially impacting everything from drug discovery to logistics optimization.
    
    The breakthrough addresses several key challenges in quantum error correction 
    and could accelerate commercial applications by 3-5 years. Healthcare researchers 
    are particularly excited about potential applications in protein folding simulations 
    and personalized medicine. Supply chain experts see opportunities for optimizing 
    complex routing problems that currently take hours to solve.
  `.trim();

  try {
    // Analyze the article
    const analysis = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
      COLLECTION_ID,
      'analyze_article',
      article
    );

    // Display results
    displayAnalysis(analysis);

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ Error:', error.message);
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
export NEWS_COLLECTION_ID="your-collection-id"

# Install dependencies
npm install

# Run the analyzer
npm start
```

---

## Part 2: Intermediate Features

### Adding Article Metadata

The `analyze_article` method accepts optional metadata:

```typescript
async function analyzeWithMetadata(
  collectionId: string,
  articleText: string,
  sourceUrl: string,
  publishedDate: string
): Promise<IMPACT_ANALYSIS_OUTPUT> {
  const metadata = {
    source_url: sourceUrl,
    published_date: publishedDate
  };

  const result = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
    collectionId,
    'analyze_article',
    articleText,
    metadata
  );

  return result;
}

// Example usage
const analysis = await analyzeWithMetadata(
  collectionId,
  articleText,
  'https://example.com/quantum-breakthrough',
  '2024-01-15'
);
```

### Batch Processing Multiple Articles

Analyze multiple articles efficiently:

```typescript
interface ArticleInput {
  text: string;
  source_url?: string;
  published_date?: string;
}

async function batchAnalyze(
  collectionId: string,
  articles: ArticleInput[]
): Promise<IMPACT_ANALYSIS_OUTPUT[]> {
  console.log(`📰 Analyzing ${articles.length} articles...\n`);

  const results: IMPACT_ANALYSIS_OUTPUT[] = [];
  
  // Process with controlled concurrency
  const concurrency = 3;
  
  for (let i = 0; i < articles.length; i += concurrency) {
    const batch = articles.slice(i, i + concurrency);
    
    const batchResults = await Promise.all(
      batch.map(async (article, idx) => {
        const articleNum = i + idx + 1;
        console.log(`   [${articleNum}/${articles.length}] Processing...`);
        
        try {
          return await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
            collectionId,
            'analyze_article',
            article.text,
            {
              source_url: article.source_url,
              published_date: article.published_date
            }
          );
        } catch (error) {
          console.error(`   [${articleNum}] Failed:`, error);
          return null;
        }
      })
    );

    results.push(...batchResults.filter((r): r is IMPACT_ANALYSIS_OUTPUT => r !== null));
  }

  console.log(`\n✅ Analyzed ${results.length}/${articles.length} articles successfully\n`);
  return results;
}
```

### Retrieving Analysis History

Get all previously analyzed articles:

```typescript
async function getAnalysisHistory(collectionId: string) {
  console.log('📋 Fetching analysis history...\n');

  const articles = await client.invoke_entity_method<any[]>(
    collectionId,
    'get_all_articles'
  );

  console.log(`Found ${articles.length} analyzed articles:\n`);
  
  articles.forEach((article, idx) => {
    console.log(`${idx + 1}. Article ${article.id}`);
    console.log(`   Created: ${new Date(article.created).toLocaleString()}`);
    console.log(`   Status: ${article.status}`);
    if (article.data?.source_url) {
      console.log(`   Source: ${article.data.source_url}`);
    }
    console.log();
  });

  return articles;
}
```

### Error Handling and Retries

Implement robust error handling:

```typescript
async function analyzeWithRetry(
  collectionId: string,
  articleText: string,
  maxRetries: number = 3
): Promise<IMPACT_ANALYSIS_OUTPUT> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      console.log(`Attempt ${attempt}/${maxRetries}...`);
      
      const result = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
        collectionId,
        'analyze_article',
        articleText
      );

      console.log('✅ Success!');
      return result;

    } catch (error) {
      lastError = error as Error;
      
      if (error instanceof FFError) {
        // Check if it's a retryable error
        if (error.message.includes('timeout')) {
          console.warn(`⚠️  Timeout on attempt ${attempt}`);
        } else if (error.message.includes('rate limit')) {
          console.warn(`⚠️  Rate limited on attempt ${attempt}`);
        } else {
          // Non-retryable error
          throw error;
        }
      }

      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000;
        console.log(`   Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw new Error(`Failed after ${maxRetries} attempts: ${lastError!.message}`);
}
```

### Filtering and Scoring

Post-process results to find high-impact articles:

```typescript
interface ScoredAnalysis {
  analysis: IMPACT_ANALYSIS_OUTPUT;
  score: number;
}

function scoreAnalysis(analysis: IMPACT_ANALYSIS_OUTPUT): number {
  const impactScores = {
    'none': 0,
    'low': 1,
    'medium': 2,
    'high': 3,
    'critical': 4
  };

  const significanceScores = {
    'low': 1,
    'medium': 2,
    'high': 3
  };

  // Weighted score combining individual impacts and overall significance
  const healthcareScore = impactScores[analysis.healthcare.impact_level] * analysis.healthcare.confidence;
  const logisticsScore = impactScores[analysis.shipping_logistics.impact_level] * analysis.shipping_logistics.confidence;
  const techScore = impactScores[analysis.technology.impact_level] * analysis.technology.confidence;
  const overallBonus = significanceScores[analysis.overall_significance];

  return (healthcareScore + logisticsScore + techScore) / 3 + overallBonus;
}

function findHighImpactArticles(
  analyses: IMPACT_ANALYSIS_OUTPUT[],
  threshold: number = 5
): ScoredAnalysis[] {
  const scored = analyses.map(analysis => ({
    analysis,
    score: scoreAnalysis(analysis)
  }));

  return scored
    .filter(item => item.score >= threshold)
    .sort((a, b) => b.score - a.score);
}

// Usage
const analyses = await batchAnalyze(collectionId, articles);
const highImpact = findHighImpactArticles(analyses, 5);

console.log(`\n🎯 Found ${highImpact.length} high-impact articles:`);
highImpact.forEach((item, idx) => {
  console.log(`\n${idx + 1}. Score: ${item.score.toFixed(2)}`);
  console.log(`   Summary: ${item.analysis.article_summary}`);
});
```

---

## Part 3: Advanced Integration Patterns

### Reading Articles from Files

Process articles from a directory:

```typescript
import fs from 'fs/promises';
import path from 'path';

async function analyzeArticlesFromDirectory(
  collectionId: string,
  directory: string
): Promise<Map<string, IMPACT_ANALYSIS_OUTPUT>> {
  const results = new Map<string, IMPACT_ANALYSIS_OUTPUT>();
  
  // Read all .txt files from directory
  const files = await fs.readdir(directory);
  const articleFiles = files.filter(f => f.endsWith('.txt'));
  
  console.log(`📁 Found ${articleFiles.length} article files\n`);

  for (const filename of articleFiles) {
    const filepath = path.join(directory, filename);
    const content = await fs.readFile(filepath, 'utf-8');
    
    console.log(`📰 Analyzing ${filename}...`);
    
    try {
      const analysis = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
        collectionId,
        'analyze_article',
        content,
        {
          source_url: `file://${filepath}`,
          published_date: new Date().toISOString()
        }
      );
      
      results.set(filename, analysis);
      console.log(`   ✅ Complete\n`);
      
    } catch (error) {
      console.error(`   ❌ Failed: ${error}\n`);
    }
  }

  return results;
}

// Usage
const results = await analyzeArticlesFromDirectory(
  collectionId,
  './articles'
);
```

### Exporting Results

Save analysis results in different formats:

```typescript
import fs from 'fs/promises';

async function exportToJSON(
  analyses: IMPACT_ANALYSIS_OUTPUT[],
  filename: string
): Promise<void> {
  const data = {
    generated_at: new Date().toISOString(),
    total_articles: analyses.length,
    analyses: analyses
  };

  await fs.writeFile(
    filename,
    JSON.stringify(data, null, 2),
    'utf-8'
  );

  console.log(`✅ Exported ${analyses.length} analyses to ${filename}`);
}

async function exportToCSV(
  analyses: IMPACT_ANALYSIS_OUTPUT[],
  filename: string
): Promise<void> {
  const headers = [
    'Summary',
    'Healthcare Impact',
    'Healthcare Confidence',
    'Logistics Impact',
    'Logistics Confidence',
    'Technology Impact',
    'Technology Confidence',
    'Overall Significance'
  ].join(',');

  const rows = analyses.map(a => [
    `"${a.article_summary.replace(/"/g, '""')}"`,
    a.healthcare.impact_level,
    a.healthcare.confidence,
    a.shipping_logistics.impact_level,
    a.shipping_logistics.confidence,
    a.technology.impact_level,
    a.technology.confidence,
    a.overall_significance
  ].join(','));

  const csv = [headers, ...rows].join('\n');

  await fs.writeFile(filename, csv, 'utf-8');
  console.log(`✅ Exported ${analyses.length} analyses to ${filename}`);
}

// Usage
await exportToJSON(analyses, 'analysis-results.json');
await exportToCSV(analyses, 'analysis-results.csv');
```

### Building a Report Generator

Create a comprehensive analysis report:

```typescript
interface AnalysisReport {
  summary: {
    total_articles: number;
    high_impact_count: number;
    average_confidence: number;
  };
  by_vertical: {
    healthcare: VerticalSummary;
    shipping_logistics: VerticalSummary;
    technology: VerticalSummary;
  };
  top_articles: IMPACT_ANALYSIS_OUTPUT[];
}

interface VerticalSummary {
  average_impact_score: number;
  average_confidence: number;
  critical_count: number;
  high_count: number;
}

function generateReport(analyses: IMPACT_ANALYSIS_OUTPUT[]): AnalysisReport {
  const impactScores = { none: 0, low: 1, medium: 2, high: 3, critical: 4 };

  const calculateVerticalSummary = (
    vertical: 'healthcare' | 'shipping_logistics' | 'technology'
  ): VerticalSummary => {
    const impacts = analyses.map(a => a[vertical]);
    
    return {
      average_impact_score: impacts.reduce((sum, i) => 
        sum + impactScores[i.impact_level], 0) / impacts.length,
      average_confidence: impacts.reduce((sum, i) => 
        sum + i.confidence, 0) / impacts.length,
      critical_count: impacts.filter(i => i.impact_level === 'critical').length,
      high_count: impacts.filter(i => i.impact_level === 'high').length
    };
  };

  const scored = analyses.map(a => ({ analysis: a, score: scoreAnalysis(a) }))
    .sort((a, b) => b.score - a.score);

  const allConfidences = analyses.flatMap(a => [
    a.healthcare.confidence,
    a.shipping_logistics.confidence,
    a.technology.confidence
  ]);

  return {
    summary: {
      total_articles: analyses.length,
      high_impact_count: scored.filter(s => s.score >= 5).length,
      average_confidence: allConfidences.reduce((a, b) => a + b, 0) / allConfidences.length
    },
    by_vertical: {
      healthcare: calculateVerticalSummary('healthcare'),
      shipping_logistics: calculateVerticalSummary('shipping_logistics'),
      technology: calculateVerticalSummary('technology')
    },
    top_articles: scored.slice(0, 5).map(s => s.analysis)
  };
}

function printReport(report: AnalysisReport): void {
  console.log('\n═══════════════════════════════════════════════════════');
  console.log('📊 ANALYSIS REPORT');
  console.log('═══════════════════════════════════════════════════════\n');

  console.log('📈 SUMMARY:');
  console.log(`   Total Articles: ${report.summary.total_articles}`);
  console.log(`   High Impact: ${report.summary.high_impact_count}`);
  console.log(`   Avg Confidence: ${(report.summary.average_confidence * 100).toFixed(1)}%\n`);

  console.log('🏥 HEALTHCARE:');
  printVerticalSummary(report.by_vertical.healthcare);

  console.log('🚢 SHIPPING & LOGISTICS:');
  printVerticalSummary(report.by_vertical.shipping_logistics);

  console.log('💻 TECHNOLOGY:');
  printVerticalSummary(report.by_vertical.technology);

  console.log('🏆 TOP 5 ARTICLES BY IMPACT:');
  report.top_articles.forEach((article, idx) => {
    console.log(`\n${idx + 1}. ${article.article_summary.substring(0, 100)}...`);
    console.log(`   Overall: ${article.overall_significance.toUpperCase()}`);
  });
  
  console.log('\n═══════════════════════════════════════════════════════');
}

function printVerticalSummary(summary: VerticalSummary): void {
  console.log(`   Avg Impact: ${summary.average_impact_score.toFixed(2)}/4`);
  console.log(`   Avg Confidence: ${(summary.average_confidence * 100).toFixed(1)}%`);
  console.log(`   Critical: ${summary.critical_count}, High: ${summary.high_count}\n`);
}

// Usage
const analyses = await batchAnalyze(collectionId, articles);
const report = generateReport(analyses);
printReport(report);
```

### Streaming Progress Updates (Future)

For long-running analyses, you could receive progress updates:

```typescript
// Note: This assumes the agent bundle implements streaming
// This is a future enhancement pattern

async function analyzeWithProgress(
  collectionId: string,
  articleText: string
) {
  console.log('📰 Starting analysis with live progress...\n');

  const iterator = await client.start_iterator(
    collectionId,
    'analyze_article_streaming',
    articleText
  );

  for await (const progress of iterator) {
    switch (progress.type) {
      case 'status':
        console.log(`⏳ ${progress.data.message}`);
        break;
      
      case 'partial':
        console.log(`📊 Partial results: ${JSON.stringify(progress.data)}`);
        break;
      
      case 'complete':
        console.log('✅ Analysis complete!\n');
        return progress.data.result;
    }
  }
}
```

---

## Complete Production Example

Here's a full-featured command-line tool using yargs for robust argument parsing:

**Installation:**

```bash
npm install @firebrandanalytics/ff-sdk yargs
npm install --save-dev @types/yargs
```

**Complete CLI Tool (`src/cli.ts`):**

```typescript
#!/usr/bin/env node
import yargs from 'yargs';
import { hideBin } from 'yargs/helpers';
import { RemoteAgentBundleClient, FFError } from '@firebrandanalytics/ff-sdk';
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';
import fs from 'fs/promises';
import path from 'path';

// Configuration from environment with defaults
interface Config {
  baseUrl: string;
  apiKey?: string;
  collectionId: string;
  timeout: number;
}

function loadConfig(): Config {
  return {
    baseUrl: process.env.AGENT_BUNDLE_URL || 'http://localhost:3001',
    apiKey: process.env.AGENT_BUNDLE_API_KEY,
    collectionId: process.env.NEWS_COLLECTION_ID || 'c0000000-0000-0000-0001-000000000001',
    timeout: parseInt(process.env.TIMEOUT || '60000')
  };
}

const config = loadConfig();

// Initialize client
const client = new RemoteAgentBundleClient(config.baseUrl, {
  api_key: config.apiKey,
  timeout: config.timeout
});

// Display functions
function displayAnalysis(analysis: IMPACT_ANALYSIS_OUTPUT): void {
  console.log('\n═══════════════════════════════════════════════════════');
  console.log('📊 IMPACT ANALYSIS RESULTS');
  console.log('═══════════════════════════════════════════════════════\n');

  console.log('📝 SUMMARY:');
  console.log(analysis.article_summary);
  console.log();

  console.log('🏥 HEALTHCARE IMPACT:');
  displayVerticalImpact(analysis.healthcare);

  console.log('🚢 SHIPPING & LOGISTICS IMPACT:');
  displayVerticalImpact(analysis.shipping_logistics);

  console.log('💻 TECHNOLOGY IMPACT:');
  displayVerticalImpact(analysis.technology);

  console.log('🎯 OVERALL SIGNIFICANCE:', analysis.overall_significance.toUpperCase());
  console.log('═══════════════════════════════════════════════════════\n');
}

function displayVerticalImpact(impact: any): void {
  const emoji = {
    'none': '⚪',
    'low': '🟢', 
    'medium': '🟡',
    'high': '🟠',
    'critical': '🔴'
  };

  console.log(`   Impact Level: ${emoji[impact.impact_level]} ${impact.impact_level.toUpperCase()}`);
  console.log(`   Confidence: ${(impact.confidence * 100).toFixed(0)}%`);
  console.log(`   Reasoning: ${impact.reasoning}`);
  console.log(`   Key Factors:`);
  impact.key_factors.forEach((factor: string) => {
    console.log(`      • ${factor}`);
  });
  console.log();
}

function generateReport(analyses: IMPACT_ANALYSIS_OUTPUT[]): any {
  const impactScores = { none: 0, low: 1, medium: 2, high: 3, critical: 4 };

  const calculateVerticalSummary = (
    vertical: 'healthcare' | 'shipping_logistics' | 'technology'
  ) => {
    const impacts = analyses.map(a => a[vertical]);
    
    return {
      average_impact_score: impacts.reduce((sum, i) => 
        sum + impactScores[i.impact_level], 0) / impacts.length,
      average_confidence: impacts.reduce((sum, i) => 
        sum + i.confidence, 0) / impacts.length,
      critical_count: impacts.filter(i => i.impact_level === 'critical').length,
      high_count: impacts.filter(i => i.impact_level === 'high').length
    };
  };

  const allConfidences = analyses.flatMap(a => [
    a.healthcare.confidence,
    a.shipping_logistics.confidence,
    a.technology.confidence
  ]);

  return {
    summary: {
      total_articles: analyses.length,
      average_confidence: allConfidences.reduce((a, b) => a + b, 0) / allConfidences.length
    },
    by_vertical: {
      healthcare: calculateVerticalSummary('healthcare'),
      shipping_logistics: calculateVerticalSummary('shipping_logistics'),
      technology: calculateVerticalSummary('technology')
    }
  };
}

function printReport(report: any): void {
  console.log('\n═══════════════════════════════════════════════════════');
  console.log('📊 BATCH ANALYSIS REPORT');
  console.log('═══════════════════════════════════════════════════════\n');

  console.log('📈 SUMMARY:');
  console.log(`   Total Articles: ${report.summary.total_articles}`);
  console.log(`   Avg Confidence: ${(report.summary.average_confidence * 100).toFixed(1)}%\n`);

  console.log('🏥 HEALTHCARE:');
  console.log(`   Avg Impact: ${report.by_vertical.healthcare.average_impact_score.toFixed(2)}/4`);
  console.log(`   Critical: ${report.by_vertical.healthcare.critical_count}, High: ${report.by_vertical.healthcare.high_count}\n`);

  console.log('🚢 SHIPPING & LOGISTICS:');
  console.log(`   Avg Impact: ${report.by_vertical.shipping_logistics.average_impact_score.toFixed(2)}/4`);
  console.log(`   Critical: ${report.by_vertical.shipping_logistics.critical_count}, High: ${report.by_vertical.shipping_logistics.high_count}\n`);

  console.log('💻 TECHNOLOGY:');
  console.log(`   Avg Impact: ${report.by_vertical.technology.average_impact_score.toFixed(2)}/4`);
  console.log(`   Critical: ${report.by_vertical.technology.critical_count}, High: ${report.by_vertical.technology.high_count}\n`);
  
  console.log('═══════════════════════════════════════════════════════');
}

// Command implementations
async function analyzeCommand(argv: any) {
  const { file, metadata } = argv;

  console.log(`📰 Analyzing article from ${file}...\n`);

  try {
    const article = await fs.readFile(file, 'utf-8');
    
    const metadataObj = metadata ? {
      source_url: metadata.source,
      published_date: metadata.date
    } : undefined;

    const analysis = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
      config.collectionId,
      'analyze_article',
      article,
      metadataObj
    );

    displayAnalysis(analysis);

    if (argv.output) {
      await fs.writeFile(
        argv.output,
        JSON.stringify(analysis, null, 2)
      );
      console.log(`💾 Saved results to ${argv.output}\n`);
    }

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ Analysis failed:', error.message);
    } else {
      console.error('❌ Unexpected error:', error);
    }
    process.exit(1);
  }
}

async function batchCommand(argv: any) {
  const { directory, concurrency, output, format } = argv;

  console.log(`📁 Processing articles from ${directory}...\n`);

  try {
    const files = await fs.readdir(directory);
    const articleFiles = files.filter(f => f.endsWith('.txt'));
    
    console.log(`Found ${articleFiles.length} article files\n`);

    const analyses: IMPACT_ANALYSIS_OUTPUT[] = [];

    // Process with controlled concurrency
    for (let i = 0; i < articleFiles.length; i += concurrency) {
      const batch = articleFiles.slice(i, i + concurrency);
      
      const batchResults = await Promise.all(
        batch.map(async (file, idx) => {
          const fileNum = i + idx + 1;
          console.log(`   [${fileNum}/${articleFiles.length}] Processing ${file}...`);
          
          try {
            const article = await fs.readFile(path.join(directory, file), 'utf-8');
            const analysis = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
              config.collectionId,
              'analyze_article',
              article
            );
            console.log(`   [${fileNum}/${articleFiles.length}] ✅ Complete`);
            return analysis;
          } catch (error) {
            console.error(`   [${fileNum}/${articleFiles.length}] ❌ Failed: ${error}`);
            return null;
          }
        })
      );

      analyses.push(...batchResults.filter((r): r is IMPACT_ANALYSIS_OUTPUT => r !== null));
    }

    console.log(`\n✅ Successfully analyzed ${analyses.length}/${articleFiles.length} articles\n`);

    // Generate and display report
    const report = generateReport(analyses);
    printReport(report);

    // Export results
    if (output) {
      const data = format === 'json' 
        ? JSON.stringify(analyses, null, 2)
        : generateCSV(analyses);

      await fs.writeFile(output, data);
      console.log(`\n💾 Results saved to ${output}`);
    }

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ Batch processing failed:', error.message);
    } else {
      console.error('❌ Unexpected error:', error);
    }
    process.exit(1);
  }
}

async function historyCommand(argv: any) {
  const { limit, format } = argv;

  console.log('📋 Fetching analysis history...\n');

  try {
    const articles = await client.invoke_entity_method<any[]>(
      config.collectionId,
      'get_all_articles'
    );

    const displayArticles = limit ? articles.slice(0, limit) : articles;

    console.log(`Found ${articles.length} analyzed articles:\n`);
    
    displayArticles.forEach((article, idx) => {
      console.log(`${idx + 1}. Article ${article.id}`);
      console.log(`   Created: ${new Date(article.created).toLocaleString()}`);
      console.log(`   Status: ${article.status}`);
      if (article.data?.source_url) {
        console.log(`   Source: ${article.data.source_url}`);
      }
      console.log();
    });

    if (limit && articles.length > limit) {
      console.log(`... and ${articles.length - limit} more\n`);
    }

    if (format === 'json') {
      console.log(JSON.stringify(articles, null, 2));
    }

  } catch (error) {
    if (error instanceof FFError) {
      console.error('❌ Failed to fetch history:', error.message);
    } else {
      console.error('❌ Unexpected error:', error);
    }
    process.exit(1);
  }
}

async function healthCommand() {
  try {
    console.log('🔍 Checking service health...\n');

    const isHealthy = await client.health_check();
    const isReady = await client.is_ready();

    console.log('🏥 Health Check:');
    console.log(`   Healthy: ${isHealthy ? '✅' : '❌'}`);
    console.log(`   Ready: ${isReady ? '✅' : '❌'}\n`);

    if (isHealthy && isReady) {
      const info = await client.get_info();
      console.log('📋 Service Info:');
      console.log(`   Name: ${info.app_name}`);
      console.log(`   Description: ${info.app_description}`);
      console.log(`   ID: ${info.app_id}\n`);
    }

    if (!isHealthy || !isReady) {
      process.exit(1);
    }

  } catch (error) {
    console.error('❌ Health check failed:', error);
    process.exit(1);
  }
}

function generateCSV(analyses: IMPACT_ANALYSIS_OUTPUT[]): string {
  const headers = [
    'Summary',
    'Healthcare Impact',
    'Healthcare Confidence',
    'Logistics Impact',
    'Logistics Confidence',
    'Technology Impact',
    'Technology Confidence',
    'Overall Significance'
  ].join(',');

  const rows = analyses.map(a => [
    `"${a.article_summary.replace(/"/g, '""')}"`,
    a.healthcare.impact_level,
    a.healthcare.confidence,
    a.shipping_logistics.impact_level,
    a.shipping_logistics.confidence,
    a.technology.impact_level,
    a.technology.confidence,
    a.overall_significance
  ].join(','));

  return [headers, ...rows].join('\n');
}

// CLI Setup with yargs
yargs(hideBin(process.argv))
  .scriptName('news-analyzer')
  .usage('$0 <command> [options]')
  .command(
    'analyze <file>',
    'Analyze a single news article',
    (yargs) => {
      return yargs
        .positional('file', {
          describe: 'Path to article text file',
          type: 'string',
          demandOption: true
        })
        .option('output', {
          alias: 'o',
          describe: 'Output file for results (JSON)',
          type: 'string'
        })
        .option('metadata', {
          alias: 'm',
          describe: 'Article metadata',
          type: 'object'
        })
        .example('$0 analyze article.txt', 'Analyze a single article')
        .example('$0 analyze article.txt -o results.json', 'Analyze and save results');
    },
    analyzeCommand
  )
  .command(
    'batch <directory>',
    'Analyze all articles in a directory',
    (yargs) => {
      return yargs
        .positional('directory', {
          describe: 'Directory containing article files (.txt)',
          type: 'string',
          demandOption: true
        })
        .option('concurrency', {
          alias: 'c',
          describe: 'Number of concurrent analyses',
          type: 'number',
          default: 3
        })
        .option('output', {
          alias: 'o',
          describe: 'Output file for results',
          type: 'string'
        })
        .option('format', {
          alias: 'f',
          describe: 'Output format',
          choices: ['json', 'csv'],
          default: 'json'
        })
        .example('$0 batch ./articles', 'Analyze all articles in directory')
        .example('$0 batch ./articles -o results.json', 'Analyze and save results')
        .example('$0 batch ./articles -c 5 -f csv', 'Analyze with 5 concurrent requests, output CSV');
    },
    batchCommand
  )
  .command(
    'history',
    'View analysis history',
    (yargs) => {
      return yargs
        .option('limit', {
          alias: 'l',
          describe: 'Maximum number of articles to display',
          type: 'number'
        })
        .option('format', {
          alias: 'f',
          describe: 'Output format',
          choices: ['text', 'json'],
          default: 'text'
        })
        .example('$0 history', 'Show all analyzed articles')
        .example('$0 history -l 10', 'Show last 10 articles')
        .example('$0 history -f json', 'Output as JSON');
    },
    historyCommand
  )
  .command(
    'health',
    'Check service health status',
    () => {},
    healthCommand
  )
  .option('verbose', {
    alias: 'v',
    type: 'boolean',
    description: 'Run with verbose logging'
  })
  .demandCommand(1, 'You must provide a command')
  .help()
  .alias('help', 'h')
  .version('1.0.0')
  .alias('version', 'V')
  .epilogue('Environment Variables:\n' +
    '  AGENT_BUNDLE_URL       Service URL (default: http://localhost:3001)\n' +
    '  AGENT_BUNDLE_API_KEY   API key for authentication\n' +
    '  NEWS_COLLECTION_ID     Collection entity ID\n' +
    '  TIMEOUT                Request timeout in ms (default: 60000)\n\n' +
    'For more information, visit: https://docs.firefoundry.ai')
  .strict()
  .parse();
```

**Package.json scripts:**

```json
{
  "name": "news-analyzer-cli",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/cli.js",
    "dev": "tsx src/cli.ts"
  },
  "dependencies": {
    "@firebrandanalytics/ff-sdk": "^0.8.1",
    "yargs": "^17.7.2"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/yargs": "^17.0.32",
    "tsx": "^4.7.0",
    "typescript": "^5.3.3"
  }
}
```

**Usage examples:**

```bash
# Analyze a single article
npm start analyze ./articles/quantum-breakthrough.txt

# Analyze with output file
npm start analyze ./articles/quantum-breakthrough.txt -o results.json

# Batch process with concurrency control
npm start batch ./articles -c 5

# Batch process and save as CSV
npm start batch ./articles -o results.csv -f csv

# View analysis history
npm start history

# View limited history
npm start history -l 10

# View history as JSON
npm start history -f json

# Check service health
npm start health

# Get help for any command
npm start analyze --help
npm start batch --help

# Show version
npm start --version
```

---

## Future Directions

As you build more sophisticated integrations with the News Analysis service, consider:

### Web Integration
- **REST API Wrapper**: Expose analysis via Express/Fastify for web apps
- **Real-time Dashboard**: WebSocket streaming for live analysis updates
- **React/Vue Frontend**: Build interactive UI for article submission and results

### Data Pipeline Integration
- **RSS Feed Processing**: Automatically analyze articles from news feeds
- **Webhook Integration**: Trigger analysis from external events
- **Database Storage**: Persist analyses for historical trending

### Advanced Analytics
- **Trend Analysis**: Track impact patterns over time
- **Alert System**: Notify on high-impact articles
- **Comparison**: Compare similar articles across time periods

### Performance Optimization
- **Caching**: Cache results for duplicate articles
- **Rate Limiting**: Implement client-side rate limiting
- **Connection Pooling**: Reuse client instances efficiently

---

## Troubleshooting

**"Service is not healthy"**
- Verify the agent bundle is running: `kubectl get pods`
- Check port-forwarding: `kubectl port-forward svc/news-analysis 3001:3001`
- Review agent bundle logs for startup errors

**"Entity not found"**
- Verify the collection ID is correct
- Check agent bundle logs for the created collection ID
- Ensure the agent bundle initialized successfully

**Timeout errors**
- Increase timeout: `timeout: 120000` for complex analyses
- Check network latency to agent bundle
- Review agent bundle resource limits

**Type errors**
- Ensure shared types package is up to date
- Verify agent bundle version matches expected types
- Check imports are from correct package

**Low confidence scores**
- Article may be too short or vague
- Try articles with clear industry-specific content
- Review the bot's prompt configuration if confidence is consistently low

---

## Best Practices

1. **Always check health before batch operations**
2. **Use type safety** via shared types package
3. **Implement retry logic** for production resilience
4. **Process in batches** with controlled concurrency
5. **Export results** for analysis and archival
6. **Handle errors gracefully** with meaningful messages
7. **Monitor performance** and adjust timeouts accordingly
8. **Validate inputs** before submitting to agent bundle
9. **Cache results** for duplicate articles
10. **Log correlation IDs** for distributed tracing

---

## Additional Resources

- **General FF SDK Tutorial**: For core concepts and patterns
- **AgentSDK Getting Started**: Learn how the agent bundle works internally
- **Platform Overview**: Understand the FireFoundry architecture
- **API Reference**: Complete FF SDK API documentation

---

**Generated by Claude**