# Getting Started with FireFoundry Agent Bundles

## Introduction and Core Concepts

In this tutorial, we'll build a **News Article Impact Analyzer** that demonstrates the core patterns of FireFoundry Agent Bundles. Our analyzer will:

- **Input**: A plain text news article
- **Output**: Structured impact assessment across three verticals: healthcare, shipping/logistics, and technology
- **Architecture**: Entities handle structure and state, Bots handle AI-powered analysis

The key insight of FireFoundry is the **separation of structure from behavior**:
- **Entities** represent business objects and their relationships in a persistent graph
- **Bots** implement AI-powered behaviors using LLMs
- **Runnable Entities** connect the two, providing context from the entity graph to the bot

## Understanding the Example

Our news analyzer follows this data flow:

```
News Article (text) → ArticleEntity → ImpactAnalysisBot → Structured Analysis
```

**ArticleEntity**: Stores the article text and orchestrates the analysis
**ImpactAnalysisBot**: Uses an LLM to extract structured insights
**NewsCollection**: Factory for creating and managing multiple analyses

## Creating the Output Schema

First, let's define what our structured output looks like using Zod schemas:

```typescript
import { z } from 'zod';
import { withSchemaMetadata } from '@firebrandanalytics/ff-agent-sdk';

// Individual impact assessment for each vertical
const VerticalImpactSchema = withSchemaMetadata(
  z.object({
    impact_level: z.enum(['none', 'low', 'medium', 'high', 'critical'])
      .describe('Level of impact this news will have on the vertical'),
    confidence: z.number().min(0).max(1)
      .describe('Confidence score for this assessment (0-1)'),
    reasoning: z.string()
      .describe('Brief explanation of why this impact level was assigned'),
    key_factors: z.array(z.string())
      .describe('Key factors from the article that drive this assessment')
  }),
  'VerticalImpact',
  'Impact assessment for a specific business vertical'
);

// Main output schema
const ImpactAnalysisSchema = withSchemaMetadata(
  z.object({
    article_summary: z.string()
      .describe('Brief 2-3 sentence summary of the article'),
    healthcare: VerticalImpactSchema
      .describe('Impact assessment for healthcare industry'),
    shipping_logistics: VerticalImpactSchema
      .describe('Impact assessment for shipping and logistics industry'),
    technology: VerticalImpactSchema
      .describe('Impact assessment for technology industry'),
    overall_significance: z.enum(['low', 'medium', 'high'])
      .describe('Overall significance of this news across all verticals')
  }),
  'ImpactAnalysis',
  'Comprehensive impact analysis of a news article across business verticals'
);

type IMPACT_ANALYSIS_OUTPUT = z.infer<typeof ImpactAnalysisSchema>;
```

The `withSchemaMetadata` function helps the LLM understand our schema structure in natural language.

## Creating the Analysis Prompt

The prompt contains the actual AI logic - this is where we define how the LLM should perform the analysis:

