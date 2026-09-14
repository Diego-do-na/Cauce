# HOW_CAUCE_WORKS.md — read this in full before writing or regenerating `tasks.yaml`

This is for whichever model is about to turn a project's spec into a task
board (via the bootstrap prompt in `EXAMPLE_PROMPTS.md`). It documents what
Cauce's scripts actually do, mechanically — not what a reasonable
orchestrator *might* do. Every claim below is a description of real code
in `orchestration/scripts/`, not an aspiration. Don't infer Cauce's
behavior from the shape of `tasks.yaml` or from what would be convenient;
read this instead. Getting this wrong doesn't error out — it produces a
board that looks fine and behaves wrong, silently, which is the worst kind
of mistake to make here.

`EXAMPLE_PROMPTS.md` is the companion doc: it shows the *style* a good
task is written in and the literal bootstrap prompt to use. This doc is
the mechanical ground truth that style is built on.

---

## 1. Claiming is poll-based, first-come-first-served. There is no reservation.

`claim_task.py` picks the **first** task in `tasks.yaml`'s file order that
is `status: todo` with every `depends_on` entry `status: done`. That's the
entire selection logic — no priority field, no "assigned to" concept, no
notion of importance beyond position in the file and the dependency
graph. Any agent that happens to run `work.sh` / `autopilot.sh` /
`claim_task.py` next gets whatever is first-eligible, regardless of who
"was supposed to" do it.

**Never write a task expecting a specific person to do it, and never put
that expectation in the task's `description`.** It has no mechanical
effect — nothing reads it, nothing enforces it. A task's `owner` field is
`null` until the moment it's claimed, is set automatically to whoever
claimed it, and `add_task.py` refuses to let you set it any other way (it
always writes `owner: null` on creation — there is no `--owner` flag for
task *creation*, only for the CLI scripts that act as a specific person).
If a task genuinely needs a specific human's attention before any
automation touches it, the only real lever is a human claiming it by hand
(`claim_task.py --owner "X"`) *before* an autopilot/fleet loop is running
— not a note in the board asking politely.

Corollary: if you need task A done strictly before task B *and they'd
otherwise both be eligible at once*, that has to be a real `depends_on`
edge, not an assumption that whoever picks up the board will read them in
the "right" order. If A and B have no dependency relationship but you
still want A attempted first when both are eligible, put A earlier in the
file — file order is the tiebreaker among equally-eligible tasks, and it's
the only one that exists.

## 2. What `scope` actually gates — and what it doesn't

`scope` is a list of repo-relative paths (files, or directory prefixes
like `src/api/`). Two scopes "overlap" if any path in one equals or is a
parent/child of any path in the other (`common._paths_overlap`).

The conflict check (`claim_task.py`) only compares the candidate task's
scope against tasks that are **currently `status: claimed`** — not against
every other `todo` task in the board. This means:

- Two tasks with overlapping scope can both sit in `tasks.yaml` as `todo`
  at the same time with no error. The board itself is never validated for
  scope overlap as a whole.
- What actually gets refused is claiming a task whose scope overlaps a
  task someone is *currently, actively* working on. Once that in-progress
  task finishes (merges to main), the scope is free again and the other
  task becomes claimable.
- So sequential tasks that touch the same files don't need distinct
  non-overlapping scopes — they need a `depends_on` edge (see below) if
  the second one genuinely requires the first one's *result*, or nothing
  at all if they just happen to touch the same file but could be claimed
  in either order without conflict once neither is in flight.

Write `scope` for **what could realistically be claimed in parallel** —
err toward listing more paths than fewer for anything you expect several
agents to work on at the same time, since that's the only case this check
actually protects.

## 3. What `depends_on` actually gates — and how tasks "unlock" others

`dependencies_satisfied(task)` is true only when every id in `depends_on`
has `status: done` — meaning it was not just worked on, but successfully
merged into `main` by `finish_task.py` (a real git merge, not a status
flip). A task with any unfinished dependency is never eligible, full stop.

