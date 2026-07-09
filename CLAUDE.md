# CLAUDE.md — Next Ticket (Shortcut Nudger)

Project-level instructions for any agent working in this repo. These are
**mandatory** and override default behaviour.

## Orient first

Before making changes, read **[ARCHITECTURE.md](./ARCHITECTURE.md)** — the fast
architectural guide (what the app is, how `index.html` is structured, the core
business rule, key invariants, and gotchas). [README.md](./README.md) is the
user-facing companion. Do not start editing before you've read the guide; it will
save you from breaking a load-bearing invariant (single file, no build step, no
custom fetch headers, pure-logic separation, re-wire-after-render).

## Non-negotiable invariants (summary — details in the guide)

- **One file, zero dependencies.** No build step, framework, or npm. The whole app
  is `index.html`.
- **No custom fetch headers** — the token goes in the query string to avoid CORS
  preflight against Shortcut.
- **Never log or commit a real API token**; it's sent only to `api.app.shortcut.com`.
- **Keep pure logic pure** (no DOM/network) and **re-wire listeners after every
  render**.

## After every change — commit, push, publish

We do **live testing on GitHub Pages**, so changes must reach the live site:

1. Make and verify the change locally.
2. **Commit** to `main` (use the `mcp__nimbalyst-mcp__developer_git_commit_proposal`
   tool per the workspace convention).
3. **Push** to `origin/main` — this auto-publishes to GitHub Pages.
4. **Always print the live URL** when reporting completion:
   **<https://danwatts81.github.io/shortcut-nudger/>**

## After committing — update the docs

Once code is committed, **update the project documentation to match** so it stays
accurate. In the *same* piece of work:

- Update [ARCHITECTURE.md](./ARCHITECTURE.md) if structure, CONFIG, the business
  rule, or an invariant changed.
- Update [README.md](./README.md) if user-facing behaviour, configuration, or usage
  changed.
- Bump `CONFIG.VERSION` in `index.html` for a user-visible release.

Stale docs are treated as a bug. If behaviour changed and neither doc did, the task
isn't done.

## When in agent-coding mode — parallelize with sub-agents

For non-trivial work, **orchestrate with sub-agents** to move faster, and split the
work to **minimise merge conflicts**:

- **Split by file / region, not by feature that touches the same lines.** Because
  the app is a single `index.html`, partition by *non-overlapping regions* (e.g.
  one agent on CSS/`<style>`, one on a specific pure-logic function, one on a
  specific `render*`/`wire*` pair). Give each agent an explicit, bounded scope so
  two agents never edit the same lines.
- If a change is inherently serial (same region), do **not** fan out — a single
  agent is faster than reconciling conflicts.
- The **main orchestrating agent owns final quality**: after sub-agents finish, it
  integrates their work, resolves any conflicts, and runs the quality checks
  (loads the page, sanity-checks the four tabs and the business rule, confirms no
  invariant was broken) **before** committing. Sub-agents produce; the orchestrator
  verifies and ships.

## Deployment reference

- Remote: `https://github.com/danwatts81/shortcut-nudger.git`, branch `main`.
- GitHub Pages: deploys from `main` / root. Live URL:
  **<https://danwatts81.github.io/shortcut-nudger/>**
- Local dev: `python -m http.server 8000` then open `http://localhost:8000/`
  (needs a real `http` origin — `file://` breaks token-auth CORS).