```typescript
import {
  Prompt,
  PromptTemplateNode,
  PromptTemplateSectionNode,
  PromptTemplateListNode,
  PromptTemplateSchemaSetNode,
  PromptTypeHelper
} from '@firebrandanalytics/ff-agent-sdk';

type IMPACT_PROMPT_ARGS = {
  static: { app_name?: string };
  request: {};
};

type IMPACT_PTH = PromptTypeHelper<string, IMPACT_PROMPT_ARGS>;

export class ImpactAnalysisPrompt extends Prompt<IMPACT_PTH> {
  constructor(args: IMPACT_PTH['args']['static']) {
    super('system', args);
    this.add_section(this.get_Context_Section());
    this.add_section(this.get_Analysis_Rules());
    this.add_section(this.get_Verticals_Section());
    this.add_section(this.get_Schema_Section());
  }

  get_Context_Section(): PromptTemplateNode<IMPACT_PTH> {
    return new PromptTemplateSectionNode<IMPACT_PTH>({
      semantic_type: 'context',
      content: 'Context:',
      children: [
        `You are an expert business analyst for ${this.static_args.app_name || 'a consulting firm'}.`,
        `Your task is to analyze news articles and assess their potential impact across different business verticals.`,
        `Provide objective, data-driven assessments with clear reasoning.`
      ]
    });
  }

  get_Analysis_Rules(): PromptTemplateNode<IMPACT_PTH> {
    return new PromptTemplateSectionNode<IMPACT_PTH>({
      semantic_type: 'rule',
      content: 'Analysis Rules:',
      children: [
        new PromptTemplateListNode<IMPACT_PTH>({
          semantic_type: 'rule',
          children: [
            `Focus on direct and indirect business impacts, not just general relevance`,
            `Consider both immediate and potential long-term effects`,
            `Base confidence scores on how clearly the article connects to each vertical`,
            `If impact is unclear or minimal, don't hesitate to assign 'none' or 'low'`,
            `Provide specific, actionable reasoning for each assessment`
          ],
          list_label_function: (_req, _child, idx) => `${idx + 1}. `
        })
      ]
    });
  }

  get_Verticals_Section(): PromptTemplateNode<IMPACT_PTH> {
    return new PromptTemplateSectionNode<IMPACT_PTH>({
      semantic_type: 'context',
      content: 'Business Verticals to Analyze:',
      children: [
        new PromptTemplateListNode<IMPACT_PTH>({
          semantic_type: 'context',
          children: [
            `**Healthcare**: Medical devices, pharmaceuticals, hospitals, health insurance, telemedicine, regulatory changes`,
            `**Shipping & Logistics**: Supply chain, transportation, warehousing, delivery services, freight, port operations`,
            `**Technology**: Software, hardware, cloud services, cybersecurity, AI/ML, telecommunications, fintech`
          ],
          list_label_function: () => '• '
        })
      ]
    });
  }

  get_Schema_Section(): PromptTemplateNode<IMPACT_PTH> {
    const schema_section = new PromptTemplateSectionNode<IMPACT_PTH>({
      semantic_type: 'schema',
      content: 'Output Schema:',
      children: []
    });

    const schemaSetNode = new PromptTemplateSchemaSetNode<IMPACT_PTH>({
      semantic_type: 'schema',
      content: "",
      children: []
    });

    // Add our schemas
    schemaSetNode.addSchemas([
      ImpactAnalysisSchema,
      VerticalImpactSchema
    ]);

    schema_section.add_child(schemaSetNode);
    return schema_section;
  }
}
```

The prompt is organized into clear sections that guide the LLM through the analysis process.

## Building the Analysis Bot

Now we create the bot that wraps our prompt. In this basic example, the bot is primarily a thin wrapper - the real AI logic is in the prompt we just created:

```typescript
import {
  StructuredDataBot,
  StructuredDataBotConfig,
  BotTypeHelper,
  PromptTypeHelper,
  BotTryRequest,
  PromptGroup
} from '@firebrandanalytics/ff-agent-sdk';
import { ImpactAnalysisPrompt } from '../prompts/ImpactAnalysisPrompt.js';

// Define type helpers for type safety
type IMPACT_PROMPT_INPUT = string; // The news article text
type IMPACT_PROMPT_ARGS = {
  static: { 
    app_name?: string;
  };
  request: {};
};

type IMPACT_PTH = PromptTypeHelper<IMPACT_PROMPT_INPUT, IMPACT_PROMPT_ARGS>;
type IMPACT_BTH = BotTypeHelper<IMPACT_PTH, IMPACT_ANALYSIS_OUTPUT>;

export class ImpactAnalysisBot extends StructuredDataBot<
  typeof ImpactAnalysisSchema,
  IMPACT_BTH,
  IMPACT_PTH