When `finish_task.py` finishes a task, it scans the whole board for any
`todo` task that lists the just-finished id in its own `depends_on` and
whose *other* dependencies (if any) are already `done` too, and prints
those as "newly unblocked." It never claims them automatically — that's
always a separate, later `claim_task.py` call by whatever agent runs next.

Use `depends_on` to encode every real ordering constraint: MUST before
SHOULD before COULD, a schema before the queries that read it, a parser
before the route that calls it. Don't rely on file order to express a
dependency that isn't actually one — file order only breaks ties among
tasks that are *already* independently eligible.

## 4. Granularity: what should be one task

One task is what a single agent session can claim, complete, verify
against its own description, and commit — entirely inside its own
worktree — without the file-in-progress state of some other claimed task.
Two failure modes to avoid, in order of how often they actually happen:

- **Too coarse**: a task that bundles several unrelated deliverables means
  an agent either does a partial job and calls it done, or takes far
  longer than the claim → work → review → finish cycle is built for,
  making the human review step at `finish` harder to actually check
  (`git diff --stat` stops being a quick skim once a task tries to be
  three things).
- **Too fine**: a task smaller than "one BUILD/VERIFY/DOCS unit" wastes
  the fixed overhead of claim → worktree → agent launch → human review →
  finish on something that didn't need its own session, and multiplies
  the total number of merges (and thus test-gate + merge-conflict checks)
  for no real isolation benefit, since nothing else was going to run
  against the same files concurrently anyway.

Rule of thumb: if writing the task's VERIFY step honestly takes more than
one command/check, or the BUILD step needs "and then, separately," it's
probably two tasks with a `depends_on` edge between them, not one.

## 5. Writing real test files at board-authoring time, not just declaring paths

`test_paths` (optional, per task) is not merely a note — `finish_task.py`
literally re-runs `CAUCE_TEST_CMD` (default `pytest`) against exactly
those paths, in the task's own worktree, and refuses to merge on a
nonzero exit (see `EXAMPLE_PROMPTS.md` for when a task should declare one
at all).

If the same agent that implements a task also invents its own test for
it from scratch, the test can end up trivially satisfying itself — it's
checking whatever the agent happened to build, not whatever the task
actually required. Writing the test file for real, at the point the board
itself is authored (when the full spec is still in view, before any
implementation exists to bias it), removes that. The task then becomes
"make this specific, already-written test pass" instead of "decide for
yourself what passing means."

Two cases, handled differently:

- **The expected behavior is already fully specified** (an endpoint's
  exact response shape, a fixed input/output pair, a fixed status code) —
  write the complete test, with real assertions, at board-creation time.
  The task's own `description` doesn't need to re-explain what the test
  checks; the test file is the spec for that part.
- **The exact behavior can't be pinned down yet** (it depends on an
  implementation detail that doesn't exist until some other task builds
  it, or on a measurement that's only meaningful once something real
  exists to measure) — write the test as a skeleton: the real test
  function, a clear comment naming exactly what's still undetermined and
  why, and either `pytest.mark.skip(reason=...)` or an assertion against a
  placeholder the claiming agent is explicitly told (in the task's
  description) to replace. Never leave it silently passing on nothing —
  an empty test function that asserts nothing is worse than no test at
  all, since it looks like coverage and isn't.

Concretely, this means the board-authoring pass doesn't only call
`add_task.py` for every task — for every task with `test_paths`, it also
creates the actual file(s) at those paths (via normal file-write tools)
in the same pass, before any task is ever claimed.

**Make the test runner able to resolve project imports on its own, as
part of the very first scaffolding task — before any task declares
`test_paths`.** Confirmed by actually running this end to end: a
correctly-written test that does `from src.api.x import y` fails with
`ModuleNotFoundError`, not an assertion error, unless the repo root is on
`sys.path` — pytest does not add it automatically for a plain `src/`
layout with no `__init__.py` chain up to root. That failure looks
identical to a real bug in the terminal and costs real debugging time to
tell apart from one, exactly when time is scarcest. Fix it once, in
whatever the first scaffolding task creates (a `pyproject.toml` with
`[tool.pytest.ini_options]` / `pythonpath = ["."]`, or a root
`conftest.py`, or an installed editable package) — not per task, and not
as something each claiming agent has to rediscover.
