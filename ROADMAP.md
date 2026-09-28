# Roadmap and Development Phases

Each phase ends with an app you can use every day. Don't start a phase until the previous phase's checklist in [TESTING.md](TESTING.md) passes.

| Phase | Theme | Prompts | Effort (rough) |
|---|---|---|---|
| 0 | Product and architecture | — (this spec) | ✅ Done |
| 1 | MVP: chat, projects, manual models, costs, files | 01–09 | 1–2 weekends |
| 2 | Intelligent routing | 10–15 | 1 weekend |
| 3 | Project memory | 16–22 | 1–2 weekends |
| 4 | Cost optimization and insight | 23–26 | 1 weekend |
| 5 | Agentic capabilities | 27–32 | 2+ weekends |
| 6 | Advanced orchestration | 33 (candidates) | Ongoing |

## Phase 1: MVP

**Outcome:** You can replace most day-to-day Claude use with manually chosen models. Every call is logged and capped.

**In scope:** FR-1 to FR-14. Auth, projects (including "General"), conversations, streaming chat through the `chat` function, model catalog sync and picker, sticky manual selection, project default model, stop/regenerate/edit, usage logging with actual cost, the $10 cap and $0.25 confirmation, .txt/.md/PDF upload and attachment, export, prompt logs, the Settings page, About Me, the Sensitive flag (enforced on calls).

**Out of scope:** Auto routing (the picker shows "Auto (coming in Phase 2)", disabled, and new conversations default to the project default or a Settings default model), memory items, the Brief UI, embeddings, dashboards.

> In Phase 1, Settings has a **Default model** (a free model is recommended), used when a project has no default.

**Acceptance criteria:**
- [ ] Log in and out. Sign-up is disabled. A second email gets 403 from the functions.
- [ ] Create a project, then a conversation. Pick a free model. The response streams.
- [ ] Switch to a paid model mid-conversation. The choice sticks for later messages.
- [ ] A long paid request (> $0.25 estimate) asks for confirmation first.
- [ ] Setting the cap to $0.01 blocks paid models, while free models still work.
- [ ] Stop mid-stream saves the partial text. Regenerate replaces it, and the history is kept.
- [ ] Attaching a PDF includes its text. The cost estimate reflects it.
- [ ] `llm_calls` shows the actual cost from OpenRouter for paid calls and $0 for free ones.
- [ ] Export downloads a zip containing JSON and Markdown.
- [ ] The security checklist (SECURITY.md §9) passes.

## Phase 2: Intelligent routing

**Outcome:** Auto becomes the default mode, and most requests land on free models, with a visible explanation.

**In scope:** FR-15 to FR-21. Tier lists with Suggest defaults, the rules plus classifier hybrid, complexity × priority mapping, capability and Sensitive filtering, fallback and cooldowns, automatic T1→T2 escalation, T3 confirmation, the routing card and popover, thumbs with notes, Retry stronger, images to vision models, and the routing test set in CI.

**Acceptance criteria:**
- [ ] The routing test set passes (≥ 27/30 expected tiers under Cost; all "must" cases pass).
- [ ] With a Tier-1 model forced into cooldown, the next candidate is used, and the fallback is shown.
- [ ] With all T1 models in cooldown, the request goes to T2 with `escalated=true` and a visible reason.
- [ ] A "high" request under Performance priority asks before using T3.
- [ ] An image attachment routes only to vision-capable models.
- [ ] Every Auto message has a routing card, and the popover explains the decision.
- [ ] A Sensitive project's calls include `data_collection: deny` (visible in the prompt log metadata or function logs).

## Phase 3: Project memory

**Outcome:** Any model can continue any project without re-explanation. Context stays around 3–5k tokens.

**In scope:** FR-22 to FR-28. Embeddings (`gte-small`), memory items (auto and manual, pinning), the Brief with suggested updates, rolling summaries with chunking of folded messages, the full context builder with budgets and retrieval, file chunking, and the context inspector.

**Acceptance criteria:**
- [ ] After 20+ messages, the Memory tab shows sensible extracted items, with no duplicates for repeated facts.
- [ ] A new conversation in the same project (with a *different* model) correctly answers "What did we decide about X?"
- [ ] A conversation over 6k tokens gets a summary, and older messages remain visible and searchable.
- [ ] The context inspector shows the per-block token counts. The median total is ≤ 5k.
- [ ] Project isolation: a fact from project A never appears in project B's context.
- [ ] Suggested Brief updates show as a diff and never overwrite the Brief without approval.
- [ ] The unit tests for `buildContext` pass (budgets, trimming order, minimums).

## Phase 4: Cost optimization

**Outcome:** You can see where the money goes and adjust tiers based on data.

**In scope:** FR-29 to FR-31. A usage dashboard (daily and monthly spend; by model, project and purpose; share of free requests), a savings estimate against a baseline model, a model comparison table (cost, calls, thumbs-up rate, average latency by task type), a routing report (method mix, classifier usage, escalation rate, thumbs by tier), and "Routing suggestions" (e.g. "Model X has 40% thumbs-down on coding → demote?").

**Acceptance criteria:**
- [ ] The dashboard's monthly total matches the OpenRouter activity page within ±5%.
- [ ] Savings = baseline cost of the same tokens − actual cost, with the method explained in a tooltip.
- [ ] The free-request share is shown and tracked against the ≥70% target.

## Phase 5: Agentic capabilities

**Outcome:** The app can research, fetch, read repos and run bounded multi-step tasks safely.

**In scope:** FR-32 to FR-36. A tool-calling loop (for tool-capable models only), per-project tool allowlists, built-in tools (web search via Tavily with OpenRouter web search as the fallback, fetch URL, memory read/write), agent runs with a 10-step and $0.50 cap, approval for writes, the Activity tab, read-only GitHub knowledge, and an MCP client (remote HTTP MCP servers first). Integrations come in the order GitHub → Google Drive → Gmail/Calendar.

**Acceptance criteria:**
- [ ] "Research X and save the key points to memory" runs search → fetch → memory write, and pauses for approval before the write.
- [ ] The run stops cleanly at 10 steps or $0.50 with a summary.
- [ ] Every step is visible in Activity, with inputs and outputs.
- [ ] Fetching a URL that resolves to a private IP is blocked.

## Phase 6: Advanced orchestration (exploratory)

- Learned routing from feedback and logs (ROUTING.md §11)
- Planner/executor and critic patterns
- A scheduled benchmark of candidate free models using the routing test set plus a small evaluation set
- Optional: an Ollama "Local" provider in the browser (D28), Word/Excel/CSV parsing, import

## Backlog (not scheduled)

- Import from exports, and from Claude/ChatGPT export files
- Conversation branching (the history is already preserved by `superseded_at`)
- Keyboard shortcuts and a command palette
- Prompt library / saved prompts per project
- Scheduled prompts (e.g. a daily brief)
- Multi-user mode (turn on sign-up, remove the owner check, per-user keys in Vault)
