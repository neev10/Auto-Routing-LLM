# System Architecture

Related: [DATABASE.md](DATABASE.md) · [ROUTING.md](ROUTING.md) · [MEMORY.md](MEMORY.md) · [SECURITY.md](SECURITY.md)

## 1. Overview

```
 Browser (React SPA, hosted by Lovable)
   │  supabase-js (anon/publishable key + user JWT)
   │    ├── direct table reads/writes ── protected by RLS ──┐
   │    └── fetch() to Edge Functions (JWT in header)        │
   ▼                                                         ▼
 Lovable Cloud (Supabase)                          Postgres + pgvector
   ├── Auth (email/password, sign-ups off)         (projects, messages,
   ├── Storage (private bucket: project-files)      memory, usage, …)
   └── Edge Functions (Deno)  ── OPENROUTER_API_KEY (secret)
         chat · models-sync · memory-extract · summarize
         embed · test-connection · export · reconcile-costs
                    │
                    ▼
              OpenRouter API  ──►  hundreds of models / providers
```

**Hard rules**

1. **The browser never calls OpenRouter.** Every LLM call goes through an edge function that reads `OPENROUTER_API_KEY` from secrets.
2. **The browser never writes cost, usage or routing data.** Only edge functions write to those tables, using the service role. RLS allows the user to read them but not insert them.
3. **Core logic is pure TypeScript.** The router, context builder, token estimator and cost estimator live in `supabase/functions/_shared/core/`. They have no Deno or npm-URL imports, so Vitest can test them from the frontend toolchain and Deno can import them into functions.
4. **Everything goes through a provider adapter.** No function calls `fetch("https://openrouter.ai/...")` directly except `providers/openrouter.ts`.

## 2. Components

### 2.1 Frontend (React)

| Area | Responsibility |
|---|---|
| `AppShell` | Sidebar (projects, recent chats, settings), top bar, budget meter |
| `ChatView` | Message list, streaming renderer (Markdown, code highlighting, copy), composer, attachments |
| `ModelPicker` | Auto/Manual toggle, searchable catalog with badges (FREE, 👁 vision, 🛠 tools, context, $/M) |
| `RoutingCard` | Per-message summary line and a details popover (reason, candidates, fallbacks, cost, context tokens) |
| `ProjectWorkspace` | Tabs: Chats, Brief, Memory, Files, Instructions, Models, Tools, Activity |
| `Settings` | OpenRouter status, tiers, limits, About Me, catalog, export, logs |
| `UsageDashboard` | Phase 4 charts |
| `lib/chatStream.ts` | POSTs to `chat`, parses SSE events, handles AbortController |
| `lib/pdfText.ts` | Extracts PDF text in the browser with `pdfjs-dist` |

