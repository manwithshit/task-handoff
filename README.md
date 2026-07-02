# task-handoff

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) for handing off a single task between two agents that don't talk to each other — a **planning agent** (e.g. Claude Code, doing the thinking) and a separate **execution agent** (a different model, tool, or session, doing the work) — sharing the same project directory.

No live conversation between the two agents. No message bus. Just two files: `PROMPT.md` in, `DONE.md` out.

## Why

If you run a "dual-brain" setup — one agent plans and reviews, another (possibly cheaper/different) agent executes — you need the handoff to be reliable without the two agents ever talking directly:

- The execution agent must **understand the task without extra back-and-forth**.
- It must **stay inside its lane** — not touch files outside the assigned scope.
- The planning agent (or a human) must be able to **verify completion by reading one file**, without re-reading the whole session.

`task-handoff` is a lightweight convention + skill instructions for exactly this. It is deliberately *not* a full orchestration framework (no CLI, no Stage/Task numbering, no message bus) — if you need that, look at heavier tools like [GitHub Spec-Kit](https://github.com/github/spec-kit) (single-agent spec→plan→tasks→implement pipeline) or [Agentic Project Management](https://github.com/sdi2200262/agentic-project-management) (multi-agent orchestration with git worktree isolation). This skill covers the much narrower case: **one task, two known agents, file-driven handoff.**

## How it works

```
<project-root>/tasks/<slug>/PROMPT.md   ← planning agent writes this
<project-root>/tasks/<slug>/DONE.md     ← execution agent writes this
```

**Dispatch (planning agent):**
1. Classify the task — `deterministic` (path is clear, just dispatch) or `exploratory` (not sure it's feasible, run a minimal demo first).
2. Fill in `PROMPT.md`: goal, definition of done, scope (allowed paths only), self-contained background (no "see the design doc" references — the execution agent can't read your other files), how to verify, and the requirement to write back `DONE.md`.
3. Hand the task directory to the execution agent.

**Report (execution agent):**
1. Fill in `DONE.md`: what changed (file-level summary), how to verify (reproducible steps + actual results), and a conclusion with `status: success | partial | failed`.
2. Exploratory tasks report "can it be done + what are the gotchas + recommendation" instead of a full implementation.

**Review (planning agent / human):**
1. Read `DONE.md`'s `status` field.
2. Cross-check "how to verify" against the original "how to verify" in `PROMPT.md` — a vague or mismatched write-up should be treated as `partial`, not accepted at face value.
3. On `partial`/`failed`, open a new `PROMPT.md` with only the delta to fix — don't copy the whole original prompt.

## Hard rules the skill enforces

- **No verification, no dispatch.** A task without "definition of done" + "how to verify" doesn't go out.
- **Scope isolation.** The execution agent only touches paths listed in `PROMPT.md`'s `scope` field.
- **Self-contained prompts.** Background the execution agent needs is embedded directly in `PROMPT.md`, never referenced by path to another document.
- **Shared root, no copy-paste context.** Both agents operate on the same project directory; the task files are the only channel.
- **Fixed fields.** Both `PROMPT.md` and `DONE.md` follow a fixed structure — easy for a machine to parse, easy for a human to skim.

## Install

Copy (or symlink) this directory into your Claude Code skills folder:

```bash
git clone https://github.com/manwithshit/task-handoff.git ~/.claude/skills/task-handoff
```

Or drop it into a project-local `.claude/skills/task-handoff/` if you only want it scoped to one repo.

## Files

- [`SKILL.md`](SKILL.md) — the skill instructions Claude reads
- [`assets/templates/PROMPT.md`](assets/templates/PROMPT.md) — dispatch template
- [`assets/templates/DONE.md`](assets/templates/DONE.md) — report-back template

## License

MIT
