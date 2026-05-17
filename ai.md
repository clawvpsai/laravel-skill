# AI SDK — Laravel 13 First-Party AI Integration

> **Laravel 13 introduces a first-party AI SDK** — unified API across OpenAI, Anthropic, Gemini, and more. Agents, tools, structured output, streaming, embeddings, image generation, and audio transcription — all in one place.

**Package:** `laravel/ai` (bundled in Laravel 13 core)
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

## Common Mistakes

1. **No tools on agent** — agent is just an LLM call with no actions. Always add tools for real utility.
2. **Structured output without proper DTO** — returning unstructured text is fragile. Always use `#[OutputAttribute]` DTOs.
3. **Embedding drift** — if source text changes, regenerate embeddings. Stale embeddings = bad search results.
4. **Token limits** — agents accumulate context. Use `max_tokens` or truncation for long conversations.
5. **Streaming without proper handling** — streaming responses must be consumed immediately (SSE/websocket). Don't store chunks in memory.
6. **API keys in code** — always use env vars. Never hardcode `OPENAI_API_KEY`.
7. **Missing failover** — AI providers go down. Always configure at least one fallback provider.

---

## Updated from Research (2026-05)

- Laravel 13 ships first-party AI SDK covering: agents, tools, structured output, streaming, embeddings, image generation, audio transcription
- Agents are PHP classes with `instructions()` and `tools()` methods — not closures or strings
- Structured output uses `#[OutputAttribute]` PHP attributes on DTOs
- Provider failover via `AI::withFailover(['openai', 'anthropic'])`
- RAG (Retrieval Augmented Generation) combines vector search + LLM for knowledge-grounded answers

Sources: [Laravel 13 AI SDK Docs](https://laravel.com/docs/13.x/ai-sdk) | [Laravel News - Ship AI with Laravel](https://laravel-news.com/ship-ai-with-laravel-building-your-first-agent-with-laravel-13s-ai-sdk) | [RichDynamix - Complete AI SDK Guide](https://richdynamix.com/articles/laravel-ai-sdk-complete-guide) | [Dev.to - AI SDK in Production](https://dev.to/martintonev/how-to-use-laravel-ai-sdk-in-production-agents-tools-streaming-rag-4mfk)
