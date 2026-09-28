# 21: Full context builder and retrieval

**Phase 3 · Depends on: 20**

---

This is the most important prompt in Phase 3. Read **all of** `docs/MEMORY.md` §2, and `docs/TESTING.md` §2.5.

1. `_shared/core/context.ts`: add `buildContext({preamble, aboutMe, instructions, brief, memoryItems (with score and pinned), chunks (with similarity), summary, recentMessages, currentMessage, attachments, budget})`, returning `{messages, report}`. Implement the block order, the per-block caps, the trimming order, the minimum of 2 exchanges, and the report exactly as in MEMORY.md §2.2–2.3. Add `computeBudget(model, tier, settings)` (§2.1). Keep `buildContextPhase1` working, or replace its uses.
2. **Unit tests:** every bullet in TESTING.md §2.5.
3. **Retrieval** in `chat` (MEMORY.md §2.4): build the query, embed it, call `match_memory` and `match_chunks` (excluding the current conversation), apply the thresholds and scores, then call `buildContext`. Recent messages are only those after `summary_through_at`. Update `last_used_at` and `use_count` in the background.
4. `llm_calls.context_tokens` = `report.total`. `prompt_logs.context_report` = the report.
5. The `routing` SSE event and the routing card now show the context tokens.

---

### Verify (don't paste)
- [ ] The model-switching test in TESTING.md §4 (Phase 3 extras) passes.
