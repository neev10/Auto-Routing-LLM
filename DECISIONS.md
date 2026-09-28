# Decision Log (Phase 0)

These decisions were made during Phase 0 planning. Any change to one of them should be recorded here, with its date and the reason, before the code changes.

Format: **Decision**, then *why*.

## 1. Foundation

| # | Decision | Why |
|---|---|---|
| D1 | **Single user.** Every table still has `user_id`, and RLS is on from day one. | Adding real multi-user later becomes a configuration change rather than a rewrite, which matters because the repo is public. |
| D2 | **Lovable-native stack:** React/Vite/TS/Tailwind/shadcn plus Lovable Cloud (Supabase). | Lovable produces its most reliable output on its own defaults. Supabase provides the database, auth, storage, functions and vectors in one place. |
| D3 | **Lovable hosts and deploys, and GitHub is the source of truth for code.** Two-way sync is on from day one. | Nothing proprietary is used, so the app could be self-hosted later. |
| D4 | **Keep costs as close to free as possible.** Target LLM spend is ≤ $10/month (the current Claude subscription is $20/month). Infrastructure runs on the free tiers. | Saving money is a primary goal. |
| D5 | **Desktop only.** No mobile layout work. | Keeps the scope small. |
| D6 | **A personal tool with public code**, under the MIT license. Clean setup docs, but no one-click deploy polish yet. | It's open source, but it's for personal use first. |

## 2. Keys, security, privacy

| # | Decision | Why |
|---|---|---|
| D7 | **The OpenRouter key is stored as a backend secret** (`OPENROUTER_API_KEY`) and never reaches the browser or the database. | This is the most secure and simplest option. People who fork the repo set their own secret. |
| D8 | **OpenRouter is the only provider in the MVP**, behind a `ProviderAdapter` interface. | Other providers, including local Ollama, can be added later without a rewrite. |
| D9 | **Email/password login.** Sign-ups are disabled after the owner account exists. Edge functions also check `ALLOWED_USER_EMAIL`. | The app has a public URL. Without a login, anyone who found it could spend the credits. |
| D10 | **Each project has a Sensitive toggle.** Sensitive projects route only to providers that don't store or train on data (`provider.data_collection = "deny"`). | Protects career, investment and personal material. |
| D11 | **Spend guards:** a hard credit limit on the OpenRouter key ($10), an app-level monthly cap ($10: warn at 80%, block paid models at 100% while free models keep working), and a per-request confirmation when the estimated cost is above $0.25. | Several layers of protection, with a backstop that lives outside the app. |

## 3. Routing

| # | Decision | Why |
|---|---|---|
| D12 | **Complexity detection is a hybrid.** Deterministic rules decide first, and a free classifier model runs only when the rules are unsure. Every decision records its method. | Fast and free most of the time, and it can be tuned by looking at the logs. |
| D13 | **Each tier has a curated model list** (2–4 ranked models, editable in Settings), filled from the synced OpenRouter catalog. The app offers a "Suggest defaults" helper. | Predictable and easy to debug. |
| D14 | **Cost-first priority by default**, with a per-project override (Cost / Balanced / Performance). | Cost is the main goal. |
| D15 | **Fallback and escalation:** falling back to another model in the same tier is automatic. Escalating from Tier 1 to Tier 2 is automatic and shown in the routing card. Tier 3 always asks first. Every response has a "Retry with a stronger model" button. | The $0.25 confirmation already catches expensive requests. |
| D16 | **A manual model choice sticks for the rest of the conversation.** Projects can set a default model. | Matches how people actually work. |
| D17 | **Thumbs up/down on every response**, with an optional note, linked to the routing decision. | Provides the data for Phase 6 performance-based routing. |

## 4. Memory

