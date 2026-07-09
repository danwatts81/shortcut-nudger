# Next Ticket — Shortcut Nudger

A zero-install, single-page tool that answers one question:
**"which ticket do I work on next?"** — and, in team mode, **"what is everyone
working on right now?"**

It reads your [Shortcut](https://www.shortcut.com/) board through the REST API and
applies a fixed business rule, so you don't have to eyeball columns every time.

There is no backend, no build step, and no browser extension — just one
`index.html`. Shortcut's API allows CORS for token-authenticated requests, so the
page talks to Shortcut directly from your browser.

## What it does

- **My Queue** — your next ticket plus the rest of your queue in the target
  column, ranked by the rule below.
- **Role selection** — boards name their columns `Role (Stage)` (e.g.
  `Dev (Ready)`, `QA (Active)`). Roles are **auto-detected** from those names, and
  a **Role** dropdown lets you scope "next" to a single role's column. A synthetic
  **Planner** role spans every role's target column for a cross-role overview.
  Columns that don't match `Role (Stage)` (Inbox / Backlog / Completed) are
  ignored. Your selection persists across reloads.
- **Member switcher** — your token has workspace-wide read access, so you can
  view *anyone's* queue (view-only; your token/identity never changes). The
  selection persists across reloads; a **reset to me** link jumps back.
- **Team Manager** — every member grouped by their Shortcut team, each showing
  what they're on **Now** and **Next**, with an expandable list of the rest.
  Members in no team fall into a **No team** bucket. Each member row has its own
  **role** dropdown, so a teammate's Now/Next is computed against *their* role's
  column; unassigned members follow the globally-selected role. Assignments
  persist across reloads.
- **Timing badges** — each ticket shows how long it's sat in its current column
  (`⏱ in state 3d`) and how long since anything changed (`✎ updated 2h ago`).
  Tickets stuck past the stale threshold get a highlighted badge.
- **Versioning** — the footer shows the app version and build timestamp
  (e.g. `v1.0.0 · built 2026-07-09 14:32`), derived from the file's
  `Last-Modified` — no build step.

## The rule

1. **Target column** = the **right-most (furthest-along) non-done workflow state
   for the selected role**. Roles come from `Role (Stage)` column names, so for
   role `Dev` with `Dev (Ready)` / `Dev (Active)`, the furthest-along one wins.
   The **Planner** role targets every role's column at once.
2. Among **your** stories in that column, work **top-to-bottom** (board position).
   When a pick spans multiple columns (Planner or a mixed-role team), columns are
   ordered by their board position first, then position within each column.
3. **Exception:** any story with the `urgent` label jumps the queue. Urgent
   stories come first, still ordered top-to-bottom among themselves.
4. The single top-ranked story is shown as **"Work on this next."**

## Using it

1. Open the hosted page (see **Deployment** below for the URL to bookmark).
2. Generate a personal Shortcut API token:
   **Shortcut → Settings → API Tokens**
   (direct link: <https://app.shortcut.com/settings/account/api-tokens>).
3. Paste the token once and click **Save**. It's stored only in your browser's
   `localStorage` and sent only to Shortcut — never to any other server. Your
   identity (which tickets count as "yours") is detected automatically from the
   token, so there's no name to pick.
4. The page shows your next ticket, plus the rest of your queue, and
   auto-refreshes every ~30 seconds. There's also a manual **Refresh** button and
   a **reset token** link in the footer.
5. Use the **My Queue / Team Manager** toggle to switch views, the **Role**
   dropdown to scope "next" to a role (or **Planner** for all roles), and the
   member dropdown (in My Queue) to inspect a teammate's queue. In Team Manager,
   each member row has its own role dropdown. All selections persist.

Each teammate can paste **their own** token, but any single token already has
workspace-wide read access, so one person can inspect the whole team.

## Configuration

All knobs live in a `CONFIG` block at the top of the `<script>` in `index.html`:

```js
const CONFIG = {
  VERSION: "1.0.0",         // app version shown in the footer; bump by hand per release
  COLUMN_PREFIX: "Dev",     // fallback seed only — used if no Role (Stage) columns are detected
  URGENT_LABEL: "urgent",   // label that jumps the queue
  POLL_MS: 30000,           // auto-refresh interval (ms)
  STALE_DAYS: 3,            // a ticket in-state this many days gets the "stale" highlight
  API_BASE: "https://api.app.shortcut.com/api/v3",
  TOKEN_KEY: "shortcut_nudger_token",
  VIEW_MEMBER_KEY: "shortcut_nudger_view_member",       // persists the viewed member (member switcher)
  MODE_KEY: "shortcut_nudger_mode",                     // persists "me" | "team" view mode
  ROLE_KEY: "shortcut_nudger_role",                     // persists the selected global role
  MEMBER_ROLES_KEY: "shortcut_nudger_member_roles",     // persists per-member role overrides (JSON map)
};
```

- **Roles are auto-detected** from `Role (Stage)` column names — no config needed.
  `COLUMN_PREFIX` is now only a **fallback seed** used when a board has no matching
  columns; the live target is driven by the selected role.
- Bump `VERSION` by hand per release; the build timestamp is automatic.
- Different urgency label? Change `URGENT_LABEL`.
- Poll faster/slower? Change `POLL_MS`.
- Flag stale tickets sooner/later? Change `STALE_DAYS`.
- The `*_KEY` values are `localStorage` keys — rename only if they collide.

## Running locally (development)

Token-authenticated CORS needs a real `http(s)` origin — opening the raw
`file://` is unreliable (its `Origin` is `null`). Serve it over localhost:

```sh
# from the repo root
python -m http.server 8000
# then open http://localhost:8000/
```

Then confirm:

- `/member` resolves your name (shown in the header).
- The **Role** dropdown lists the roles auto-detected from your board, plus
  **Planner**; picking one scopes the target column to that role.
- The "next" card matches a manual read of your board.

## Deployment (GitHub Pages)

1. Commit `index.html` and `README.md` to the repo root.
2. Push to GitHub.
3. Repo **Settings → Pages** → deploy from the default branch, `/` (root).
4. Share the resulting `https://<user>.github.io/<repo>/` URL. Teammates
   bookmark it and paste their own token once.

Hosting on GitHub Pages (a real `https` origin) rather than `file://` keeps
token-auth CORS reliable.

## How it works

- `pickTargetState(workflows, prefix)` — picks the right-most non-done state
  whose name starts with the prefix, across all workflows.
- `detectRoles(workflows)` — parses `Role (Stage)` column names into the set of
  roles, resolving each role's target column via `pickTargetState` and excluding
  roles that have only done-type states.
- `resolveTargets(detected, roleName)` — returns the target column(s) for a role:
  one column for a normal role, or every role's column (deduped) for **Planner**.
- `pickNext(stories, myId, targetStateIds, urgentLabel, stateRank?)` — filters to
  a member's active stories across one *or more* target columns and sorts (urgent
  first, then by cross-column rank, then by position). Team Manager runs one
  state-scoped search per distinct target column (bounded by roles, not members),
  so there are still no per-member API calls.
- `formatDuration(fromISO, nowMs)` — compact elapsed time (`just now`, `5m`,
  `3h`, `4d`) powering the timing badges. `nowMs` is injected for testability.
- `formatBuildStamp(date)` — formats the footer build timestamp as
  `YYYY-MM-DD HH:MM` (24-hour, no seconds).

These are pure functions with no DOM or network dependencies, so a future
[Tampermonkey](https://www.tampermonkey.net/) overlay (a true in-page nudge via
`GM_xmlhttpRequest` + `@connect api.app.shortcut.com`) could reuse the same core
logic verbatim.

## Security note

The token grants access to your Shortcut workspace, so treat it like a password.
Using **per-user** tokens (rather than one shared workspace secret) keeps each
person's identity automatic and avoids embedding a shared full-access secret in a
public page.
