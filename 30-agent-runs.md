# 30: Agent runs with guardrails

**Phase 5 · Depends on: 29**

---

Read `docs/DECISIONS.md` D27, `docs/SECURITY.md` §7, and `docs/PRD.md` US-21.

1. **Start a run:** a composer toggle "🤖 Run as task" (or the `/task` prefix). This creates an `agent_runs` row (goal = the message, `max_steps` 10, `max_cost_usd` 0.50).
2. **Loop in an `agent` edge function:** plan (one model call producing a short numbered plan, stored as a `plan` step) → execute steps with the tool loop → each model call, tool call and tool result becomes an `agent_steps` row with its cost → stop at the final answer, `max_steps`, `max_cost_usd` (`limit_reached`, with a summary of progress so far), or a user Stop. Write tools pause the run (`awaiting_approval`) using the approval mechanics from prompt 29.
3. **Routing:** the planning step uses Auto with at least T2 (it may ask for T3). Execution steps use Auto normally. Everything counts toward the monthly cap.
4. **Live UI:** a run card in the chat showing the plan checklist, the current step, steps used out of 10, cost out of $0.50, and Stop. The project's **Activity** tab lists runs, and each run expands to show every step with its inputs and outputs.
5. The final answer is posted as an assistant message linked to the run.

---

### Verify (don't paste)
- [ ] The "Research X and save key points to memory" test in ROADMAP.md Phase 5.
- [ ] Set `max_cost_usd` to $0.01, and the run stops cleanly with `limit_reached`.
