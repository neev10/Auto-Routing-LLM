# 03: Shared core, OpenRouter adapter, model sync and test connection

**Phase 1 · Depends on: 02**

---

Create the backend foundation. Read `docs/ARCHITECTURE.md` §2.2–2.4, `docs/SECURITY.md` §2 and §6, and `docs/DATABASE.md` (the `models` table).

**A. Pure core** in `supabase/functions/_shared/core/` (no imports other than relative imports between these files):
- `types.ts`: `CatalogModel`, `ChatMessage`, `ChatRequest`, `StreamEvent`, `Usage`.
- `tokens.ts`: `estimateTokens(text) = Math.ceil(text.length / 4)`, and `estimateMessagesTokens(messages)`.
- `cost.ts`: `estimateCost(model, promptTokens, maxOutputTokens)` using `expectedOutput = min(maxOutputTokens, 1000)` and per-million prices; `capCheck({monthSpend, cap, warnPct, estimate, isPaid, confirmThreshold, confirmed})` returns `{allowed, needsConfirmation?: 'cost'|'cap_reached', warn: boolean, blockPaid: boolean}`. Follow `docs/ROUTING.md` §7 exactly.

**B. Adapter** `supabase/functions/_shared/providers/`:
- `adapter.ts`: the `ProviderAdapter` interface from ARCHITECTURE.md §2.4.
- `openrouter.ts`: `listModels()` (GET `/api/v1/models`, normalized into `models` columns: prices ×1e6 for per-million, `is_free` when both prices are 0 or the id ends with `:free`, capability flags from `supported_parameters` and `architecture.input_modalities`, and `vendor` from the id prefix); `chat()` (non-streaming); `chatStream()` (streaming SSE parser that yields `{type:'delta', text}` and a final `{type:'usage', usage, generationId, provider}`); `keyInfo()` (GET `/api/v1/key`). Add `provider: {data_collection: "deny"}` when `sensitive` is true. Send the `HTTP-Referer: APP_PUBLIC_URL` and `X-Title: Switchyard` headers.

**C. Helpers** in `_shared/`: `auth.ts` (`requireOwner(req)` per the Knowledge rules), `db.ts` (service-role client), `cors.ts` (origins from `ALLOWED_ORIGINS`), and `sse.ts`.

**D. Edge functions:**
- `models-sync`: owner check → `listModels()` → upsert all models; set `is_active=false` on models that weren't returned; set `app_settings.models_synced_at`. Returns `{count}`.
- `test-connection`: owner check → `keyInfo()` → return only `{ok, limit, usage, usage_monthly, is_free_tier}`. Never return the key.

**E. Tests:** add Vitest, and an `npm test` script. Add `src/test/cost.test.ts` covering the cases in `docs/TESTING.md` §2.1 and §2.4. Add `.github/workflows/test.yml` that runs `npm ci && npm test` on push and pull request.

I've added the secrets `OPENROUTER_API_KEY`, `ALLOWED_USER_EMAIL`, `ALLOWED_ORIGINS` and `APP_PUBLIC_URL` in Cloud → Secrets.

---

### Verify (don't paste)
- [ ] `npm test` passes, and GitHub Actions is green.
- [ ] Searching the code for `openrouter.ai` finds it only in `providers/openrouter.ts`.
- [ ] Calling `test-connection` without logging in returns 401.
