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
## Advanced JSON Schema — `anyOf()` and `JsonSchema::union()` (Laravel 13.17+)

The `Illuminate\JsonSchema` builder powers every agent `schema()` method. The two most common "this can't be expressed with primitive types" cases — **discriminated unions** (a field that's one of several distinct shapes) and **multi-type unions** (a field that accepts more than one JSON type) — added first-class support in Laravel 13.17. Both additions are exactly what AI assistants hallucinate: most reach for a single `enum` (loses per-shape fields) or a `string|null` (loses the union). The correct idioms are below.

### `JsonSchema::anyOf()` — Discriminated Unions (PR #60509)

Use `anyOf()` when a single field can resolve to one of several distinct shapes — e.g., a CMS that returns either an `article` (with `body`), a `video` (with `duration_seconds`), or a `podcast` (with `audio_url`). Modeling this as a single `enum` collapses the per-shape fields, and hand-rolling the JSON bypasses Laravel's built-in validation. `anyOf()` was always the right tool; Laravel 13.17+ promotes it to first-class on `Illuminate\JsonSchema` so the structured-output call doesn't have to drop the shape or hand-roll JSON.

Both OpenAI and Gemini support `anyOf` in their structured output APIs, so this unlocks more expressive schemas when building AI-powered features with Laravel AI.

```php
use Illuminate\JsonSchema\JsonSchema;

public function schema(JsonSchema $schema): array
{
    return [
        'content' => $schema->anyOf([
            // article variant
            $schema->object(fn ($s) => [
                'type'          => $s->string()->enum(['article'])->required(),
                'title'         => $s->string()->required(),
                'body'          => $s->string()->required(),
                'word_count'    => $s->integer(),
            ]),
            // video variant
            $schema->object(fn ($s) => [
                'type'             => $s->string()->enum(['video'])->required(),
                'title'            => $s->string()->required(),
                'duration_seconds' => $s->integer()->required(),
                'thumbnail_url'    => $s->string(),
            ]),
            // podcast variant
            $schema->object(fn ($s) => [
                'type'        => $s->string()->enum(['podcast'])->required(),
                'title'       => $s->string()->required(),
                'audio_url'   => $s->string()->required(),
                'episode_num' => $s->integer(),
            ]),
        ])->required(),
    ];
}
```

The LLM picks the matching shape and returns its fields; you branch on the `type` discriminator in your response handler:

```php
$result = $agent->run($input);

match ($result['content']['type']) {
    'article' => renderArticle($result['content']),
    'video'   => embedVideo($result['content']),
    'podcast' => queueAudioTranscription($result['content']),
};
```

OpenAI and Gemini both honor `anyOf` in structured-output mode. Anthropic has limited `anyOf` support — use `discriminator + oneOf` as a fallback there, or test with a small fixture.

### `JsonSchema::union([...])` — Multi-Type Unions (PR #60455)

Use `union()` when a field accepts more than one JSON type but stays in the same shape — e.g., a metadata value that can be a `string`, `number`, or `boolean`. This comes up constantly with third-party MCP tool schemas where mixed-type fields are common. Before 13.17, `JsonSchema::fromArray(['type' => ['string', 'number', 'boolean']])` **threw a deserialization exception** — now it round-trips cleanly. The same applies when you build the schema directly.

```php
use Illuminate\JsonSchema\JsonSchema;

// Round-trip a third-party MCP tool schema that uses multi-type unions
$schema = JsonSchema::fromArray([
    'type'        => ['string', 'number', 'boolean'],
    'description' => 'A flexible metadata value',
]);

// Or build one directly via the SDK
$value = JsonSchema::union(['string', 'number'])->nullable();
```

Both `fromArray()` (deserialize-then-reserialize) and direct construction now work on 13.17+. The `union()` method also chains with `->nullable()`, `->description(...)`, and the rest of the builder API.

### When to Use Which — Decision Matrix

| Need                                                                              | Use                                          | Example                                          |
|-----------------------------------------------------------------------------------|----------------------------------------------|--------------------------------------------------|
| Field is one of several **distinct shapes** (each with its own fields)            | `$schema->anyOf([...])`                       | Content block: `article` / `video` / `podcast`   |
| Field accepts more than one **JSON type** (same shape, type varies)               | `JsonSchema::union(['string', 'number'])`     | Metadata value: `string` OR `number` OR `boolean` |
| Field has a fixed set of options of the **same shape**                            | `$schema->string()->enum(['a', 'b', 'c'])`    | Status: `pending` / `running` / `done`            |
| Field is optional                                                                 | chain `->nullable()` (any of the above)       | Optional `description` field                     |

**Why this matters:** When the schema is wrong, the LLM silently degrades. With `anyOf`, the model picks the right variant and returns its full shape; with `union`, it can return any of the listed types without losing data. Modeling discriminated unions as a single `enum` collapses the per-variant fields and forces the response handler to re-fetch them. Modeling type unions as `string|null` makes the LLM stringify every number and boolean.

Source: [Laravel Changelog — Add `anyOf` Support to JSON Schema](https://laravel.com/docs/changelog#june-2026-add-anyof-support-to-json-schema) | [Laravel Changelog — Add Multi-Type Union Support to `Illuminate\JsonSchema`](https://laravel.com/docs/changelog#june-2026-add-multi-type-union-support-to-illuminatejsonschema) | PR [#60509](https://github.com/laravel/framework/pull/60509) + [#60455](https://github.com/laravel/framework/pull/60455)

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

## Conversation Memory — Multi-Turn Chat That Remembers

By default, every `Agent::prompt(...)` call is **stateless** — the LLM only sees the single user message. For any real chatbot / support agent / in-product assistant, you need the agent to remember previous turns. The Laravel AI SDK gives you two ways to add memory, both backed by database tables created by `php artisan migrate`.

### Option 1: `RemembersConversations` trait (zero-config)

The trait handles load + persist automatically. New user/assistant messages are written after each interaction; previous messages are loaded on the next prompt. Pair with `Conversational` to mark the agent as stateful:

```php
use Laravel\Ai\Agents\Agent;
use Laravel\Ai\Attributes\Conversational;
use Laravel\Ai\Concerns\RemembersConversations;

#[Conversational]  // marks this agent as having multi-turn context
class SupportAgent extends Agent
{
    use RemembersConversations;  // auto-loads + auto-saves history

    protected function instructions(): string
    {
        return 'You are a helpful customer support agent. Use prior turns '
             . 'to avoid asking the user to repeat themselves.';
    }
}

// First turn
$agent = new SupportAgent;
$reply = $agent->prompt('My order #1234 never arrived.');
// → assistant reply, both user + assistant messages persisted

// Second turn — same instance (or new one with same conversation key)
$reply = $agent->prompt('What should I do next?');
// → assistant can see the prior "My order #1234" turn and answer coherently
```

**Under the hood:** the trait reads from / writes to `agent_conversations` + `agent_conversation_messages` tables (created by the AI SDK's published migrations). Conversations are keyed by a stable identifier (e.g. user ID, session ID, ticket ID).

### Option 2: `messages()` method (full control)

If you need a custom history source (Redis, an external chat service, your own `messages` table with tenant scoping), implement `Conversational` and define `messages()` yourself. **When you provide `messages()` manually, the trait will NOT auto-load — you take full control:**

```php
use Laravel\Ai\Agents\Agent;
use Laravel\Ai\Attributes\Conversational;
use Laravel\Ai\Contracts\Conversational as ConversationalContract;
use Laravel\Ai\Message;

#[Conversational]
class TenantScopedAgent extends Agent implements ConversationalContract
{
    public function instructions(): string
    {
        return 'You are a tenant-aware assistant.';
    }

    /**
     * Load prior turns from YOUR table (with tenant scoping, filters, etc.).
     * Return an iterable of Message instances in chronological order.
     */
    public function messages(): iterable
    {
        return ChatMessage::query()
            ->where('tenant_id', $this->tenantId)
            ->where('user_id', $this->userId)
            ->orderBy('created_at')
            ->limit(50)
            ->get()
            ->map(fn ($m) => new Message($m->role, $m->content));
    }
}
```

**Critical warning from the docs:** *"When using the `RemembersConversations` trait, do not manually define a `messages` method in your agent class. If a `messages` method is present, it will take precedence over the trait's implementation and conversation history will not be loaded from the database."* — if you want the trait, leave `messages()` alone; if you define `messages()`, drop the trait.

### Production gotchas

- **Cap the history.** A 200-turn conversation burns serious tokens. Pass a `limit(50)` (or whatever your model can stomach) in your custom `messages()` query. Claude 3.5 Sonnet fits ~20-40 turns comfortably; GPT-4o handles more, but still rate-limit.
- **Don't leak across users.** Every agent instance should be scoped to a tenant + user key. The trait uses `agent_conversation_key` (configurable) to namespace — make sure you set it per-request via `$agent->forUser($user->id)` or your constructor.
- **Persist tool calls too.** If your agent uses tools, the conversation history should include the assistant's tool-call + tool-response pairs, not just the final text. The trait handles this for you; manual `messages()` implementations must include them.
- **Don't store secrets in history.** If a tool returns an API key, token, or PII in its payload, the trait will persist it. Strip sensitive fields before returning from tool `handle()` methods, or filter them in your custom `messages()` query.

Source: [Laravel AI SDK Docs — Conversation Context](https://laravel.com/docs/13.x/ai-sdk#conversation-context) | [Introducing the Laravel AI SDK — Conversation Memory](https://laravel.com/blog/introducing-the-laravel-ai-sdk)

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

The Laravel AI SDK ships **per-resource fakes** — fake agents, images, transcriptions, embeddings, rerankings, and files without ever touching a real API. Fakes make CI fast (sub-millisecond instead of multi-second HTTP calls) and deterministic (no flaky network or rate-limit flakiness). **Every AI test should use fakes** — never hit a real provider in CI.

### `Agent::fake()` — Per-Class Fake (Recommended)

The cleanest way to fake an agent is the per-class static `fake()` method. It accepts either a response array (consumed in order, FIFO) or a closure that receives the prompt and returns a response:

```php
use App\Ai\Agents\TicketClassifier;
use Tests\TestCase;

class SupportAgentTest extends TestCase
{
    public function test_agent_classifies_ticket_correctly(): void
    {
        // Queue of canned responses (consumed one per prompt, FIFO)
        TicketClassifier::fake([
            ['category' => 'billing', 'sentiment' => 'frustrated', 'priority' => 'high'],
            ['category' => 'shipping', 'sentiment' => 'neutral',   'priority' => 'low'],
        ]);

        $first  = (new TicketClassifier)->prompt('Charged twice on my card');
        $second = (new TicketClassifier)->prompt('Where is my package?');

        $this->assertSame('billing', $first->category);
        $this->assertSame('shipping', $second->category);
    }
}
```

### Dynamic responses with a closure

```php
TicketClassifier::fake(function ($prompt) {
    // Return a different response based on the prompt content
    return str_contains($prompt->prompt, 'refund')
        ? ['category' => 'refund', 'priority' => 'medium']
        : ['category' => 'other',  'priority' => 'low'];
});
```

### `preventStrayPrompts()` — Catch Unmocked Agent Calls

By default, an unfaked agent call falls back to a real provider. **`preventStrayPrompts()`** flips that to "throw an exception" — essential for CI to ensure every agent interaction is mocked:

```php
public function test_no_unmocked_agent_runs(): void
{
    TicketClassifier::fake()->preventStrayPrompts();

    (new TicketClassifier)->prompt('Valid input');      // ✅ uses fake
    (new TicketClassifier)->prompt('Unmocked input');   // 💥 throws — caught in test
}
```

Pair with `expectException(...)` for clean failure reporting, or let it fail the test naturally.

### Assertions — Did the Agent Actually Run?

After faking, assert that the agent was (or wasn't) called and inspect what it was sent:

```php
public function test_handler_invokes_classifier(): void
{
    TicketClassifier::fake([['category' => 'billing', 'priority' => 'high']]);

    (new \App\Actions\ProcessTicket)->handle($ticket);

    TicketClassifier::assertPrompted('charged twice');           // contains substring
    TicketClassifier::assertPrompted(fn ($prompt) =>            // or a closure
        str_contains($prompt->prompt, 'charged twice')
    );
    TicketClassifier::assertNeverPrompted('refund');              // negative assertion
}
```

### Structured Output Fakes — Auto-Generate Matching Schema

If your agent implements `HasStructuredOutput` and defines a `schema()` method, `Agent::fake()` automatically generates fake data that matches the schema — no need to hand-write the JSON:

```php
// Given TicketClassifier implements HasStructuredOutput with schema() returning
// {category: string, priority: 'low|medium|high', sentiment: 'positive|neutral|negative|frustrated'}

TicketClassifier::fake();  // no arguments — auto-generates schema-matching fake
$result = (new TicketClassifier)->prompt('anything');
// $result is a real TicketClassification DTO with valid faked values
```

Pass an explicit array when you want to control the values (assertions about specific fields).

### Per-Resource Fakes — Image, Audio, Embeddings, Reranking, Files

Every AI resource has its own fake class. Use them when testing code paths that hit image gen / transcription / embeddings / reranking / file uploads without touching a real provider:

```php
use Laravel\Ai\Images;
use Laravel\Ai\Audio\Transcription;
use Laravel\Ai\Embeddings;
use Laravel\Ai\Reranking;
use Laravel\Ai\Files;
use Laravel\Ai\RankedDocument;

// Images — return base64 (or use the closure form to inspect the prompt)
Image::fake(function (ImagePrompt $prompt) {
    return base64_encode('fake-png-bytes');
});

// Transcriptions — provide text responses
Transcription::fake(['Hello world transcript']);
Transcription::fake()->preventStrayTranscriptions();   // require all to be faked

// Embeddings — auto-generate vectors of the right dimensions, or supply explicit ones
Embeddings::fake();                                    // auto-dimension fakes
Embeddings::fake([                                     // explicit response vectors
    [0.1, 0.2, 0.3, /* ... 1536 dims for ada-002 */],
]);

// Reranking — return RankedDocument instances with scores
Reranking::fake([
    [
        new RankedDocument(index: 0, document: 'Most relevant doc', score: 0.95),
        new RankedDocument(index: 1, document: 'Second',          score: 0.80),
    ],
]);

// Files (vector store uploads / deletes)
Files::fake();   // intercepts add() / remove() without hitting the provider
// then assert: Files::assertUploaded(...), Files::assertDeleted(...)
```

### Facade-Level Fake (Legacy Pattern)

The `AI::fake()` facade-level approach still works — it's just lower-level. **Prefer per-class fakes for new code:**

```php
use Laravel\AI\Facades\AI;

AI::fake([
    'choices' => [
        ['message' => ['content' => '{"category":"billing","priority":"high"}']],
    ],
]);

$result = AI::generate('openai', 'gpt-4o', [
    'prompt' => 'Billing issue',
    'output' => TicketClassification::class,
]);
```

**Why per-class is better:** clearer intent (you read `TicketClassifier::fake()` and know exactly which agent is being faked), better assertions (`TicketClassifier::assertPrompted()` vs digging through a generic prompt list), and safer scoping (each agent class can have its own fake independently — no cross-test pollution).

Source: [Laravel AI SDK Docs — Testing](https://laravel.com/docs/13.x/ai-sdk#testing) | [Lexo.ch — Getting Started with the Laravel AI SDK](https://www.lexo.ch/blog/2026/03/getting-started-with-the-laravel-ai-sdk-agents-tools-structured-output)
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
15. **No conversation memory on chat agents** — every `prompt()` call is stateless by default. For any real chatbot / support agent / in-product assistant, add `RemembersConversations` (auto) or implement `messages()` (manual). Without it, your agent "forgets" between turns and users get frustrated.
16. **Defining `messages()` AND using `RemembersConversations`** — they conflict. If `messages()` exists, the trait is bypassed silently. Pick one: trait = automatic DB-backed history, `messages()` = you own the storage layer (Redis, custom table with tenant scoping, etc.).
17. **Real API calls in CI tests** — every `Agent::fake()` without `preventStrayPrompts()` will silently hit a real provider if you add a new agent call. Always use `Agent::fake()->preventStrayPrompts()` to catch unmocked agent interactions before they reach production CI.
18. **No per-resource fakes for image/audio/embedding tests** — `AI::fake()` only fakes text completions. For image gen / transcription / embeddings / reranking / files, use the resource-specific fakes: `Image::fake()`, `Transcription::fake()`, `Embeddings::fake()`, `Reranking::fake()`, `Files::fake()` — each has its own assertions.
19. **Modeling discriminated unions (article vs video vs podcast) as a single `enum`** — you lose type-specific fields. Use `$schema->anyOf([...])` (Laravel 13.17+, PR #60509) so each variant keeps its own shape, and branch on the discriminator in your response handler.
20. **Assuming `JsonSchema::fromArray()` rejects multi-type unions** — it does NOT, as of Laravel 13.17 (PR #60455). `['type' => ['string', 'number', 'boolean']]` now round-trips correctly; previously it threw a deserialization exception. The fix only backported to 13.x — `JsonSchema::fromArray()` on 12.x with multi-type unions still throws.
21. **`anyOf()` doesn't include `null` by default** — chain `->nullable()` on the `$schema->anyOf([...])` result if the field can be missing. `anyOf()` describes one of N shapes; making the field optional/nullable is a separate decision (`->required()` vs `->nullable()`).
22. **Confusing `anyOf()` (shape union) with `JsonSchema::union(['string', 'number'])` (type union)** — they look similar but answer different questions. `anyOf` = which schema matches? `union` = which JSON types are valid? Mixing them up produces either over-permissive schemas (extra `null` accepted) or under-permissive schemas (multiple types expressed as one shape).

---

## Updated from Research (2026-07-11, cycle 35)

### Cycle 35 additions (2026-07-11)

- **`Illuminate\JsonSchema::anyOf()` — Discriminated Unions (Laravel 13.17+, PR #60509)** — the only correct way to express "one field, several distinct shapes" (article / video / podcast, success / error envelope, polymorphic records). AI assistants overwhelmingly hallucinate this as a single `$schema->string()->enum([...])` — that collapses the per-shape fields. `anyOf()` accepts an array of `object()` schemas, each with its own `type` discriminator. Both OpenAI and Gemini honor `anyOf` in structured-output mode; Anthropic has limited support (fallback to `discriminator + oneOf`).
- **`JsonSchema::union(['string', 'number'])` — Multi-Type Unions (Laravel 13.17+, PR #60455)** — fields that accept more than one JSON type (`'type' => ['string', 'number', 'boolean']`). Constantly used by third-party MCP tool schemas. Before 13.17, `JsonSchema::fromArray(['type' => [...]])` **threw a deserialization exception**. Now round-trips cleanly on 13.17+. 12.x still throws.
- **Decision matrix** — 4-row table mapping common structural needs (discriminated shape vs multi-type union vs fixed enum vs optional field) to the correct schema-builder call.
- **Worked discriminator-branch example** — `match ($result['content']['type']) { 'article' => ..., 'video' => ..., 'podcast' => ... }` so the response-handler pattern is concrete, not abstract.
- **Anthropic compatibility note** — flagged up front because `anyOf` is OpenAI/Gemini-only by default. Documented the `discriminator + oneOf` fallback for Anthropic-only flows.
- **Common Mistakes list grew 18 → 22** — added "Modeling discriminated unions as a single `enum`", "Assuming `JsonSchema::fromArray()` rejects multi-type unions" (12.x still throws; 13.17+ only), "`anyOf()` doesn't include `null` by default", and "Confusing `anyOf()` with `union()`" (shape union vs type union).

## Updated from Research (2026-07-05)

### Cycle 26 additions (2026-07-05)

- **Conversation memory** — `Conversational` attribute + `RemembersConversations` trait for zero-config auto-persisted multi-turn chat (backed by `agent_conversations` + `agent_conversation_messages` tables from the AI SDK migration). For full control, implement the `Conversational` interface and define `messages()` yourself — but then the trait is bypassed. Critical warning: defining `messages()` AND using the trait = the trait silently loses (per official docs).
- **Per-class testing fakes (`Agent::fake()`)** — preferred over the `AI::fake([...])` facade-level pattern. Accepts an array of canned responses (FIFO) or a closure receiving `$prompt`. Pairs with `preventStrayPrompts()` to catch unmocked calls in CI, and `assertPrompted()` / `assertNeverPrompted()` for assertions (string contains OR closure). Structured-output fakes auto-generate schema-matching data when the agent implements `HasStructuredOutput`.
- **Per-resource fakes** — `Image::fake(closure)`, `Transcription::fake([...])->preventStrayTranscriptions()`, `Embeddings::fake()` (auto-dimension or explicit vectors), `Reranking::fake([RankedDocument, ...])`, `Files::fake()`. Each has its own assertions. Facade-level `AI::fake()` still works but is the lower-level legacy path.

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
