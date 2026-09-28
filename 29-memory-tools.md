# 29: Memory tools

**Phase 5 · Depends on: 28**

---

1. **`search_memory`** (read): `{query}` → `match_memory` plus `match_chunks` for the current project. Returns the items and excerpts with their ids.
2. **`save_memory`** (**write**, needs approval): `{type, content, importance}` → the UI shows an approval card with the exact item, and **Approve** / **Edit** / **Reject**. On approval it inserts with `source='auto'` and embeds.
3. **`update_brief`** (**write**, needs approval): `{proposed_brief}` → it goes into the existing suggestion flow (`brief_suggestion`) rather than writing directly.
4. **Approval mechanics:** when a write tool is called, `chat` saves the pending call, sends `needs_approval{toolCallId, tool, args}`, and ends the stream. The client shows the card. Approving calls `chat` with `{approveToolCall: id, edits?}`, which executes the call and continues the loop. Rejecting sends the tool result "User rejected" and continues.

---

### Verify (don't paste)
- [ ] "Remember that we chose Tavily for search" produces an approval card, and nothing is saved until you approve.
