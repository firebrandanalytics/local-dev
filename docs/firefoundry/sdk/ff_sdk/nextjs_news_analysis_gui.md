# Building a Next.js GUI for the News Analysis Agent Bundle

## Introduction

This tutorial demonstrates how to build a web interface for the News Analysis Agent Bundle using Next.js. We'll create a clean, modern UI that allows users to submit news articles and view structured impact analysis results.

**What we'll build:**
- Article submission form with validation
- Real-time loading states during analysis
- Structured display of impact analysis results
- Clean, responsive design using Tailwind CSS

**Prerequisites:**
- News Analysis Agent Bundle deployed and accessible (e.g., at `http://localhost:3001`)
- Node.js 20+ and pnpm installed
- Basic familiarity with React and Next.js
- Completed monorepo setup with AgentSDK

---

## Part 1: Project Setup

### Adding the Next.js Template to Your Monorepo

FireFoundry CLI provides a command to scaffold the Next.js template into your existing monorepo:

```bash
# From your monorepo root
ff-cli add-nextjs-template

# This creates:
# packages/web-ui/          (Next.js application)
```

**Manual Setup Alternative:**

If you're setting up manually, create the workspace structure:

```bash
# From monorepo root
mkdir -p packages/web-ui packages/shared-types

# Copy Next.js template files to packages/web-ui
# (or use the ff-cli command above)
```

### Workspace Configuration

Ensure your root `package.json` has the workspace configuration:

```json
{
  "name": "news-analysis-monorepo",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev:web": "pnpm --filter web-ui dev",
    "build:web": "pnpm --filter web-ui build",
    "dev:bundle": "pnpm --filter news-analysis-bundle dev"
  }
}
```

### Creating the Shared Types Package

Create `packages/shared-types/package.json`:

```json
{
  "name": "@my-workspace/shared-types",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  }
}
```

Create `packages/shared-types/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["dist", "node_modules"]
}
```

Create `packages/shared-types/src/index.ts`:

```typescript
/**
 * Shared types for News Analysis
 * Used by both agent bundle and web UI
 */

export interface VerticalImpact {
  impact_level: 'none' | 'low' | 'medium' | 'high' | 'critical';
  confidence: number; // 0-1
  reasoning: string;
  key_factors: string[];
}

export interface IMPACT_ANALYSIS_OUTPUT {
  article_summary: string;
  healthcare: VerticalImpact;
  shipping_logistics: VerticalImpact;
  technology: VerticalImpact;
  overall_significance: 'low' | 'medium' | 'high';
}

export interface AnalysisRequest {
  article: string;
  metadata?: {
    source_url?: string;
    published_date?: string;
  };
}

export interface AnalysisResponse {
  success: boolean;
  data?: IMPACT_ANALYSIS_OUTPUT;
  error?: string;
}
```

Build the shared types:

```bash
cd packages/shared-types
pnpm install
pnpm build
```

### Installing Dependencies

Add the shared types dependency to your Next.js app:

```bash
cd packages/web-ui
pnpm add @my-workspace/shared-types@workspace:*
pnpm add @firebrandanalytics/ff-sdk
```

---

## Part 2: Configuration

### Environment Variables

Create `packages/web-ui/.env.local`:

```bash
# Agent Bundle Connection
NEXT_PUBLIC_AGENT_BUNDLE_URL=http://localhost:3001
NEXT_PUBLIC_AGENT_BUNDLE_API_KEY=your-api-key-here
NEXT_PUBLIC_COLLECTION_ID=c0000000-0000-0000-0001-000000000001

# App Configuration
NEXT_PUBLIC_APP_NAME=News Analysis
```

**Note:** Replace the values with your actual agent bundle URL, API key, and collection ID from your deployment.

### API Client Setup

Create `packages/web-ui/src/lib/analysisClient.ts`:

