# A Guide to Writing Bots: From Simple Data to Interactive Behavior

Welcome! In the previous tutorial, we crafted a powerful and dynamic `CodeReviewAssistantPrompt`. Now, it's time to bring that prompt to life by building a Bot that uses it.

### The Goal
We will create a **`CodeReviewBot`** that takes our prompt and turns it into a fully functional AI behavior. We'll start with the simplest implementation and incrementally add advanced capabilities like tool usage and automatic error recovery.

### The Role of a Bot
If a Prompt is the "blueprint" for an AI task, the Bot is the "engine" or "brain" that executes it. A Bot is responsible for:
-   Interacting with the LLM.
-   Parsing and validating the LLM's response.
-   Calling external tools (like linters or APIs).
-   Handling errors and retrying automatically.

### The Scope
We will build and run the Bot in isolation. The final step of connecting it to your application's state via an Entity is covered in other guides, like the main **[Getting Started Guide]**.

---

## Chapter 1: The Simplest Bot for a Structured Task

Our first goal is simple: create a bot that takes code as input and reliably returns the structured JSON data defined by our prompt's schema. We'll use a specialized base class to make this incredibly easy.

### Step 1: Type Helpers (`BotTypeHelper`)
Just as our prompt had a PTH, our bot needs a `BotTypeHelper` (BTH). The BTH connects the prompt's types to the bot's final, expected output type—in our case, the TypeScript type inferred from our `CodeReviewOutputSchema`.

```typescript
import { BotTypeHelper } from '@firebrandanalytics/ff-agent-sdk';
import { z } from 'zod';
import { CodeReviewPTH } from '../prompts/CodeReviewAssistantPrompt.js'; // From previous tutorial
import { CodeReviewOutputSchema } from '../prompts/schemas.js'; // From previous tutorial

// The final output type is inferred directly from our Zod schema.
type CodeReviewOutput = z.infer<typeof CodeReviewOutputSchema>;

// The BTH links the Prompt's types (PTH) with the final Output type.
type CodeReviewBTH = BotTypeHelper<CodeReviewPTH, CodeReviewOutput>;
```

### Step 2: The "Quick Win" with `StructuredDataBot`
For any task that should return validated JSON, the `StructuredDataBot` base class is the "easy button." It automatically handles prompting the LLM, parsing the JSON response, and validating it against your Zod schema.

```typescript
import {
  StructuredDataBot,
  StructuredDataBotConfig,
  PromptGroup,
  PromptInputText,
} from '@firebrandanalytics/ff-agent-sdk';
import { CodeReviewAssistantPrompt } from '../prompts/CodeReviewAssistantPrompt.js';

export class CodeReviewBot extends StructuredDataBot<
  typeof CodeReviewOutputSchema,
  CodeReviewBTH,
  CodeReviewPTH
> {
  constructor() {
    // We assemble a PromptGroup, just like in the last tutorial.
    const promptGroup = new PromptGroup<CodeReviewPTH>([
      { name: 'system_instructions', prompt: new CodeReviewAssistantPrompt({ programming_language: 'TypeScript' }) },
      { name: 'user_input', prompt: new PromptInputText<CodeReviewPTH>({}) }
    ]);

    const config: StructuredDataBotConfig<typeof CodeReviewOutputSchema, CodeReviewPTH> = {
      name: "CodeReviewBot",
      schema: CodeReviewOutputSchema, // The schema to validate against
      schema_description: "A structured code review with overall feedback and specific comments.",
      base_prompt_group: promptGroup,
      model_pool_name: "firebrand_completion_default"
    };
    
    super(config);
  }
}
```

### Step 3: Running the Bot
Let's see it in action! A simple function can instantiate and run our bot.

```typescript
import { BotRequest } from '@firebrandanalytics/ff-agent-sdk';

async function runSimpleBot() {
  const bot = new CodeReviewBot();
  const sampleCode = `function add(a, b) { return a+b; }`;

  const request = new BotRequest<CodeReviewBTH>({
    id: 'review-request-1',
    input: sampleCode, // This is passed to the `PromptInputText`
    args: { user_name: 'Alex' } // These are passed to our system prompt
  });

  console.log('🤖 Running CodeReviewBot...');
  const response = await bot.run(request);
  
  console.log('✅ Review Complete! Output:');
  console.log(response.output);
}

runSimpleBot();
```

Just like that, you have a working bot that reliably returns a clean, typed, and validated JavaScript object.

---

## Chapter 2: Adding Custom Business Logic Validation

