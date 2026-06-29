# AI SDK — Laravel 13 First-Party AI Integration

> **Laravel 13 introduces a first-party AI SDK** — unified API across OpenAI, Anthropic, Gemini, and more. Agents, tools, structured output, streaming, embeddings, image generation, audio transcription, **MCP servers**, **sub-agent delegation**, and **provider-hosted vector stores** — all in one place.

**Package:** `laravel/ai` (first-party Laravel package, installed separately)
**Install:** `composer require laravel/ai` (if not present)

---

## Setup

```php
// .env — configure at least one AI provider
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_API_KEY=...

// config/ai.php (published via artisan vendor:publish --tag=ai-config)
'providers' => [
    'openai' => ['api_key' => env('OPENAI_API_KEY')],
    'anthropic' => ['api_key' => env('ANTHROPIC_API_KEY')],
],
'default' => 'openai',
```

---

## Agents — The Core Building Block

Agents are PHP classes that encapsulate instructions, conversation context, tools, and output schema for interacting with an LLM.

```php
use Laravel\AI\Agents\Agent;
use Laravel\AI\Support\AgentAttributes;

class SupportAgent extends Agent
{
    // System prompt — what the agent is and does
    protected function instructions(): string
    {
        return 'You are a helpful customer support agent. '
            . 'You have access to tools to look up orders and knowledge base articles. '
            . 'Be concise, friendly, and accurate.';
    }

    // Available tools for this agent
    protected function tools(): array
    {
        return [
            LookUpOrderTool::class,
            SearchKnowledgeBaseTool::class,
        ];
    }
}
```

### Dispatching an Agent

```php
use Laravel\AI\Facades\AI;

// Simple chat (stateless)
$response = AI::prompt('openai', 'gpt-4o', [
    'messages' => [
        ['role' => 'user', 'content' => 'What is my order status?'],
    ],
]);

// Agent with tools (stateful, tool-use enabled)
$agent = new SupportAgent();

$reply = $agent->run('I want to check my order #12345');
// Agent may call LookUpOrderTool internally, then respond

// Streaming response
foreach ($agent->runStreamed('What is my order status?') as $chunk) {
    echo $chunk;
}
```

### Agent with Tools — The Full Pattern

```php
// app/AI/Tools/LookUpOrderTool.php
use Laravel\AI\Tools\Tool;
use Laravel\AI\Attributes\Tool as ToolAttribute;

#[ToolAttribute(
    name: 'look_up_order',
    description: 'Look up an order by its order number. Returns order status, items, and shipping info.',
)]
class LookUpOrderTool extends Tool
{
    public function handle(string $orderNumber): string
    {
        $order = Order::where('order_number', $orderNumber)->first();

        if (!$order) {
            return "Order #{$orderNumber} not found.";
        }

        return "Order #{$orderNumber}: Status={$order->status}, "
            . "Total={$order->total}, Items=" . $order->items->count();
    }
}
```

### Structured Output — Return Typed Data

```php
use Laravel\AI\Support\OutputSchema;
use App\Dtos\TicketClassification;

$result = AI::generate('openai', 'gpt-4o', [
    'prompt' => "Classify this support ticket: {$ticket->subject}\n\n{$ticket->body}",
    'output' => TicketClassification::class,  // typed output
]);

// $result is a TicketClassification DTO — not a raw string
echo $result->category;     // "billing"
echo $result->sentiment;    // "frustrated"
echo $result->priority;     // "high"
```

**DTO for structured output:**
```php
// app/Dtos/TicketClassification.php
use Laravel\AI\Support\Attributes\OutputAttribute;

#[OutputAttribute]
class TicketClassification
{
    public function __construct(
        #[OutputAttribute\Property(description: 'billing|technical|shipping|general')]
        public string $category,

        #[OutputAttribute\Property(description: 'positive|neutral|negative|frustrated')]
        public string $sentiment,

        #[OutputAttribute\Property(description: 'low|medium|high|critical')]
        public string $priority,
    ) {}
}
```

---

## Embeddings & Vector Search

_(Also covered in `performance.md` — summary here for completeness)_

