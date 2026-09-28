# Routing Architecture

Implemented in `supabase/functions/_shared/core/complexity.ts`, `router.ts` and `cost.ts`, all as pure functions. Tests are described in [TESTING.md](TESTING.md).

## 1. Principles

1. **Use free models when they're good enough.** Escalate only when there's a reason to.
2. **Be transparent.** Every decision stores its method, signals, candidates and a reason that a person can read.
3. **Settings are data.** Tiers, thresholds and priorities live in the database, not in code.
4. **Degrade gracefully.** One model failing never stops a request while alternatives remain.

## 2. Pipeline

```
selectModel(input) ─┬─ 1. Explicit choice?  message override > conversation sticky > project default
                     │        └─ yes → use it (still subject to requirements check + cost guard)
                     ├─ 2. Requirements      vision? minContext? tools? sensitive?
                     ├─ 3. Complexity        rules → confident? else classifier
                     ├─ 4. Tier              complexity × priority → tier (+ cap/budget constraints)
                     ├─ 5. Candidate pick    tier list filtered by requirements & health, tag-sorted
                     └─ 6. Fallback plan     remaining candidates → next tier (auto T1→T2; T3 asks)
then cost guard (§7) → stream → runtime fallback (§6)
```

## 3. Explicit choices (Mode = Manual, or defaults)

| Source | Method stored | Behavior |
|---|---|---|
| The user picks a model in the picker | `manual` | Sets `conversations.mode='manual'` and `sticky_model_id`. Applies to this message and to later ones (D16). |
| A later message in a manual conversation | `sticky` | Uses `sticky_model_id`. |
| A new conversation in a project that has `default_model_id` | `project_default` | Starts in manual mode with that model. The user can switch to Auto. |
| "Retry with a stronger model" | `retry_stronger` | Moves one tier up from the previous decision's tier (from a manual pick, it uses the tier that model belongs to, or T2 if it isn't in any tier). |

Explicit choices **still** go through:
- **The requirements check.** If an image is attached and the chosen model has no vision support, the app returns `error{code:'model_lacks_vision'}` and suggests vision models. It never silently switches.
- **The Sensitive check.** A Sensitive project sends `data_collection: "deny"`. If OpenRouter has no qualifying endpoint, the request fails with a clear message.
- **The cost guard** (§7).

Explicit choices do **not** fall back automatically to a different model. On failure, the user sees **Retry** and **Switch to Auto**.

## 4. Complexity detection (hybrid, D12)

### 4.1 Rules (`scoreComplexity`)

The input is the user message text, attachment token estimates, and the conversation's previous tier.

