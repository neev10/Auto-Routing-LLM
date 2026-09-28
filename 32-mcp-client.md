# 32: MCP client (remote servers)

**Phase 5 · Depends on: 31**

---

Read `docs/DECISIONS.md` D26 and `docs/SECURITY.md` §7.

1. **Settings → Integrations → MCP servers:** add a remote MCP server (a name, a Streamable HTTP URL, and an optional auth header whose value is stored as a secret or in Vault by name; never in `integrations.config`).
2. In the edge function, use the official MCP TypeScript SDK client (via an npm specifier) to connect, `listTools`, and `callTool`. Cache the tool list per server for 10 minutes.
3. Expose the MCP tools through the registry as `mcp__{server}__{tool}`. By default **every MCP tool is treated as a write tool** (it needs approval) unless the user marks it read-only in the Tools tab.
4. **Per-project enablement** in the Tools tab, the same as the built-in tools.
5. The first targets are the Google Drive, then Gmail/Calendar MCP servers, if hosted remote versions are available to you. Otherwise, document this as a follow-up.

---

### Verify (don't paste)
- [ ] Connect one public test MCP server, list its tools, and call one after approval.
- [ ] Run the whole Phase 5 checklist, and update DECISIONS.md.
