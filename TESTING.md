# Testing Plan

There are three layers of tests:

1. **Automated unit tests** (Vitest, run in CI) for the pure core modules: complexity, router, cost and context.
2. **The routing test set**: 30 prompts with expected outcomes. The unit tests check them against the rules deterministically, and a live evaluation checks them end-to-end.
3. **Manual checklists** at the end of each phase, since Lovable writes the code and a person confirms the behavior.

## 1. Setup

- Vitest runs in the frontend toolchain. Tests live in `src/test/` and import `supabase/functions/_shared/core/*.ts` directly. Those files must stay free of Deno-specific or URL imports (ARCHITECTURE.md rule 3).
- CI: `.github/workflows/test.yml` runs `npm ci && npm test` on every push and pull request.
- The fixture is [`docs/fixtures/routing-test-set.json`](fixtures/routing-test-set.json).

## 2. Unit tests

### 2.1 `tokens.ts`
- `estimateTokens('')` = 0. `estimateTokens('abcd'.repeat(100))` = 100.

### 2.2 `complexity.ts` (`scoreComplexity`)
- For every fixture case: `score` and `band` match exactly, and `taskType` matches when it's specified.
- High keywords count once, even if several appear. The same applies to low keywords.
- Code detection: a ``` block gives +1, three code-like lines give +1, and ordinary prose gives 0.
- Attachment thresholds: 4,000 tokens gives +0, 4,001 gives +1, and 20,001 gives +2.

### 2.3 `router.ts` (`selectModel`)
Use a fixed catalog: T1 = `free-a` (tags: light), `free-b` (strong), `free-coder` (coding), `free-vision` (vision); T2 = `mid-a`, `mid-coder` (coding); T3 = `prem-a`. `free-vision` also has `supports_vision=true`; the others don't, except `mid-a` and `prem-a`.

- Precedence: a message override beats sticky, which beats the project default, which beats Auto.
- Every fixture case, with the classifier **mocked** to return `expectComplexity`, gives the expected tier for Cost, Balanced and Performance.
- `taskType=coding` in T1 picks `free-coder`. `medium` under Cost picks `free-b`.
- Vision required → `free-vision`. Vision required with a manual pick of a non-vision model → error `model_lacks_vision`.
- `minContext` larger than every T1 model's context → escalates to T2, with a reason that mentions context.
- Every T1 model in cooldown → T2, with `escalated=true`.
- Cap reached → only free models are candidates, the tier is forced to T1, and the reason mentions the cap.
- High under Performance → the result is `needs_confirmation: tier3`, and never a T3 model selected directly.
- A classifier timeout or bad JSON → the rules' fallback mapping is used, and `signals.classifier='failed'`.
- A Sensitive request → `requirements.sensitive=true` is passed through to the chat request.
- The `reason` string is never empty.

### 2.4 `cost.ts`
- The estimate for a free model is 0.
- For a paid model with 1M prompt tokens at $3/M and 1,000 expected output tokens at $15/M, the estimate is $3.015.
- Cap checks: spend 9.99 with an estimate of 0.02 → `cap_reached` confirmation. Spend 10.00 → paid models blocked. Estimate 0.26 → `cost` confirmation, unless `confirm=true`.
- The 80% warning flag.

### 2.5 `context.ts` (`buildContext`)
- Block order matches MEMORY.md §2.3.
- Each block's cap is respected. The total stays within the budget, not counting block 9.
- Trimming order: chunks → unpinned memory → recent messages (down to the last 2 exchanges) → summary → brief → instructions. Pinned items and About Me are never dropped.
- Budget clamp: a model with an 8k context gets a budget of 2,000; one with 200k gets 4,000 by default and 8,000 when premium.
- The report's per-block token counts add up to the total.
- An empty memory and summary (Phase 1 mode) gives only the preamble, About Me, instructions and recent messages.
- Messages with `superseded_at` set are never included.

### 2.6 `memoryMerge.ts`
- Similarity ≥ 0.92 → update instead of insert.
- `supersedes` sets `superseded_by` on the old item. `resolves` sets the status to resolved.

## 3. The routing test set

Columns:
- **Rules** = the expected `score` → `band`. "?" means the classifier decides.
- **Expect** = the final complexity (the mocked classifier output for "?" cases).
- **Cost / Bal / Perf** = the expected tier for each priority. "T3?" means `needs_confirmation(tier3)`.
- **Must** cases must pass. The rest must pass at ≥ 27/30 overall.

`+Ntok` = filler prose appended to reach N estimated tokens (with no keywords). `+code(N)` = a fenced code block of N neutral lines.

| # | Prompt (abbreviated) | Rules | Expect | Cost | Bal | Perf | Must | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | "Rewrite this sentence to sound more professional: 'we cant make the meeting tmrw'" | −1 → low | low | T1 | T1 | T2 | ✓ | |
| 2 | "What is a Roth IRA?" | −1 → low | low | T1 | T1 | T2 | ✓ | |
| 3 | "Translate 'good morning, how are you' into Nepali." | −1 → low | low | T1 | T1 | T2 | | |
| 4 | "Summarize this in 3 bullets:" +200tok | 0 → low | low | T1 | T1 | T2 | | |
| 5 | "Give me a title for a blog post about AI model routing." | −1 → low | low | T1 | T1 | T2 | | |
| 6 | "Fix the grammar: their going to the store tomorow." | −1 → low | low | T1 | T1 | T2 | | |
| 7 | "Define idempotency in one sentence." | −1 → low | low | T1 | T1 | T2 | | |
| 8 | "Suggest 5 names for a business brokerage firm." | 0 → low | low | T1 | T1 | T2 | | |
| 9 | "Write a Python function that reverses a string." | 0 → low | low | T1 | T1 | T2 | | taskType=coding → free-coder |
| 10 | "Draft a short thank-you email to my recruiter." | 0 → low | low | T1 | T1 | T2 | | taskType=writing |
| 11 | "Debug this, why does it return undefined?" +code(8) +100tok total | 4 → ? | medium | T1 | T2 | T2 | | coding |
| 12 | "Compare Supabase vs Firebase for a solo developer building an AI app." | 2 → ? | medium | T1 | T2 | T2 | | |
| 13 | "What are the trade-offs between pgvector and Pinecone?" | 2 → ? | medium | T1 | T2 | T2 | | |
| 14 | "Refactor this function to be more readable." +code(15) | 4 → ? | medium | T1 | T2 | T2 | | coding |
| 15 | "Research the latest OpenRouter free model limits." | 1 → low | low | T1 | T1 | T2 | | taskType=research (a rules "under-call"; Phase 5 search helps) |
| 16 | "Create an implementation plan for adding OAuth to my app." | 2 → ? | medium | T1 | T2 | T2 | | |
| 17 | "Summarize the attached report." + attachment 15,000tok | 0 → low | low | T1 | T1 | T2 | ✓ | minContext ≥ ~19k; T1 model must fit |
| 18 | "Write a cover letter for a Director of Technology Transformation role using my resume below." +600tok | 2 → ? | medium | T1 | T2 | T2 | | writing |
| 19 | "Analyze these quarterly numbers and tell me what stands out." +400tok | 2 → ? | medium | T1 | T2 | T2 | | analysis |
| 20 | "Optimize this SQL query." +code(10) | 4 → ? | medium | T1 | T2 | T2 | | coding |
| 21 | "Design a system architecture for a multi-tenant SaaS…" +1600tok | 5 → high | high | T2 | T2 | T3? | ✓ | Never T3 without asking |
| 22 | "Debug this stack trace and find the root cause." +code(30) +400tok total | 5 → high | high | T2 | T2 | T3? | | coding → mid-coder |
| 23 | "Refactor this module." +code(150) +1800tok total | 6 → high | high | T2 | T2 | T3? | | coding |
| 24 | (previous tier T2) "Compare these three approaches in depth." +350tok | 5 → high | high | T2 | T2 | T3? | | +1 from the previous tier |
| 25 | "Prove that the algorithm below terminates and derive its complexity." +code(8) +100tok total | 4 → ? | high | T2 | T2 | T3? | | classifier → high |
| 26 | [image attached] "What's in this screenshot?" | 0 → low | low | T1 (free-vision) | T1 (free-vision) | T2 (mid-a) | ✓ | vision filter |
| 27 | [Sensitive project] "Rewrite my salary negotiation email to sound confident." | −1 → low | low | T1 | T1 | T2 | ✓ | `sensitive=true` passed through |
| 28 | [sticky model = mid-a] "Rewrite this sentence." | n/a | n/a | mid-a | mid-a | mid-a | ✓ | method=sticky, no classifier call |
| 29 | [spend = cap] content of #21 | 5 → high | high | T1 | T1 | T1 | ✓ | Forced free; reason mentions the cap |
| 30 | [all T1 in cooldown] content of #1 | −1 → low | low | T2 | T2 | T2 | ✓ | escalated=true |

### Live routing evaluation (manual, after Phase 2 and whenever tiers change)

1. Settings → Routing → **Run routing eval** (a Phase 2 feature). It sends each fixture's text to a `route-preview` mode of `chat` (`dryRun: true`, which routes without generating) and shows the actual band, complexity and tier next to the expected ones.
2. The target is ≥ 27/30 matches under Cost. Record the result in the Activity tab. Don't overfit the keyword lists to this set; add new cases from real mis-routes instead.

`dryRun` does call the classifier for uncertain cases (about 11 free calls per run).

## 4. Manual checklists per phase

Use the acceptance criteria in [ROADMAP.md](ROADMAP.md) for each phase, plus:

**Every phase**
- [ ] SECURITY.md §9 checklist
- [ ] CI is green
- [ ] No console errors on the main screens
- [ ] Exporting a project still works (the schema may have changed)
- [ ] DECISIONS.md is updated if anything changed

**Phase 1 extras**
- [ ] Streaming renders Markdown tables and code blocks with a copy button
- [ ] Editing the last user message hides the old exchange and generates a new one
- [ ] A free-model 429 shows a clear message (Auto fallback arrives in Phase 2)
- [ ] The model catalog refreshes automatically after 24h (set `models_synced_at` back in time to test)

**Phase 3 extras**
- [ ] The model-switching test: two conversations about a made-up project fact in project A using model X; a new conversation using model Y; asking "remind me what we decided" gets it right
- [ ] The token budget test: the context inspector total is ≤ budget + block 9

## 5. Regression habit

When something misroutes or the memory misses something in real use:
1. Rate it 👎 with a note.
2. Add a fixture case (routing) or a unit test (context).
3. Fix the rule, threshold or prompt, and confirm the new case passes.
