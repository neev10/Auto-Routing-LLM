# Memory Architecture

> **The model is replaceable. The project's memory is not.**

Memory belongs to the app. On every request, a context builder assembles a compact, relevant package for whichever model is selected. Switching models needs no handoff, because every model receives the same package.

Implemented in `_shared/core/context.ts` (pure), `memory-extract`, `summarize` and `embed`.

## 1. Layers

| Layer | What it is | Stored in | Created by | Included |
|---|---|---|---|---|
| **About Me** (global) | ≤300 tokens: role, answer style, tools used | `profiles.about_me` | The user | Always |
| **Project instructions** | How to behave in this project | `projects.instructions` | The user | Always |
| **Project Brief** | 300–500 words: what the project is, goals, current state, key decisions, constraints | `projects.brief` | The user; auto-suggested updates | Always |
| **Memory items** | Typed, atomic entries: fact, decision, preference, open question, action item | `memory_items` | Auto-extraction, manual entry, or pinning a message | Pinned items always; others by relevance |
| **Rolling summary** | A summary of the older part of the current conversation | `conversations.summary` | `summarize` | When it exists |
| **Recent messages** | The last turns, word for word | `messages` | Chat | Always (within budget) |
| **Searchable chunks** | Pieces of files, and older messages from other conversations | `chunks` | File ingest; summarize | By relevance |

This maps to the layers in the original brief:
- short-term memory = recent messages
- working memory = rolling summary plus pinned items
- long-term memory = memory items plus the Brief
- semantic memory = embeddings over items and chunks
- project knowledge = files, the Brief and instructions

**Isolation (D23):** every retrieval query filters by `project_id`. Only About Me is shared across projects.

## 2. The context builder

### 2.1 Budget

```
budget = clamp(app_settings.context_budget_tokens (default 4000),
               min = 1500,
               max = floor(model.context_length * 0.25))
if tier == 3 or user chose a model with context_length ≥ 200k:
   budget = min(8000, floor(model.context_length * 0.25))       // premium can take more
```

The budget covers blocks 2–8 below. The current user message and any attachments are counted separately; the cost guard sees the full total.

### 2.2 Assembly order and per-block caps

| # | Block | Cap (tokens) | If over the cap |
|---|---|---|---|
| 1 | App preamble (fixed, ~120 tokens) | — | — |
| 2 | About Me | 300 | Truncated |
| 3 | Project instructions | 500 | Truncated, with a note |
| 4 | Project Brief | 700 | Truncated, with a note |
| 5 | Memory items: pinned first, then by relevance | 800 | Lowest score dropped |
| 6 | Retrieved chunks | 800 | Lowest similarity dropped |
| 7 | Rolling summary | 600 | Truncated (summaries are kept ≤600 when written) |
| 8 | Recent messages, newest first until the budget runs out | remainder (min: last 2 exchanges) | Oldest dropped |
| 9 | Current user message + attachment text | not budgeted | Attachments over 12k tokens are truncated with a note, and the rest goes to chunks (Phase 3) |

**Trimming order when the total is over budget:** 6 → 5 (unpinned) → 8 (down to the minimum) → 7 → 4 → 3. Blocks 1, 2 and 9 are never trimmed.

### 2.3 Output shape

The builder returns OpenAI-format `messages[]` plus a `report`:

```
system:
  You are an assistant inside the user's personal AI workspace. Earlier replies in this
  conversation may have been written by different AI models; treat them as your own prior work.
  Use the project context below. If it conflicts with the user's latest message, follow the user.

  ## About the user
  {about_me}

  ## Project: {name}
  ### Instructions
  {instructions}
  ### Brief
  {brief}
  ### Relevant memory
  - [decision] …   - [fact] …   - [open_question] …
  ### Relevant excerpts
  > (file: spec.pdf) …
  ### Earlier in this conversation (summary)
  {summary}
user/assistant: … recent messages …
user: {current message}
  [Attached: notes.md]
  {attachment text}
```

`report = { budget, blocks: [{name, tokens, items, trimmed}], total }`. This report is stored in `prompt_logs.context_report`, shown in the context inspector, and its total becomes `llm_calls.context_tokens`.

### 2.4 Retrieval (Phase 3)

1. Build the query: the current message, plus the last assistant message if the current one is short (<30 tokens, e.g. "ok do that"), truncated to 512 tokens.
2. Embed it with `gte-small` (the `embed` function, `mean_pool` + `normalize`).
3. `match_memory(project, q, 12)` → score = `similarity + 0.05 × importance + (pinned ? 1 : 0)`. Keep items with similarity ≥ 0.75 or pinned, and fill up to the block cap.
4. `match_chunks(project, q, 8, exclude current conversation)` → keep items with similarity ≥ 0.78, and fill up to the cap.
5. Update `last_used_at` and `use_count` on the items that were included, in the background.

Thresholds are constants in `context.ts`. Tune them by using the context inspector.

**Phase 1 (before memory exists):** blocks 1, 2, 3 and 8 only, with the recent messages capped at 6,000 tokens. This is already cheaper than sending the full history.

## 3. Memory items

### 3.1 Types

