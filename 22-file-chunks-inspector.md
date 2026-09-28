# 22: File chunking and the context inspector

**Phase 3 · Depends on: 21**

---

Read `docs/MEMORY.md` §6 and §8.

1. **File chunking:** a `file-ingest` function `{fileId}`: split `extracted_text` into chunks of about 500 tokens with 50 overlap (splitting on headings or paragraphs first), embed them, insert them into `chunks` (`source_type='file'`, `source_id` = the file id), and set `files.chunked_at`. It's called after upload. Add a "Re-index" action per file, and "Index all" for files that were uploaded before this step. Deleting a file deletes its chunks.
2. Attachments that are truncated at 12k tokens now say "(rest of the file is available via search)".
3. **Context inspector:** a "🔍 What did the model see?" link on each assistant message opens a drawer showing the budget, a per-block token bar (with trimmed blocks highlighted), the memory items and chunks included with their scores (each links to the item or file), and the full messages from `prompt_logs` (or a note explaining why they aren't available).
4. **Settings → Memory:** the context budget, summary threshold and extraction interval (from `app_settings`), plus a "Reset to defaults" button.

---

### Verify (don't paste)
- [ ] Run the whole Phase 3 checklist in ROADMAP.md and TESTING.md, and update DECISIONS.md.
