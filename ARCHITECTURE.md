# Architecture Guide — Next Ticket (Shortcut Nudger)

> **Read this first.** This is the fast-orientation doc for any agent working on
> this repo. It explains *what* the app is, *how* it's built, and *where* things
> live so you can make correct changes without re-reading the whole file.
> Keep it accurate — see [Keeping this guide accurate](#keeping-this-guide-accurate).

---

## 1. What it is (in one breath)

A **zero-install, single-page web app** that answers *"which ticket do I work on
next?"* by reading a [Shortcut](https://www.shortcut.com/) board via its REST API
and applying a fixed business rule. It also shows what a teammate, a team, or a
role is working on right now, across four tabs.

- **No backend, no build step, no framework, no dependencies.** The entire app is
  one file: [`index.html`](./index.html) (~1850 lines: HTML + CSS + vanilla JS).
- The browser talks **directly** to `https://api.app.shortcut.com/api/v3`.
  Shortcut allows CORS for token-authenticated requests, so no proxy is needed.
- Auth is a **personal Shortcut API token** the user pastes once. It lives only
  in `localStorage` and is sent only to Shortcut.
- Hosted on **GitHub Pages** (a real `https` origin — required for reliable
  token-auth CORS; `file://` sends `Origin: null` and is flaky).

**Live URL:** <https://danwatts81.github.io/shortcut-nudger/>

---

## 2. The core business rule (the whole point of the app)

Board convention: workflow **state (column) names are `Role (Stage)`** —
e.g. `Dev (Ready)`, `Dev (Active)`, `QA (Active)`, `Design (Paused)`. Roles are
**auto-detected** from these names via the regex `ROLE_STATE_RE`.

1. **Target column** = the **right-most (furthest-along) non-done** state for the
   selected role. A role's target is *all* its non-done columns; the pick prefers
   the most-advanced one.
2. Among a member's stories in the target column(s), work **top-to-bottom** by
   board `position`. Across multiple columns (Planner / mixed teams), order by
   column board position first, then position within the column.
3. **Urgent exception:** a story with the `urgent` label jumps the queue.
4. The single top-ranked story is shown as **"Work on this next."**
5. **Paused rule:** any state whose *stage* matches `/paused/i` is "paused." A
   paused ticket is **never** "next" (even if urgent). Paused tickets surface in a
   secondary dashed section *only* when the member has no non-paused ticket
   strictly more advanced than the entry/"Ready" column.

The **Planner** synthetic role spans every role's target column for a cross-role
overview. Columns not matching `Role (Stage)` (Inbox / Backlog / Completed) are
ignored — and additionally, only the **Development** workflow is used (see §4).

---

## 3. The four tabs (UI surfaces)

| Tab | Selector(s) | What it shows |
| --- | --- | --- |
| **My Queue** | Role dropdown | *Your* next ticket + rest of your queue for the chosen role. |
| **My Teammates** | Teammate dropdown | Any teammate's next + queue (view-only; your token/identity never changes). Uses their assigned role, falling back to yours. |
| **Team Manager** | Team dropdown | Every member's Now / Next / paused work; per-member inline role dropdowns. |
| **Role Manager** | Role dropdown | Everyone assigned to a role + "Missing Role Assignment" list + hidden-others toggle. |

The active tab and every selector persist in `localStorage`.

---

## 4. Code structure inside `index.html`

The file is three concatenated blocks. Line numbers are approximate — grep for the
named markers/functions rather than trusting them.

```
<style> ............... all CSS (dark theme, CSS custom props in :root)
<body> ................ static DOM skeleton (#app, #metaBar, #banner, modal, API tray)
<script>
  CONFIG .............. single tweakable-knobs object (see §5)
  PURE LOGIC .......... no DOM, no network — unit-testable, Tampermonkey-reusable
  API LAYER ........... apiFetch + typed getters
  STATE + RENDER ...... the `state` object + all render/wire functions
  refresh() / boot() .. orchestration + polling
  static wiring ....... footer, modal, API tray event listeners
</script>
```

### 4a. Pure logic (no DOM / no network — reuse verbatim)
Marked with the `PURE LOGIC` banner comment. These are the testable heart:

- `detectRoles(workflows, excludedRoles)` — buckets non-done states by role from
  `Role (Stage)` names. Exact role match (`Dev` ≠ `DevOps`).
- `findDevWorkflow(workflows)` — picks the **Development** workflow by name
  (case-insensitive), falling back to the first workflow. **Only this workflow's
  states/roles drive the app.**
- `resolveTargets(detected, roleName)` — target state array for a role; Planner
  flattens+dedupes all roles.
- `stateRankFromTargets(targets)` — `{stateId: rank}`, rank 0 = most advanced.
- `pickNext(stories, myId, targetStateIds, urgentLabel, stateRank)` — filters to a
  member's active stories and sorts (urgent → cross-column rank → position).
- `isPausedState(st)` — stage matches `/paused/i`.
- `pickNextPaused(...)` — composes `pickNext` over non-paused/paused subsets;
  returns `{ next, queue, paused, all }`.
- `buildStateNameById(workflows)` — flat `{stateId: name}` for state chips.
- `formatDuration` / `daysSince` / `formatBuildStamp` — display/time helpers
  (`nowMs` injected for testability).

### 4b. API layer
- `apiFetch(pathOrUrl, token)` — the one fetch chokepoint. Adds `token` as a query
  param (no custom headers → simple request → **no CORS preflight**). Logs every
  call to the **API tray** (token redacted), honors a shared `AbortController`
  (`apiAbort`) so the **Stop** button cancels in-flight requests, and maps
  401/403 to an `AUTH` error that bounces back to the token setup screen.
- Typed getters: `getMember` (identity from token), `getMembers`, `getGroups`
  (teams), `getWorkflows`, `searchStories` (cursor-paginated operator search).
- **Query broad, filter precise:** searches fetch by owner+state, then
  `pickNext()` filters client-side so an imperfect query can't corrupt ranking.

### 4c. State + render
- Single mutable `state` object (see it defined near the `STATE + RENDER` banner) —
  holds identity, cached members/groups, detected roles, per-tab selections,
  the API log, and loading/stopped flags. **No framework: renders are
  `innerHTML` string builders + explicit `wire*()` event re-binding after each
  paint.** If you add interactive DOM, you must add a matching `wire*` call.
- Persistence: thin `getStored*/setStored*` wrappers over `localStorage`, keyed by
  the `*_KEY` values in CONFIG.

### 4d. Orchestration
- `refresh(manual)` — the main loop. Resolves identity once, fetches workflows,
  scopes to the Development workflow, resolves roles/teams, then branches per tab
  to search stories and call the matching `render*`. Auto-refreshes every
  `POLL_MS`; `boot()` kicks it off after a token exists.

---

## 5. CONFIG — the only knobs

All behaviour tuning lives in the `CONFIG` object at the top of `<script>`:

| Key | Meaning |
| --- | --- |
| `VERSION` | Footer version string — **bump by hand per release**. |
| `WORKFLOW_NAME` | Only this workflow's states/roles are used (`"Development"`). |
| `EXCLUDED_ROLES` | Role names dropped from detection (Inbox/Backlog/Completed). |
| `COLUMN_PREFIX` | **Fallback seed only** — used when nothing is detected/stored. |
| `URGENT_LABEL` | Label that jumps the queue (`"urgent"`). |
| `POLL_MS` | Auto-refresh interval. |
| `STALE_DAYS` | Days-in-column before the "stale" highlight. |
| `API_BASE` | Shortcut API root. |
| `*_KEY` | `localStorage` keys (token, tab, roles, team, member-role map). |

The build timestamp in the footer is derived from `document.lastModified` — no
build step.

---

## 6. Key invariants & gotchas (don't break these)

- **One file, no dependencies.** Do not add a build step, framework, npm, or split
  files unless explicitly asked. The zero-install property is the product.
- **No custom fetch headers** — the token goes in the query string on purpose to
  keep requests "simple" and avoid CORS preflight. Adding an `Authorization`
  header will break CORS against Shortcut.
- **Token is a secret.** Never log it un-redacted, never send it anywhere but
  `api.app.shortcut.com`, never commit a real token.
- **Pure logic stays pure.** Keep the functions in §4a free of DOM/network so they
  remain testable and reusable by a future Tampermonkey overlay.
- **Re-wire after render.** Every `innerHTML` paint drops event listeners; the
  matching `wire*()` must run afterward.
- **Team Manager scales by #roles, not #members** — it runs one search per
  distinct target column, never per member. Keep it that way.
- **`state` is populated progressively** — `apiFetch` guards `typeof state` because
  it can be called before `state` is fully initialized.

---

## 7. Local development

Token-auth CORS needs a real `http(s)` origin (not `file://`):

```sh
python -m http.server 8000   # from repo root, then open http://localhost:8000/
```

Sanity checks: `/member` resolves your name in the header; the Role dropdown lists
auto-detected roles + Planner; the "next" card matches a manual board read.

---

## 8. Deployment (live testing happens here)

Committing to `main` and pushing auto-publishes to GitHub Pages (Settings → Pages,
deploy from `main` / root). After any change: **commit, push, and give the user the
live URL** so they can live-test:

**<https://danwatts81.github.io/shortcut-nudger/>**

---

## 9. Keeping this guide accurate

This document and [`README.md`](./README.md) are the project's memory. When you
change behaviour, structure, CONFIG, or the business rule, **update both in the
same change** so the next agent isn't misled. README is user-facing (how to use
it); this guide is agent-facing (how it's built and why).
