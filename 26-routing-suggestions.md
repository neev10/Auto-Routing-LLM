# 26: Routing suggestions

**Phase 4 · Depends on: 25**

---

Close the loop, keeping a person in charge. Read `docs/ROUTING.md` §8 and §11.

1. A pure function `suggestTierChanges(stats, tiers, catalog)`, with tests, that returns suggestions like:
   - "Model X: 40% 👎 on coding over 15 ratings → move below Y, or remove the `coding` tag."
   - "Model Z hits rate limits 30% of the time → move it down in T1."
   - "T2 is used for 60% of medium requests under Balanced, but 👍 is the same as T1 → consider Cost priority for project P."
   - "Free model W left the catalog → replace it in T1."
   Only suggest when there are at least 10 data points.
2. **Settings → Routing & Tiers:** a "Suggestions (n)" panel. Each suggestion has **Apply** (it edits `tier_models` or project settings) and **Dismiss**. Applying one also appends a line to the Activity log.
3. Add a "Classifier confidence" figure to the Routing tab. If the classifier and later feedback disagree often, suggest trying a different classifier model.

---

### Verify (don't paste)
- [ ] Run the Phase 4 checklist, and update DECISIONS.md.
