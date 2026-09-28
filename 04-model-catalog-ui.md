# 04: Model catalog, picker and OpenRouter settings

**Phase 1 · Depends on: 03**

---

Build the model UI. Read `docs/PRD.md` (FR-5, FR-6, FR-7, FR-14) and `docs/ROUTING.md` §3.

1. **Settings → OpenRouter tab:** shows "Key configured ✓" and a **Test connection** button (calls `test-connection` and shows the limit, monthly usage and whether the account is on the free tier). Also shows the text "Key is stored in Lovable Cloud Secrets and never reaches the browser."
2. **Stale sync:** on app load, if `app_settings.models_synced_at` is null or more than 24h old, call `models-sync` in the background.
3. **Settings → Models tab:** a searchable, sortable table of the catalog: name, id, context, $/M in, $/M out, badges (FREE, 👁 vision, 🛠 tools, 🧠 reasoning), and active status. Include a **Refresh models** button and filters (free only, vision, tools, minimum context).
4. **`ModelPicker` component** for the chat header: a button showing the current model, which opens a command-style dialog with search, the same filters and badges, sections for "Free" and "Paid", and the price per million shown. The first item is "Auto (coming in Phase 2)", shown disabled.
5. **Settings → Limits tab (partial):** a **Default model** selector (`app_settings.default_model_id`), with a hint recommending a free model.
6. **Project default model:** add a "Default model" selector to the project settings (a simple dialog from the project's ⋯ menu for now), with options for Sensitive toggle, priority (stored now, used in Phase 2), description and instructions.

---

### Verify (don't paste)
- [ ] The catalog shows hundreds of models, and the free filter works.
- [ ] Test connection shows your $10 limit.
