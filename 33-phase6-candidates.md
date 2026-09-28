# 33: Phase 6 candidates (pick one at a time)

**Phase 6 · Exploratory.** Each section is its own prompt. Read `docs/ROUTING.md` §11 first.

---

## 33a: Feedback-weighted routing
Add `learnedRank(tierModels, stats)` in `_shared/core/router.ts` (pure, with tests): within a tier, reorder models for a (taskType, complexity) pair by `thumbsUpRate − λ × normalizedCost`, using only models with ≥10 ratings. Add a Settings toggle "Learn from my feedback" (default off), and show "learned" as the method detail on the routing card.

---

## 33b: Critic pass
For projects with a new setting "Double-check high-stakes answers", after a `high` answer, run a cheaper critic model with the prompt "List factual errors, missing steps, or risky advice. Reply NONE if fine." If the critic finds issues, show a collapsible "Reviewer notes" under the answer. The critic's cost is logged with `purpose='agent'`.

---

## 33c: Scheduled model benchmark
A "Benchmark" page: pick candidate models, run the routing-eval prompts plus a 10-question quality set (stored in `docs/fixtures/quality-set.json`, which you create) on each, score them with a judge model, and show a table of cost, latency and score, with an "Add to tier" button. Run it manually. Scheduling it is optional.

---

### Verify (don't paste)
- [ ] Each feature is behind a toggle, off by default, and documented in DECISIONS.md when adopted.