```typescript
import { RemoteAgentBundleClient } from '@firebrandanalytics/ff-sdk';
import type { IMPACT_ANALYSIS_OUTPUT, AnalysisRequest } from '@my-workspace/shared-types';

// Configuration from environment variables
const config = {
  baseUrl: process.env.NEXT_PUBLIC_AGENT_BUNDLE_URL || 'http://localhost:3001',
  apiKey: process.env.NEXT_PUBLIC_AGENT_BUNDLE_API_KEY,
  collectionId: process.env.NEXT_PUBLIC_COLLECTION_ID || 'c0000000-0000-0000-0001-000000000001'
};

// Create singleton client instance
const client = new RemoteAgentBundleClient(config.baseUrl, {
  api_key: config.apiKey,
  timeout: 120000 // 2 minutes for analysis
});

/**
 * Analyze a news article
 */
export async function analyzeArticle(
  request: AnalysisRequest
): Promise<IMPACT_ANALYSIS_OUTPUT> {
  const result = await client.invoke_entity_method<IMPACT_ANALYSIS_OUTPUT>(
    config.collectionId,
    'analyze_article',
    request.article,
    request.metadata
  );

  return result;
}

/**
 * Health check
 */
export async function checkHealth(): Promise<boolean> {
  try {
    return await client.health_check();
  } catch {
    return false;
  }
}

export { config };
```

---

## Part 3: Building the UI Components

### Main Page

Replace `packages/web-ui/src/app/page.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { AnalysisForm } from '@/components/AnalysisForm';
import { AnalysisResults } from '@/components/AnalysisResults';
import { ErrorDisplay } from '@/components/ErrorDisplay';
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';

export default function HomePage() {
  const [analysis, setAnalysis] = useState<IMPACT_ANALYSIS_OUTPUT | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleAnalysisComplete = (result: IMPACT_ANALYSIS_OUTPUT) => {
    setAnalysis(result);
    setError(null);
  };

  const handleError = (errorMessage: string) => {
    setError(errorMessage);
    setAnalysis(null);
  };

  const handleReset = () => {
    setAnalysis(null);
    setError(null);
  };

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white border-b shadow-sm">
        <div className="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8 py-6">
          <h1 className="text-3xl font-bold text-gray-900">
            📰 News Impact Analyzer
          </h1>
          <p className="mt-2 text-gray-600">
            Analyze news articles for business impact across healthcare, logistics, and technology sectors
          </p>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
        <div className="space-y-8">
          {/* Analysis Form */}
          <AnalysisForm
            onAnalysisComplete={handleAnalysisComplete}
            onError={handleError}
            onLoadingChange={setIsLoading}
            disabled={isLoading}
          />

          {/* Error Display */}
          {error && <ErrorDisplay error={error} onDismiss={() => setError(null)} />}

          {/* Results Display */}
          {analysis && (
            <AnalysisResults 
              analysis={analysis} 
              onReset={handleReset}
            />
          )}
        </div>
      </main>

      <footer className="bg-white border-t mt-12">
        <div className="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8 py-6">
          <p className="text-center text-gray-500 text-sm">
            Powered by FireFoundry Agent Bundle
          </p>
        </div>
      </footer>
    </div>
  );
}
```

### Analysis Form Component

Create `packages/web-ui/src/components/AnalysisForm.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { analyzeArticle } from '@/lib/analysisClient';
import type { IMPACT_ANALYSIS_OUTPUT } from '@my-workspace/shared-types';

interface AnalysisFormProps {
  onAnalysisComplete: (result: IMPACT_ANALYSIS_OUTPUT) => void;
  onError: (error: string) => void;
  onLoadingChange: (loading: boolean) => void;
  disabled?: boolean;
}

export function AnalysisForm({
  onAnalysisComplete,
  onError,
  onLoadingChange,
  disabled = false
}: AnalysisFormProps) {
  const [articleText, setArticleText] = useState('');
  const [sourceUrl, setSourceUrl] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!articleText.trim()) {
      onError('Please enter article text');
      return;
    }

    onLoadingChange(true);

    try {
      const result = await analyzeArticle({
        article: articleText,
        metadata: sourceUrl ? { source_url: sourceUrl } : undefined
      });

      onAnalysisComplete(result);
    } catch (error) {
      onError(
        error instanceof Error 
          ? error.message 
          : 'Failed to analyze article. Please try again.'
      );
    } finally {
      onLoadingChange(false);
    }
  };

  const handleClear = () => {
    setArticleText('');
    setSourceUrl('');
  };

  const exampleArticle = `Breaking: Major tech company announces breakthrough in quantum computing that could revolutionize data processing across industries. The new quantum processor demonstrates unprecedented speed improvements in complex calculations, potentially impacting everything from drug discovery to logistics optimization.

