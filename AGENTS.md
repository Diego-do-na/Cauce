# AGENTS.md — shared conventions for Cauce contributors

## Stack

TODO: define stack.

## Build / test

TODO: define stack (commands below are placeholders until then).

```bash
pip install -r requirements.txt   # install shared deps for scripts + both apps
pytest                            # run tests
```

## Coordination rules

- **Never edit `tasks.yaml` by hand.** Only `scripts/claim_task.py`,
  `scripts/finish_task.py`, and `scripts/add_task.py` may write to it.
  Hand edits race with other people's pushes and will be overwritten or
  will corrupt someone else's claim.
- Claim a task with `python scripts/claim_task.py`, do the work in the
  worktree it creates, then finish with `python scripts/finish_task.py
  --task-id <id>`.
- One git branch per task: `task/<id>`.
- See `README.md` for full setup and workflow.
