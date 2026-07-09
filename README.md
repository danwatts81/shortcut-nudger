# Next Ticket — Shortcut Nudger

A zero-install, single-page tool that answers one question:
**"which ticket do I work on next?"**

It reads your [Shortcut](https://www.shortcut.com/) board through the REST API and
applies a fixed business rule, so you don't have to eyeball columns every time.

There is no backend, no build step, and no browser extension — just one
`index.html`. Shortcut's API allows CORS for token-authenticated requests, so the
page talks to Shortcut directly from your browser.

## The rule

1. **Target column** = the **right-most workflow state whose name starts with
   "Dev"** (case-insensitive). If your board has several — e.g. `Dev Ready`,
   `Dev In Progress`, `Dev Review` — the furthest-along one wins.
2. Among **your** stories in that column, work **top-to-bottom** (board position).
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

Each teammate pastes **their own** token, so everyone sees their own queue.

## Configuration

All knobs live in a `CONFIG` block at the top of the `<script>` in `index.html`:

```js
const CONFIG = {
  COLUMN_PREFIX: "Dev",     // target column = right-most state whose name starts with this
  URGENT_LABEL: "urgent",   // label that jumps the queue
  POLL_MS: 30000,           // auto-refresh interval (ms)
  API_BASE: "https://api.app.shortcut.com/api/v3",
  TOKEN_KEY: "shortcut_nudger_token",
};
```

- Different column naming? Change `COLUMN_PREFIX` (e.g. `"In Dev"`).
- Different urgency label? Change `URGENT_LABEL`.
- Poll faster/slower? Change `POLL_MS`.

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
- The right-most `Dev*` state is chosen as the target column.
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
- `pickNext(stories, myId, targetStateId, urgentLabel)` — filters to your
  active stories in the target column and sorts (urgent first, then by position).

These are pure functions with no DOM or network dependencies, so a future
[Tampermonkey](https://www.tampermonkey.net/) overlay (a true in-page nudge via
`GM_xmlhttpRequest` + `@connect api.app.shortcut.com`) could reuse the same core
logic verbatim.

## Security note

The token grants access to your Shortcut workspace, so treat it like a password.
Using **per-user** tokens (rather than one shared workspace secret) keeps each
person's identity automatic and avoids embedding a shared full-access secret in a
public page.
