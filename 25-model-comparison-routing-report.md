# 25: Model comparison and routing report

**Phase 4 · Depends on: 24**

---

Read `docs/PRD.md` FR-30 and FR-31, and `docs/ROUTING.md` §11.

1. **Usage → Models tab:** a table per model with calls, total cost, average cost per call, average latency and TTFT, error or rate-limit rate, 👍 rate (from `feedback`), and the tiers it belongs to. Filter by `taskType` and `complexity` (join through `routing_decisions`).
2. **Usage → Routing tab:**
   - The method mix (rules vs. classifier vs. manual…), the classifier share, and the average classifier latency.
   - The escalation rate and the most common fallback reasons.
   - 👍 rate by tier and by complexity band.
   - Manual overrides after Auto: count the cases where the user switched model or used Retry stronger right after an Auto answer. These are "the router picked badly" signals. List the last 20, each with a link to its message.
3. **Export CSV** for both tables.

---

### Verify (don't paste)
- [ ] The numbers make sense against the Logs tab.
