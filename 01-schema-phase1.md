# 01: Phase 1 database schema

**Phase 1 · Depends on: 00 (Knowledge pasted)**
Paste everything between the lines into Lovable.

---

Set up the Phase 1 database. Read `docs/DATABASE.md`, sections "Phase 1 tables", "RLS policies" and "Triggers and housekeeping".

Create a migration that:
1. Enables the `vector` extension.
2. Creates exactly these tables with the columns, defaults and check constraints in the doc: `profiles`, `app_settings`, `models`, `projects`, `conversations`, `routing_decisions`, `messages`, `llm_calls`, `prompt_logs`, `files`. Include the `default_model_id` column in `app_settings`.
3. Adds the indexes listed.
4. Enables RLS on every table:
   - Owner full access (`user_id = auth.uid()`) on `profiles`, `app_settings`, `projects`, `conversations`, `messages`, `files`.
   - **Select only** for the owner on `routing_decisions`, `llm_calls`, `prompt_logs`.
   - `models`: select for any authenticated user. No write policies.
5. Adds `updated_at` triggers, and a trigger on `messages` insert that increments `conversations.message_count` and bumps `conversations.updated_at`.
6. Creates a private Storage bucket `project-files`, with policies that allow an authenticated user to read, write and delete objects only when the first path segment equals their `auth.uid()`.

Don't build any UI in this step.

---

### Verify (don't paste)
- [ ] The Cloud → Database view shows all 10 tables, with RLS enabled on each.
- [ ] `models` has no insert policy.
- [ ] The `project-files` bucket is private.