```php
use Laravel\AI\Facades\Embeddings;

// Generate embedding
$vector = Embeddings::embed('Your search query here'); // float[]

// Store in pgvector-backed column
Article::create([
    'title' => $title,
    'content' => $content,
    'embedding' => $vector,
]);

// Semantic similarity search
$results = Article::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding)
    ->orderByRelevance('embedding', $queryEmbedding)
    ->limit(5)
    ->get();
```

---

## Image Generation

```php
use Laravel\AI\Facades\Images;

// Generate image
$imageUrl = Images::generate('openai', [
    'model' => 'dall-e-3',
    'prompt' => 'A clean Laravel app dashboard, dark mode, terminal in corner',
    'size' => '1024x1024',
    'quality' => 'standard',
]);

// Or: save to storage directly
$path = Images::generateAndSave('openai', [
    'prompt' => 'Hero image for tech blog post',
    'size' => '1792x1024',
], 'generated-images');
```

---

## Audio Transcription

```php
use Laravel\AI\Facades\Audio;

// Transcribe audio file
$transcript = Audio::transcribe('openai', [
    'file' => fopen(storage_path('app/voicemail.mp3'), 'r'),
    'model' => 'whisper-1',
    // Optional: prompt to guide transcription
    'prompt' => 'Technical product names: Laravel, Vue, Tailwind, Stripe',
]);

echo $transcript->text;
// $transcript->duration, $transcript->language also available
```

---

## Streaming Responses

```php
// Stream text response (Server-Sent Events)
$agent = new SupportAgent();
foreach ($agent->runStreamed('How do I reset my password?') as $chunk) {
    echo $chunk; // each chunk is a token
}

// Or via AI facade
foreach (AI::stream('openai', 'gpt-4o', [
    'messages' => [['role' => 'user', 'content' => 'Write a haiku about Laravel']],
]) as $chunk) {
    echo $chunk;
}
```

---

---

## Sub-Agents (June 3, 2026) — Multi-Agent Orchestration

Agents can **delegate to specialized sub-agents** by returning them from `tools()`. The parent agent decides when to call a sub-agent; the sub-agent runs in isolation (no parent conversation history) and returns its response back to the parent.

This turns the SDK into a real **orchestration layer** — a general-purpose agent can route to specialists (refunds agent, search agent, billing agent, etc.) without hand-wiring function calls in your own code.

### Basic sub-agent (no customization)

```php
namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Promptable;

class CustomerSupportAgent implements Agent, HasTools
{
    use Promptable;

    public function instructions(): string
    {
        return 'You help customers with account, order, and billing questions. '
             . 'Delegate refund policy questions to the refunds specialist.';
    }

    public function tools(): iterable
    {
        return [
            new RefundsAgent, // ← just return the agent instance
        ];
    }
}
```

If a sub-agent does **not** implement `CanActAsTool`, Laravel uses the agent's class basename as the tool name and a generic description that asks the parent agent to pass a clear, self-contained task. Each sub-agent invocation runs in isolation — it does **not** receive the parent's conversation history.

### Customizing the tool name/description with `CanActAsTool`

```php
namespace App\Ai\Agents;

use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\CanActAsTool;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)] // ← sub-agent can use a different provider/model
class RefundsAgent implements Agent, CanActAsTool, HasTools
{
    use Promptable;

    public function instructions(): string
    {
        return 'You are a refunds specialist. Apply our refund policy to '
             . 'the customer situation and return a clear verdict.';
    }

    // Tool-facing name and description shown to the parent agent
    public function actAsToolName(): string
    {
        return 'refunds_specialist';
    }

    public function actAsToolDescription(): string
    {
        return 'Use this tool to evaluate refund eligibility. '
             . 'Pass a complete self-contained description of the situation '
             . '(customer history, order, complaint). Returns a refund verdict.';
    }

    public function tools(): iterable
    {
        return [/* refunds-only tools */];
    }
}
```

