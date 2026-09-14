# EXAMPLE_PROMPTS.md — how to actually write for Cauce

Two examples. The first shows the level of detail a single task's
`description` needs to survive being handed to an agent with zero other
context. The second is the prompt you give *yourself* (in Claude Code, at
the repo root, before anyone claims anything) to turn a fresh project's
`AGENTS.md` into the first real batch of tasks in `tasks.yaml`.

Both come from what actually worked running Cauce under a real hackathon
deadline — not aspirational advice, examples calibrated on what an agent with
zero other context actually needed to not go off scope.

---

## 1. A well-written task

An agent that claims a task sees **only** its `title` + `description` (that's
what `work.sh`/`autopilot.sh` pass as the initial prompt) and its `scope`
(which files it's allowed to touch). It cannot see other tasks, cannot see
your mental model of the architecture, and will not ask a human before
guessing at anything left ambiguous. Write for that.

The shape that held up: **(a) BUILD** — exactly what to create/change, with
concrete names, not vague nouns; **(b) VERIFY** — a command or check the
agent itself can run to know it succeeded, so "done" isn't a matter of
opinion; **(c) DOCS** — what to write down for the *next* task/agent that
depends on this one, since they won't have this session's context either.

```yaml
- id: T014
  title: "POST /detect — defensive request parsing for all 4 input shapes"
  description: >
    (a) BUILD — Implement request parsing for POST /detect that accepts,
    without any client-side configuration, all four shapes: (1) a raw
    base64 string as the entire body: text/plain body, no wrapper; (2)
    JSON with the audio under any ONE of these keys: audio, audio_base64,
    wav, data, file, clip, content — check them in that order, first
    match wins; (3) multipart/form-data with a single file field (any
    field name); (4) raw binary WAV (Content-Type audio/wav or
    application/octet-stream, or no Content-Type header at all). On
    anything that isn't one of these four (empty body, corrupt base64,
    an unparseable multipart body, JSON with none of the known keys),
    return the fallback verdict {"is_synthetic": false, "confidence":
    0.5} with HTTP 200 — never a 4xx/5xx from this endpoint — and log
    the parse failure with tracing::warn! including which of the 4
    shapes was attempted and why it failed. Put the shape-detection
    logic in src/http/parse.rs as parse_detect_request(headers, body) ->
    Result<Vec<u8>, ParseError>; src/routes/detect.rs only calls it and
    maps Err to the fallback response.

    (b) VERIFY — cargo test parse_request runs 4 passing cases (one per
    shape above) plus 3 failing cases (empty body, garbage base64,
    JSON with an unrecognized key only) and asserts the failing cases
    still produce HTTP 200 with the exact fallback body, not a 4xx.

    (c) DOCS — src/http/parse.rs's module doc comment: one line per
    shape naming which field/content-type it matches, so T015 (which
    calls this from /analyze too) doesn't have to re-read the match
    arms to know what's supported.
  scope:
    - src/http/parse.rs
    - src/routes/detect.rs
    - tests/parse_request_test.rs
  depends_on: ["T009"]
  test_paths:
    - tests/parse_request_test.rs
  suggested_model: sonnet
```

Notice what's absent: no explanation of *why* `/detect` must always return
200 (that belongs in `AGENTS.md`/the spec, which the agent is expected to
have read as project context — don't re-derive frozen contracts inside a
single task). The task only says what THIS task builds, how to check it,
and what to leave behind for the next one.

`test_paths` and the (b) VERIFY line aren't the same mechanism, and both
matter: (b) is what the agent reads and acts on *during* the session —
free text, whatever proves the work to a human skimming it later. `test_paths`
is what `finish_task.py` itself *enforces*, mechanically, before it will
merge anything — it re-runs exactly those paths (via `CAUCE_TEST_CMD`,
default `pytest`; this example assumes the project set it to `cargo test`
for a Rust codebase) and refuses to proceed on a nonzero exit, no human
judgment involved. Not every task needs the second one — a task that only
scaffolds config files has nothing worth gating a merge on; a task whose
whole point is a measurable property (an endpoint's latency, a parser's
behavior on malformed input) should have one.

What made tasks fail in practice, in order of how often it happened:
a vague `scope` (agent edits a file another claimed task also touches →
`claim_task.py`'s conflict check saves you, but only if scopes were listed
completely in the first place); no VERIFY step (agent declares victory on
vibes, `finish_task.py` merges it, the bug surfaces two tasks later); and
a `description` that assumes context from a conversation that already
scrolled out of the agent's window.

---

## 2. Bootstrapping a new project's task board

Run this once, at the very start of a new hackathon/project, in a normal
Claude Code session at the repo root — **not** through Cauce itself (there's
nothing to claim yet). Point it at whatever design doc/spec you have; if you
only have a rough idea, describe it inline instead of citing a file.

```
Read orchestration/HOW_CAUCE_WORKS.md in full first — it's the mechanical
ground truth for how claiming, scope, depends_on, and test_paths actually
behave. Don't infer any of that from this prompt or from tasks.yaml's
shape; that doc is more precise than either.

Then read AGENTS.md and <path to the spec/design doc, or paste a
description of the project here> in full.

Break the MUST/SHOULD/COULD scope into a task board for orchestration/tasks.yaml,
one task per meaningful unit of work, using add_task.py for every task (never
hand-edit tasks.yaml — see AGENTS.md's Coordination rules).

For each task:
- title: short, specific, names the concrete deliverable.
- description: (a) BUILD exactly what to create/change with concrete
  file/function names, not vague nouns — the agent that claims this sees
  ONLY this text, nothing else; (b) VERIFY a command or check the agent
  can run itself to know it's done, not a matter of opinion; (c) DOCS what
  to leave written down for whichever later task depends on this one.
- scope: the exact file/directory paths this task touches — err on the
  side of listing more, not fewer, since two tasks with overlapping scope
  can never be claimed at the same time (that's the whole point).
- depends_on: every task that must be `done` first — sequence the MUST
  items before SHOULD before COULD, and split anything that would force a
  lower-priority item to start before a higher-priority one is finished.
  Among tasks with no dependency relationship to each other, add the ones
  you want attempted first earlier in the file — that's the only tiebreak
  that exists among equally-eligible tasks (see HOW_CAUCE_WORKS.md §1).
- test_paths: OPTIONAL, leave empty for most tasks. Only add it when the
  task's whole point is a measurable property a real test can pin down (an
  endpoint's latency, a parser's behavior on malformed input, a
  calculation's correctness) — not for scaffolding, config, or anything
  whose "done" is really a human judgment call. When set, finish_task.py
  re-runs exactly these paths (via CAUCE_TEST_CMD, default `pytest`) and
  refuses to merge the task's branch on a nonzero exit. **Whenever you set
  this, also create the actual test file(s) at those paths yourself, in
  this same pass** — a real test with real assertions where the spec is
  already unambiguous, or a skeleton with a clear comment on what's still
  undetermined otherwise (see HOW_CAUCE_WORKS.md §5). Never just declare
  the path and leave writing the test to whoever claims the task — that's
  the agent grading its own homework.
- suggested_model: haiku for mechanical/boilerplate work, sonnet for real
  design-and-implementation work, opus only for genuinely hard/high-stakes
  tasks (these are Claude Code's own --model aliases — see work.sh's
  resolve_model_flag if you're also planning to run cursor-agent, which
  uses a completely different model catalog).

Never write a task expecting a specific person to do it, or note anywhere
that it's "reserved" for someone — there is no such mechanism (see
HOW_CAUCE_WORKS.md §1). Any eligible task can be claimed by whichever
agent runs next, regardless of who you had in mind.

Keep scopes non-overlapping across everything you'd expect to run in
parallel — that's what actually lets multiple agents work at once without
merge conflicts. If finishing one piece of work will obviously surface a
follow-up (e.g. a schema task revealing the need for a query layer), make
that follow-up its own task with depends_on, instead of quietly growing
the first task's scope.

Print the final list of task ids + titles + depends_on as a table before
adding anything, so I can sanity-check the sequencing first.
```

That last line matters: reviewing the planned board *before* forty
`add_task.py` calls fire off is much cheaper than discovering a bad
dependency chain after three agents have already claimed against it.
