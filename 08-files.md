# 08: Files (.txt, .md, PDF) and attachments

**Phase 1 · Depends on: 07**

---

Add file support. Read `docs/MEMORY.md` §6 and `docs/DATABASE.md` (the `files` table and storage).

1. **Upload** from the project's Files tab (drag-and-drop) and from the composer's 📎 button. Accept `.txt`, `.md` and `.pdf` up to 10 MB, and reject anything else with a clear message.
2. **Storage path:** `{user_id}/{project_id}/{file_id}/{filename}` in the `project-files` bucket.
3. **Text extraction in the browser:** read .txt and .md directly. For PDFs, use `pdfjs-dist` in `src/lib/pdfText.ts` (concatenate the page text with page markers). If a PDF yields almost no text, set `text_status='none'` and show "Scanned PDF, no text found". Save `extracted_text`, `token_estimate` and `text_status`.
4. **Files tab:** a list with the name, size, token estimate, status, a **Preview text** drawer, and delete (which removes the storage object and the row).
5. **Attach to a message:** attached files appear as chips above the composer, and are sent as `attachmentFileIds`. The user message stores `attachments` `[{fileId, name, kind}]`, which are shown as chips on the sent message. The `chat` function already includes the text (prompt 05).
6. Show the attachment's token estimate on its chip, so the user can see why a request might need a cost confirmation.

---

### Verify (don't paste)
- [ ] Upload a text PDF, attach it, ask for a summary, and the answer uses its content.
- [ ] A file over 10 MB is rejected.
