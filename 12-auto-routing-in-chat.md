# 12: Auto routing in the `chat` function

**Phase 2 · Depends on: 11**

---

Wire routing into `chat`. Read `docs/ROUTING.md` §2–§7 and §10, and `docs/ARCHITECTURE.md` §3.

1. **Auto becomes available:** conversations whose `mode='auto'` (the new default for conversations without a project default model) are routed with `selectModel`. Manual, sticky and project-default choices behave as before, but now go through the requirements check (for example `model_lacks_vision`).
2. **Classifier call:** when `band='uncertain'`, call the classifier model through the adapter (non-streaming, `temperature 0`, `max_tokens 80`, a 4s timeout, the system prompt from ROUTING.md §4.2, the message truncated to 1,500 tokens, and `data_collection: deny` if sensitive). Log it in `llm_calls` with `purpose='classifier'`. Parse the reply with `parseClassifierOutput`, and on failure use `classifierFallback`.
3. **Health:** load `models.cooldown_until`. Implement runtime fallback (ROUTING.md §6.2): 429 → 10-minute cooldown; 5xx, timeout or no first token within 20s → `consecutive_failures++` and a 2-minute cooldown (15 minutes after 3 failures); 400 "no endpoints" → next candidate. Allow at most 4 attempts. When T1 is exhausted, escalate to T2 automatically (unless cap-blocked). When T2 is exhausted, return `needs_confirmation{kind:'tier3'}`. Send a `fallback` SSE event for each switch, and store `fallback_chain`. Reset `consecutive_failures` after a success. Only the service role updates `models`.
4. **T3 confirmation:** when the router returns `needsConfirmation: 'tier3'`, send `needs_confirmation{kind:'tier3', modelId, estimatedCost}`. The client can resend with `confirm: true` to use T3, or with `forceTier: 2`.
5. **Persist everything** in `routing_decisions` (priority, complexity, score, signals, requirements, tier, candidates, fallback_chain, escalated, reason, classifier_latency_ms, estimated_cost_usd).
6. The `routing` SSE event carries the fields listed in ARCHITECTURE.md §3.
7. **Title and utility calls** use `utility_model_id` with free-only fallback (ROUTING.md §10).
8. **`dryRun: true`:** route only (including the classifier), and return the decision as JSON without generating. This is used by the routing eval.

---

### Verify (don't paste)
- [ ] Phase 2 acceptance items 2, 3 and 7 in ROADMAP.md. To force a cooldown, set `cooldown_until` in the future on a T1 model in the database.
