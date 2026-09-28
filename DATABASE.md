# Database Schema

Postgres (Lovable Cloud / Supabase) with the `pgvector` extension. Every user-owned table has `user_id uuid not null default auth.uid()` and RLS enabled.

Each table is labelled with the **phase** in which it is created. Lovable should create only the tables for the phase being built.

Conventions:
- Primary keys are `uuid default gen_random_uuid()`.
- Timestamps are `timestamptz default now()`.
- Money is stored in USD as `numeric(12,6)`. Prices are stored **per million tokens** as `numeric(12,4)`, because OpenRouter reports them per token and per-million is easier to read.
- Enums are stored as `text` with `check` constraints, which makes them easier to change later than Postgres enums.

## Entity overview

```
auth.users ─1─1─ profiles / app_settings
     │
     └─< projects ─< conversations ─< messages >─ routing_decisions
            │             │              │
            │             │              └─< feedback
            ├─< memory_items (vector)    
            ├─< files ─< chunks (vector)
            └─< llm_calls >─ prompt_logs
models (global catalog) ─< tier_models (per user)
```

---

## Phase 1 tables

```sql
create extension if not exists vector;   -- needed from Phase 3; harmless to enable early

-- ── profiles: About Me + display prefs ─────────────────────────────
create table profiles (
  user_id      uuid primary key default auth.uid() references auth.users on delete cascade,
  display_name text,
  about_me     text default '',            -- global memory, ≤ ~300 tokens (UI shows counter)
  created_at   timestamptz default now(),
  updated_at   timestamptz default now()
);

-- ── app_settings: one row per user ─────────────────────────────────
create table app_settings (
  user_id                  uuid primary key default auth.uid() references auth.users on delete cascade,
  default_priority         text not null default 'cost' check (default_priority in ('cost','balanced','performance')),
  monthly_cap_usd          numeric(12,6) not null default 10,
  confirm_threshold_usd    numeric(12,6) not null default 0.25,
  cap_warn_pct             int not null default 80,
  default_max_tokens       int not null default 4096,
  context_budget_tokens    int not null default 4000,   -- MEMORY.md
  summarize_threshold_tokens int not null default 6000,
  extract_every_n_messages int not null default 16,
  prompt_log_retention_days int not null default 30,
  default_model_id         text,          -- Phase 1: used when project has no default (Auto arrives in Phase 2)
  classifier_model_id      text,          -- null → first active Tier-1 model
  utility_model_id         text,          -- title/summary/extraction; null → first active Tier-1 model
  baseline_model_id        text,          -- for "estimated savings" (Phase 4)
  models_synced_at         timestamptz,
  costs_reconciled_at      timestamptz,
  prompt_logs_purged_at    timestamptz,
  updated_at               timestamptz default now()
);

-- ── models: global OpenRouter catalog (written only by models-sync) ─
create table models (
  id                  text primary key,              -- e.g. 'qwen/qwen3-coder:free'
  provider_adapter    text not null default 'openrouter',
  name                text not null,
  vendor              text,                          -- prefix before '/', e.g. 'qwen'
  description         text,
  context_length      int,
  max_output_tokens   int,
  prompt_price_per_m  numeric(12,4) not null default 0,
  completion_price_per_m numeric(12,4) not null default 0,
  is_free             boolean not null default false, -- both prices = 0 or id ends ':free'
  supports_tools      boolean not null default false, -- 'tools' in supported_parameters
  supports_vision     boolean not null default false, -- 'image' in input_modalities
  supports_structured boolean not null default false, -- 'structured_outputs' or 'response_format'
  supports_reasoning  boolean not null default false, -- 'reasoning' in supported_parameters
  input_modalities    text[] default '{}',
  raw                 jsonb,                         -- full catalog entry for future use
  is_active           boolean not null default true,
  cooldown_until      timestamptz,                   -- set after 429/5xx (ROUTING.md §6)
  consecutive_failures int not null default 0,
  last_seen_at        timestamptz default now(),
  updated_at          timestamptz default now()
);
create index on models (is_free) where is_active;

-- ── projects ───────────────────────────────────────────────────────
create table projects (
  id                uuid primary key default gen_random_uuid(),
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  name              text not null,
  description       text default '',
  instructions      text default '',               -- system-level project instructions
  brief             text default '',               -- Phase 3 uses it; column exists now
  brief_suggestion  text,                          -- Phase 3: pending auto-update
  brief_updated_at  timestamptz,
  is_sensitive      boolean not null default false,
  priority_override text check (priority_override in ('cost','balanced','performance')),
  default_model_id  text references models(id) on delete set null,
  is_default        boolean not null default false,  -- the auto-created "General"
  archived_at       timestamptz,
  created_at        timestamptz default now(),
  updated_at        timestamptz default now()
);
create index on projects (user_id) where archived_at is null;

-- ── conversations ──────────────────────────────────────────────────
create table conversations (
  id                  uuid primary key default gen_random_uuid(),
  user_id             uuid not null default auth.uid() references auth.users on delete cascade,
  project_id          uuid not null references projects on delete cascade,
  title               text default 'New chat',
  mode                text not null default 'auto' check (mode in ('auto','manual')),
  sticky_model_id     text references models(id) on delete set null, -- manual pick persists (D16)
  summary             text,                        -- Phase 3 rolling summary
  summary_through_at  timestamptz,                 -- messages created ≤ this are summarized
  last_extracted_at   timestamptz,                 -- Phase 3 extraction watermark
  message_count       int not null default 0,
  created_at          timestamptz default now(),
  updated_at          timestamptz default now()
);
create index on conversations (project_id, updated_at desc);

-- ── routing_decisions (written by edge fn only) ────────────────────
create table routing_decisions (
  id                  uuid primary key default gen_random_uuid(),
  user_id             uuid not null references auth.users on delete cascade,
  project_id          uuid not null references projects on delete cascade,
  conversation_id     uuid not null references conversations on delete cascade,
  mode                text not null check (mode in ('auto','manual')),
  method              text not null check (method in ('manual','sticky','project_default','rules','classifier','retry_stronger')),
  priority            text not null,
  complexity          text check (complexity in ('low','medium','high')),
  complexity_score    int,
  task_type           text,                        -- general|coding|writing|analysis|vision|research
  signals             jsonb default '{}',          -- rule signals that fired
  requirements        jsonb default '{}',          -- {vision, tools, minContext, sensitive}
  tier                int check (tier in (1,2,3)),
  selected_model_id   text not null,
  candidates          jsonb default '[]',          -- [{id, rank, skippedReason?}]
  fallback_chain      jsonb default '[]',          -- [{from, to, reason}]
  escalated           boolean not null default false,
  reason              text not null,               -- human-readable "why"
  estimated_cost_usd  numeric(12,6),
  classifier_latency_ms int,
  created_at          timestamptz default now()
);

-- ── messages ───────────────────────────────────────────────────────
create table messages (
  id                  uuid primary key default gen_random_uuid(),
  user_id             uuid not null default auth.uid() references auth.users on delete cascade,
  project_id          uuid not null references projects on delete cascade,
  conversation_id     uuid not null references conversations on delete cascade,
  role                text not null check (role in ('user','assistant','system','tool')),
  content             text not null default '',
  attachments         jsonb default '[]',          -- [{fileId, name, kind}]
  model_id            text,                        -- assistant messages
  routing_decision_id uuid references routing_decisions on delete set null,
  status              text not null default 'complete' check (status in ('complete','streaming','stopped','error')),
  error               text,
  token_estimate      int,                         -- estimateTokens(content)
  tool_calls          jsonb,                       -- Phase 5
  pinned_to_memory    boolean not null default false,
  superseded_at       timestamptz,                 -- regenerate / edit keeps history hidden
  created_at          timestamptz default now()
);
create index on messages (conversation_id, created_at) where superseded_at is null;

-- ── llm_calls: every provider call (written by edge fn only) ───────
create table llm_calls (
  id                  uuid primary key default gen_random_uuid(),
  user_id             uuid not null references auth.users on delete cascade,
  project_id          uuid references projects on delete set null,
  conversation_id     uuid references conversations on delete set null,
  message_id          uuid references messages on delete set null,
  routing_decision_id uuid references routing_decisions on delete set null,
  agent_run_id        uuid,                        -- Phase 5
  purpose             text not null check (purpose in ('chat','classifier','title','extraction','summary','brief','embedding','tool','agent')),
  model_id            text not null,
  upstream_provider   text,                        -- provider OpenRouter actually used, if returned
  generation_id       text,                        -- OpenRouter response id
  prompt_tokens       int,
  completion_tokens   int,
  reasoning_tokens    int,
  cached_tokens       int,
  context_tokens      int,                         -- estimated tokens of memory/history we injected
  cost_usd            numeric(12,6),               -- from usage.cost; null until known
  is_free             boolean not null default false,
  latency_ms          int,
  ttft_ms             int,                         -- time to first token
  status              text not null check (status in ('ok','error','stopped','rate_limited','timeout')),
  error               text,
  sensitive           boolean not null default false,
  created_at          timestamptz default now()
);
create index on llm_calls (user_id, created_at desc);
create index on llm_calls (project_id, created_at desc);

-- ── prompt_logs: full assembled prompt, 30-day retention, never for sensitive ─
create table prompt_logs (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users on delete cascade,
  llm_call_id  uuid not null references llm_calls on delete cascade,
  messages     jsonb not null,           -- exactly what was sent (minus images as data URLs)
  context_report jsonb,                  -- per-block token counts from buildContext
  expires_at   timestamptz not null,
  created_at   timestamptz default now()
);
create index on prompt_logs (expires_at);

-- ── files ──────────────────────────────────────────────────────────
create table files (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid not null default auth.uid() references auth.users on delete cascade,
  project_id     uuid not null references projects on delete cascade,
  name           text not null,
  mime_type      text not null,
  size_bytes     int not null check (size_bytes <= 10485760),   -- 10 MB
  storage_path   text not null,          -- project-files/{user_id}/{project_id}/{file_id}/{name}
  kind           text not null check (kind in ('text','pdf','image','other')),
  extracted_text text,
  text_status    text not null default 'pending' check (text_status in ('pending','ready','failed','none')),
  token_estimate int,
  chunked_at     timestamptz,            -- Phase 3
  created_at     timestamptz default now()
);
```

