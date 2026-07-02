# task-handoff

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) for a **dual-brain workflow**: one agent plans (e.g. Claude Code), a *different* model executes (Qwen, Codex, Gemini…), and they hand a task off through files — never talking directly.

You finish planning and just say *"hand this to Qwen"*. The skill then:

1. Creates a task folder under the executor's name,
2. Writes a scoped `PROMPT.md` constraining exactly what that model may do,
3. Prints a **ready-to-paste launch prompt** you forward to the executor.

Next morning you say *"review yesterday's Qwen tasks"* and it scans that executor's folders, reads each `DONE.md`, and reports back in one pass.

## Why

If you run a planner/executor split — one agent thinks, another (often cheaper/different) does the work — the handoff has to be reliable without the two agents ever talking:

- The executor must **understand the task with no back-and-forth**.
- It must **stay in its lane** — never touch files outside the assigned scope.
- You must be able to **review a whole batch by executor**, not chase down individual task names.

`task-handoff` is a lightweight convention + skill instructions for exactly this. It is deliberately *not* a full orchestration framework (no CLI, no message bus, no Stage/Task numbering). If you need that, look at [GitHub Spec-Kit](https://github.com/github/spec-kit) (single-agent spec→plan→tasks→implement) or [Agentic Project Management](https://github.com/sdi2200262/agentic-project-management) (multi-agent orchestration with git-worktree isolation). This skill covers the narrow case: **one task, a known executor model, file-driven handoff.**

## Folder layout

```
<project-root>/handoffs/<executor>/<YYMMDD>_<slug>/
    PROMPT.md   ← planner writes this (the constraint)
    DONE.md     ← executor writes this (the result)
```

Grouping by `<executor>` + date is what makes *"review yesterday's Qwen tasks"* a one-line scan.

## Flow

**Dispatch** — you say *"hand this to Qwen"*:
1. Classify the task — `deterministic` (path is clear) or `exploratory` (feasibility unknown, run a minimal demo first).
2. Auto-create `handoffs/qwen/<YYMMDD>_<slug>/` and write `PROMPT.md`: goal, definition of done, scope (allowed paths only), **self-contained** background (no "see the design doc" — the executor can't read your other files), how to verify, and the requirement to write `DONE.md`.
3. Print a launch prompt you paste to the executor — it contains the absolute path to the folder and the scope rule.

**Execute** — a separate model reads `PROMPT.md`, works only inside `scope`, writes `DONE.md`: what changed, how to verify (reproducible steps + actual results), and `status: success | partial | failed`.

**Review** — you say *"review yesterday's Qwen tasks"*:
1. Scan `handoffs/qwen/` for the matching date.
2. Read each `DONE.md`; cross-check its "how to verify" against the original in `PROMPT.md`.
3. Get a single summary: which succeeded, which are partial/failed (and why), which never reported back. A vague or mismatched write-up is treated as `partial`, not accepted at face value.

## Hard rules the skill enforces

- **No verification, no dispatch.** A task without "definition of done" + "how to verify" doesn't go out.
- **Scope isolation.** The executor only touches paths listed in `PROMPT.md`'s `scope`.
- **Self-contained prompts.** Background the executor needs is embedded in `PROMPT.md`, never referenced by path to another doc it can't read.
- **Shared root, no copy-paste context.** Both agents operate on the same project directory; the task files are the only channel.
- **Fixed fields.** Both files follow a fixed structure — easy for a machine to parse, easy for a human to skim.

## Install

```bash
git clone https://github.com/manwithshit/task-handoff.git ~/.claude/skills/task-handoff
```

Or drop it into a project-local `.claude/skills/task-handoff/` to scope it to one repo.

## Files

- [`SKILL.md`](SKILL.md) — the skill instructions Claude reads
- [`assets/templates/PROMPT.md`](assets/templates/PROMPT.md) — dispatch/constraint template
- [`assets/templates/DONE.md`](assets/templates/DONE.md) — report-back template

## License

MIT
