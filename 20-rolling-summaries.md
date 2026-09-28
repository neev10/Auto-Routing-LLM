# 20: Rolling conversation summaries

**Phase 3 · Depends on: 19**

---

Read `docs/MEMORY.md` §5, and `docs/DATABASE.md` (`conversations.summary`, and `chunks` with `conversation_id`).

1. **`summarize` edge function** `{conversationId}`: if the tokens of non-superseded messages newer than `summary_through_at` exceed `summarize_threshold_tokens` (6,000), take the oldest of them, leaving the most recent ~2,000 tokens unsummarized. Call the utility model with the summary prompt (old summary plus the messages being folded in, ≤600 tokens, third person; sensitive respected; free-only; `purpose='summary'`). Save `summary`, and advance `summary_through_at` to the `created_at` of the last message folded in.
2. **Chunk the folded messages:** group them into chunks of about 500 tokens with 50 tokens of overlap (keeping each message whole where possible, and prefixing each with `role:`). Embed them, and insert them into `chunks` (`source_type='message'`, `source_id` = the conversation id, `conversation_id` set).
3. **Trigger:** `chat` calls it in `waitUntil` after `done`.
4. **UI:** a small "📝 Summarized through <time>" marker in the chat at the boundary. Clicking it shows the summary. All messages remain visible.

---

### Verify (don't paste)
- [ ] In a long conversation, the marker appears, `chunks` has rows, and nothing disappears from the UI.
