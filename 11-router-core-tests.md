# 11: Complexity rules and router (pure core, with tests)

**Phase 2 · Depends on: 10**

---

Implement the routing brain as pure functions. Read **all of** `docs/ROUTING.md`, and `docs/TESTING.md` §2.2, §2.3 and §3. The fixture is `docs/fixtures/routing-test-set.json`.

1. `_shared/core/complexity.ts`: `scoreComplexity({text, attachmentTokens, previousTier})` returns `{score, band: 'low'|'uncertain'|'high', confident, taskType, signals: string[]}`. Implement the table in ROUTING.md §4.1 **exactly**, including word-boundary keyword matching and the taskType order. Export the regexes as constants.
2. `_shared/core/router.ts`: `selectModel(input)`, where `input` is `{mode, overrideModelId?, stickyModelId?, projectDefaultModelId?, retryStronger?, previousTier?, priority, complexity: {band, score, taskType, signals}, classifierResult?, requirements: {vision, minContext, tools, sensitive}, catalog: CatalogModel[], tiers: {tier, modelId, rank, tags}[], health: {modelId, cooldownUntil}[], now, capBlocked}`. It returns a `RoutingDecision` (`method`, `tier`, `complexity`, `selectedModelId`, `candidates`, `escalated`, `reason`), or `{needsConfirmation: 'tier3', modelId}`, or `{error: code, reasons}`. Implement §3, §5 and §6.1, including the vision auto-fill and the cap behavior.
3. `classifierFallback(score)`: 2 → low, 3–4 → medium. Add `parseClassifierOutput(text)` to parse and validate the classifier's JSON reply (returns null if invalid).
4. **Tests** in `src/test/`:
   - `complexity.test.ts`: every fixture case's `expectScore`, `expectBand` and `expectTaskType`. Build the message text exactly as the fixture's `description` field says.
   - `router.test.ts`: using the fixture's `testCatalog`, every case's `expectTier` for all three priorities (mock the classifier with `expectComplexity`), plus every bullet in TESTING.md §2.3.
   - Must cases must pass. Report how many of the 30 pass.

Don't wire this into `chat` yet.

---

### Verify (don't paste)
- [ ] `npm test` is green, and 30/30 pass against the fixture (the rules are deterministic, so all 30 should pass).
