# 31: GitHub read-only knowledge

**Phase 5 · Depends on: 30**

---

Read `docs/DECISIONS.md` D29.

1. Secret: `GITHUB_TOKEN` (a fine-grained token, read-only for contents and metadata; I'll add it).
2. **Project → Tools → GitHub:** link one or more repos (`owner/repo`, branch) to the project. Store this in `integrations` (`kind='github'`, config `{repos: [...]}`, with no token).
3. **Tools (read):** `github_list_files {repo, path?}`, `github_read_file {repo, path}` (≤ 200 KB, truncated to 8k tokens), and `github_search_code {repo, query}`.
4. **Index repo docs (optional button):** chunk and embed the repo's `README*` and `docs/**/*.md` into `chunks` (`source_type='file'`, with a virtual file record per path), so that retrieval brings in repo knowledge automatically. Add "Re-index" and "Remove index" actions.

---

### Verify (don't paste)
- [ ] In the Software Dev project, "What does docs/ROUTING.md say about Tier 3?" answers correctly with the file path.
