# 23: Usage dashboard

**Phase 4 · Depends on: Phase 3 complete**

---

Start Phase 4. Read `docs/PRD.md` FR-29 and §3 (metrics), and `docs/DATABASE.md` "Phase 4 views".

1. **Migration:** create the `v_usage_daily` and `v_month_spend` views, with `security_invoker = on`.
2. **`/usage` page** (in the sidebar under Settings):
   - Month selector. KPI tiles: total spend, spend vs. cap, number of requests, % served by free models (target ≥70%), and median context tokens.
   - Spend per day (a bar chart), stacked or split by model.
   - Spend by model: a table plus a bar chart, including free models with their request counts (matching the "September AI Usage" layout in the original brief: Claude $x, GPT $x, Gemini $x, other paid $x, free: N requests, total $x). Group by `vendor`.
   - Spend by project and by purpose (chat, classifier, extraction, summary…).
3. Use Recharts (a shadcn chart). Keep it readable; no 3D, no pie charts with more than 5 slices.

---

### Verify (don't paste)
- [ ] The monthly total matches the OpenRouter activity page within ±5%.
