# 07: Project workspace

**Phase 1 · Depends on: 06**

---

Build the project workspace at `/p/:projectId`. Read `docs/PRD.md` §6, §10 and US-8.

1. **Header:** the project name (editable), the description, a 🔒 Sensitive toggle (with a confirmation that explains it limits which providers are used), the priority selector (Cost/Balanced/Performance, with a note that it's used from Phase 2), the default model, and a ⋯ menu with Archive and Export (Export is wired up in prompt 09).
2. **Tabs:** Chats · Instructions · Files · Memory · Brief · Activity.
   - **Chats:** the conversation list (title, last updated, message count, the model last used), "+ New chat", rename and delete.
   - **Instructions:** a Markdown editor for `projects.instructions`, with a token estimate.
   - **Files:** an empty state for now (prompt 08).
   - **Memory** and **Brief:** a placeholder saying "Coming in Phase 3".
   - **Activity:** a table of this project's `llm_calls` (time, purpose, model, tokens, cost, status). Clicking a row opens a drawer with the prompt log if one exists.
3. **Sidebar:** clicking a project opens its workspace. Archived projects are hidden, with a "Show archived" toggle. General can't be archived.
4. **Deleting a conversation** asks for confirmation, and cascades.

---

### Verify (don't paste)
- [ ] Toggling Sensitive shows 🔒 in the sidebar. A chat in that project writes no `prompt_logs` row, and its `llm_calls.sensitive` is true.
