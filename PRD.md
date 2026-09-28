# Product Requirements Document

**Product:** Switchyard, a personal BYOK multi-LLM workspace
**Owner:** Sandy Paudel
**Status:** Approved for build (Phase 0 complete)
**Related:** [DECISIONS.md](DECISIONS.md), [ROADMAP.md](ROADMAP.md)

---

## 1. Problem

- Claude costs about $20/month, and most everyday requests (rewrites, summaries, quick questions) don't need a frontier model.
- Trying other models means juggling several apps, and none of them remembers the others' conversations.
- Project context lives inside each vendor's app. Switching models means re-explaining everything.

## 2. Vision

One workspace where **projects own their memory**, **models are interchangeable**, and **free models handle everything they're good enough for**. The app is always open about which model it used, why, and what it cost.

## 3. Goals and success metrics

| Goal | Metric | Target (by end of Phase 4) |
|---|---|---|
| Cut AI spend | Monthly OpenRouter spend | ≤ $10/month |
| Use free models where they're good enough | Share of chat requests served by free models | ≥ 70% |
| Keep routing quality acceptable | Thumbs-up rate on Auto-routed responses | ≥ 80% |
| Make model switching seamless | Switching models mid-conversation needs no re-explanation | Pass on manual test |
| Keep context efficient | Median context tokens per request | ≤ 5,000 |
| Become the daily tool | Days per week the app is used | ≥ 5 |

## 4. Non-goals (for now)

- Multi-user support, teams, sharing and billing
- Mobile layouts or native apps
- Providers other than OpenRouter (the design allows them later)
- Local models
- Image generation, voice, video
- Fine-tuning or training
- A plugin marketplace

## 5. User

A single power user: a senior technical program leader who works on several long-running projects (an AI app, a career search, investment research, business ideas, software development). He is comfortable with technical detail and wants transparency and control rather than magic.

## 6. Core concepts

| Concept | Definition |
|---|---|
| **Project** | A persistent workspace containing conversations, instructions, memory, files, settings and activity. There is always at least one project, **General**, which is created automatically. |
| **Conversation** | A chat thread inside one project. |
| **Mode** | **Auto** (the router picks the model) or **Manual** (the user picks). A manual pick sticks for the rest of the conversation. |
| **Tier** | A curated, ranked list of models: T1 = free/lightweight, T2 = strong/moderate cost, T3 = premium. |
| **Routing decision** | A record of how and why a model was chosen for a message. |
| **Project Brief** | A 300–500 word living description of the project, included in every request. |
| **Memory item** | A short typed entry: fact, decision, preference, open question or action item. |
| **About Me** | A short global profile included in every project. |
| **Sensitive project** | A project that only routes to providers that don't store or train on data. |

## 7. User stories

### Chat and models
- **US-1** As the user, I can start a conversation inside a project and get a streamed response.
- **US-2** I can pick a specific model from a searchable list that shows price, context size, free/paid and capability badges.
- **US-3** I can leave the mode on Auto and have the app pick a sensible model.
- **US-4** For each Auto-routed response, I can see which model was used, why it was chosen, the tier, the complexity, whether it escalated, the cost, and the context tokens.
- **US-5** I can switch models mid-conversation, and the new model still understands the project.
- **US-6** I can stop a response, regenerate it, retry it with a stronger model, or edit my last message and resend it.
- **US-7** I can rate a response with thumbs up or down and an optional note.

### Projects and memory
- **US-8** I can create, rename, archive and configure projects: instructions, Sensitive flag, priority, default model.
- **US-9** Each project has a Brief that I can edit. The app suggests updates, and I accept or reject them.
- **US-10** The app automatically extracts facts, decisions, preferences, open questions and action items. I can see, edit, pin, resolve and delete them.
- **US-11** I can add a memory manually, or pin any message to memory.
- **US-12** Long conversations get summarized automatically, and nothing is lost.
- **US-13** I can upload files to a project, and relevant parts get used as context.
- **US-14** I can maintain a short About Me profile that is used in every project.

### Cost and control
- **US-15** I'm asked to confirm before any request estimated to cost more than $0.25.
- **US-16** Paid models are blocked once I hit the monthly cap, while free models keep working.
- **US-17** I can see spend by day, month, model and project, and an estimate of my savings.
- **US-18** I can export a project, or everything, as JSON plus Markdown.
- **US-19** I can see a log of every LLM call, including the assembled prompt (for 30 days, and not for Sensitive projects).

### Later phases
- **US-20** The app can search the web, fetch URLs and read GitHub repos as tools.
- **US-21** The app can run multi-step tasks within step and cost limits, and asks me before writing to any outside system.
- **US-22** The router can learn from my ratings and logs which models work best for which tasks.

## 8. Functional requirements

IDs are referenced from the Lovable prompts and tests. The **Phase** column says when each requirement ships.

