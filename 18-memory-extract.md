# 18: Automatic memory extraction

**Phase 3 · Depends on: 17**

---

Read `docs/MEMORY.md` §3.2 in full, and `docs/TESTING.md` §2.6.

1. `_shared/core/memoryMerge.ts` (pure): given proposed items (each with an embedding) and existing items (id, content, embedding, type), return the operations `{insert[], update[], supersede[], resolve[]}` using similarity ≥ 0.92 → update, plus the `supersedes` and `resolves` hints. Add unit tests.
2. **`memory-extract` edge function** `{conversationId}`: owner check; load the messages after `last_extracted_at` (batches of 8k tokens or less), the Brief, and up to 40 active items; call the utility model with the extraction prompt from MEMORY.md (JSON reply, sensitive respected, free-only fallback, logged as `purpose='extraction'`); embed the proposals; apply the `memoryMerge` operations; set `last_extracted_at`; return a count summary. If ≥5 items changed (or the Brief is older than 7 days and ≥1 item changed), also generate a Brief suggestion (this is implemented in prompt 19; for now, leave a TODO hook that prompt 19 fills).
3. **Triggers:** in `chat`, after `done`, in `waitUntil`: if the number of messages since `last_extracted_at` is at least `extract_every_n_messages` (16), call the extraction. In the client: when leaving a conversation (on route change) with ≥4 unextracted messages, call it too, and on tab close use `fetch(url, {keepalive: true})` with the auth header.
4. **Memory tab:** a toast or badge: "3 new memories from 'Conversation X'". New items get a "new" highlight until they're viewed.

---

### Verify (don't paste)
- [ ] Have a 20-message conversation with decisions in it. Sensible items appear, and repeating a fact doesn't create a duplicate.
