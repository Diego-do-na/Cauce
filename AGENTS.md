# AGENTS.md — shared conventions for this project's contributors

TODO: replace this whole top section with the real project's summary —
what it is, for which challenge/deadline, and the one or two sentences
that explain the core technical approach any agent should internalize
before touching code (the "read this before touching modeling/architecture
code" section Cauce itself doesn't need, but almost every real project
does).

## Repo layout

TODO: describe the real layout once product code exists. The one rule that
doesn't change regardless of what goes here: **`orchestration/`** is Cauce
— the task board, claim/finish scripts, the boss-dashboard and
model-requester web apps. It is coordination tooling, not product code. You
claim a task by running something under `orchestration/scripts/`, but the
*work itself* — everything a task's `scope` points at — happens outside
`orchestration/`. Don't add product code under `orchestration/`, and don't
add Cauce mechanics anywhere else.

## Stack

TODO: define stack.

## Build / test

TODO: define stack (commands below are placeholders until then).

```bash
pip install -r orchestration/requirements.txt   # Cauce scripts + boss-dashboard + model-requester deps
pytest                                           # run the real project's tests
```

## Coordination rules (Cauce)

- **Never edit `orchestration/tasks.yaml` by hand.** Only
  `orchestration/scripts/claim_task.py`,
  `orchestration/scripts/finish_task.py`, and
  `orchestration/scripts/add_task.py` may write to it. Hand edits race with
  other people's pushes and will be overwritten or corrupt someone else's
  claim.
- **One-time per machine**: `./orchestration/scripts/setup.sh` — creates
  `orchestration/.venv`, installs `orchestration/requirements.txt`, and
  sets `git config user.name` if unset (that name becomes the default
  Cauce owner).
- **The daily loop, one command each way**:
  - `./orchestration/scripts/work.sh [owner] [agent]` — claims the next
    eligible task, rebases its worktree onto the latest `origin/main`
    (always, every launch — a dependency's merged code needs this to
    actually show up in your files, not just in `tasks.yaml`'s status),
    and launches the agent (`claude` by default, pass `cursor-agent` as
    the second arg) directly inside it with `--model` set from the task's
    `suggested_model` and the task's title+description as its initial
    prompt. Safe to run from anywhere (it always resolves the main
    checkout first).
  - `./orchestration/scripts/finish.sh <task-id>` — refuses if the branch
    has no commits beyond `main` (nothing to finish). If the task declared
    its own `test_paths` in `tasks.yaml` (optional, per-task — not every
    task needs one), re-runs exactly those (via `CAUCE_TEST_CMD`, default
    `pytest`) in the task's own worktree first and refuses to proceed on a
    nonzero exit — same treatment as a merge conflict: nothing pushed,
    task stays claimed. Then if it merges cleanly: pushes the branch,
    **merges it into `main` for real and pushes `main`** (not just a
    status flip — this is what actually lands your code where every new
    worktree branches from and where anyone browses it on GitHub), then
    marks the task done in `tasks.yaml`. Safe to run from inside the
    task's own worktree (the common case) or from the main checkout. Run
    `python orchestration/scripts/task_log.py <task-id>` first if you want
    a deterministic summary of what a session actually did before
    finishing it.
  - `./orchestration/scripts/autopilot.sh [owner] [agent]` — the loop
    version of `work.sh`: claims a task, launches the agent interactively
    (normal permission prompts, nothing bypassed), and when the agent's
    session ends it stops and asks you to confirm the diff stayed inside
    the task's declared scope before running `finish_task.py` and moving
    to the next eligible task. One invocation, keeps going until you quit
    it. If it's interrupted (or you answer "quit") with a task still
    claimed, re-running it resumes that same task's worktree instead of
    claiming a new one.
  - `./orchestration/scripts/fleet.sh [owner] [agent...]` — runs one
    `autopilot.sh` loop per agent CLI **in parallel** (auto-detects
    `claude`/`cursor-agent` on PATH if none given), each in its own Terminal.app
    window (macOS; prints the commands to run yourself otherwise), each
    under its own Cauce owner identity (`<owner>-<agent>`) so the two
    loops never mistake each other's in-flight claim for their own. When
    one agent finishes its task before the other, it grabs the next
    eligible one immediately instead of waiting. Safe by construction —
    verified under an actual concurrent race, not just sequential turns —
    because Cauce's scope-conflict check and its pull/rebase/retry on
    every `tasks.yaml` write already serialize concurrent claims
    correctly; nothing here bypasses permission prompts.
  - `./orchestration/scripts/dashboard.sh` — one person runs this to serve
    the read-only board monitor at `http://localhost:8000`.
  - The raw Python entry points (`claim_task.py`, `finish_task.py`,
    `poller.py`, `add_task.py`, `notify_discord.py`, `check_merge.py`,
    `task_log.py`) still work directly under `orchestration/scripts/` if
    you need more control than the wrappers give — but only from the main
    checkout, never from inside a task's own worktree (they refuse with a
    clear error if run from the wrong place, since every git operation
    they do — including the merge into `main` — has to happen there).
- Every git command that mutates refs (pull/fetch/push/rebase/merge/worktree
  add) in the scripts runs under one machine-wide lock, `orchestration/.git.lock`
  (`common.local_repo_lock()`, re-entrant; shell scripts go through
  `orchestration/scripts/locked_git.py`). This is what lets two `autopilot.sh`
  loops (claude + cursor-agent under `fleet.sh`) share one checkout without
  "cannot lock ref" / "divergent branches" collisions. If you add a git call to
  a script, route it through `run_git` or `locked_git.py`.
- One git branch per task: `task/<id>`.
- Every task's `scope` in `tasks.yaml` is a set of non-overlapping file
  paths — this is what lets Cauce grant parallel claims across worktrees
  without merge conflicts. **This is a human/agent review responsibility,
  not something any script verifies for you** — whoever reviews a task
  before finishing it (in `autopilot.sh`'s prompt, or by hand) must check
  the diff only touches files inside the declared scope. If a task turns
  out to need work outside its own scope — e.g. finishing a DB schema
  surfaces the need for a query layer that's really a separate task —
  don't silently expand the current task's scope to cover it and don't let
  the agent edit those files. Either narrow the task's own Definition of
  Done so it self-verifies within its own scope (a schema task proves
  itself with its own fixture/migration test, not by writing the real
  query layer), or create the follow-up as its own task via `add_task.py`
  with `depends_on` pointing at the current one. Sequential dependency
  chains are the intended way to model "B needs A finished first" — not
  overlapping scope.
- TODO once the real team exists: agents' architectural decision
  authority, and who owns which kind of deviation (Cauce itself has none —
  this is a placeholder for whatever the real project's RACI ends up
  being).
- Discord notifications (`orchestration/scripts/notify_discord.py`) are
  optional and degrade to a console print when `DISCORD_WEBHOOK_URL` /
  `DISCORD_BOT_TOKEN` aren't set (put them in `orchestration/.env`, which
  is gitignored). Nothing in this repo requires Discord to function.

## Where things live

- Task board: `orchestration/tasks.yaml`.
- Mechanical ground truth for how claiming/scope/depends_on/test_paths
  actually behave — read before writing or regenerating the board:
  `orchestration/HOW_CAUCE_WORKS.md`.
- Task-writing example + a prompt that bootstraps a whole project's
  initial board from its spec: `orchestration/EXAMPLE_PROMPTS.md`.
- TODO: point at the real project's spec/design doc once one exists.
