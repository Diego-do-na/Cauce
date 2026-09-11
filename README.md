# Cauce

Cauce is a task orchestrator for a small team running Claude Code and
Cursor in parallel. **There is no central server.** Coordination happens
entirely through `tasks.yaml`, committed and pushed like any other file —
git and GitHub *are* the backend. The only thing anyone might leave
running is the boss dashboard (a read-only monitor); every script in
`scripts/` is a short-lived, one-off command run by whoever needs it.

```
                    ┌───────────────────────┐
                    │   GitHub (origin)     │
                    │   tasks.yaml + code   │
                    └───────────▲───────────┘
                     git push / │ \ git pull
                    ┌───────────┼──┼───────────────┐
                    │           │  │               │
             ┌──────┴─────┐ ┌───┴──┴────┐   ┌──────┴──────┐
             │ Dev A       │ │ Dev B      │   │ boss-dashboard│
             │ claim_task  │ │ claim_task │   │ (read-only,   │
             │ poller      │ │ finish_task│   │ local mirror  │
             │ Claude/Cursor│ │ Claude/Cursor│  │ clone, polls  │
             └─────────────┘ └────────────┘   │ every 5s)     │
                                                └───────────────┘
             ┌──────────────────────────┐
             │ model-requester           │  <- writes new tasks
             │ (form -> add_task.py ->   │     straight to tasks.yaml,
             │  commit + push)           │     never reads for monitoring
             └──────────────────────────┘

Nothing above talks to anything else directly — every arrow is a git
pull or a git push. The two dashboards are just local read/write views
into that same tasks.yaml.
```

## Install

Requires **Python 3.11+** and **git** on macOS, Windows, or Linux.

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Discord notifications (optional)

`scripts/notify_discord.py` never crashes your workflow if unconfigured —
with neither variable set, it just prints to the console. Pick one mode:

- **Webhook (posts to a channel):**
  ```bash
  export DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/..."
  ```
- **Bot DM (messages the task owner directly):**
  ```bash
  export DISCORD_BOT_TOKEN="your-bot-token"
  ```
  Also fill in `discord_map.yaml` at the repo root, mapping each
  teammate's name (as used for `owner` / `git config user.name`) to
  their Discord user id.

## Day-to-day workflow

1. **Claim a task:**
   ```bash
   python scripts/claim_task.py --owner "Alice"   # or omit --owner to use git config user.name
   ```
   This finds the first `todo` task whose dependencies are all `done`,
   checks its `scope` doesn't collide with anything currently `claimed`,
   marks it claimed in `tasks.yaml`, and creates a worktree at
   `../task-<id>` on branch `task/<id>`. It prints the task's full
   description and worktree path.

2. **Do the work.** `cd ../task-<id>` and point Claude Code or Cursor at
   the printed description. Commit as you go, inside that worktree.

3. **Finish the task**, from your main checkout (not the worktree):
   ```bash
   python scripts/finish_task.py --task-id T002
   ```
   This test-merges your branch against `main` in an isolated temp
   clone first. If it conflicts, **nothing is pushed** and the task
   stays `claimed` — resolve the conflict by hand, then re-run the same
   command. If it merges cleanly, your branch is pushed, `tasks.yaml` is
   updated to `done`, and any tasks that just became unblocked are
   printed (they are not auto-claimed).

4. **Or let the poller do the claiming for you:**
   ```bash
   python scripts/poller.py --interval 30 --owner "Alice"
   ```
   Every interval it pulls, looks for an eligible task, and claims one
   automatically if you don't already have a claim in progress. It
   rings the terminal bell and prints `TASK READY: ...` — it never
   starts Claude Code or Cursor itself; you still do that by hand.

5. **Add a new task** (or use the model-requester form below):
   ```bash
   python scripts/add_task.py \
     --title "Add pagination to /tasks" \
     --description "..." \
     --scope src/api/tasks.py --scope tests/test_tasks.py \
     --depends-on T001 \
     --suggested-model claude-sonnet
   ```

**Hard rule:** never hand-edit `tasks.yaml`. All three scripts above
handle concurrent pushes safely (`git pull --rebase` + retry, up to 5
attempts); a hand edit can race with someone else's claim and corrupt it.

## Running the two dashboards

They are separate apps, on separate ports, and neither needs to run
24/7 unless someone wants it available continuously.

**boss-dashboard** (read-only monitor, port 8000):
```bash
cd boss-dashboard
uvicorn app:app --port 8000
```
Open http://localhost:8000. It keeps its own local mirror clone
(`boss-dashboard/_mirror` by default — see `CAUCE_REPO_URL` /
`CAUCE_MIRROR_PATH` env vars to point it elsewhere) so it never touches
anyone's actual working checkout. It polls `/api/state` every 5 seconds:
a kanban board (todo/claimed/done), a progress bar, and an activity feed
showing time-since-last-commit per active branch. No websockets — pure
polling, no write endpoints.

**model-requester** (write-only task intake, port 8001):
```bash
cd model-requester
uvicorn app:app --port 8001
```
Open http://localhost:8001. A single form — title, description, one or
more scope paths, a multi-select of existing tasks for `depends_on`, and
a suggested-model dropdown — that calls `add_task.py`'s logic on submit,
commits, pushes, and shows the generated task id. It never polls for
monitoring; the only `GET` it exposes just lists current task ids/titles
to populate the dependency dropdown.

## Repo layout

```
tasks.yaml              the task board (never hand-edit)
discord_map.yaml         name -> Discord user id, for bot-DM mode
AGENTS.md / CLAUDE.md    shared conventions for coding agents
.claude/skills/          placeholder Skills for the team to fill in
scripts/                 claim_task.py, finish_task.py, add_task.py,
                         poller.py, check_merge.py, notify_discord.py
boss-dashboard/          read-only monitor (FastAPI, port 8000)
model-requester/         write-only task intake (FastAPI, port 8001)
```
