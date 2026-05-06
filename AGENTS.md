# AGENTS.md

## Project Rules

- Keep work direct and small.
- Do not over-engineer simple repo tasks.
- Do not delegate trivial inspection, Beads creation, or single-file edits to a subagent.
- Use subagents only when there are independent tasks that benefit from parallel execution.
- For Beads work in this repo, create or update the smallest useful issue set, then act.
- Do not wait on a subagent for a task that can be completed faster locally.
- Follow `PLAN.md` as the source of truth unless the user gives a newer explicit instruction.
- The intended boot fan speed is `40%`; the intended fallback fan speed is `60%`.
- Never alter `secrets.yaml`: do not write, delete, replace, re-encrypt, format, or use it as a workaround unless the user names `secrets.yaml` and approves the exact operation.
