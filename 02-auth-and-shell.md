# 02: Auth, bootstrap and app shell

**Phase 1 · Depends on: 01**

---

Build authentication and the app shell. Read `docs/SECURITY.md` §3 and `docs/PRD.md` §10.

1. **Login page** with email and password: sign in, plus a sign-up form that I'll use once. Hide the sign-up option behind a small "First-time setup" link. There is no social login.
2. **Protected routes.** Everything except `/login` requires a session.
3. **First-login bootstrap** (runs client-side after login, and is idempotent): upsert `profiles` and `app_settings` for the user. If the user has no projects, create one named "General" with `is_default = true`.
4. **App shell:**
   - Left sidebar (fixed width, about 260px): "+ New chat" button; a PROJECTS list (non-archived, with 🔒 on sensitive ones, "+" to add a project); a RECENT list (the 15 most recently updated conversations across projects); ⚙ Settings at the bottom with a Sign out button.
   - Main area: a placeholder "Select or start a conversation."
   - Routes: `/`, `/p/:projectId`, `/c/:conversationId`, `/settings/:tab?`.
5. "+ New chat" creates a conversation in the currently selected project (or General) and navigates to it.
6. Settings page skeleton with tabs: OpenRouter, Models, Limits, About Me, Export, Logs. Each tab is empty for now, except **About Me**, which edits `profiles.about_me` with a character counter (warn above about 1,200 characters, roughly 300 tokens).

---

### Verify (don't paste)
- [ ] You can sign up once, log out and log in again.
- [ ] "General" appears automatically.
- [ ] Creating a project and a new chat works.
- [ ] **Now disable sign-ups** in Cloud → Auth settings (see SETUP.md §4).