**Storage:** a private bucket `project-files`. Its policy allows access only when the first path segment equals `auth.uid()::text`.

## Phase 2 tables

```sql
create table tier_models (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null default auth.uid() references auth.users on delete cascade,
  tier       int not null check (tier in (1,2,3)),
  model_id   text not null references models(id) on delete cascade,
  rank       int not null,                 -- 1 = first choice
  tags       text[] not null default '{}', -- 'coding','strong','vision','long','light'
  unique (user_id, tier, model_id)
);

create table feedback (
  id                  uuid primary key default gen_random_uuid(),
  user_id             uuid not null default auth.uid() references auth.users on delete cascade,
  message_id          uuid not null references messages on delete cascade,
  routing_decision_id uuid references routing_decisions on delete set null,
  rating              smallint not null check (rating in (-1, 1)),
  note                text,
  created_at          timestamptz default now(),
  unique (user_id, message_id)
);
```

## Phase 3 tables

```sql
create table memory_items (
  id                     uuid primary key default gen_random_uuid(),
  user_id                uuid not null default auth.uid() references auth.users on delete cascade,
  project_id             uuid not null references projects on delete cascade,
  type                   text not null check (type in ('fact','decision','preference','open_question','action_item')),
  content                text not null,          -- one or two sentences, self-contained
  source                 text not null default 'auto' check (source in ('auto','manual','pinned_message')),
  source_conversation_id uuid references conversations on delete set null,
  source_message_id      uuid references messages on delete set null,
  status                 text not null default 'active' check (status in ('active','resolved','archived')),
  pinned                 boolean not null default false,  -- always included in context
  importance             smallint not null default 2 check (importance between 1 and 3),
  superseded_by          uuid references memory_items on delete set null,
  embedding              vector(384),
  last_used_at           timestamptz,
  use_count              int not null default 0,
  created_at             timestamptz default now(),
  updated_at             timestamptz default now()
);
create index on memory_items (project_id, status);
create index on memory_items using hnsw (embedding vector_ip_ops);

create table chunks (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references auth.users on delete cascade,
  project_id   uuid not null references projects on delete cascade,
  source_type  text not null check (source_type in ('message','file','summary')),
  source_id    uuid not null,             -- files.id, or conversations.id for message/summary chunks
  conversation_id uuid references conversations on delete cascade, -- set for message/summary chunks
  ordinal      int not null default 0,    -- position within source
  content      text not null,             -- ~500 tokens, 50 overlap
  token_estimate int,
  embedding    vector(384),
  created_at   timestamptz default now()
);
create index on chunks (project_id, source_type);
create index on chunks using hnsw (embedding vector_ip_ops);

-- Semantic search RPC (embeddings are normalized → inner product == cosine)
create or replace function match_memory(p_project uuid, p_query vector(384), p_limit int default 12)
returns table (id uuid, type text, content text, pinned boolean, importance smallint, similarity float)
language sql stable security invoker as $$
  select m.id, m.type, m.content, m.pinned, m.importance, -(m.embedding <#> p_query) as similarity
  from memory_items m
  where m.project_id = p_project and m.status = 'active' and m.superseded_by is null
    and m.embedding is not null
  order by m.embedding <#> p_query
  limit p_limit;
$$;

create or replace function match_chunks(p_project uuid, p_query vector(384), p_limit int default 8,
                                        p_exclude_conversation uuid default null)
returns table (id uuid, source_type text, source_id uuid, content text, similarity float)
language sql stable security invoker as $$
  select c.id, c.source_type, c.source_id, c.content, -(c.embedding <#> p_query) as similarity
  from chunks c
  where c.project_id = p_project and c.embedding is not null
    and (p_exclude_conversation is null or c.conversation_id is distinct from p_exclude_conversation)
  order by c.embedding <#> p_query
  limit p_limit;
$$;
```

