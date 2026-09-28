# 00: Project Knowledge (paste into Lovable → Project settings → Knowledge)

Paste everything below the line into Lovable's **Knowledge** field. Lovable includes it in every prompt. Keep it under Lovable's size limit; if you need to trim, remove the "Style" section first.

---

## Project: Switchyard, a personal BYOK multi-LLM workspace

Switchyard is a single-user web app. It routes chat requests to many LLMs through OpenRouter, keeps model-independent project memory, and tracks cost. The full specification is in the repo under `docs/`. **Read the doc sections named in each prompt before writing code.** When a prompt and the docs disagree, ask me.

### Non-negotiable rules
1. **The browser never calls OpenRouter.** Every LLM call goes through a Supabase Edge Function that reads `Deno.env.get("OPENROUTER_API_KEY")`. Never put the key in frontend code, `.env` or the database.
2. **RLS on every table.** User-owned tables: `user_id = auth.uid()` for all operations. Tables written only by functions (`llm_calls`, `routing_decisions`, `prompt_logs`, `chunks`, `agent_runs`, `agent_steps`) get a SELECT policy only, and functions write them with the service role.
3. **Every edge function starts with the owner check:** verify the JWT with `supabase.auth.getUser()`, then return 403 unless the user's email equals `Deno.env.get("ALLOWED_USER_EMAIL")` (case-insensitive). Never trust a `user_id` from the request body.
4. **Pure core modules.** Routing, complexity scoring, cost estimation, token estimation and context building live in `supabase/functions/_shared/core/*.ts`. They are pure TypeScript with **no imports** from Deno, URLs, npm specifiers or Supabase, so that Vitest can test them from `src/test/`.
5. **Provider adapter.** Only `supabase/functions/_shared/providers/openrouter.ts` may call `https://openrouter.ai/api/v1/...`.
6. **Sensitive projects.** When `projects.is_sensitive` is true, every OpenRouter call for that project includes `provider: { data_collection: "deny" }`, and no `prompt_logs` row is written.
7. **Cost comes from OpenRouter.** Actual cost is `usage.cost` from the response (for streams, the final SSE chunk). Estimates are used only for the pre-send guard.
8. **Settings are data.** Thresholds and limits come from `app_settings`. Don't hard-code them.
9. **Don't remove or rename existing columns or tables** unless the prompt says so. Add migrations; don't rewrite old ones.
10. **Build only what the current prompt asks for.** No speculative features, no mock data in production code paths, no placeholder API keys.

### Stack
React + Vite + TypeScript + Tailwind + shadcn/ui, TanStack Query, react-markdown with syntax highlighting, Lovable Cloud (Supabase: Postgres + pgvector, Auth, Storage, Edge Functions in Deno), Vitest.

### Style
- Desktop-only layout, clean and dense, light and dark mode. The left sidebar holds projects and recent chats, and the main pane holds the chat.
- Money is shown to 4 decimals below $0.01 and to 2 decimals above. Tokens use a `k` suffix.
- Every Auto-routed assistant message gets a compact routing card (Phase 2+).
- Components go in `src/components/<area>/`, hooks in `src/hooks/`, helpers in `src/lib/`.

### Key numbers (defaults)
Monthly cap $10 · confirmation above $0.25 · warning at 80% · context budget 4,000 tokens · summarize above 6,000 tokens · extract every 16 messages · prompt logs kept 30 days · classifier timeout 4s · file limit 10 MB · agent limits: 10 steps / $0.50.
