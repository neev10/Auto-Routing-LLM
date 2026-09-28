# 28: Web search and fetch URL tools

**Phase 5 · Depends on: 27**

---

Read `docs/DECISIONS.md` D25 and `docs/SECURITY.md` §7.

1. **`web_search`** (read): `{query, max_results ≤ 5}` → calls the Tavily API with `TAVILY_API_KEY` (I'll add this secret), and returns titles, URLs and snippets. Fallback: if Tavily isn't configured or fails, and the user has allowed paid search in Settings, use OpenRouter's web search feature on the next model call instead (charged per request; show the estimate).
2. **`fetch_url`** (read): `{url}` → fetch with a 10s timeout and a 2 MB cap, extract readable text (strip scripts, styles and nav; use a small readability-style extractor), and truncate to 8k tokens. **Block** non-http(s) schemes, `localhost`, private and link-local IP ranges (resolve DNS first where possible), and cloud metadata IPs.
3. Citations: when an answer uses search or fetch results, the model is told to cite URLs, and the UI renders them as a "Sources" list under the message.
4. **Settings → Tools:** the search provider status and the "Allow paid search fallback" toggle (default off).

---

### Verify (don't paste)
- [ ] "What changed in OpenRouter's free limits recently?" searches, answers and lists its sources.
- [ ] `fetch_url` on `http://169.254.169.254` is blocked.