The breakthrough addresses several key challenges in quantum error correction and could accelerate commercial applications by 3-5 years. Healthcare researchers are particularly excited about potential applications in protein folding simulations and personalized medicine. Supply chain experts see opportunities for optimizing complex routing problems that currently take hours to solve.`;

  const handleLoadExample = () => {
    setArticleText(exampleArticle);
    setSourceUrl('https://example.com/quantum-breakthrough');
  };

  return (
    <div className="bg-white rounded-lg shadow-sm border p-6">
      <form onSubmit={handleSubmit} className="space-y-6">
        {/* Article Text Input */}
        <div>
          <label htmlFor="article" className="block text-sm font-medium text-gray-700 mb-2">
            Article Text *
          </label>
          <textarea
            id="article"
            value={articleText}
            onChange={(e) => setArticleText(e.target.value)}
            placeholder="Paste news article text here..."
            disabled={disabled}
            className="w-full h-64 px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:bg-gray-50 disabled:text-gray-500"
            required
          />
          <p className="mt-2 text-sm text-gray-500">
            {articleText.length} characters
          </p>
        </div>

        {/* Source URL Input */}
        <div>
          <label htmlFor="source" className="block text-sm font-medium text-gray-700 mb-2">
            Source URL (optional)
          </label>
          <input
            id="source"
            type="url"
            value={sourceUrl}
            onChange={(e) => setSourceUrl(e.target.value)}
            placeholder="https://example.com/article"
            disabled={disabled}
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:bg-gray-50 disabled:text-gray-500"
          />
        </div>

        {/* Action Buttons */}
        <div className="flex gap-3">
          <button
            type="submit"
            disabled={disabled || !articleText.trim()}
            className="flex-1 bg-blue-600 text-white px-6 py-3 rounded-lg font-medium hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
          >
            {disabled ? (
              <span className="flex items-center justify-center gap-2">
                <svg className="animate-spin h-5 w-5" viewBox="0 0 24 24">
                  <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" fill="none" />
                  <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z" />
                </svg>
                Analyzing...
              </span>
            ) : (
              'Analyze Article'
            )}
          </button>
          
          <button
            type="button"
            onClick={handleClear}
            disabled={disabled}
            className="px-6 py-3 border border-gray-300 rounded-lg font-medium hover:bg-gray-50 disabled:bg-gray-100 disabled:cursor-not-allowed transition-colors"
          >
            Clear
          </button>

          <button
            type="button"
            onClick={handleLoadExample}
            disabled={disabled}
            className="px-6 py-3 border border-blue-300 text-blue-600 rounded-lg font-medium hover:bg-blue-50 disabled:bg-gray-100 disabled:cursor-not-allowed transition-colors"
          >
            Load Example
          </button>
        </div>
      </form>
    </div>
  );
}
```

### Analysis Results Component

Create `packages/web-ui/src/components/AnalysisResults.tsx`:

```typescript
'use client';

import type { IMPACT_ANALYSIS_OUTPUT, VerticalImpact } from '@my-workspace/shared-types';

interface AnalysisResultsProps {
  analysis: IMPACT_ANALYSIS_OUTPUT;
  onReset?: () => void;
}

export function AnalysisResults({ analysis, onReset }: AnalysisResultsProps) {
  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="bg-white rounded-lg shadow-sm border p-6">
        <div className="flex items-start justify-between">
          <div className="flex-1">
            <h2 className="text-xl font-bold text-gray-900 mb-2">
              Analysis Results
            </h2>
            <p className="text-gray-600">{analysis.article_summary}</p>
          </div>
          {onReset && (
            <button
              onClick={onReset}
              className="ml-4 px-4 py-2 text-sm border border-gray-300 rounded-lg hover:bg-gray-50"
            >
              New Analysis
            </button>
          )}
        </div>

        {/* Overall Significance Badge */}
        <div className="mt-4">
          <SignificanceBadge significance={analysis.overall_significance} />
        </div>
      </div>

      {/* Vertical Impact Cards */}
      <div className="grid md:grid-cols-3 gap-6">
        <ImpactCard
          title="Healthcare"
          icon="🏥"
          impact={analysis.healthcare}
        />
        <ImpactCard
          title="Shipping & Logistics"
          icon="🚢"
          impact={analysis.shipping_logistics}
        />
        <ImpactCard
          title="Technology"
          icon="💻"
          impact={analysis.technology}
        />
      </div>
    </div>
  );
}

function SignificanceBadge({ significance }: { significance: 'low' | 'medium' | 'high' }) {
  const styles = {
    low: 'bg-gray-100 text-gray-800',
    medium: 'bg-yellow-100 text-yellow-800',
    high: 'bg-red-100 text-red-800'
  };

  return (
    <span className={`inline-flex items-center px-3 py-1 rounded-full text-sm font-medium ${styles[significance]}`}>
      Overall Significance: {significance.toUpperCase()}
    </span>
  );
}

function ImpactCard({ title, icon, impact }: { 
  title: string; 
  icon: string; 
  impact: VerticalImpact;
}) {
  const impactColors = {
    none: 'bg-gray-100 text-gray-800 border-gray-200',
    low: 'bg-green-100 text-green-800 border-green-200',
    medium: 'bg-yellow-100 text-yellow-800 border-yellow-200',
    high: 'bg-orange-100 text-orange-800 border-orange-200',
    critical: 'bg-red-100 text-red-800 border-red-200'
  };

  const impactEmojis = {
    none: '⚪',
    low: '🟢',
    medium: '🟡',
    high: '🟠',
    critical: '🔴'
  };

  return (
    <div className="bg-white rounded-lg shadow-sm border p-6">
      {/* Header */}
      <div className="flex items-center gap-2 mb-4">
        <span className="text-2xl">{icon}</span>
        <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
      </div>

      {/* Impact Level */}
      <div className={`px-3 py-2 rounded-lg border-2 mb-4 ${impactColors[impact.impact_level]}`}>
        <div className="flex items-center justify-between">
          <span className="font-medium">
            {impactEmojis[impact.impact_level]} {impact.impact_level.toUpperCase()}
          </span>
          <span className="text-sm">
            {Math.round(impact.confidence * 100)}% confidence
          </span>
        </div>
      </div>

      {/* Reasoning */}
      <div className="mb-4">
        <h4 className="text-sm font-medium text-gray-700 mb-2">Reasoning</h4>
        <p className="text-sm text-gray-600">{impact.reasoning}</p>
      </div>

      {/* Key Factors */}
      <div>
        <h4 className="text-sm font-medium text-gray-700 mb-2">Key Factors</h4>
        <ul className="space-y-1">
          {impact.key_factors.map((factor, idx) => (
            <li key={idx} className="text-sm text-gray-600 flex items-start gap-2">
              <span className="text-blue-600 mt-1">•</span>
              <span>{factor}</span>
            </li>
          ))}
        </ul>
      </div>

      {/* Confidence Bar */}
      <div className="mt-4 pt-4 border-t">
        <div className="flex items-center justify-between mb-1">
          <span className="text-xs text-gray-500">Confidence</span>
          <span className="text-xs font-medium text-gray-700">
            {Math.round(impact.confidence * 100)}%
          </span>
        </div>
        <div className="w-full bg-gray-200 rounded-full h-2">
          <div
            className="bg-blue-600 h-2 rounded-full transition-all"
            style={{ width: `${impact.confidence * 100}%` }}
          />
        </div>
      </div>
    </div>
  );
}
```

### Error Display Component