Note: `p_exclude_conversation` stops chunks from the *current* conversation being retrieved, because that conversation is already represented by its recent messages and rolling summary. Message chunks are created when old messages are folded into a summary (MEMORY.md §5), so the originals stay searchable from *other* conversations.

## Phase 4 views

```sql
create view v_usage_daily with (security_invoker = on) as
select user_id, date_trunc('day', created_at) as day, model_id, project_id, purpose,
       count(*) as calls, sum(prompt_tokens) as prompt_tokens, sum(completion_tokens) as completion_tokens,
       sum(coalesce(cost_usd,0)) as cost_usd, bool_and(is_free) as all_free
from llm_calls group by 1,2,3,4,5;

create view v_month_spend with (security_invoker = on) as
select user_id, sum(coalesce(cost_usd,0)) as spend_usd,
       count(*) filter (where is_free) as free_calls, count(*) as calls
from llm_calls
where created_at >= date_trunc('month', now() at time zone 'utc')
group by user_id;
```

(The `chat` function computes the month's spend directly with the same query as `v_month_spend`. The view is for the UI.)

## Phase 5 tables

```sql
create table agent_runs (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references auth.users on delete cascade,
  project_id    uuid not null references projects on delete cascade,
  conversation_id uuid references conversations on delete set null,
  goal          text not null,
  status        text not null check (status in ('running','awaiting_approval','done','failed','stopped','limit_reached')),
  max_steps     int not null default 10,
  max_cost_usd  numeric(12,6) not null default 0.50,
  steps_used    int not null default 0,
  cost_usd      numeric(12,6) not null default 0,
  created_at    timestamptz default now(),
  finished_at   timestamptz
);

create table agent_steps (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references auth.users on delete cascade,
  run_id        uuid not null references agent_runs on delete cascade,
  ordinal       int not null,
  kind          text not null check (kind in ('plan','tool_call','tool_result','model','approval','final')),
  tool_name     text,
  input         jsonb,
  output        jsonb,
  is_write      boolean not null default false,
  approved_at   timestamptz,
  created_at    timestamptz default now()
);

create table integrations (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null default auth.uid() references auth.users on delete cascade,
  kind        text not null,               -- 'github','gdrive','gmail','mcp'
  config      jsonb not null default '{}', -- non-secret config only; tokens live in secrets/vault
  enabled     boolean not null default true,
  created_at  timestamptz default now()
);

create table project_tools (
  project_id  uuid not null references projects on delete cascade,
  user_id     uuid not null default auth.uid() references auth.users on delete cascade,
  tool_name   text not null,
  enabled     boolean not null default true,
  requires_approval boolean not null default false,
  primary key (project_id, tool_name)
);
```

---

## RLS policies

**User-owned, read-write** (`profiles`, `app_settings`, `projects`, `conversations`, `messages`, `files`, `tier_models`, `feedback`, `memory_items`, `integrations`, `project_tools`):

```sql
alter table <t> enable row level security;
create policy "<t> owner all" on <t> for all to authenticated
  using (user_id = auth.uid()) with check (user_id = auth.uid());
```

**Written by functions, read-only for the user** (`routing_decisions`, `llm_calls`, `prompt_logs`, `chunks`, `agent_runs`, `agent_steps`):

```sql
alter table <t> enable row level security;
create policy "<t> owner read" on <t> for select to authenticated using (user_id = auth.uid());
-- no insert/update/delete policies → only service role can write
```

Exception: `messages` is read-write so that the client can pin a message and set `superseded_at`. Assistant messages, however, are **inserted only by `chat`**.

**Global catalog** (`models`):

```sql
alter table models enable row level security;
create policy "models read" on models for select to authenticated using (true);
-- writes: service role only (models-sync, chat cooldown updates)
```

## Triggers and housekeeping

- `updated_at` trigger on tables that have that column.
- On `messages` insert: `conversations.message_count += 1` and `updated_at = now()`.
- On first login (the client checks this): upsert `profiles` and `app_settings`, and create the "General" project if the user has none.
- Prompt-log purge: `chat` runs `delete from prompt_logs where expires_at < now()` when `app_settings.prompt_logs_purged_at` is more than 24h old. No cron is needed. If pg_cron is available, it can be used instead.
- Cost reconcile: the same "stale-on-read" pattern, using `costs_reconciled_at` (hourly).