**Key benefits:**
- Specialized instructions per sub-agent (no giant system prompt)
- Per-sub-agent model/provider (`#[Provider(Lab::Anthropic)]` etc.)
- Per-sub-agent toolset (sub-agent only sees what it needs)
- Isolated context (no token bloat from parent's full conversation)
- Parent agent stays the orchestrator — you don't hand-wire routing

---

## MCP Servers (May 14, 2026) — Connect Agents to MCP

Laravel AI agents can connect to **MCP (Model Context Protocol) servers** to access tools, resources, and prompts from external systems. Supports both **STDIO** and **HTTP** transports, with bearer and OAuth auth, and built-in caching.

```php
use Laravel\Ai\Providers\Tools\Mcp;

public function tools(): iterable
{
    return [
        // Local STDIO MCP server (e.g., filesystem, git, sqlite MCP)
        new Mcp(command: 'npx', args: ['-y', '@modelcontextprotocol/server-filesystem', '/data']),

        // Remote HTTP MCP server with bearer auth
        new Mcp(
            url: 'https://mcp.example.com',
            auth: ['type' => 'bearer', 'token' => env('MCP_TOKEN')],
        ),

        // HTTP MCP server with OAuth
        new Mcp(
            url: 'https://mcp.example.com',
            auth: ['type' => 'oauth'],
        ),
    ];
}
```

The MCP server's tools become available to the agent automatically — no need to wrap each tool manually. Built-in response caching keeps repeat calls cheap.

---

## Vector Stores & File Search (June 3, 2026)

First-class support for **provider-hosted vector stores** (OpenAI, Gemini) and **`FileSearch` provider tools** for RAG. The AI provider hosts the vector store; you upload files and add metadata, then expose `FileSearch` as a tool to your agents.

### Adding files to a vector store

```php
use Laravel\Ai\Documents\Document;

$store = AI::vectorStore('knowledge-base');

$store->add(
    Document::fromPath('/docs/refund-policy.pdf'),
    metadata: [
        'author' => 'Legal Team',
        'department' => 'Support',
        'year' => 2026,
    ],
);

// Add raw content
$store->add(
    Document::fromText('Our refund policy allows returns within 30 days...'),
    metadata: ['source' => 'internal-wiki'],
);

// Remove a file
$store->remove($documentId);
```

**Note:** Some providers may return a different "document ID" than the file's original ID. Store **both** IDs in your DB for future reference.

### `FileSearch` provider tool

```php
use Laravel\Ai\Providers\Tools\FileSearch;

public function tools(): iterable
{
    return [
        // Search across one or more stores
        new FileSearch(stores: ['store_id_1', 'store_id_2']),

        // Filter by metadata (simple equality)
        new FileSearch(
            stores: ['store_id'],
            where: ['author' => 'Taylor Otwell', 'year' => 2026],
        ),

        // Complex filters via closure
        new FileSearch(
            stores: ['store_id'],
            where: fn (FileSearchQuery $q) => $q->where('department', 'Engineering')
                                               ->where('year', '>=', 2025),
        ),
    ];
}
```

Supported providers: **OpenAI**, **Gemini**. The agent can now do RAG over your uploaded documents — the provider runs the vector search, you just expose the tool.

---

## Provider Failover

```php
// If OpenAI fails, automatically tries Anthropic
$result = AI::withFailover([
    'openai',
    'anthropic',
])->prompt('user', 'What is Laravel?');

// Or per-call
$result = AI::prompt('anthropic', 'claude-3-5-sonnet', [
    'messages' => [['role' => 'user', 'content' => 'Explain queues in Laravel']],
]);
```

---

## Testing AI Agents

```php
use Laravel\AI\Facades\AI;
use Tests\TestCase;

class SupportAgentTest extends TestCase
{
    public function test_agent_classifies_ticket_correctly(): void
    {
        // Mock the LLM response
        AI::fake([
            'choices' => [
                ['message' => ['content' => '{"category":"billing","sentiment":"frustrated","priority":"high"}']]
            ]
        ]);

        $result = AI::generate('openai', 'gpt-4o', [
            'prompt' => 'Billing issue: charged twice',
            'output' => TicketClassification::class,
        ]);

        $this->assertEquals('billing', $result->category);
        $this->assertEquals('high', $result->priority);
    }
}
```

---

---

## AI SDK vs Laravel MCP vs Laravel Boost — Don't Confuse Them

Laravel ships **three distinct AI-related products** that are easy to mix up. Use this matrix to pick the right one:

| Product | Package | Purpose | Direction |
|---|---|---|---|
| **Laravel AI SDK** | `laravel/ai` | Build agents that **call** AI providers (OpenAI, Anthropic, Gemini). Embeddings, structured output, RAG, streaming. | Your app → AI |
| **Laravel MCP** | `laravel/mcp` | Expose your Laravel app as an **MCP server** so external AI clients (Claude Desktop, Cursor) can call your tools/resources/prompts. | AI client → Your app |
| **Laravel Boost** | `laravel/boost` (dev) | Dev-time MCP server that gives **AI coding assistants** (Claude Code, Cursor, Copilot) deep context about your Laravel app — real routes, schema, config, logs, artisan commands. | AI assistant ↔ Your dev environment |

**When to use which:**

- **AI SDK** → you want your app to USE AI features (chat, embeddings, image gen, RAG, agents).
- **MCP** → you want AI clients to invoke your app's actions (e.g., a Cursor user runs an "Update user profile" tool that hits your Laravel endpoint).
- **Boost** → install in your dev repo so AI coding assistants stop hallucinating Laravel patterns; they get real routes/schema/config via MCP.

**They compose**: install Boost for dev, use the AI SDK in production code, AND expose parts of your app as MCP servers if external AI clients should drive it.

Source: [Laravel AI SDK, Boost, and MCP: Which Tool Do You Need?](https://laravel.com/blog/laravel-ai-sdk-boost-or-mcp-which-tool-do-you-need) | [Laravel MCP Docs](https://laravel.com/docs/13.x/mcp) | [Laravel Boost](https://laravel.com/docs/13.x/boost)

---

## Embedding Caching — Avoid Duplicate API Calls

Embedding generation is one of the most expensive AI operations (per-token pricing + rate limits). Laravel 13 ships first-class caching via the AI SDK config + `->cache(...)`:

```php
// config/ai.php
'caching' => [
    'embeddings' => [
        'cache' => true,            // enable global caching
        'driver' => null,            // null = use default cache store
        'ttl' => 86400,              // 1 day default
    ],
],

// Then anywhere you generate embeddings:
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache(seconds: 3600) // per-call override: cache for 1 hour
    ->generate();

// Stringable helper — also accepts cache argument
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: true);
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: 3600); // explicit TTL
```

**When the cache hits:**
- No outbound API call → no token cost, no rate-limit pressure
- Latency drops from ~300ms to ~1ms (cache lookup)

**When to override TTL:**
- Short TTL (60s) for ephemeral text like live chat messages (rarely re-queried)
- Long TTL (7d+) for static knowledge base content (refunds policy, product docs)
- Never cache for unique-per-user inputs (chat history) — the cache hit rate will be 0% and you're paying for the cache lookup

Source: [Laravel AI SDK Docs — Caching Embeddings](https://laravel.com/docs/13.x/ai-sdk)

---

## Queue Long-Running AI Calls — `->queue()` Instead of `->prompt()`

Audio transcription, large document analysis, image generation, and full-document summarization can take 5–60 seconds. Don't block the HTTP request — queue them:

```php
// ❌ Blocks the request for 30s on transcription
$transcript = AI::transcribe('openai', ['file' => $audio, 'model' => 'whisper-1']);

// ✅ Returns immediately; result is processed on a queue worker
$job = AI::transcribe('openai', ['file' => $audio, 'model' => 'whisper-1'])
    ->queue();

// Or with completion callbacks — fired after the queued job finishes
$job = AI::generate('openai', 'gpt-4o', ['prompt' => "Summarize this article..."])
    ->queue(
        onComplete: fn($result) => $article->update(['summary' => $result]),
        onFailure:  fn($e)      => Log::error('AI summary failed', ['error' => $e->getMessage()]),
    );
```

**How it works under the hood:**
- `->queue()` dispatches a queued job to your default queue connection
- The job re-invokes the AI SDK with the same parameters
- `onComplete` / `onFailure` callbacks run in the queue worker context — they have full DB access, can dispatch notifications, etc.

**Required setup:**
- A queue worker must be running: `php artisan queue:work` (or Horizon / Cloud managed queues)
- For long AI calls, bump the job timeout: `->tries(1)->timeout(300)` (5 min)

Source: [Laravel AI SDK, Boost, and MCP](https://laravel.com/blog/laravel-ai-sdk-boost-or-mcp-which-tool-do-you-need)

---

## Prompt Caching — Cut AI Costs 90% on Repeated System Prompts

If you have a large system prompt (legal context, brand voice guide, knowledge dump) processed thousands of times per day, every request pays full price for those input tokens. **Anthropic** and **OpenAI** both support prompt caching — and the Laravel AI SDK surfaces it via `providerOptions`:

```php
// Agent with Anthropic prompt caching enabled
use Laravel\Ai\Enums\Lab;

class LegalReviewAgent
{
    // ...
    public function providerOptions(): array
    {
        return [
            // Anthropic — mark the system prompt as cacheable for 5 minutes
            'cache_control' => ['type' => 'ephemeral', 'ttl' => '5m'],
        ];
    }
}

#[Provider(Lab::Anthropic)]
class CachedSummarizerAgent
{
    // OpenAI — automatic caching for prompts >1024 tokens (no config needed)
    // Just make sure your system prompt is consistent across calls
}
```

**How the savings work (Anthropic example):**
- First call: full price for the system prompt (~10K tokens = $0.03)
- Subsequent calls within 5 min: **90% discount** on cached tokens (~$0.003 instead of $0.03)
- At 10K calls/day with 5K-token cached prompt: **~$270/day saved** vs uncached

**When prompt caching pays off:**
- Agent with a fixed large system prompt + many requests per minute
- RAG pipelines that always prepend the same instructions before varying context
- Anything where the system prompt is the same across >100 requests/hour

**When it doesn't help:**
- Unique system prompts per call (different agents, different instructions)
- Very small prompts (<500 tokens — caching overhead may exceed savings)
- Very low traffic (<10 calls/hour — cache rarely hits before TTL expires)

**Note:** As of mid-2026, the Laravel AI SDK passes `cache_control` to Anthropic automatically when set in `providerOptions()`. For OpenAI, caching is automatic when the prompt exceeds 1024 tokens and the prefix matches a recent request — no explicit config needed.

Source: [Laravel AI issue #119 — Prompt caching](https://github.com/laravel/ai/issues/119) | [Anthropic Prompt Caching Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

---

## Common Mistakes

1. **No tools on agent** — agent is just an LLM call with no actions. Always add tools for real utility.
2. **Structured output without proper DTO** — returning unstructured text is fragile. Always use `#[OutputAttribute]` DTOs.
3. **Embedding drift** — if source text changes, regenerate embeddings. Stale embeddings = bad search results.
4. **Token limits** — agents accumulate context. Use `max_tokens` or truncation for long conversations.
5. **Streaming without proper handling** — streaming responses must be consumed immediately (SSE/websocket). Don't store chunks in memory.
6. **API keys in code** — always use env vars. Never hardcode `OPENAI_API_KEY`.
7. **Missing failover** — AI providers go down. Always configure at least one fallback provider.
8. **No sub-agent isolation** — putting too many tools on one agent bloats token count and confuses the model. Use sub-agents (`CanActAsTool`) for specialized workflows.
9. **Forgetting provider filter in `FileSearch`** — searching across all stores is slow and noisy. Always filter by metadata (`where: [...]`) when possible.
10. **Sub-agent without `CanActAsTool`** — works, but the parent gets a generic description. Always implement `CanActAsTool` for production agents so the LLM knows exactly when to delegate.
11. **Blocking AI calls in HTTP requests** — audio transcription, image gen, and large-doc summarization take 5–60s. Use `->queue()` instead of `->prompt()` so the HTTP request returns immediately and a queue worker handles the call.
12. **No embedding cache for repeated inputs** — knowledge base articles, FAQs, and product descriptions get re-embedded on every search query. Use `->cache(seconds: 3600)` or enable globally via `ai.caching.embeddings.cache = true` in config.
13. **Confusing AI SDK with Laravel MCP/Boost** — AI SDK is for your app to call AI providers; Laravel MCP is for exposing your app as an MCP server for external AI clients; Laravel Boost is a dev tool for AI coding assistants. Installing the wrong one wastes hours.
14. **Unique system prompts killing prompt-cache savings** — Anthropic/OpenAI cache only hits on a consistent prompt prefix. If every request has a different system prompt, you pay full price on every call.

---

## Updated from Research (2026-06-29)

### Cycle 9 additions (2026-06-29)

- **AI SDK vs Laravel MCP vs Laravel Boost** — three distinct products. AI SDK = your app calls AI providers. Laravel MCP = expose your app as an MCP server for external AI clients. Laravel Boost = dev-time MCP server giving AI coding assistants deep context about your Laravel app. They compose.
- **Embedding caching** — `Embeddings::for([...])->cache(seconds: 3600)->generate()` or globally via `ai.caching.embeddings.cache = true` in config. Also `Str::of($text)->toEmbeddings(cache: true)`. Avoid duplicate API calls + rate-limit pressure for repeated inputs (knowledge base, FAQs).
- **`->queue()` for long-running AI calls** — replaces blocking `->prompt()` for audio transcription, large doc analysis, image gen. Dispatched as a queue job with `onComplete` / `onFailure` callbacks. Requires a running queue worker (`php artisan queue:work`).
- **Prompt caching for cost** — Anthropic via `providerOptions(['cache_control' => ['type' => 'ephemeral', 'ttl' => '5m']])`. OpenAI automatic for prompts >1024 tokens. 90% discount on cached input tokens; pays off for agents with large fixed system prompts + high request volume.

- Laravel 13 ships first-party AI SDK covering: agents, tools, structured output, streaming, embeddings, image generation, audio transcription
- Agents are PHP classes with `instructions()` and `tools()` methods — not closures or strings
- Structured output uses `#[OutputAttribute]` PHP attributes on DTOs
- Provider failover via `AI::withFailover(['openai', 'anthropic'])`
- RAG (Retrieval Augmented Generation) combines vector search + LLM for knowledge-grounded answers
- **Sub-agents (June 3, 2026)** — agents can return other agents from `tools()` for delegation. Implement `CanActAsTool` to customize the tool-facing name/description shown to the parent. Sub-agents run in isolation (no parent history).
  - Use `#[Provider(Lab::Anthropic)]` (or another provider) to give a sub-agent a different model/provider from the parent.
- **Vector stores (June 3, 2026)** — first-class `AI::vectorStore('name')` for provider-hosted stores (OpenAI, Gemini). Upload `Document::fromPath()` / `Document::fromText()` with metadata.
- **`FileSearch` provider tool** — expose a vector store search as an agent tool: `new FileSearch(stores: ['id'], where: [...])`. Filter by metadata for precision.
- **MCP server support (May 14, 2026)** — `new Mcp(command: 'npx', args: [...])` for STDIO servers, `new Mcp(url: ..., auth: ...)` for HTTP servers (bearer or OAuth). Server tools become agent tools automatically.
- **Provider-specific tools** — `#[Provider(Lab::Anthropic)]`, `#[Provider(Lab::OpenAI)]`, etc., let you pin an agent or sub-agent to a specific provider's tool ecosystem (FileSearch, web search, code execution, etc.).

Sources: [Laravel 13 AI SDK Docs](https://laravel.com/docs/13.x/ai-sdk) | [Laravel AI Agents Now Support MCP Servers](https://laravel.com/blog/laravel-ai-agents-now-support-mcp-servers) | [Laravel News - Ship AI with Laravel](https://laravel-news.com/ship-ai-with-laravel-building-your-first-agent-with-laravel-13s-ai-sdk) | [Laravel News - Sub-agents](https://laravel-news.com/laravel-ai-sdk-sub-agents) | [RichDynamix - Complete AI SDK Guide](https://richdynamix.com/articles/laravel-ai-sdk-complete-guide) | [Dev.to - AI SDK in Production](https://dev.to/martintonev/how-to-use-laravel-ai-sdk-in-production-agents-tools-streaming-rag-4mfk)
