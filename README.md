# Switchyard — a personal BYOK multi-LLM workspace

> *Working name. Rename freely.* A switchyard is where trains get routed to the right track — which is what this app does with your requests.

Switchyard is a lightweight, web-based AI workspace that uses **your own OpenRouter key** to talk to hundreds of models. It has three ideas at its core:

1. **Auto routing.** Simple requests go to free models. Harder ones escalate to paid models only when they need to, and the app always shows you which model it picked and why.
2. **Projects with model-independent memory.** Each project has a Brief, typed memory items, rolling conversation summaries and searchable knowledge. The memory belongs to the app, not to any model. You can switch from Claude to Gemini to a free Qwen model in the middle of a conversation, and the new model picks up where the last one stopped.
3. **Cost you can see.** Every call is logged with its tokens and its real cost. There is a monthly cap, a confirmation step for expensive requests, and a dashboard that estimates how much you saved.

> **The model is replaceable. The project, context, memory, tools and data belong to the application.**

**Status:** Phase 0 (specification) is complete. The build happens in phases with Lovable. See [docs/ROADMAP.md](docs/ROADMAP.md).

---

## Stack

| Layer | Choice |
|---|---|
| Frontend | React + Vite + TypeScript + Tailwind + shadcn/ui (Lovable default) |
| Backend | Lovable Cloud (built on Supabase): Postgres, Auth, Storage, Edge Functions (Deno) |
| Vector search | pgvector + `gte-small` embeddings (384-dim), generated for free inside Edge Functions |
| LLM gateway | OpenRouter (the only provider in the MVP, behind a provider adapter) |
| Hosting | Lovable (hosting + deploy), with GitHub as the source of truth for code |
| License | MIT |

## Documentation map

| Doc | What it answers |
|---|---|
| [docs/PRD.md](docs/PRD.md) | What we're building, for whom, and what "done" means |
| [docs/DECISIONS.md](docs/DECISIONS.md) | Every decision made in Phase 0, with the reasoning behind it |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Components, request flow, edge functions, repo structure |
| [docs/DATABASE.md](docs/DATABASE.md) | Full schema, RLS, indexes, SQL |
| [docs/MEMORY.md](docs/MEMORY.md) | The memory layers, context builder, summarization, extraction |
| [docs/ROUTING.md](docs/ROUTING.md) | Complexity rules, classifier, tiers, fallback, cost guards |
| [docs/SECURITY.md](docs/SECURITY.md) | Keys, auth, RLS, privacy routing, threat model |
| [docs/SETUP.md](docs/SETUP.md) | Environment variables and step-by-step setup |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Phases 1–6 with scope and acceptance criteria |
| [docs/TESTING.md](docs/TESTING.md) | Unit tests, the routing test set, and per-phase manual checklists |
| [docs/lovable-prompts/](docs/lovable-prompts/) | Small, ordered prompts to paste into Lovable, one step at a time |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to change things without breaking them |

## Quick start

1. Read [docs/SETUP.md](docs/SETUP.md). It takes about 20 minutes.
2. Create the Lovable project and connect it to GitHub.
3. Commit this `docs/` folder, this README and `LICENSE` to the repo.
4. In Lovable, paste [`docs/lovable-prompts/00-project-knowledge.md`](docs/lovable-prompts/00-project-knowledge.md) into **Project Knowledge**.
5. Run the Phase 1 prompts in order. After each prompt, check the result against its "Verify" list before moving on.

## Guiding principle

**Build something useful quickly, use it every day, learn what's missing, then improve the architecture.** Each phase has to leave the app usable. If a feature isn't needed until a later phase, it waits for that phase.

## License

MIT. See [LICENSE](LICENSE).
