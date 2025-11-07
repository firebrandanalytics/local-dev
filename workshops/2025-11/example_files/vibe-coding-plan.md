# Agent Bundle: “Math Word Problem Solver”

### What the user does

Paste a word problem → click “Solve”.

### What the bundle does (high-level workflow)

A → B → C linear flow:

1. Analyze the text and extract a solvable equation + reasoning
2. Render that equation into a Newton API GET URL (per your link)
3. Fetch the URL deterministically and return the numeric answer + the reasoning trail

This matches FireFoundry’s simple sequential workflow pattern (step-by-step runnable entities, appendOrRetrieveCall, and yield\* to stream progress). ￼ ￼

⸻

### Minimal entity model (non-technical)

- Problem (Entity): stores the original text, derived equation, API URL, and final result.
- EquationDeriver (Runnable Entity/Bot): turns text into (equation, variable, reasoning).
- NewtonInvoker (Runnable Entity): URL-encodes the equation, calls the GET endpoint, parses JSON.
- SolveWorkflow (Runnable Orchestrator): A→B→C coordinator that calls the two steps in order and assembles the response.

This “graph of doers + data” approach is exactly how FireFoundry likes you to separate stateful entities from active runnables/bots. ￼

⸻

### API surface

Expose an ID-less POST endpoint on the bundle (clean for a UI or cURL):

- POST /solve-word-problem
- Body: { "problem_text": "..." }
- Returns:

```json
{
  "success": true,
  "reasoning": "...why the equation is correct...",
  "equation": "2x + 3 = 11",
  "api_url": "/solve/2x%2B3=11",
  "answer": "x = 4"
}
```

Use FireFoundry’s `@ApiEndpoint` decorator for ID-less routes; it’s the recommended pattern for “create + run” style workflows. ￼ ￼ ￼

⸻

### Deterministic fetches & testing

When you run the workflow, bind to Mock Cache so the LLM parts (derivation) are reproducible in tests and CI; the Newton GET itself is deterministic but the derivation isn’t. Mock Cache lets you replay Broker calls and get stable runs. ￼ ￼

⸻

### Vibe-coding blueprint (by time box)

15 minutes — MVP (one file prompt+bot, one endpoint)

- Goal: E2E text → answer.
- Pieces:
- EquationDeriver as a StructuredDataBot with a tiny schema: { equation, variable, reasoning }.
- NewtonInvoker step uses fetch() to call the rendered GET URL; returns { answer, raw }.
- Simple linear SolveWorkflow: run derivation → run invoker → return merged DTO.
- Add a single POST /solve-word-problem via @ApiEndpoint.

This matches the quick “bundle exposes clean ID-less APIs” pattern. ￼

1 hour — “Reliable MVP”

- Add input validation and nicer errors.
- Stream progress messages (“Deriving equation…”, “Calling Newton…”) using the workflow’s yield\* pattern for great UX later. ￼
- Turn on Mock Cache for the derivation step to keep tests deterministic. ￼
- Add a basic /health and log the custom routes (the standalone server template shows both). ￼

1 day — Polished app + GUI

- Next.js front-end with a single page: textarea, “Solve” button, streaming progress, and a result card (Tailwind). The tutorial gives you a ready pattern for wiring a web UI to an Agent Bundle. ￼
- Use the FF SDK in the UI for type-safe calls to your endpoint (when deployed behind Kong), or direct HTTP locally. ￼ ￼
- Add a shared types package for the returned schema. ￼

⸻

Concrete schemas (keep it tiny)

- Derivation output (Zod):
  `{ equation: string, variable?: string, reasoning: string }`
- Newton output (parsed):
  `{ answer: string, raw: any }`
- Final response:
  `{ success: boolean, reasoning: string, equation: string, api_url: string, answer: string }`

FireFoundry encourages strongly-typed contracts end-to-end (Zod + shared types + FF SDK). ￼

⸻

Implementation sketch (non-technical, step order)

1. Bundle & endpoint
   - Create `MathAgentBundle` and add `@ApiEndpoint('POST','solve-word-problem').
2. SolveWorkflow (A→B→C)
   - A: `EquationDeriver` bot (StructuredData) extracts {equation, reasoning}
   - B: `NewtonInvoker` builds `/{operation}/{encodeURIComponent(expression)}` (e.g., solve/2x%2B3=11) and fetch()es it.
   - C: Merge outputs; return the structured result.
     Use `appendOrRetrieveCall` and `yield*` exactly like the linear example.
3. Determinism & testing
   - Bind runs to Mock Cache for the derivation LLM call so repeated runs produce the same equation/reasoning during tests.
4. Consumer
   - Local dev: call the endpoint directly (cURL/fetch) as shown in the tutorial. ￼
   - Staging/Prod (behind Kong): use FF SDK from your UI or server to call /solve-word-problem. ￼

⸻

Nice extras (optional)

- Show the Newton API URL you generated and the raw response for transparency.
- Add a “Try again” that keeps the same problem text but re-derives (turn Mock Cache off for that run to allow creative retries).
- Export to PDF of the worked solution; your front-end can POST HTML to a bundle endpoint that renders and returns a PDF (pattern is covered under binary downloads in FF SDK docs). ￼