Schema validation is great for structure, but what about logic? An AI can produce a schema-correct response that is still logically flawed. For example, it could comment on a line number that doesn't exist. Let's add a custom check for this.

### Step 1: Extending `postprocess_generator`
We can add our own validation logic by overriding the `postprocess_generator` method. We'll still use the `StructuredDataBot`'s powerful parsing and schema validation by calling `super`.

```typescript
// Inside the CodeReviewBot class...
  import { BotTryRequest, FFError } from '@firebrandanalytics/ff-agent-sdk';
  import { BotPostprocessGenerator } from '@firebrandanalytics/ff-agent-sdk';

  protected override async *postprocess_generator(
    broker_content: any,
    request: BotTryRequest<CodeReviewBTH>
  ): BotPostprocessGenerator<CodeReviewBTH> {
    // 1. Get the schema-validated data from the parent StructuredDataBot.
    //    This is guaranteed to be a valid object that matches our Zod schema.
    const validatedData: CodeReviewOutput = yield* super.postprocess_generator(broker_content, request);

    // 2. Add our own custom business logic validation.
    const lineCount = request.parent.input.split('\n').length;
    for (const comment of validatedData.comments) {
      if (comment.line_number > lineCount) {
        // 3. If the logic fails, throw an error. The bot framework will
        //    catch this and trigger an automatic retry with this error message.
        throw new FFError(
          `The review contains a comment for line ${comment.line_number}, but the code is only ${lineCount} lines long. Please correct the line numbers.`
        );
      }
    }

    // 4. If all checks pass, return the data.
    return validatedData;
  }
```
This pattern is incredibly powerful. It lets you leverage the framework's helpers for the heavy lifting (JSON parsing, schema validation) while easily layering on your own domain-specific rules. If our custom check fails, the bot will automatically retry and "tell" the LLM about its mistake, guiding it toward a logically correct answer.

---

## Chapter 3: Agentic Behavior with Dynamic Tool Calls

Our bot is getting smarter, but we can make it truly agentic. What if, after generating a code suggestion, the AI could validate its *own* suggestion to ensure it's correct before showing it to the user? This is a perfect use case for a dynamic **Tool Call**.

To get this power, we need to graduate from the convenient `StructuredDataBot` to the more flexible base `Bot`.

### Step 1: Refactor to Extend the Base `Bot`
This refactoring is well-motivated: we're trading convenience for the power to create dynamic, multi-step agent behaviors.

```typescript
import { Bot, BotConfig } from '@firebrandanalytics/ff-agent-sdk';
// We now extend the base Bot class
export class CodeReviewBot extends Bot<CodeReviewBTH> {
  constructor() {
    // ... same PromptGroup as before ...
    const config: BotConfig<CodeReviewBTH> = {
      name: "CodeReviewBot",
      base_prompt_group: promptGroup,
      model_pool_name: "firebrand_completion_default",
      max_tries: 3 // It's good practice to set a retry limit
    };
    super(config);
  }
  // ... we will re-implement postprocess_generator and add tools next ...
}
```

### Step 2: Define and Register a Validation Tool
Our tool will check if a suggested code snippet is valid. For this tutorial, our mock "validator" will check for TypeScript type annotations.

```typescript
import { DispatchTable } from '@firebrandanalytics/ff-agent-sdk';

// This is our mock validation tool.
async function validateSuggestion(request: any, args: { suggestion: string }): Promise<string[]> {
  console.log('🔧 Validating AI suggestion...');
  if (args.suggestion.includes(': number')) {
    return []; // No errors, suggestion is valid!
  }
  return ['Suggestion is missing explicit type annotations (e.g., `: number`).'];
}

const reviewDispatchTable: DispatchTable<CodeReviewPTH, CodeReviewOutput> = {
  validateSuggestion: {
    func: validateSuggestion,
    spec: { // The schema tells the LLM how to use the tool
      name: 'validateSuggestion',
      description: 'Validates a TypeScript code suggestion to ensure it follows best practices like including type annotations.',
      parameters: {
        type: 'object',
        properties: {
          suggestion: { type: 'string', description: 'The suggested code snippet to validate.' }
        },
        required: ['suggestion']
      }
    }
  }
};

// Now, update the bot's constructor to include the dispatch table
export class CodeReviewBot extends Bot<CodeReviewBTH> {
  constructor() {
    // ...
    const config: BotConfig<CodeReviewBTH> = {
      // ...
      dispatch_table: reviewDispatchTable // Register the tool
    };
    super(config);
  }
  // ...
}
```

### Step 3: Update the Prompt to Use the Tool
The LLM won't use the tool unless we tell it to! This is the most critical step: we instruct the AI on its new agentic workflow.