State: TanStack Query for server data (Lovable's default). Local component state for streaming.

### 2.2 Edge Functions

| Function | Trigger | Purpose | Phase |
|---|---|---|---|
| `chat` | POST from the client | Auth check → load settings → route → build context → cost guard → stream from OpenRouter → persist message, routing decision and usage → background jobs | 1 (routing 2, memory 3) |
| `models-sync` | Client, when the catalog is stale (>24h) or Refresh is clicked | Fetch `/api/v1/models`, upsert `models`, mark missing models inactive | 1 |
| `test-connection` | Settings button | Call OpenRouter `/api/v1/key` and return the limit, usage and free-tier status (never the key) | 1 |
| `export` | Settings / project menu | Build JSON + Markdown for one project or for everything, and return a zip | 1 |
| `title` | Background, from `chat` | Generate a conversation title with a free model | 1 |
| `memory-extract` | Background from `chat` (every 16 messages) or client (on leaving a conversation) | Extract typed memory items, deduplicate, embed, and propose a Brief update | 3 |
| `summarize` | Background from `chat` | Fold old messages into the rolling summary | 3 |
| `embed` | Called by other functions, and by the client for file chunks | Generate `gte-small` embeddings (384-dim) | 3 |
| `reconcile-costs` | Opportunistically from `chat`, at most once an hour | Fill in missing costs (for stopped streams) through OpenRouter `/api/v1/generation?id=` | 1 |
| `memory-pin` | Client (📌 on a message) | Condense a message into one pinned memory item | 3 |
| `brief-suggest` | Background from `memory-extract`, or the Brief tab | Draft a revised Brief into `brief_suggestion` | 3 |
| `file-ingest` | Client, after upload / re-index | Chunk and embed file text | 3 |
| `agent` | Client ("Run as task") | Bounded multi-step runs with approvals | 5 |
| `_shared/tools/*` | Imported by `chat` / `agent` | Web search, fetch URL, memory tools, GitHub read, MCP | 5 |

Background work uses `EdgeRuntime.waitUntil(promise)`, so the response isn't delayed. If that proves unreliable, the client calls the background function after receiving `done`.

### 2.3 Shared modules (`supabase/functions/_shared/`)

```
_shared/
  core/                       # PURE TS — unit tested with Vitest
    types.ts                  # Model, Tier, RoutingInput, RoutingDecision, ContextBlock…
    tokens.ts                 # estimateTokens(text) = ceil(chars / 4)
    complexity.ts             # rule scoring → {score, band, confident, taskType, signals}
    router.ts                 # selectModel(input, catalog, tiers, settings) → decision
    cost.ts                   # estimateCost(model, promptTokens, maxOutput); capCheck()
    context.ts                # buildContext(blocks, budget) → messages[] + report
    memoryMerge.ts            # dedupe / supersede logic for extracted items
  providers/
    adapter.ts                # interface ProviderAdapter { listModels, chatStream, chat, keyInfo }
    openrouter.ts             # the only implementation in the MVP
  db.ts                       # service-role client, typed queries
  auth.ts                     # verify JWT + ALLOWED_USER_EMAIL
  sse.ts                      # SSE encode/parse helpers
  prompts.ts                  # system prompts for classifier, extraction, summary, title, brief
```

### 2.4 Provider adapter

```ts
interface ProviderAdapter {
  id: 'openrouter' | 'ollama' | 'anthropic' | 'openai' | 'google';
  listModels(): Promise<CatalogModel[]>;
  chat(req: ChatRequest): Promise<ChatResult>;                 // non-streaming (classifier, extraction…)
  chatStream(req: ChatRequest, signal: AbortSignal): AsyncIterable<StreamEvent>;
  keyInfo?(): Promise<KeyInfo>;
}

interface ChatRequest {
  model: string;
  messages: ChatMessage[];
  maxTokens?: number;
  temperature?: number;
  responseFormat?: 'json' | 'text';
  sensitive: boolean;            // → OpenRouter provider.data_collection = "deny"
  tools?: ToolSpec[];            // Phase 5
}
```

The OpenRouter adapter adds:
- `provider: { data_collection: "deny" }` when `sensitive` is true.
- `HTTP-Referer` and `X-Title` headers (OpenRouter's app attribution headers).
- It parses `usage` from the final SSE chunk (tokens and `cost`) and captures the response `id` as `generation_id`.

## 3. The `chat` request lifecycle

```
Client                          chat (edge fn)                                OpenRouter
  │ POST {conversationId, content, attachmentFileIds?,
  │       modelOverride?, confirm?, forceTier?, retryStronger?,
  │       regenerateOf?, editOf?, dryRun?, approveToolCall?}
  │────────────────────────────►│
  │                             │ 1. auth: JWT valid & email == ALLOWED_USER_EMAIL
  │                             │ 2. load project, conversation, settings, month spend
  │                             │ 3. persist user message
  │                             │ 4. ROUTE (ROUTING.md): manual/sticky/default or Auto
  │                             │      rules → (classifier call if unsure) → tier → model
  │                             │ 5. BUILD CONTEXT (MEMORY.md) within budget for model
  │                             │ 6. COST GUARD: cap check + estimate > $0.25 && !confirm
  │◄──── event: needs_confirmation {estimate, model}  (stream ends; client asks user)
  │                             │ 7. insert routing_decisions row
  │◄──── event: routing {decision summary}
  │                             │ 8. stream ─────────────────────────────────────►│
  │◄──── event: delta {text}  ◄─┤◄──────────────────────────── SSE chunks ────────│
  │                             │    on error before first token → fallback model
  │◄──── event: fallback {from, to, reason}
  │                             │ 9. final chunk: usage {tokens, cost}, id
  │                             │10. persist assistant message + llm_calls (+ prompt_logs)
  │◄──── event: done {messageId, usage, cost}
  │                             │11. waitUntil: title / summarize / extract / reconcile
```

**SSE event types** (client ⇄ `chat`):

| Event | Payload |
|---|---|
| `routing` | `{decisionId, modelId, modelName, tier, complexity, method, reason, isFree, escalated, contextTokens, estimatedCost}` |
| `delta` | `{text}` |
| `fallback` | `{fromModel, toModel, reason}` |
| `needs_confirmation` | `{kind: 'cost' \| 'tier3' \| 'cap_reached', estimatedCost, modelId, message}` |
| `done` | `{userMessageId, assistantMessageId, usage: {promptTokens, completionTokens, costUsd}}` |
| `error` | `{code, message, retryable}` |
| `tool` *(Phase 5)* | `{name, status: 'running' \| 'done' \| 'error'}` |
| `needs_approval` *(Phase 5)* | `{toolCallId, tool, args}` |

**Stop:** the client aborts the fetch. The function sees `req.signal` fire, aborts the upstream request, and saves the partial text with `status='stopped'`. If no usage chunk arrived, `cost_usd` stays null, and `reconcile-costs` fills it in later from the generation id.

**Mid-stream failure** (after tokens have already been sent): no automatic fallback, because that would mix outputs. The function saves the partial text with `status='error'`, and the client shows **Retry** and **Retry stronger** buttons.

## 4. Limits to keep in mind

- **Edge Function wall-clock and CPU limits.** Very long generations from slow models could hit the wall-clock limit. Mitigation: set `max_tokens` (default 4,096, configurable), and treat a timeout like a stop. Check the current limits in the Lovable/Supabase docs.
- **OpenRouter free limits:** 20 requests/minute, and 50/day or 1,000/day (see DECISIONS.md). Classifier, title, extraction and summary calls count against the same daily limit. Budget: with the 1,000/day limit, those background calls use at most about 25% of it.
- **Free models change often.** The catalog sync marks models that have disappeared as `is_active=false`. The router skips inactive models and ones in cooldown.

## 5. Repository structure

```
/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docs/                         # this specification
│   ├── fixtures/routing-test-set.json   # 30 prompts with expected outcomes (TESTING.md)
│   └── lovable-prompts/
├── .github/workflows/test.yml    # npm ci && npm test (Vitest)
├── src/
│   ├── components/
│   │   ├── chat/                 # ChatView, MessageItem, Composer, RoutingCard, ConfirmCostDialog
│   │   ├── models/               # ModelPicker, ModelBadge
│   │   ├── projects/             # ProjectSidebar, ProjectWorkspace, tabs/*
│   │   ├── memory/               # MemoryList, MemoryItemEditor, BriefEditor, ContextInspector
│   │   ├── settings/             # tabs/*
│   │   ├── usage/                # BudgetMeter, UsageDashboard (Phase 4)
│   │   └── ui/                   # shadcn
│   ├── hooks/                    # useProjects, useConversation, useModels, useSettings…
│   ├── lib/                      # supabase client, chatStream, pdfText, format
│   ├── pages/                    # Login, App, Settings, Usage
│   └── test/                     # Vitest tests importing supabase/functions/_shared/core
├── supabase/
│   ├── migrations/               # SQL (created by Lovable)
│   └── functions/
│       ├── _shared/              # see §2.3
│       ├── chat/index.ts
│       ├── models-sync/index.ts
│       ├── test-connection/index.ts
│       ├── export/index.ts
│       ├── title/index.ts
│       ├── memory-extract/index.ts
│       ├── summarize/index.ts
│       ├── embed/index.ts
│       ├── memory-pin/index.ts
│       ├── brief-suggest/index.ts
│       ├── file-ingest/index.ts
│       ├── reconcile-costs/index.ts
│       └── agent/index.ts            # Phase 5
```

## 6. Extensibility hooks (built in now, used later)

| Future capability | Hook that exists from Phase 1 |
|---|---|
| Other providers | `ProviderAdapter` + `models.provider_adapter` column |
| Local Ollama | Adapter interface. A future `BrowserProvider` runs in the client for `local/*` models. |
| Tools / MCP | `ChatRequest.tools`, `messages.role = 'tool'`, and `tool_calls jsonb` columns |
| Agents | `agent_runs` / `agent_steps` tables (Phase 5), with `llm_calls.agent_run_id` reserved |
| Learned routing | `feedback` + `routing_decisions` + `llm_calls` joined by ids |
| Multi-user | `user_id` everywhere + RLS. Turn sign-up back on and remove the `ALLOWED_USER_EMAIL` check. |