| Signal | Points | Notes |
|---|---|---|
| Message length (estimated tokens) | <60: 0 · 60–300: +1 · 300–1500: +2 · >1500: +3 | Counts only the text typed or pasted, not attachments |
| A code block (```) or ≥3 lines that look like code | +1, and `taskType=coding` | Code heuristic: lines ending in `;`, `{`, `}`, or starting with `def `, `function `, `class `, `import `, `SELECT ` |
| High-complexity keywords | +2 (applied once) | `architect`, `design a system`, `system design`, `debug`, `root cause`, `refactor`, `optimi[sz]e`, `trade-?offs?`, `prove`, `derive`, `step[- ]by[- ]step plan`, `multi-step`, `in depth`, `comprehensive`, `evaluate .* options`, `compare .* (and\|vs\|versus)`, `strategy`, `implementation plan` |
| Low-complexity keywords | −1 (applied once) | `rewrite`, `rephrase`, `reword`, `summari[sz]e`, `tl;dr`, `translate`, `fix (the )?grammar`, `proofread`, `shorten`, `make it (shorter\|more formal\|friendlier)`, `what is`, `define`, `synonym`, `title for` |
| Attachment text tokens | >4,000: +1 · >20,000: +2 | Also sets `requirements.minContext` |
| The previous decision in this conversation was T2 or higher | +1 | Conversations tend to stay hard |
| Research intent (`research`, `find sources`, `latest`, `current`) | +1, and `taskType=research` | Web search comes in Phase 5 |

`taskType` hints (these add no points), checked in order, first match wins:
1. **coding**: a code block or code-like lines, or the words `python`, `javascript`, `typescript`, `sql`, `regex`, `function`, `script`, `stack trace`, `compile`
2. **research**: `research`, `find sources`, `latest`, `current`
3. **analysis**: `analy[sz]e`, `compare`, `evaluate`
4. **writing**: `rewrite`, `draft`, `email`, `cover letter`, `proofread`, `rephrase`
5. otherwise **general**

Vision (an image attachment) sets `taskType=vision` and `requirements.vision=true`.

Keyword matching is case-insensitive, and each pattern must start at a word boundary (a leading `\b`). Without that, `undefined` would match `define`. The exact regexes are in `complexity.ts` and are checked against [`fixtures/routing-test-set.json`](fixtures/routing-test-set.json).

**Bands:**

| Score | Band | Confident? |
|---|---|---|
| ≤ 1 | low | yes → no classifier |
| 2–4 | uncertain | **no → classifier** |
| ≥ 5 | high | yes → no classifier |

For **Sensitive** projects the classifier is still allowed (with `data_collection: deny`). If it fails, the rules' best guess is used: score 2 → low, 3–4 → medium.

### 4.2 Classifier (only when the rules are unsure)

- Model: `app_settings.classifier_model_id`, or the first active Tier-1 model.
- Timeout: **4s**, with `max_tokens: 80` and `temperature: 0`.
- Input: the system prompt below plus the user message truncated to 1,500 tokens. Project context is **not** included; the message alone is enough to judge difficulty, and it keeps the call cheap.
- The call is logged in `llm_calls` with `purpose='classifier'`.

```
System: You classify the difficulty of a request for an AI model router.
Reply with JSON only: {"complexity":"low|medium|high","task_type":"general|coding|writing|analysis|research","needs_reasoning":true|false,"confidence":0-1}
low = a small/free model will do well (rewrites, short answers, simple code).
medium = needs a capable model (multi-part questions, moderate code, careful writing).
high = needs a frontier model (hard reasoning, complex debugging/architecture, long multi-step work, high-stakes accuracy).
```

If parsing fails or the call times out, the app uses the rules' fallback mapping (score 2 → low, 3–4 → medium) and records `method='rules'` with `signals.classifier='failed'`.

## 5. Complexity × priority → tier

| Complexity | Cost (default) | Balanced | Performance |
|---|---|---|---|
| low | T1 | T1 | T2 |
| medium | T1 (prefer `strong` tag) | T2 | T2 |
| high | T2 (+ "Try premium?" chip) | T2 (+ chip) | **T3 (asks first)** |

Priority comes from `projects.priority_override`, or else `app_settings.default_priority`.

**Budget constraint:** if the month's spend is at or above the monthly cap, the tier is forced to T1 and paid models are removed everywhere. The routing card then says "Monthly cap reached, free models only."

**Tier 3 rule (D15):** Auto never goes to T3 without asking. The `chat` function returns `needs_confirmation{kind:'tier3'}` with the T3 model and its estimated cost. The client offers **Use premium** and **Use T2 instead**.

## 6. Candidate selection and fallback

### 6.1 Picking a candidate

For the chosen tier:
1. Start from `tier_models` for that tier, ordered by `rank`.
2. Remove models that are `is_active = false`, still in `cooldown_until > now()`, or that fail a requirement:
   - vision required → `supports_vision`
   - `minContext` → `context_length ≥ estimatedPromptTokens + max_tokens`
   - tools required (Phase 5) → `supports_tools`
3. Sort (a stable sort, keeping rank order within each group) by tag match: `taskType=coding` prefers the `coding` tag, `medium` under Cost prefers `strong`, and long context prefers `long`.
4. The first model is selected. The rest become `candidates` (skipped models are stored with `skippedReason`).
5. If the list is empty: under Cost, `escalate` to the next tier (T1→T2 is automatic; T2→T3 asks). Otherwise, the app returns `error{code:'no_eligible_model'}` with the reasons.

**Auto-fill for vision:** if vision is needed and no tier model supports it, the app picks the cheapest active vision model in the catalog (free first), and the reason says so.

### 6.2 Runtime fallback (before the first token)

| Upstream result | Action |
|---|---|
| 429 (rate limited) | Mark the model `cooldown_until = now() + 10 min` and try the next candidate |
| 5xx, network error, or no first token within 20s | `consecutive_failures += 1`, cooldown for 2 min (15 min after 3 failures), then try the next candidate |
| 402 (credits) | Stop. Return `error{code:'credits'}` with a link to OpenRouter credits |
| 400 "no endpoints" (usually because of the Sensitive filter) | Try the next candidate. If none remain, return the message: "No private provider available for these models." |
| Tier-1 candidates used up | **Automatically escalate to T2** (D15). Set `escalated=true` and send a `fallback` event with the reason "free models unavailable". Blocked if the cap has been reached. |
| Tier-2 candidates used up | Return `needs_confirmation{kind:'tier3'}` |

At most **4 attempts** per request. Each attempt is added to `fallback_chain`. After a success, `consecutive_failures = 0`.

## 7. Cost guard (D11)

Runs after the model is selected and the context is built, before any upstream call.

```
promptTokens   = estimateTokens(all assembled messages)
expectedOutput = min(max_tokens, 1000)              // typical answer
estimate       = promptTokens/1e6 * prompt_price_per_m + expectedOutput/1e6 * completion_price_per_m

if model is paid:
  if monthSpend >= monthly_cap:                     → reroute to T1 (Auto) or needs_confirmation{cap_reached} (Manual: offer free alternative)
  if monthSpend + estimate > monthly_cap:           → needs_confirmation{cap_reached} ("this request may exceed your cap")
  if estimate > confirm_threshold (0.25) && !confirm → needs_confirmation{cost}
warn banner when monthSpend >= cap_warn_pct% of cap
```

After a successful call, the actual cost is taken from `usage.cost` (verified: returned in the final SSE chunk).

The OpenRouter key's own credit limit ($10) is the backstop outside the app. Even a bug in the guard can't spend more than that.

## 8. Tier lists and "Suggest defaults"

Tier lists are editable in **Settings → Routing & Tiers** (drag to reorder, add from the catalog, set tags). **Suggest defaults** fills empty tiers from the current catalog using this heuristic, and the user then reviews and adjusts the result:

| Tier | Heuristic |
|---|---|
| T1 | Active `:free` models with `context_length ≥ 32k`. Prefer well-known vendors (deepseek, qwen, meta-llama, google, mistralai, openai open-weight). Pick 4: one tagged `coding` (the id contains `coder`/`code`), one `strong` (the largest parameter count in the name, or a reasoning-capable model), one `vision` if available, one `light`. |
| T2 | Paid models with `prompt_price_per_m ≤ $1` and `context_length ≥ 64k`. Pick 3, one of them tagged `coding`. |
| T3 | Paid frontier models (anthropic, openai, google top-line models) with `prompt_price_per_m ≤ $15`. Pick 2. |

Free model lineups change often. When a tier model disappears from the catalog, the app shows a banner under Settings → Tiers.

**The utility model** (titles, summaries, extraction) defaults to the Tier-1 model tagged `strong`, or else the first Tier-1 model. It should be good at following JSON instructions. If extraction quality is poor, pick a better one in Settings.

## 9. The routing card (UI)

Summary line on each assistant message:

```
⚡ Auto → deepseek/…:free · T1 · medium · classifier · $0.0000 · ctx 3.4k  [👍][👎][↻][⤴ stronger]
```

The popover shows: the reason sentence, the signals that fired, the classifier output, the candidates and why others were skipped, the fallback chain, estimated vs. actual cost, context tokens by block (Phase 3), and a link to the prompt log.

**Reason sentence template:** `"{Complexity} {taskType} request ({method}); {priority} priority → Tier {n}; picked {model} ({why: first eligible | coding tag | vision required | fallback after X})."`

## 10. Title, extraction and summary calls

These calls never go through Auto. They use `utility_model_id` (or `classifier_model_id` for titles), with the same fallback list as T1, respect the Sensitive flag, and **never use paid models** unless the user chooses a paid utility model in Settings.

## 11. Phase 6 extensions (not built yet)

- **Learned routing:** for each (taskType, complexity) pair, compute the thumbs-up rate per model over the last N ratings, and re-rank tier lists using the rating with a cost penalty.
- **Critic pass:** for high-stakes requests, a second model reviews the answer.
- **Planner/executor:** a T3 model plans, and T1/T2 models execute the steps.
- **Benchmarks:** run the routing test set and a small evaluation set against candidate models on a schedule.
