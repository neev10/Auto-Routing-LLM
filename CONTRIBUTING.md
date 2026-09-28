# Contributing

This is a personal project whose code is public. Forks and pull requests are welcome, but the roadmap follows its owner's daily use.

## Ground rules

1. **The spec comes first.** If a change contradicts `docs/`, update the doc (and `docs/DECISIONS.md`) in the same PR.
2. **Keep the core pure.** `supabase/functions/_shared/core/` must not import Deno, npm URLs, Supabase or `fetch`. Everything it needs comes in as arguments. This is what lets Vitest test it.
3. **The browser never talks to an LLM provider.** Every call goes through an edge function and the `ProviderAdapter`.
4. **No secrets in code, docs, issues or chat prompts.** Use Lovable Cloud → Secrets.
5. **RLS on every new table.** Tables written only by functions get read-only policies.
6. **Settings are data.** New thresholds go in `app_settings` with a sensible default. Don't hard-code them.
7. **Every Auto decision must be explainable** from its `routing_decisions` row.

## Working with Lovable

- Use the prompts in `docs/lovable-prompts/` **one at a time**. Check each result against the prompt's **Verify** list before moving on.
- If Lovable drifts (for example it calls OpenRouter from the browser, skips RLS, or merges files), reply with the specific rule from `00-project-knowledge.md` and ask it to fix only that.
- Small, focused follow-up prompts work better than big ones. When something breaks, ask Lovable to *explain* the bug before it fixes it.
- Changes made locally, in Cursor, Claude Code or the GitHub UI, sync back to Lovable through GitHub. Pull before prompting Lovable, so the two don't diverge.

## Branches and commits

- `main` is deployed by Lovable. For bigger changes, work on a branch and open a PR. CI (`npm test`) must pass.
- Commit messages: `feat(routing): …`, `fix(memory): …`, `docs: …`, `test: …`.

## Adding a model provider (future)

1. Implement `ProviderAdapter` in `_shared/providers/<name>.ts`.
2. Add catalog normalization so the provider's models map to the `models` columns. Set `provider_adapter`.
3. Add the secret name to `docs/SETUP.md`.
4. Add the provider to the adapter registry. The router doesn't change.

## Adding a tool (Phase 5+)

1. Define a `ToolSpec` (name, description, JSON schema, `isWrite`).
2. Implement the handler. Treat all external content as untrusted.
3. Register it, and add it to the `project_tools` defaults (off by default when `isWrite`).
4. Add a unit test for input validation.

## Reporting security issues

Please don't open a public issue. Contact the maintainer privately through the email on their GitHub profile.