| ID | Requirement | Phase |
|---|---|---|
| FR-1 | Email/password auth. All data is scoped to `auth.uid()` through RLS. Edge functions reject any user whose email isn't `ALLOWED_USER_EMAIL`. | 1 |
| FR-2 | Projects CRUD. A "General" project is created automatically on first login. | 1 |
| FR-3 | Conversations CRUD inside projects, with titles generated automatically by a free model after the first exchange. | 1 |
| FR-4 | Streaming chat through the `chat` edge function (SSE). The browser never calls OpenRouter directly. | 1 |
| FR-5 | Model catalog synced from OpenRouter `/api/v1/models` into the `models` table. The sync runs when the catalog is more than 24h old, and on a manual Refresh. | 1 |
| FR-6 | A model picker with search, filters (free, vision, tools, context ≥ N) and price display. | 1 |
| FR-7 | Manual model selection that sticks for the conversation. The project default model is applied to new conversations. | 1 |
| FR-8 | Messages persisted with the model used, status (complete/stopped/error) and usage. | 1 |
| FR-9 | Stop, regenerate and edit-last-message. Superseded messages are kept, hidden, with `superseded_at` set. | 1 |
| FR-10 | Usage log (`llm_calls`) recording tokens, cost from `usage.cost`, latency, status and purpose for every call. | 1 |
| FR-11 | Monthly cap and per-request confirmation (see ROUTING.md §7). | 1 |
| FR-12 | Upload .txt, .md and PDF files (≤ 10 MB). Text is extracted (PDF in the browser). Files can be attached to a message. | 1 |
| FR-13 | Project export and export everything (JSON + Markdown zip). | 1 |
| FR-14 | Settings: key status and test connection, spend limits, About Me, default priority. | 1 |
| FR-15 | Auto mode with hybrid complexity detection (rules + classifier). | 2 |
| FR-16 | Tier lists editable in Settings, with a "Suggest defaults" helper. | 2 |
| FR-17 | Capability matching (vision, context length, tools) and the Sensitive filter. | 2 |
| FR-18 | Fallback chain, automatic T1→T2 escalation, T3 confirmation, "Retry stronger". | 2 |
| FR-19 | A routing card on every assistant message, with details in a popover. | 2 |
| FR-20 | Feedback (thumbs and note) linked to the routing decision. | 2 |
| FR-21 | Image attachments routed to vision models. | 2 |
| FR-22 | Project Brief, with suggested updates. | 3 |
| FR-23 | Automatic memory extraction and manual memories, shown in the Memory tab. | 3 |
| FR-24 | Rolling conversation summaries. | 3 |
| FR-25 | Embeddings (`gte-small`) for memory items and chunks, and semantic retrieval. | 3 |
| FR-26 | The context builder with token budgets (see MEMORY.md). | 3 |
| FR-27 | File chunking and retrieval. | 3 |
| FR-28 | A context inspector: "what did the model see?" | 3 |
| FR-29 | A usage dashboard: by day, month, model and project; free vs. paid; savings estimate. | 4 |
| FR-30 | Model comparison: cost and rating per model and task type. | 4 |
| FR-31 | Routing report: method accuracy, escalation rate, thumbs by tier. | 4 |
| FR-32 | Tool calling framework and built-in tools. | 5 |
| FR-33 | Web search (Tavily plus OpenRouter fallback). | 5 |
| FR-34 | Agent runs with guardrails and an Activity log. | 5 |
| FR-35 | GitHub read-only knowledge. | 5 |
| FR-36 | MCP client. | 5 |
| FR-37 | Multi-model patterns (planner/executor, critic), benchmarking, learned routing. | 6 |

## 9. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | The key exists only in backend secrets. RLS is on every table. There is no public sign-up. See SECURITY.md. |
| Privacy | The Sensitive flag is enforced for every call made for that project: chat, classifier, extraction, summary and title. |
| Cost | The infrastructure stays on free tiers. The app never makes a paid call without passing the cap check. |
| Performance | Time to first token is under 2.5s when rules decide and under 5s when the classifier runs (excluding model latency). The classifier times out after 4s. |
| Reliability | A failure of one model never stops a request while free alternatives remain. |
| Portability | No Lovable-proprietary APIs. Core logic is pure TypeScript and can be unit-tested outside Deno. |
| Maintainability | The router and context builder are pure functions with unit tests. Settings are data, not code. |
| Transparency | Every Auto decision is explainable from the stored `routing_decisions` row. |

## 10. UX outline

```
┌───────────────┬─────────────────────────────────────────────────────────┐
│ + New chat    │ Project ▸ Conversation title            [Model: AUTO ▼] │
│               │─────────────────────────────────────────────────────────│
│ PROJECTS      │  user: …                                                │
│ ● General     │  assistant: …                                           │
│ ● AI App      │   ┌──────────────────────────────────────────────────┐  │
│ ● Career 🔒   │   │ ⚡ Auto → qwen/…:free · T1 · low · $0.00 · 3.1k ctx│  │
│               │   │ Why: short rewrite, free model available  👍 👎 ↻↑│  │
│ RECENT        │   └──────────────────────────────────────────────────┘  │
│ …             │                                                         │
│               │ [📎] [ Type a message…                     ] [Send ▸]   │
│ ⚙ Settings    │ Month: $3.12 / $10 ███░░░░░░░                           │
└───────────────┴─────────────────────────────────────────────────────────┘
```

**Project workspace tabs:** Chats · Brief · Memory · Files · Instructions · Models · Tools (Phase 5) · Activity
**Settings tabs:** OpenRouter · Routing & Tiers · Limits · About Me · Models catalog · Export · Logs

🔒 marks a Sensitive project.

## 11. Open questions (to be decided later)

- Which three integrations come first in Phase 5? The current plan is GitHub, Google Drive, then Gmail/Calendar.
- Should the Brief update itself without asking once trust is established?
- Is an Ollama "Local" provider worth building? (Deferred.)
