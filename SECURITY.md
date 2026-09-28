# Security & Privacy

This is a single-user app, but it has a public URL, a paid API key, and personal data (career, investments, family). The design below reflects that.

## 1. Assets and threats

| Asset | Threat | Control |
|---|---|---|
| OpenRouter key | Leaked through the browser, a repo, logs or the chat | Stored only as a backend secret. Never returned by any function. Never logged. Never in `.env` with a `VITE_` prefix. |
| Credits | A stranger uses the deployed app | Login required, sign-ups disabled, `ALLOWED_USER_EMAIL` check in every function, OpenRouter key credit limit |
| Credits | A bug or loop runs up spend | App-level cap, per-request confirmation, max 4 attempts per request, agent step and cost caps, key credit limit as the backstop |
| Personal data | Read by another user of the database | RLS on every table. Only one account exists. |
| Personal data | Used for training by a model provider | Sensitive projects → `data_collection: "deny"` on **every** call for that project |
| Personal data | Stored in prompt logs | Logs expire after 30 days. Sensitive projects are never logged in full. |
| Personal data | Exposed through the public GitHub repo | No data or secrets in the repo. The `.gitignore` covers `.env*` except `.env.example`. |
| Files | Accessed by guessing URLs | Private bucket, path-scoped policy, short-lived signed URLs (10 minutes) |
| Behavior | Prompt injection from files or web pages (Phase 5) | Tool outputs are marked as untrusted data. Writes to outside systems need approval. Agents can't change settings or read secrets. |

## 2. API key handling (D7)

- Store it as the `OPENROUTER_API_KEY` secret in Lovable **Cloud → Secrets**. The value is write-only there and can't be read back.
- **Never** paste the key into the Lovable chat, a prompt, the database or a file.
- The Settings page shows **Key configured ✓** plus a **Test connection** button, which calls `test-connection`. That function returns only `{ ok, label?, limit, usage, is_free_tier }` from OpenRouter's `/api/v1/key`.
- **Removing or rotating the key:** delete or replace the secret in Cloud → Secrets, and revoke the old key on openrouter.ai. The functions read the secret on each invocation.
- **Recommended on OpenRouter:** give this app its own key, named `switchyard`, with a credit limit of **$10** (use a monthly reset if OpenRouter offers one). In OpenRouter's privacy settings, also consider turning off providers that train on prompts across the whole account. The per-request `data_collection` setting then becomes a second layer of protection for Sensitive projects.

**Future multi-provider or multi-user keys:** if users ever paste their own keys into the app, store them encrypted with Supabase Vault (`vault.create_secret`). Read them only inside functions using the service role, and never select them back to the client. This isn't built now.

## 3. Authentication (D9)

1. Email/password auth.
2. Create the owner account, then **turn off sign-ups** in the Lovable Cloud auth settings.
3. As a second layer, every edge function calls `requireOwner(req)`:
   - It verifies the JWT with `supabase.auth.getUser(token)`.
   - It rejects the request with 403 unless `user.email` equals the `ALLOWED_USER_EMAIL` secret (compared case-insensitively).
4. Session: supabase-js defaults (refresh tokens in local storage). Settings has a **Sign out everywhere** button.

## 4. Database access

- RLS is enabled on **every** table (see DATABASE.md). A test in TESTING.md checks this.
- Tables written only by functions (`llm_calls`, `routing_decisions`, `prompt_logs`, `chunks`, agent tables) have **no** insert/update/delete policy for `authenticated`.
- Functions use the service-role client, **but always filter by the verified `user.id`**. They never take a `user_id` from the request body.
- Functions validate that the `conversationId` and `projectId` they receive belong to the user before doing anything.

## 5. Sensitive projects (D10)

`projects.is_sensitive = true` means:
- Every OpenRouter call for the project (chat, classifier, title, extraction, summary, brief) sends `provider: { data_collection: "deny" }`.
- No `prompt_logs` rows are written. `llm_calls` metadata is still written.
- If no provider qualifies, the call fails with a message instead of quietly falling back to a provider that keeps data.
- A 🔒 badge appears in the sidebar and the composer.
- Embeddings are computed inside the edge runtime (`gte-small`), so nothing leaves the backend.
- Export still includes the project (the export belongs to the user). The export dialog warns that the file contains sensitive data.

Optional: a "Strict" setting (future) adds `zdr: true` for zero-data-retention endpoints only. It isn't built now, because it removes most free models.

## 6. Edge function hygiene

- CORS: allow only the app's origin(s), read from the `ALLOWED_ORIGINS` secret. Localhost is allowed only in development.
- Validate inputs (zod or manual checks): message length ≤ 200k characters, attachments that belong to the user, known enum values.
- Never log request bodies, prompts or keys to the function console. Log only ids, model, status and latency.
- Error messages sent to the client are generic, plus an error code. Details go to `llm_calls.error`.
- Timeouts: 4s for the classifier, 20s for the first token, and `max_tokens` on every call.

## 7. Agents and tools (Phase 5)

- Each project has a tool allowlist (`project_tools`).
- **Read-only tools** (search, fetch URL, read memory, GitHub read) run without approval.
- **Write tools** (write memory, create a GitHub issue, send an email, create a calendar event…) pause the run with `awaiting_approval`. The user sees the exact payload and chooses **Approve** or **Reject**.
- Guardrails: 10 steps and $0.50 per run (D27). When a limit is hit, the run stops with `limit_reached` and a summary of the progress so far.
- Tool results are wrapped in `<tool_result untrusted="true">…</tool_result>`, and the system prompt tells the model never to follow instructions found inside tool results.
- `fetch_url` blocks private IP ranges and `localhost`, which protects against SSRF.
- Integration tokens (GitHub PAT, Google OAuth) are stored in secrets or Vault, never in `integrations.config`.

## 8. Open-source hygiene

- `.env.example` lists variable **names** only.
- Secret-scanning / push protection is enabled on the GitHub repo.
- Dependabot alerts are on.
- GitHub also recognizes `docs/SECURITY.md` as the repo's security policy. The reporting contact is in `CONTRIBUTING.md`.

## 9. Security checklist (run at the end of every phase)

- [ ] `grep -r "sk-or-" src/ supabase/` finds nothing.
- [ ] The browser network tab shows no requests to `openrouter.ai`.
- [ ] Sign-up is disabled, and the login page has no "Sign up" link.
- [ ] Calling a function with no JWT returns 401. A JWT for a different email returns 403.
- [ ] Every table has RLS enabled (`select relname from pg_class where relrowsecurity = false and relnamespace = 'public'::regnamespace` returns only views).
- [ ] A Sensitive project's calls show `sensitive=true` in `llm_calls`, and it has no `prompt_logs` rows.
- [ ] Files can't be fetched without a signed URL.
