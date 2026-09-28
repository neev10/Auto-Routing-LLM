# 13: Routing card, Auto picker and feedback

**Phase 2 · Depends on: 12**

---

Make routing visible. Read `docs/ROUTING.md` §9, and `docs/PRD.md` US-4 and US-7.

1. **ModelPicker:** "⚡ Auto" is now enabled, and is the default. Picking a model switches to manual (sticky), and picking Auto switches back (`mode='auto'`, clear `sticky_model_id`). Show the current mode clearly in the chat header.
2. **RoutingCard** under every assistant message (compact, one line): `⚡ Auto → {model} · T{n} · {complexity} · {method} · ${cost} · ctx {k}`. Manual messages show `✋ Manual → {model} · ${cost}`. Add a ↪ marker if there was a fallback or an escalation.
3. **Popover** on click: the reason sentence, the signals, the classifier result and its latency, the candidates (✓ chosen, ✗ skipped with the reason), the fallback chain, estimated vs. actual cost, context tokens, and a "View prompt log" link.
4. **Feedback:** 👍 and 👎 on the card. 👎 opens an optional one-line note. Upsert into `feedback` with `routing_decision_id`. The selected state stays visible.
5. **"Try premium?" chip:** when the decision was `high` and the tier chosen was below 3, show a small chip that resends the request with `forceTier: 3` (which goes through the T3 confirmation).
6. **Tier-3 confirmation dialog:** "This looks complex. Use {T3 model} (~$x.xx) or stay on Tier 2?"

---

### Verify (don't paste)
- [ ] Phase 2 acceptance items 4 and 6.