> {
  constructor() {
    const prompt_group = new PromptGroup([
      { 
        name: "impact_analysis_prompt", 
        prompt: new ImpactAnalysisPrompt({ app_name: "News Impact Analyzer" })
      },
    ]);

    const config: StructuredDataBotConfig<typeof ImpactAnalysisSchema, IMPACT_PTH> = {
      name: "ImpactAnalysisBot",
      schema: ImpactAnalysisSchema,
      schema_description: "Analyzes news articles for business impact across verticals",
      base_prompt_group: prompt_group,
      model_pool_name: "firebrand_completion_default"
    };
    
    super(config);
  }

  override get_semantic_label_impl(_request: BotTryRequest<IMPACT_BTH>): string {
    return "ImpactAnalysisBotSemanticLabel";
  }
}
```

The `StructuredDataBot` automatically handles:
- Sending the prompt to the LLM
- Extracting JSON from the response
- Validating the output against our schema
- Retrying on errors

For this basic example, we don't need custom validation or error handling - the framework handles it all.

## Creating the Article Entity

The entity stores our article and connects it to the bot:

```typescript
import {
  AddInterface,
  BotRequestArgs,
  EntityNode,
  EntityNodeTypeHelper,
  EntityFactory,
  RunnableEntityBotWrapperDecorator,
  RunnableEntityTypeHelper,
  Context,
  IRunnableEntity
} from '@firebrandanalytics/ff-agent-sdk';
import {
  type UUID,
  type EntityInstanceNodeDTO,
  type JSONObject,
  type JSONValue
} from '@firebrandanalytics/shared-types';
import { ImpactAnalysisBot } from '../bots/ImpactAnalysisBot.js';

// Define the DTO structure for our entity
export interface ArticleEntityDTOData {
  article_text: string;
  source_url?: string;
  published_date?: string;
  [key: string]: JSONValue; // Required for JSONObject compatibility
}

export type ArticleEntityDTO = EntityInstanceNodeDTO<ArticleEntityDTOData> & {
  node_type: "ArticleEntity";
};

// Define type helpers
type ARTICLE_RETH = RunnableEntityTypeHelper<
  EntityNodeTypeHelper<
    any, // ETH
    ArticleEntityDTO,
    "ArticleEntity",
    {}, // edges_from
    {} // edges_to
  >,
  IMPACT_ANALYSIS_OUTPUT,
  {}
>;

@RunnableEntityBotWrapperDecorator(
  {
    generalType: "ArticleEntity",
    specificType: "ArticleEntity",
    allowedConnections: {}
  },
  new ImpactAnalysisBot()
)
export class ArticleEntity
  extends AddInterface<
    typeof EntityNode<ARTICLE_RETH["enh"]>,
    IRunnableEntity<ARTICLE_RETH["enh"]["eth"]["bth"], IMPACT_ANALYSIS_OUTPUT>
  >(EntityNode<ARTICLE_RETH["enh"]>)
{
  constructor(factory: EntityFactory<any>, idOrDto: UUID | ArticleEntityDTO) {
    super(factory, idOrDto);
  }

  // This method connects entity data to bot input
  protected async get_bot_request_args(): Promise<BotRequestArgs<IMPACT_BTH>> {
    const dto = await this.get_dto();

    return {
      id: "impact_analysis_request",
      args: {},
      input: dto.data.article_text, // Send article text to the bot
      context: new Context(dto),
      parent: undefined
    };
  }

  // Helper methods for working with the entity
  async set_article_text(article_text: string): Promise<void> {
    await this.update_data({
      ...(this._dto?.data || {}),
      article_text
    });
  }

  async get_article_text(): Promise<string> {
    const dto = await this.get_dto();
    return dto.data.article_text;
  }
}
```

The `@RunnableEntityBotWrapperDecorator` automatically connects our entity to the bot, and `get_bot_request_args()` defines how entity data flows to the bot.

## Building the Collection Entity

The collection entity acts as a factory for creating and managing article analyses:

```typescript
import {
  EntityNode,
  EntityNodeTypeHelper,
  EntityFactory,
  EntityDecorator
} from '@firebrandanalytics/ff-agent-sdk';
import {
  type UUID,
  type EntityNodeDTO
} from '@firebrandanalytics/shared-types';

