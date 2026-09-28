# 09: Export, logs and limits

**Phase 1 · Depends on: 08**

---

Finish Phase 1. Read `docs/DECISIONS.md` D30–D31, `docs/ROUTING.md` §7 and `docs/SECURITY.md` §9.

1. **`export` edge function:** takes `{projectId?}` (all projects when omitted). It returns a zip containing:
   - `export.json`: projects, conversations, non-superseded messages, files metadata plus extracted text, profile `about_me` and `app_settings`. Memory tables are added in Phase 3. Keys and secrets are never included.
   - `markdown/<project>/<conversation>.md`: the readable transcripts, with the model shown per message.
   - A `README.txt` that describes the format.
   The client downloads it as `switchyard-export-<date>.zip`. If the project is sensitive, show a warning first.
2. **Settings → Export tab:** "Export everything". The project ⋯ menu → Export project.
3. **Settings → Limits tab:** edit `monthly_cap_usd`, `confirm_threshold_usd`, `cap_warn_pct`, `default_max_tokens` and `prompt_log_retention_days`. Show this month's spend and a link to the OpenRouter credits page.
4. **Settings → Logs tab:** a table of all `llm_calls` (filters: project, purpose, model, status, date). The row drawer shows the full record, and the `prompt_logs.messages` JSON if it exists (with a note when it has expired or the project is sensitive).
5. A **global warning banner** when spend is at or above the warning percentage, and a red one at the cap ("Paid models are blocked until next month; free models still work").

---

### Verify (don't paste)
- [ ] Run the whole Phase 1 checklist in ROADMAP.md and TESTING.md §4.
- [ ] Update DECISIONS.md if anything changed. **Phase 1 is done → use the app daily for a few days before Phase 2.**
