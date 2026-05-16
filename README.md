# worker_v1-example-reddit-engager-worker

Docker compose stack for the autonomous Reddit engager — a
Playwright-equipped Claude Code agent that, every 4 hours,
scrolls a Reddit home feed, picks an interesting post, reads its
comments, replies to a select few, commits an audit log, and
notifies `@clauderemote` of its activity.

Single-container architecture (no fan-out children) — multiple
parallel browser sessions on the same Reddit account would
themselves trigger anti-bot flags, so we use one body and let
cron pace the cycles.

## Companion repo

The agent's playbook + Playwright wrapper live in the sibling
repo:

```
https://github.com/clawborrator/worker_v1-example-reddit-engager-repo
```

`docker compose up` clones that repo into `/workspace/repo` and
hands control to the playbook in its `CLAUDE.md`.

## Setup (one time)

### 1. Capture Reddit cookies

Use a browser you're already logged into on the account you
want the engager to post from. Pick an export tool:

- Chrome / Edge: install **Cookie-Editor** extension →
  open reddit.com → click extension icon → Export →
  "Export as JSON"
- Firefox: install **Cookie Quick Manager** → Search "reddit.com"
  → Export selection

Save the result as:

```
./secrets/reddit.cookies.json
```

It's gitignored. Don't commit it.

If the JSON shape your extension exports isn't the Playwright
shape (`{name, value, domain, path, httpOnly, secure, expires}`
per cookie, top-level array), see the wrapper script's
`auth-check` command output for a hint — it'll tell you what's
missing.

### 2. Mint a clawborrator channel token

From your laptop (any host that has `claw` installed):

```bash
claw token mint --name reddit-engager
```

Copy the `ck_live_…` value.

### 3. Create a GitHub PAT for the audit log

The engager commits `data/posted/<date>.json` to its repo each
cycle. Needs `repo` scope. Generate at:

GitHub → Settings → Developer settings → Personal access tokens →
Tokens (classic) → Generate new token

### 4. Configure .env

```bash
cp .env.example .env
$EDITOR .env
```

Fill in: `CLAUDE_CODE_OAUTH_TOKEN`, `CLAWBORRATOR_TOKEN`,
`REPO_PAT`, `GIT_USER_EMAIL`, `GIT_USER_NAME`. Defaults are fine
for everything else.

### 5. Start

```bash
docker compose up -d
docker compose logs -f reddit-engager
```

First warmup cycle hits in ~30-60s. After that, `CronCreate
0 */4 * * *` paces the recurring cycles.

## Stopping / restarting

```bash
docker compose down                  # stop + remove
docker compose up -d                 # restart (re-clones repo, fresh cron)
docker compose logs -f reddit-engager
```

A stop in the middle of a cycle is fine — any in-progress
Playwright browser is killed with the container, no orphan
processes. The next start runs a warmup cycle immediately.

## What to expect in `@clauderemote`

After each cycle, you'll get a message like:

> Posted on **r/programming**. On a thread *"Why I switched
> from Vim to Helix"*, replied to a comment by `u/modalfan`
> explaining the tradeoffs of modal editing once muscle memory
> kicks in.

If the cycle skipped (no interesting post found, or auth
expired), you'll get a brief explanation:

> Cycle skipped: cookies expired. Refresh `secrets/reddit.cookies.json`.

## When cookies expire

Reddit invalidates sessions on suspicious activity, password
changes, or just slow attrition. Realistic cookie lifetime when
posting (not just reading) is days-to-weeks.

When the engager notices a 401-equivalent on `auth-check`, it
will notify @clauderemote and skip cycles until you re-export
cookies and `docker compose restart reddit-engager` (no rebuild
needed — the cookies file is bind-mounted, so a fresh export +
restart is enough).

## See also

- `../worker_v1-playwright/` — the image this stack uses
- `../worker_v1-example-reddit-engager-repo/` — the agent's playbook
- `../worker_v1-example-heartbeat-worker/` — fan-out example (3
  children per cycle, periodic system metrics)
