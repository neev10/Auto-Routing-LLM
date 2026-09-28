# 14: Retry stronger and the routing eval

**Phase 2 · Depends on: 13**

---

Read `docs/ROUTING.md` §3 (retry_stronger) and `docs/TESTING.md` §3 ("Live routing evaluation").

1. **⤴ Retry with a stronger model** on the routing card: it regenerates the message with `retryStronger: true`. The router picks the tier one above the previous decision's tier (T3 still asks first), with `method='retry_stronger'`. The old answer is superseded but can still be reached through a "previous versions (n)" toggle on the message.
2. **Settings → Routing → Run routing eval:** loads `docs/fixtures/routing-test-set.json` (bundle it as a static import), builds each case's text, calls `chat` with `dryRun: true` against a throwaway conversation in General (created and deleted by the eval, or flagged as hidden), and shows a table comparing expected and actual values (band, complexity, tier under the current priority), with a pass count. Skip cases that need special state (26–30), and mark them "unit-tested only".
3. Show the classifier calls the eval used ("used N free requests").

---

### Verify (don't paste)
- [ ] The eval shows ≥ 18/25 matches for the live-runnable cases. The classifier decides the uncertain ones, so some variation is normal. Note any misses for tuning.
