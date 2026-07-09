# Next Ticket — Shortcut Nudger

A zero-install, single-page tool that answers one question:
**"which ticket do I work on next?"** — and, across four purpose-built tabs,
**"what is my team / a role working on right now?"**

It reads your [Shortcut](https://www.shortcut.com/) board through the REST API and
applies a fixed business rule, so you don't have to eyeball columns every time.

There is no backend, no build step, and no browser extension — just one
`index.html`. Shortcut's API allows CORS for token-authenticated requests, so the
page talks to Shortcut directly from your browser.

## What it does

The app is organised into **four tabs**, each owning only the selector(s) it
needs. Your active tab and each selector persist across reloads.

- **My Queue** — your next ticket plus the rest of your queue, ranked by the rule
  below. A **Role** dropdown scopes "next" to a single role's column.
- **My Teammates** — pick any teammate from a dropdown and see *their* next +
  queue (view-only; your token/identity never changes). A **reset to me** link
  jumps back. A teammate's Now/Next is computed against *their* assigned role
  (see **Manage Teammates**), falling back to your role if unassigned.
- **Team Manager** — pick a team (your teams are listed first, then all others,
  plus a **No team** bucket) and see every member's **Now** / **Next** / paused
  work, with an expandable list of the rest. Each member row has its own inline
  **role** dropdown.
- **Role Manager** — pick a role and see everyone assigned to it, a **Missing
  Role Assignment** list (teammates with no role yet), and a **"N Teammates in
  Other Roles Hidden"** reveal toggle.

- **Role detection** — boards name their columns `Role (Stage)` (e.g.
  `Dev (Ready)`, `QA (Active)`). Roles are **auto-detected** from those names. A
  synthetic **Planner** role spans every role's target column for a cross-role
  overview. Columns that don't match `Role (Stage)` (Inbox / Backlog / Completed)
  are ignored.
- **Manage Teammates** — a **Manage Teammates** button (in the controls row)
  opens a modal for bulk role assignment across every member, with a name filter.
  The same per-member role dropdowns also appear inline on Team Manager / Role
  Manager rows; both write the same persisted assignments.
- **Workflow-state chips** — every queue / "Then" / rest row shows its
  `Role (Stage)` state name as a chip, so you can see which column a ticket is in
  at a glance (useful under Planner and mixed-role team views).
- **Timing badges** — each ticket shows how long it's sat in its current column
  (`⏱ in state 3d`) and how long since anything changed (`✎ updated 2h ago`).
  Tickets stuck past the stale threshold get a highlighted badge.
- **API log tray** — a pull-up tray pinned to the bottom of the page logs every
  Shortcut API call this session (method, path, status, elapsed ms), with the
  auth token redacted and a live ticking timer for in-flight calls. It's
  session-only (in-memory) and clears on reload.
- **Versioning** — the footer shows the app version and build timestamp
  (e.g. `v1.1.0 · built 2026-07-09 14:32`), derived from the file's
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
5. **Paused rule (generalized):** any state whose stage matches `/paused/i`
   (`Dev (Paused)`, `QA (Paused)`, `Design (Paused)`, …) is treated as paused. A
   paused ticket is **never** "next" (even if urgent). Paused tickets surface as a
   secondary, dashed **Paused** section **only** when the member has no non-paused
   ticket *strictly more advanced than the entry/"Ready" column* — i.e. someone
   who's actively mid-flight isn't nagged about their parked ticket, but someone
   whose only work is a Ready ticket (or nothing but a paused one) still sees it.
   Under **Planner** the "entry column" is the global least-advanced position, so
   paused semantics are coarser there (documented, acceptable).

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
5. Use the **My Queue / My Teammates / Team Manager / Role Manager** tabs to
   switch views. Each tab shows only its own selector(s): a **Role** dropdown
   (My Queue / Role Manager), a **teammate** dropdown (My Teammates), or a **team**
   dropdown (Team Manager). Pick **Planner** in a role dropdown to span all roles.
   Assign roles to teammates via the **Manage Teammates** modal or the inline
   dropdowns on Team/Role Manager rows. Pull up the **API tray** at the bottom to
   watch Shortcut calls live. All selections persist across reloads.

Each teammate can paste **their own** token, but any single token already has
workspace-wide read access, so one person can inspect the whole team.

## Configuration

All knobs live in a `CONFIG` block at the top of the `<script>` in `index.html`:

```js
const CONFIG = {
  VERSION: "1.1.0",         // app version shown in the footer; bump by hand per release
  COLUMN_PREFIX: "Dev",     // fallback seed only — used if no Role (Stage) columns are detected
  URGENT_LABEL: "urgent",   // label that jumps the queue
  POLL_MS: 30000,           // auto-refresh interval (ms)
  STALE_DAYS: 3,            // a ticket in-state this many days gets the "stale" highlight
  API_BASE: "https://api.app.shortcut.com/api/v3",
  TOKEN_KEY: "shortcut_nudger_token",
  VIEW_MEMBER_KEY: "shortcut_nudger_view_member",       // persists the viewed teammate (My Teammates tab)
  TAB_KEY: "shortcut_nudger_tab",                       // persists the active tab (replaces the old MODE_KEY)
  ROLE_KEY: "shortcut_nudger_role",                     // persists the My Queue role (also the global fallback role)
  ROLEVIEW_KEY: "shortcut_nudger_roleview",             // persists the Role Manager's viewed role
  TEAM_KEY: "shortcut_nudger_team",                     // persists the Team Manager's selected team
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
- `TAB_KEY` **replaces** the old `MODE_KEY`. `getStoredTab()` validates the stored
  value against the four known tabs and falls back to `myqueue`, so an existing
  `shortcut_nudger_mode` (`"me"`/`"team"`) value is simply ignored — no breakage.

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

- `detectRoles(workflows)` — parses `Role (Stage)` column names (across all
  workflows) and buckets the non-done states by role, so a role's target is
  **all** of its non-done columns. Matching is by exact role equality (role
  `Dev` does not swallow `DevOps (Ready)`), and roles with only done-type states
  are excluded. (The old single-column `pickTargetState` helper is removed.)
- `resolveTargets(detected, roleName)` — returns the target column(s) for a role:
  all of a normal role's non-done columns, or every role's columns (deduped) for
  **Planner**. Columns are ranked **most-advanced-first** so the "next" ticket is
  the furthest-along one across the whole role, with urgent tickets always winning.
- `pickNext(stories, myId, targetStateIds, urgentLabel, stateRank?)` — filters to
  a member's active stories across one *or more* target columns and sorts (urgent
  first, then by cross-column rank, then by position). Team Manager runs one
  state-scoped search per distinct target column (bounded by roles, not members),
  so there are still no per-member API calls.
- `isPausedState(st)` — true when a state's stage (group 2 of `Role (Stage)`)
  matches `/paused/i`, so the paused rule works for any role generically.
- `pickNextPaused(stories, memberId, roleTargets, urgentLabel)` — **composes**
  `pickNext` over the non-paused and paused subsets of a role's columns and
  returns `{ next, queue, paused, all }`. `next`/`queue` are drawn only from the
  non-paused base (so paused is never "next"); `paused` is populated only when the
  member has no non-paused ticket strictly more advanced than the entry column.
- `buildStateNameById(workflows)` — flat `{ stateId: name }` lookup spanning every
  workflow, powering the workflow-state chips on queue rows.
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
