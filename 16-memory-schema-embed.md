# 16: Memory schema and the embed function

**Phase 3 · Depends on: Phase 2 complete**

---

Start Phase 3 (project memory). Read `docs/MEMORY.md` §1 and §2.4, and `docs/DATABASE.md` "Phase 3 tables".

1. **Migration:** create `memory_items` and `chunks` with the HNSW indexes, and the `match_memory` and `match_chunks` SQL functions exactly as in the doc. RLS: `memory_items` gets owner full access; `chunks` gets owner select only.
2. **`embed` edge function** (also exported as a helper in `_shared/embed.ts` for other functions to import): `const session = new Supabase.ai.Session('gte-small')`; `embed(texts: string[]) → number[][]` using `session.run(text, {mean_pool: true, normalize: true})`. Truncate each input to about 2,000 characters (the model's limit is 512 tokens). The HTTP endpoint (owner-checked) takes `{texts}` and returns `{embeddings}`.
3. A small **health check** in Settings → Memory (a new tab): "Embeddings: OK (384 dims)", done by embedding the word "test".

---

### Verify (don't paste)
- [ ] The health check shows 384 dims.
