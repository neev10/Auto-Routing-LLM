# 05: The `chat` edge function (streaming, manual models, cost guard)

**Phase 1 · Depends on: 04**

---

Build the `chat` edge function. Read `docs/ARCHITECTURE.md` §3 (the lifecycle and SSE events), `docs/ROUTING.md` §3 and §7, `docs/MEMORY.md` §2.4 ("Phase 1" paragraph), and `docs/SECURITY.md` §5–6.

**Request body:** `{conversationId, content, attachmentFileIds?: string[], modelOverride?: string, confirm?: boolean, regenerateOf?: string, editOf?: string}`

**Steps:**
1. `requireOwner`. Load the conversation and project (verify ownership), `app_settings`, `profiles.about_me`, and this month's spend (the sum of `llm_calls.cost_usd` since the start of the UTC month).
2. **Model resolution (Phase 1):** `modelOverride` (then set `conversations.mode='manual'` and `sticky_model_id`) → `sticky_model_id` → `projects.default_model_id` → `app_settings.default_model_id`. If none is set → `error{code:'no_model'}`. Record the method (`manual`, `sticky` or `project_default`; use `project_default` for the settings default too).
3. Handle `editOf` and `regenerateOf`: set `superseded_at` on the old user or assistant message (and every message after it). For an edit, insert the new user message. For a regenerate, reuse the existing user message. Otherwise, insert the user message.
4. **Context (Phase 1):** system = the preamble (from MEMORY.md §2.3, first paragraph) + About Me + project instructions. Then the non-superseded messages, newest first, until 6,000 tokens (keeping at least the last 2 exchanges). Then the current message plus the attachment text (`files.extracted_text`, truncated to 12,000 tokens with a note). Put this in `_shared/core/context.ts` as a pure `buildContextPhase1()`.
5. **Cost guard** with `capCheck`. If confirmation is needed and `confirm` isn't true → send a `needs_confirmation` event and end. If the cap is reached and the model is paid → `needs_confirmation{kind:'cap_reached'}` and a message suggesting a free model.
6. Insert a `routing_decisions` row (mode manual, the method, `tier` null, and a reason like "Manual choice: X").
7. Stream: send the `routing` event, then proxy `delta` events. Save the assistant message with `status='streaming'` at the start, and update it at the end.
8. On the final usage chunk: update the message (content, `status='complete'`, `model_id`) and insert `llm_calls` (`purpose='chat'`, tokens, `cost_usd = usage.cost`, `is_free`, `latency_ms`, `ttft_ms`, `generation_id`, `context_tokens`, `sensitive`). Unless the project is sensitive, insert `prompt_logs` (messages, `expires_at = now + retention days`). Send the `done` event.
9. **Abort:** if `req.signal` aborts, abort upstream, save the partial text with `status='stopped'`, and insert `llm_calls` with `status='stopped'` and a null cost.
10. **Upstream errors before the first token:** `error` event with a code (`rate_limited`, `credits`, `no_endpoints`, `upstream`) and `retryable`. Record `llm_calls` with the error status. (Automatic fallback comes in Phase 2.)
11. **Background** (`EdgeRuntime.waitUntil`): if the conversation title is still "New chat" after the first exchange, generate a title of 6 words or fewer with the default free model (respecting sensitive); purge expired `prompt_logs` if `prompt_logs_purged_at` is more than 24h old.

Also create `reconcile-costs`: for this user's `llm_calls` with `cost_usd is null` and a `generation_id`, fetch `/api/v1/generation?id=` through the adapter and fill in the cost. `chat` triggers it in the background when `costs_reconciled_at` is more than 1h old.

Add unit tests in `src/test/context.phase1.test.ts`: ordering, the 6k cap, the minimum of 2 exchanges, and exclusion of superseded messages.

---

### Verify (don't paste)
- [ ] Use curl or the function test UI to see SSE events in order: routing → delta… → done.
- [ ] `llm_calls` has the real cost for a paid model.
