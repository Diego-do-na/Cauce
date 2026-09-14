# Cauce

Cauce is a task orchestrator for a small team running Claude Code and/or
Cursor in parallel. **There is no central server.** Coordination happens
entirely through `orchestration/tasks.yaml`, committed and pushed like any
other file — git and GitHub *are* the backend. The only thing anyone might
leave running is the boss dashboard (a read-only monitor); every script
under `orchestration/scripts/` is a short-lived command run by whoever
needs it, or by the wrapper scripts below on your behalf.

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
             │ work.sh /   │ │ autopilot.sh│  │ (read-only,   │
             │ autopilot.sh│ │ + Claude/Cursor│ local mirror  │
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

## Using this as a template for a new project

This repo is meant to be **copied wholesale** into a new hackathon/project
repo (or cloned and repointed at a new `origin`), not rebuilt from scratch:

1. Copy everything here into the new repo, alongside your actual product
   code (`src/`, `api/`, whatever it ends up being called) — `orchestration/`
   is entirely self-contained and never assumes anything about the rest of
   the tree beyond "it's a git repo with an `origin` remote".
2. Fill in the TODOs in `AGENTS.md` (stack, build/test commands) and in
   `.claude/skills/*/SKILL.md` for the new project's actual conventions.
3. Replace the two example tasks in `orchestration/tasks.yaml` with a real
   first batch — read `orchestration/HOW_CAUCE_WORKS.md` first (the
   mechanical ground truth: how claiming/scope/depends_on/test_paths
   actually behave), then see `orchestration/EXAMPLE_PROMPTS.md` for a
   task-writing example and a ready-to-use prompt that generates the whole
   initial board
   from your `AGENTS.md`/spec in one shot.
4. Run `./orchestration/scripts/setup.sh` (once per machine) and you're
   coordinating for real. If the team wants a per-hackathon runbook
   (Discord setup with this project's real repo URL/names, a day-to-day
   walkthrough) beyond what this README already covers generically, ask
   Claude Code to write one once the project exists — it'll have the real
   details to fill in, instead of a template guessing at them in advance.

## Install

Requires **Python 3.11+** and **git** on macOS, Windows, or Linux.
`./orchestration/scripts/setup.sh` does the steps below for you (creates
`orchestration/.venv`, installs deps, and sets `git config user.name` if
unset) — run it manually only if you want more control:

```bash
cd orchestration
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Discord notifications (optional)

`orchestration/scripts/notify_discord.py` never crashes your workflow if
unconfigured — with neither variable set, it just prints to the console.
Pick one mode:

- **Webhook (posts to a channel):**
  ```bash
  export DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/..."
  ```
- **Bot DM (messages the task owner directly):**
  ```bash
  export DISCORD_BOT_TOKEN="your-bot-token"
  ```
  Also fill in `orchestration/discord_map.yaml`, mapping each teammate's
  name (as used for `owner` / `git config user.name`) to their Discord
  user id.

Both go in `orchestration/.env` (copy from `orchestration/.env.example`) —
gitignored, one per machine, only the machine sending notifications needs
it filled in.

## Day-to-day workflow

The one-command path (recommended — handles worktree sync, model
selection, and the initial prompt for you):

```bash
./orchestration/scripts/work.sh                    # claim next task, launch `claude` in its worktree
./orchestration/scripts/work.sh '' cursor-agent    # ...or launch Cursor instead
./orchestration/scripts/finish.sh T002             # mark it done from anywhere (worktree included)
./orchestration/scripts/autopilot.sh               # loop: claim -> agent -> your review -> finish -> repeat
./orchestration/scripts/fleet.sh                   # one autopilot.sh loop per agent CLI, in parallel
./orchestration/scripts/dashboard.sh               # (one person) serve the board monitor
```

None of these bypass permission prompts — the agent still asks before every
action, exactly as if you'd launched it by hand. `autopilot.sh` only
automates the claim → launch → (**you** review the diff) → finish → claim-next
cycle; it stops and asks you to confirm the work stayed inside the task's
declared scope before it ever runs `finish_task.py`.

The manual path, if you want to see every step:

1. **Claim a task:**
   ```bash
   python orchestration/scripts/claim_task.py --owner "Alice"   # or omit --owner to use git config user.name
   ```
   This finds the first `todo` task whose dependencies are all `done`,
   checks its `scope` doesn't collide with anything currently `claimed`,
   marks it claimed in `tasks.yaml`, and creates a worktree at
   `../task-<id>` on branch `task/<id>`. It prints the task's full
   description and worktree path.

2. **Do the work.** `cd ../task-<id>` and point Claude Code or Cursor at
   the printed description. Commit as you go, inside that worktree.

3. **Finish the task**, from your main checkout (not the worktree — or
   just run `finish.sh`, which does this from anywhere):
   ```bash
   python orchestration/scripts/finish_task.py --task-id T002
   ```
   This refuses if the branch has no commits beyond `main` (nothing to
   finish). If the task declared its own `test_paths` (not every task
   needs one — see `orchestration/EXAMPLE_PROMPTS.md`), it re-runs exactly
   those (via `CAUCE_TEST_CMD`, default `pytest`) in the task's own
   worktree first — a failure blocks the same way a conflict does: nothing
   pushed, task stays `claimed`, fix and re-run. Then it test-merges your
   branch against `main` in an isolated temp clone. If it conflicts,
   **nothing is pushed** and the task stays `claimed` — resolve the
   conflict by hand, then re-run the same command. If it merges cleanly:
   your branch is pushed, **merged into `main` for real and pushed** (not
   just a status flip), the now-redundant worktree and branch are cleaned
   up, `tasks.yaml` is updated to `done`, and any tasks that just became
   unblocked are printed (they are not auto-claimed).

4. **Or let the poller do the claiming for you:**
   ```bash
   python orchestration/scripts/poller.py --interval 30 --owner "Alice"
   ```
   Every interval it pulls, looks for an eligible task, and claims one
   automatically if you don't already have a claim in progress. It rings
   the terminal bell and prints `TASK READY: ...` — it never starts Claude
   Code or Cursor itself; you still do that by hand (or use `work.sh`
   instead of the poller, which does start the agent for you).

5. **Add a new task** (or use the model-requester form below):
   ```bash
   python orchestration/scripts/add_task.py \
     --title "Add pagination to /tasks" \
     --description "..." \
     --scope src/api/tasks.py --scope tests/test_tasks.py \
     --test-path tests/test_tasks.py \
     --depends-on T001 \
     --suggested-model sonnet
   ```
   `--test-path` is optional and repeatable — only add it for tasks worth
   gating on a real test run (see point 3 above and
   `orchestration/EXAMPLE_PROMPTS.md`).
   See `orchestration/EXAMPLE_PROMPTS.md` for what a task worth claiming
   actually looks like, and for a prompt that generates a whole first batch
   from your project's spec.

**Hard rule:** never hand-edit `tasks.yaml`. Every script above handles
concurrent pushes safely (`git pull --rebase` + retry, up to 5 attempts,
plus a machine-local lock so two processes sharing one checkout — e.g. two
`fleet.sh` agents — never race each other's git commands); a hand edit can
still race with someone else's claim and corrupt it.

## Running the two dashboards

They are separate apps, on separate ports, and neither needs to run 24/7
unless someone wants it available continuously.

**boss-dashboard** (read-only monitor, port 8000):
```bash
./orchestration/scripts/dashboard.sh
```
Open http://localhost:8000. It keeps its own local mirror clone
(`orchestration/boss-dashboard/_mirror` by default — see `CAUCE_REPO_URL` /
`CAUCE_MIRROR_PATH` env vars to point it elsewhere) so it never touches
anyone's actual working checkout. It polls `/api/state` every 5 seconds: a
kanban board (todo/claimed/done), a progress bar, and an activity feed
showing time-since-last-commit per active branch. No websockets — pure
polling, no write endpoints.

**model-requester** (write-only task intake, port 8001):
```bash
cd orchestration/model-requester
uvicorn app:app --port 8001
```
Open http://localhost:8001. A single form — title, description, one or
more scope paths, a multi-select of existing tasks for `depends_on`, and a
suggested-model dropdown — that calls `add_task.py`'s logic on submit,
commits, pushes, and shows the generated task id. It never polls for
monitoring; the only `GET` it exposes just lists current task ids/titles
to populate the dependency dropdown.

## Repo layout

```
AGENTS.md / CLAUDE.md        shared conventions for coding agents (fill in the TODOs)
.claude/skills/               placeholder Skills for the team to fill in
.github/workflows/ci.yml      placeholder CI (adjust to the real stack)
orchestration/                everything Cauce — self-contained, never product code
  tasks.yaml                  the task board (never hand-edit)
  discord_map.yaml            name -> Discord user id, for bot-DM mode
  HOW_CAUCE_WORKS.md          mechanical ground truth: claiming/scope/depends_on/test_paths
  EXAMPLE_PROMPTS.md           a well-written task, and a board-bootstrapping prompt
  scripts/
    setup.sh                  one-time per machine: venv, deps, git user.name
    work.sh                   claim the next task and launch the agent in its worktree
    finish.sh                 mark a task done, from anywhere
    autopilot.sh              the claim -> agent -> your review -> finish loop
    fleet.sh                  one autopilot.sh loop per agent CLI, in parallel
    dashboard.sh              serve boss-dashboard
    claim_task.py / finish_task.py / add_task.py / poller.py
    check_merge.py / notify_discord.py / task_log.py / locked_git.py / common.py
  boss-dashboard/             read-only monitor (FastAPI, port 8000)
  model-requester/            write-only task intake (FastAPI, port 8001)
```