| Type | Example |
|---|---|
| `fact` | "The app is hosted on Lovable Cloud; GitHub is the source of truth." |
| `decision` | "Decided: hybrid routing — rules first, classifier when unsure." |
| `preference` | "Prefers concise answers with tables for comparisons." |
| `open_question` | "Which search API for Phase 5 — Tavily or Brave?" |
| `action_item` | "Buy $10 OpenRouter credits to unlock 1,000 free requests/day." |

Rules: each item is 1–2 sentences, **self-contained** (no "it" or "the above"), and never contains secrets.

### 3.2 Automatic extraction (`memory-extract`)

**When:**
- In the background after every **16 new messages** in a conversation (those created after `last_extracted_at`).
- When the user leaves a conversation that has ≥4 unextracted messages (the client calls the function when switching conversations, and on tab close with `fetch(..., {keepalive: true})`, since `sendBeacon` cannot send the auth header).

**How:**
1. Load the unextracted messages (≤8k tokens; if there are more, process them in batches), the project Brief, and up to 40 existing active items (id and content).
2. Call the utility model (Sensitive → deny) with the extraction prompt below. It returns JSON.
3. For each proposed item:
   - Embed it. If an existing active item has similarity ≥ 0.92 → **update** that item's content (or skip it if they're the same).
   - If the model marks `supersedes: <id>` → insert the new item and set `superseded_by` on the old one.
   - `resolves: <id>` → set the open question or action item to `status='resolved'`.
   - Otherwise → insert with `source='auto'`.
4. Set `conversations.last_extracted_at`.
5. If ≥5 items changed, or the Brief is older than 7 days and ≥1 item changed → **propose a Brief update** (§4).

**Extraction prompt (abbreviated):**

```
You maintain long-term memory for a project. From the NEW MESSAGES, extract only durable,
useful items. Types: fact, decision, preference, open_question, action_item.
- Each item: 1–2 self-contained sentences. No pronouns without referents. No secrets/keys.
- Skip chit-chat, transient details, and anything already in EXISTING ITEMS unless it changed.
- If an item replaces an existing one, set "supersedes": "<id>". If it answers an open
  question or completes an action item, set "resolves": "<id>".
- importance: 3 = core to the project, 2 = useful, 1 = minor.
Return JSON: {"items":[{"type","content","importance","supersedes"?,"resolves"?}]}
Return {"items":[]} if nothing qualifies.
```

### 3.3 Manual memory

- **+ Add memory** in the Memory tab (choose a type and write the content).
- **📌 Pin to memory** on any message → the utility model condenses the message into one item (`source='pinned_message'`, `pinned=true`). The user can edit the result.
- Every item can be edited, pinned or unpinned, resolved, archived or deleted. Edits re-embed the item.

## 4. Project Brief

- The user writes it. A template is offered when a project is created: *Purpose · Goals · Current state · Key decisions · Constraints · Open problems*.
- **Suggested updates:** the utility model receives the current Brief plus the changed items and returns a revised Brief of ≤500 words. It's stored in `projects.brief_suggestion`. The Brief tab shows a diff with **Accept**, **Edit** and **Dismiss**. The Brief is never overwritten silently (D18).
- A **Regenerate from memory** button can build a Brief for a project that doesn't have one.

## 5. Rolling summaries (`summarize`)

**Trigger:** after an assistant message, if the tokens of messages newer than `summary_through_at` exceed **6,000** (D21).

**How:**
1. Take the oldest unsummarized messages, leaving the most recent ~2,000 tokens unsummarized.
2. Prompt: *"Update the running summary of this conversation. Keep: goals, decisions made, conclusions, open threads, key facts, code/file names. ≤600 tokens. Write in third person ('The user…', 'The assistant…')."* The input is the old summary plus the messages being folded in.
3. Save `conversations.summary` and advance `summary_through_at`.
4. **Chunk and embed the folded messages** into `chunks` (`source_type='message'`, `conversation_id` set). Other conversations can then retrieve them, and nothing is lost.

The originals are never deleted. The UI always shows the full conversation.

## 6. Files as knowledge

| Phase | Behavior |
|---|---|
| 1 | Upload .txt, .md or PDF (≤10 MB). Text is extracted (PDF in the browser with `pdfjs-dist`; scanned PDFs get `text_status='none'`). **Attach to message** puts the text into block 9. |
| 2 | Images: stored in the bucket and sent to vision models as short-lived signed URLs. |
| 3 | On upload, and for existing files: the text is chunked (~500 tokens, 50 overlap, split on headings or paragraphs) and embedded into `chunks`. Retrieval (block 6) then brings in relevant parts automatically. |
| Later | .docx, .xlsx, .csv parsers. |

## 7. Model switching

Nothing special happens. When the model changes, the next request is built from the same blocks. The preamble tells the model that earlier replies may have come from other models. The routing card shows the switch ("Switched from X"), so the history stays readable.

## 8. The context inspector (FR-28)

A "🔍 What did the model see?" link on each assistant message opens:
- the budget and each block's token count, with trimmed blocks highlighted
- the memory items and chunks included, with their similarity scores
- the full prompt, from `prompt_logs` (not available for Sensitive projects or after 30 days)

This is the main tool for tuning thresholds and budgets.

## 9. Costs of memory

All background memory work uses free Tier-1 models (Sensitive projects use deny-routed free models). Embeddings are free because they run inside the edge runtime. A rough estimate of daily free-request usage: extraction once per 16 messages, plus a summary about once per 20 messages, plus a title per conversation. That's roughly 10–15% of chat volume.