Create `packages/web-ui/src/components/ErrorDisplay.tsx`:

```typescript
'use client';

interface ErrorDisplayProps {
  error: string;
  onDismiss?: () => void;
}

export function ErrorDisplay({ error, onDismiss }: ErrorDisplayProps) {
  return (
    <div className="bg-red-50 border-2 border-red-200 rounded-lg p-4">
      <div className="flex items-start gap-3">
        <div className="flex-shrink-0">
          <svg className="h-6 w-6 text-red-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
        </div>
        <div className="flex-1">
          <h3 className="text-sm font-medium text-red-800 mb-1">
            Analysis Error
          </h3>
          <p className="text-sm text-red-700">{error}</p>
        </div>
        {onDismiss && (
          <button
            onClick={onDismiss}
            className="flex-shrink-0 text-red-600 hover:text-red-800"
          >
            <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
            </svg>
          </button>
        )}
      </div>
    </div>
  );
}
```

---

## Part 4: Running the Application

### Development Workflow

From your monorepo root:

```bash
# Install all dependencies
pnpm install

# Start the agent bundle (if not already running)
pnpm dev:bundle

# In another terminal, start the Next.js app
pnpm dev:web
```

The Next.js application will be available at `http://localhost:3000`.

### Testing the Application

1. **Load the example article** using the "Load Example" button
2. **Submit for analysis** - You should see a loading spinner
3. **View results** - After ~10-30 seconds, the analysis will appear below
4. **Try different articles** - Clear and paste new content to analyze

### Troubleshooting

**Connection Errors:**
- Verify the agent bundle is running at the URL in `.env.local`
- Check the API key is correct
- Ensure the collection ID exists in your agent bundle

**Type Errors:**
- Rebuild shared types: `cd packages/shared-types && pnpm build`
- Restart Next.js dev server after type changes

**Slow Analysis:**
- Analysis typically takes 10-30 seconds depending on article length
- Check agent bundle logs for errors
- Increase timeout in `analysisClient.ts` if needed

---

## Part 5: Enhancements and Next Steps

### Adding Loading State Improvements

Update the form to show better progress feedback:

```typescript
// In AnalysisForm.tsx, add a progress message state
const [progressMessage, setProgressMessage] = useState('');

// Update handleSubmit:
setProgressMessage('Connecting to agent bundle...');
// ... then
setProgressMessage('Analyzing article impact...');
// ... then
setProgressMessage('Generating insights...');
```

### Adding Local Storage for Recent Analyses

```typescript
// In page.tsx, add localStorage persistence
useEffect(() => {
  if (analysis) {
    const recent = JSON.parse(localStorage.getItem('recentAnalyses') || '[]');
    recent.unshift({
      timestamp: new Date().toISOString(),
      summary: analysis.article_summary,
      data: analysis
    });
    localStorage.setItem('recentAnalyses', JSON.stringify(recent.slice(0, 10)));
  }
}, [analysis]);
```

### Validation and Character Limits

```typescript
// Add to AnalysisForm.tsx
const MIN_CHARS = 100;
const MAX_CHARS = 10000;

const isValid = articleText.length >= MIN_CHARS && articleText.length <= MAX_CHARS;

// Show validation message
{articleText.length > 0 && (
  <p className={`text-sm ${isValid ? 'text-gray-500' : 'text-red-600'}`}>
    {articleText.length < MIN_CHARS && `Minimum ${MIN_CHARS} characters required`}
    {articleText.length > MAX_CHARS && `Maximum ${MAX_CHARS} characters exceeded`}
  </p>
)}
```

### Export Results Feature

```typescript
// Add export button to AnalysisResults
function exportToJSON() {
  const dataStr = JSON.stringify(analysis, null, 2);
  const dataBlob = new Blob([dataStr], { type: 'application/json' });
  const url = URL.createObjectURL(dataBlob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `analysis-${Date.now()}.json`;
  link.click();
}

// Add button
<button onClick={exportToJSON} className="...">
  Export Results
</button>
```

---

## Part 6: Production Considerations

### Environment-Specific Configuration

