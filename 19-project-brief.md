# 19: Project Brief

**Phase 3 · Depends on: 18**

---

Read `docs/MEMORY.md` §4.

1. **Project → Brief tab:** a Markdown editor for `projects.brief`, with a token counter (warn above 700 tokens) and `brief_updated_at`. New projects get the template (Purpose · Goals · Current state · Key decisions · Constraints · Open problems).
2. **Suggested update:** fill in the hook from prompt 18. A `brief-suggest` function takes the current Brief plus the changed items and asks the utility model for a revised Brief of ≤500 words that keeps the user's structure. The result is stored in `projects.brief_suggestion`.
3. When a suggestion exists: a banner "Suggested update to the Brief" with a side-by-side or inline diff, and **Accept** (copies it into `brief`, clears the suggestion, sets `brief_updated_at`), **Edit then accept**, and **Dismiss**. The Brief is never overwritten automatically.
4. **Regenerate from memory:** available when the Brief is empty, or on demand. It builds a draft from the active items and puts it into the suggestion flow.
5. **Export** includes the Brief.

---

### Verify (don't paste)
- [ ] A suggestion appears after extraction, and nothing changes until you accept it.