@EntityDecorator({
  specificType: 'NewsCollection',
  generalType: 'Class',
  allowedConnections: {
    'Contains': ['ArticleEntity']
  }
})
export class NewsCollection extends EntityNode<
  EntityNodeTypeHelper<any, EntityNodeDTO, 'NewsCollection', {}, {}>
> {
  constructor(factory: EntityFactory<any>, idOrDto: UUID | EntityNodeDTO) {
    super(factory, idOrDto);
  }

  /**
   * Creates a new article analysis and runs it
   */
  async analyze_article(
    article_text: string, 
    metadata?: { source_url?: string; published_date?: string }
  ): Promise<IMPACT_ANALYSIS_OUTPUT> {
    console.log(`📰 Creating analysis for article (${article_text.length} chars)`);

    const dto = await this.get_dto();
    
    if (!article_text || article_text.trim().length === 0) {
      throw new Error('Article text is required for analysis.');
    }

    // Create the ArticleEntity
    const article_dto = await this.factory.create_entity_node({
      app_id: dto.app_id!,
      name: `article-analysis-${Date.now()}`,
      specific_type_name: 'ArticleEntity',
      general_type_name: 'ArticleEntity',
      status: 'Pending',
      data: {
        article_text,
        source_url: metadata?.source_url || null,
        published_date: metadata?.published_date || null
      }
    });

    // Get the entity instance
    const article_entity = await this.factory.get_entity(article_dto.id);

    // Connect it to this collection
    console.log(`🔗 Connecting article ${article_dto.id} to collection ${this.id}`);
    await this.connect_to({
      general_type_name: 'Edge',
      specific_type_name: 'Contains',
      to_node_id: article_dto.id,
      to_node_type: 'ArticleEntity',
      data: {}
    });

    // Run the analysis
    console.log(`🤖 Running impact analysis...`);
    const result = await article_entity.run();
    
    console.log('✅ Analysis complete');
    return result;
  }

  /**
   * Get all analyzed articles in this collection
   */
  async get_all_articles(): Promise<any[]> {
    await this.load();
    const contains_edges = (this.edges_from as any)['Contains'] || [];
    return Promise.all(
      contains_edges.map((edge: any) => edge.get_to())
    );
  }
}
```

This collection entity demonstrates the factory pattern - it creates new entities, connects them via edges, and orchestrates their execution.

## Assembling the Agent Bundle

Finally, let's create the main Agent Bundle that ties everything together:

```typescript
import {
  FFAgentBundle,
  logger,
  app_provider,
  HealthStatus,
  FFConstructors
} from '@firebrandanalytics/ff-agent-sdk';
import { ArticleEntity } from './entities/ArticleEntity.js';
import { NewsCollection } from './entities/NewsCollection.js';

// Register our custom entities
export const NewsAnalysisConstructors = {
  ...FFConstructors,
  'ArticleEntity': ArticleEntity,
  'NewsCollection': NewsCollection
} as const;

export class NewsAnalysisAgentBundle extends FFAgentBundle<any> {
  constructor() {
    super(
      {
        id: "c0000000-0000-0000-0001-000000000000",
        name: "NewsAnalysisService",
        description: "News article impact analysis service using FireFoundry"
      },
      NewsAnalysisConstructors,
      app_provider
    );
  }

  override async init() {
    await super.init();
    logger.info("NewsAnalysisAgentBundle initialized!");
    await this.create_sample_collection();
  }

  async check_readiness(): Promise<HealthStatus> {
    try {
      await this.entity_client.get_node_by_name("news-analysis-root");
      return {
        healthy: true,
        message: "News analysis service ready",
        details: { entity_client_status: "connected" }
      };
    } catch (error) {
      return {
        healthy: false,
        message: "Database connection failed",
        details: {
          error: error instanceof Error ? error.message : "Unknown error",
          entity_client_status: "disconnected"
        }
      };
    }
  }