For different environments, create multiple env files:

```bash
# .env.local (development)
NEXT_PUBLIC_AGENT_BUNDLE_URL=http://localhost:3001

# .env.production (production)
NEXT_PUBLIC_AGENT_BUNDLE_URL=https://api.example.com

# .env.staging (staging)
NEXT_PUBLIC_AGENT_BUNDLE_URL=https://staging-api.example.com
```

### Adding Express Middleware Layer

**Common Production Pattern:** In real deployments, we typically place an Express middleware layer between the Next.js UI and the agent bundle:

```
Next.js UI → Express Middleware → Agent Bundle
```

This provides:
- Authentication and authorization
- Rate limiting
- Request/response transformation
- Caching
- WebSocket streaming support

See the **Express.js Middleware Tutorial** for implementing this pattern.

### Error Boundary

Add a React error boundary for better error handling:

```typescript
// src/components/ErrorBoundary.tsx
'use client';

import { Component, ReactNode } from 'react';

export class ErrorBoundary extends Component<
  { children: ReactNode },
  { hasError: boolean }
> {
  constructor(props: { children: ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="p-8 text-center">
          <h2 className="text-xl font-bold text-red-600 mb-2">
            Something went wrong
          </h2>
          <button
            onClick={() => window.location.reload()}
            className="mt-4 px-4 py-2 bg-blue-600 text-white rounded-lg"
          >
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Analytics and Monitoring

Add basic analytics tracking:

```typescript
// src/lib/analytics.ts
export function trackAnalysis(success: boolean, duration: number) {
  // Send to your analytics service
  console.log('Analysis tracked:', { success, duration });
}

// Use in AnalysisForm:
const startTime = Date.now();
// ... analysis code
trackAnalysis(true, Date.now() - startTime);
```

---

## Complete Project Structure

```
news-analysis-monorepo/
├── packages/
│   ├── news-analysis-bundle/    # Agent bundle (AgentSDK)
│   ├── shared-types/             # Shared TypeScript types
│   │   ├── src/
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web-ui/                   # Next.js application
│       ├── src/
│       │   ├── app/
│       │   │   ├── page.tsx      # Main page
│       │   │   ├── layout.tsx
│       │   │   └── globals.css
│       │   ├── components/
│       │   │   ├── AnalysisForm.tsx
│       │   │   ├── AnalysisResults.tsx
│       │   │   └── ErrorDisplay.tsx
│       │   └── lib/
│       │       └── analysisClient.ts
│       ├── .env.local
│       ├── package.json
│       └── tsconfig.json
├── package.json              # Root workspace config
└── pnpm-workspace.yaml
```

---

## Best Practices

1. **Type Safety**: Always use shared types from the workspace package
2. **Error Handling**: Provide clear, actionable error messages to users
3. **Loading States**: Show progress feedback during long operations
4. **Validation**: Validate input before sending to the agent bundle
5. **Responsive Design**: Ensure the UI works on mobile and desktop
6. **Accessibility**: Use semantic HTML and ARIA labels
7. **Performance**: Consider memoization for expensive calculations
8. **Testing**: Add unit tests for components and integration tests

---

## Additional Resources

- **Express Middleware Tutorial**: Add middleware layer for production
- **WebSocket Streaming Tutorial**: Real-time progress updates
- **FF SDK Documentation**: Complete API reference
- **Agent Bundle Tutorial**: Understanding the backend
- **Next.js Documentation**: Framework features and patterns

---

## Summary

You now have a fully functional Next.js GUI for the News Analysis Agent Bundle! The application provides:

✅ Clean, modern interface for article submission  
✅ Real-time loading states and progress feedback  
✅ Structured display of multi-vertical impact analysis  
✅ Type-safe integration using shared workspace types  
✅ Error handling and validation  
✅ Responsive design with Tailwind CSS  

**Next steps:**
- Add history view to see past analyses
- Implement WebSocket streaming for real-time updates
- Add Express middleware for production deployment
- Create batch upload feature for multiple articles
- Add export and sharing capabilities

---

**Generated by Claude**