```typescript
// In CodeReviewAssistantPrompt, add a new section:
  get_Validation_Workflow_Section(): PromptTemplateNode<CodeReviewPTH> {
    return new PromptTemplateSectionNode<CodeReviewPTH>({
      semantic_type: 'rule',
      content: 'Validation Workflow:',
      children: [
        'IMPORTANT: When you decide to provide a code suggestion, you MUST first validate it using the `validateSuggestion` tool.',
        'If the tool returns any errors, you MUST modify your suggestion to fix the errors and then call the tool again.',
        'Do NOT include a suggestion in your final JSON output unless it has been successfully validated by the tool (i.e., the tool returned an empty array).',
      ]
    });
  }
```

### Step 4: The Generate-Then-Validate Lifecycle
By adding this tool and prompt, we've created a sophisticated, agentic loop that happens between our Bot and the LLM:

1.  **SDK -> LLM:** "Review this code: `function add(a,b){return a+b}`."
2.  **LLM (reasoning):** "This code lacks types. I will suggest `function add(a: number, b: number): number { return a + b; }`. My instructions say I must validate this suggestion before I can finish."
3.  **LLM -> SDK:** "Please call the `validateSuggestion` tool for me on this code snippet: `function add(a: number, b: number): number { return a + b; }`."
4.  **SDK:** The framework executes your `validateSuggestion` function. It passes validation.
5.  **SDK -> LLM:** "The `validateSuggestion` tool returned `[]` (no errors)."
6.  **LLM (reasoning):** "Excellent. My suggestion is valid. Now I can confidently construct the final JSON output."
7.  **LLM -> SDK:** (Sends the final, complete JSON object with the validated suggestion).

This is the power of dynamic tool calls: the AI uses tools as part of its own internal reasoning process to improve the quality of its final answer.

### Step 5: Implement `postprocess_generator` with Tool Validation
Since we are no longer using `StructuredDataBot`, we are now responsible for the full parsing and validation logic. This is also where we can add our most advanced validation: ensuring the LLM followed our workflow instructions.

```typescript
// Inside the CodeReviewBot class
import { extractJSON } from '@firebrandanalytics/ff-agent-sdk'; // A helpful utility

protected override async *postprocess_generator(
    broker_content: any,
    request: BotTryRequest<CodeReviewBTH>
): BotPostprocessGenerator<CodeReviewBTH> {
    // 1. Manually parse the LLM's text response.
    let parsedJson;
    try {
        const jsonString = extractJSON(broker_content.content);
        parsedJson = JSON.parse(jsonString);
    } catch (error) {
        throw new FFError(`Response was not valid JSON. Error: ${error.message}`);
    }

    // 2. Manually validate against our Zod schema.
    const validationResult = CodeReviewOutputSchema.safeParse(parsedJson);
    if (!validationResult.success) {
        throw new FFError(`JSON did not match the required schema. Errors: ${validationResult.error.toString()}`);
    }
    const reviewOutput = validationResult.data;

    // 3. ADVANCED VALIDATION: Check if the tool workflow was followed.
    // The BotRequest object tracks all tools that were called during this try.
    const toolCalls = request.get_tool_call_results();
    const validatorWasCalled = Object.values(toolCalls).some(
        (call) => call.func_name === 'validateSuggestion'
    );

    const hasSuggestion = reviewOutput.comments.some(c => !!c.suggestion);
    
    // If the LLM provided a suggestion but skipped our mandatory validation step,
    // we reject the output and force a retry.
    if (hasSuggestion && !validatorWasCalled) {
        throw new FFError("The review contains a code suggestion, but the `validateSuggestion` tool was not called. You must validate all suggestions.");
    }

    return reviewOutput;
}
```

---

## Chapter 4: Conclusion and Next Steps

Congratulations! You've built a truly robust and powerful bot.

**Our bot evolved from:**
*   A simple, schema-validated data-fetcher using `StructuredDataBot`.
*   To a more intelligent agent with custom business logic validation.
*   To a powerful, tool-using bot that can orchestrate a dynamic, agentic workflow.
*   And finally, to a resilient, self-correcting system that validates both the structure and the *process* of the AI's output.

You now understand the core patterns of the Bot framework: starting simple with helpers, extending them for custom logic, and graduating to the base classes for maximum power and control.

**The Path Forward:**
You now have a complete, self-contained AI behavior. The final step is to connect it to your application's data and state. To learn how to wrap this Bot in a stateful, resumable Entity, see the **[Getting Started Guide]**.