  private async create_sample_collection() {
    try {
      const collection_name = "news-analysis-root";
      const existing = await this.entity_client.get_node_by_name(collection_name);
      
      if (existing) {
        logger.info("News collection already exists:", existing.id);
        this.print_usage_example(existing.id);
        return;
      }

      const collection_dto = await this.entity_client.create_node({
        app_id: this.get_app_id(),
        name: collection_name,
        specific_type_name: 'NewsCollection',
        general_type_name: 'NewsCollection',
        status: 'Completed',
        data: {}
      });

      logger.info("Created news collection:", collection_dto.id);
      this.print_usage_example(collection_dto.id);
    } catch (error) {
      logger.error("Failed to create collection:", error);
    }
  }

  private print_usage_example(collection_id: string) {
    logger.info("✅ You can now analyze news articles with this API call:");
    logger.info(`curl -X POST http://localhost:3001/invoke \\
  -H "Content-Type: application/json" \\
  -d '{
    "entity_id": "${collection_id}",
    "method_name": "analyze_article",
    "args": [
      "Breaking: Major tech company announces breakthrough in quantum computing that could revolutionize data processing across industries. The new quantum processor demonstrates unprecedented speed improvements in complex calculations, potentially impacting everything from drug discovery to logistics optimization.",
      {
        "source_url": "https://example.com/quantum-breakthrough",
        "published_date": "2024-01-15"
      }
    ]
  }'`);
  }
}
```

## Complete Working Example

Here's how all the pieces work together:

1. **Input**: A news article about quantum computing breakthroughs
2. **NewsCollection.analyze_article()**: Creates an ArticleEntity with the article text
3. **ArticleEntity**: Stores the article and connects to ImpactAnalysisBot
4. **ImpactAnalysisBot**: Uses the LLM to analyze impact across verticals
5. **ImpactAnalysisPrompt**: Structures the LLM request with context and rules
6. **Output**: Structured analysis with impact levels, confidence scores, and reasoning

Example output:
```json
{
  "article_summary": "Major tech company announces quantum computing breakthrough with unprecedented processing speeds, potentially transforming multiple industries.",
  "healthcare": {
    "impact_level": "high",
    "confidence": 0.8,
    "reasoning": "Quantum computing could accelerate drug discovery and medical research",
    "key_factors": ["drug discovery acceleration", "complex medical calculations"]
  },
  "shipping_logistics": {
    "impact_level": "medium", 
    "confidence": 0.7,
    "reasoning": "Optimization algorithms could improve route planning and supply chain efficiency",
    "key_factors": ["logistics optimization", "complex calculations"]
  },
  "technology": {
    "impact_level": "critical",
    "confidence": 0.9,
    "reasoning": "Direct disruption to computing industry with new processing capabilities",
    "key_factors": ["quantum processor", "data processing revolution"]
  },
  "overall_significance": "high"
}
```

## Key Patterns and Takeaways

The FireFoundry Agent Bundle architecture provides several key benefits:

**1. Separation of Concerns**
- Entities handle data structure and persistence
- Bots handle AI-powered logic
- Prompts handle LLM communication

**2. Type Safety**
- Schema-driven development with Zod
- TypeScript type helpers throughout
- Compile-time validation

**3. Composability**  
- Entities can be connected via edges
- Bots can be reused across entities
- Prompts can be modular and reusable

**4. Persistence**
- Entity relationships are automatically persisted
- Working memory stores intermediate results
- Full audit trail of operations

**5. Scalability**
- Background job processing
- Resumable computations
- Progress tracking and monitoring

This architecture shines when building applications that require:
- Complex AI workflows with multiple steps
- Persistent state management
- Structured data extraction and analysis
- Composable business logic
- Audit trails and monitoring

The entity-bot separation pattern makes it easy to test, debug, and extend your AI-powered applications while maintaining clean architectural boundaries.