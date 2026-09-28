# 24: Estimated savings

**Phase 4 · Depends on: 23**

---

Read `docs/PRD.md` FR-29 and `docs/DATABASE.md` (`app_settings.baseline_model_id`).

1. **Settings → Limits:** a **Baseline model for savings** selector (suggest the Claude Sonnet-class model from your T3 or T2 list).
2. **Savings calculation** (pure function in `_shared/core/cost.ts`, with a unit test): for each `purpose='chat'` call, `baselineCost = prompt_tokens/1e6 × baseline_in + completion_tokens/1e6 × baseline_out`. Savings = Σ baselineCost − Σ actual cost (including classifier, extraction and summary costs, which are usually 0). Use today's baseline prices, and say so.
3. **A dashboard tile:** "If everything had gone to {baseline}: $X · Actual: $Y · Estimated savings: $Z", with a tooltip explaining the method and its limits (it assumes the same number of output tokens).
4. Also show a month-over-month comparison with the $20/month subscription as a reference line.

---

### Verify (don't paste)
- [ ] Recompute one month by hand from the Logs tab, and it matches.
