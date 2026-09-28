# Setup Guide

Time: about 20 minutes before the first Lovable prompt.

## 1. OpenRouter

1. Create an account at openrouter.ai.
2. **Buy $10 of credits (once, recommended).** This raises the free-model limit from 50 to 1,000 requests/day for good, and the same $10 funds your paid-model budget.
3. Go to **Keys → Create key**, name it `switchyard`, and set a **credit limit of $10** (with a monthly reset if offered). Copy the key; it starts with `sk-or-`.
4. Optional: in **Settings → Privacy**, review the data-training and logging settings for the whole account.

## 2. Lovable project

1. Create a new Lovable project named `switchyard`.
2. Make sure **Lovable Cloud** is enabled. It is the default and provides the Supabase backend.
3. Connect **GitHub** (Settings → GitHub → Connect). Lovable creates a repo and syncs both ways.
4. Clone the repo locally, or use the GitHub web UI, and add:
   - `docs/` (this folder)
   - `CONTRIBUTING.md`
   - `LICENSE`
   - replace Lovable's `README.md` with the one from this spec
   Commit and push. Lovable pulls the files in, and they become readable to Lovable's agent.
5. In Lovable, open **Project settings → Knowledge** and paste the contents of [`lovable-prompts/00-project-knowledge.md`](lovable-prompts/00-project-knowledge.md).

## 3. Secrets (Lovable → Cloud → Secrets)

| Name | Value | Used by |
|---|---|---|
| `OPENROUTER_API_KEY` | your `sk-or-…` key | all LLM functions |
| `ALLOWED_USER_EMAIL` | the email address you'll log in with | every function (owner check) |
| `ALLOWED_ORIGINS` | comma-separated app URLs, e.g. `https://switchyard.lovable.app,https://id-preview--….lovable.app` | CORS |
| `APP_PUBLIC_URL` | your app URL | OpenRouter `HTTP-Referer` attribution header |
| `TAVILY_API_KEY` | *(Phase 5)* | the web search tool |
| `GITHUB_TOKEN` | *(Phase 5, fine-grained, read-only)* | GitHub knowledge |

`SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` are provided to functions automatically by Lovable Cloud.

**Frontend variables** (`.env`, safe to expose): `VITE_SUPABASE_URL` and `VITE_SUPABASE_PUBLISHABLE_KEY`. Lovable manages these.

`.env.example` (committed):

```
# Frontend (public)
VITE_SUPABASE_URL=
VITE_SUPABASE_PUBLISHABLE_KEY=
# Backend secrets — set in Lovable Cloud → Secrets, never here
# OPENROUTER_API_KEY, ALLOWED_USER_EMAIL, ALLOWED_ORIGINS, APP_PUBLIC_URL
# Phase 5: TAVILY_API_KEY, GITHUB_TOKEN
```

## 4. First run

1. Run Phase 1 prompts 01–02 (schema and auth).
2. Sign up **once** with the `ALLOWED_USER_EMAIL` address.
3. **Disable sign-ups** in Lovable Cloud → Users/Auth settings. If you can't find the toggle, ask Lovable: *"Disable new user sign-ups in auth settings."*
4. Continue with the remaining Phase 1 prompts.
5. In Settings, click **Test connection**. You should see your key's limit and usage.
6. Click **Refresh models**. The catalog should show a few hundred models.

## 5. Running tests locally (optional)

```bash
npm ci
npm test          # Vitest: router + context builder + cost guard
```

The tests also run on GitHub Actions on every push (`.github/workflows/test.yml`), which is free for public repos.

## 6. Troubleshooting

| Symptom | Likely cause |
|---|---|
| 401 from functions | Not logged in, or the JWT expired → reload |
| 403 from functions | `ALLOWED_USER_EMAIL` doesn't match the login email |
| All free models fail with 429 | The daily free limit was reached (50 or 1,000/day) → wait until UTC midnight, or buy credits |
| "No private provider available" | The project is Sensitive, and those models have no providers that don't store data → pick other models for the tiers |
| 402 errors | OpenRouter credits exhausted, or the key limit reached |
| Streaming stops at a fixed length | Edge function wall-clock limit → lower `max_tokens` |