| # | Decision | Why |
|---|---|---|
| D18 | **Memories are created both automatically and manually**, and all of them are visible and editable. Extraction for Sensitive projects also uses no-training routing. | The user keeps control, and the privacy protection is consistent. |
| D19 | **Four layers:** Project Brief, memory items, rolling conversation summaries, and searchable chunks. | The Brief lets a new model get up to speed quickly. The other layers are retrieved only when relevant. |
| D20 | **Context budget of about 4k tokens** (range 3–5k), adapted to the model's context window. | This is where most of the token savings come from. |
| D21 | **Summarize once unsummarized history passes about 6k tokens.** Originals are kept and remain searchable. | Model switching then needs no special handoff logic. |
| D22 | **Semantic search is in Phase 3**, using pgvector and `gte-small` inside Edge Functions at no cost. | Memory is the point of the app. Verified: Supabase supports `gte-small` natively in Edge Functions (384 dimensions). |
| D23 | **A short global "About Me"** is included in every project. Project memories never cross between projects. | Personal context without leaking between projects. |

## 5. Capabilities

| # | Decision | Why |
|---|---|---|
| D24 | **Files:** .txt, .md and PDF in Phase 1 (10 MB limit, PDF text extracted in the browser); images in Phase 2; searchable in Phase 3; .docx, .xlsx and .csv later. | Useful immediately, without fragile server-side parsing. |
| D25 | **Web search in Phase 5:** a free-tier search API (for example Tavily) as a tool, with OpenRouter's built-in web search as a fallback. | Search is almost never free at scale. |
| D26 | **Tools:** built-in tools first (web search, fetch URL, read/write memory), then an MCP client. The first integrations are GitHub, Google Drive and Gmail/Calendar. | Built-in tools are simpler and are needed anyway. |
| D27 | **Agent guardrails:** at most 10 steps and $0.50 per run. Read-only actions run freely. Any write to an outside system needs approval. Every step is logged. | Safe autonomy. |
| D28 | **Local models (Ollama) are deferred**, but the adapter leaves room for a browser-side "Local" provider. | Cloud functions can't reach the user's laptop. |
| D29 | **GitHub repos become readable project knowledge in Phase 5**, read-only at first. | Useful for the Software Dev project. |

## 6. Operations

| # | Decision | Why |
|---|---|---|
| D30 | **Logging:** call metadata is kept forever. Full assembled prompts are kept for 30 days, and never for Sensitive projects. The app has an Activity/Logs view. No external monitoring tools. | Debugging memory and routing requires seeing the actual prompt. |
| D31 | **Export:** a per-project export plus "Export everything" (JSON plus Markdown) from Phase 1. Import comes later. | The free tier may not include backups, so the data must stay portable. |
| D32 | **The model catalog syncs when it's more than 24h old** (checked when the app loads), plus a manual Refresh button. Actual cost comes from OpenRouter's `usage.cost`. Estimates are used only for the pre-send check. | Verified: OpenRouter returns usage and cost in every response, including the last SSE chunk when streaming. |
| D33 | **Chat experience:** streaming, Markdown and code highlighting, stop, regenerate, and edit-and-resend of the last message. | Streaming affects how the functions are built, so it was decided early. |
| D34 | **Testing:** unit tests for the router and the context builder, a 30-prompt routing test set, and a manual checklist per phase. CI runs on GitHub Actions. | These two modules carry the most logic. |
| D35 | **Format of this spec:** a `docs/` folder of Markdown files plus small Lovable prompts. | It lives in the repo and evolves with the code. |

## Facts verified during Phase 0 (September 2026)

- OpenRouter `:free` models allow 20 requests/minute. The daily limit is **50 requests/day** if you have bought less than $10 of credits in total, and **1,000/day** once you have bought $10 or more at any point. → **Recommendation: buy $10 of credits once.** That raises the free limit and funds the paid budget.
- OpenRouter's `provider.data_collection: "deny"` restricts routing to providers that don't store data, and `provider.zdr: true` restricts it to zero-data-retention endpoints.
- OpenRouter includes `usage` (tokens and `cost`) in every response, and in the final SSE chunk when streaming. No extra parameter is needed.
- Supabase Edge Functions can run `gte-small` embeddings natively (`new Supabase.ai.Session('gte-small')`, 384 dimensions).
- Lovable Cloud is built on Supabase. Secrets are managed in the **Cloud → Secrets** panel and are available to functions through `Deno.env.get()`.

Re-check these facts every few months. Providers change their limits and policies.
