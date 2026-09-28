# 17: Memory tab and manual memory

**Phase 3 · Depends on: 16**

---

Read `docs/MEMORY.md` §3.1 and §3.3.

1. **Project → Memory tab:** items grouped by type (Decisions, Facts, Preferences, Open questions, Action items), with filters (status, source, pinned) and search. Each item shows its content, a type badge, 📌 when pinned, importance dots, its source (auto / manual / pinned message, with a link to the conversation), and when it was last used.
2. **Actions:** edit (re-embeds through the `embed` helper, using a small `memory-upsert` function or the embed endpoint followed by a client update), pin or unpin, resolve (for open questions and action items), archive, delete, and change the type.
3. **+ Add memory:** a type selector and the content. It's saved with `source='manual'` and embedded.
4. **📌 Pin to memory** on any chat message: calls a `memory-pin` function that condenses the message into one self-contained item with the utility model (respecting sensitive and free-only), then saves it with `source='pinned_message'`, `pinned=true` and the embedding. The toast lets the user edit the result.
5. **Export** now also includes `memory_items` (without embeddings) in JSON and Markdown (`memory.md` per project).

---

### Verify (don't paste)
- [ ] Add a manual item, pin a message, edit an item, and confirm the embedding is not null afterwards.
