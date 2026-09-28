# 06: Chat UI

**Phase 1 · Depends on: 05**

---

Build the chat experience. Read `docs/ARCHITECTURE.md` §2.1 and §3, and `docs/PRD.md` US-1, US-6 and US-15.

1. `src/lib/chatStream.ts`: POSTs to the `chat` function with the user's JWT, parses the SSE events (`routing`, `delta`, `fallback`, `needs_confirmation`, `done`, `error`), and supports an `AbortController`.
2. **ChatView** at `/c/:id`: a header with a breadcrumb (Project ▸ Title, with an inline rename) and the `ModelPicker` (selecting a model sends `modelOverride` on the next message). Below it, the message list and the composer.
3. **Messages:** Markdown rendering with GFM tables, code blocks with syntax highlighting and a copy button, and a copy button per message. Assistant messages show a small footer: the model name · cost · tokens. Hide messages where `superseded_at` is set.
4. **Streaming:** show text as it arrives, and a **Stop** button while streaming.
5. **Actions:** ↻ Regenerate on the last assistant message, and ✎ Edit on the last user message (edit inline, then resend with `editOf`).
6. **Confirm dialog:** on `needs_confirmation`, show the estimated cost and model, with **Send anyway** (resend with `confirm: true`), **Use a free model instead** (opens the picker filtered to free models) and **Cancel**. For `cap_reached`, explain the monthly cap.
7. **Errors:** an inline error card with **Retry**. Show a clear message for `rate_limited` ("Free model limit reached; try another model or wait") and for `credits`.
8. **BudgetMeter** at the bottom of the sidebar: "Month: $x.xx / $10" with a progress bar. It turns amber at 80% and red at 100%. It reads from `llm_calls`.
9. **Composer:** a textarea that grows with the text; Enter sends and Shift+Enter adds a new line; it's disabled while streaming; and it has a 📎 button, which does nothing yet (it's wired up in prompt 08). Show 🔒 in the composer when the project is sensitive.

---

### Verify (don't paste)
- [ ] Phase 1 acceptance items 2–6 in ROADMAP.md.
