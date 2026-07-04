# task-handoff

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) for a **dual-brain workflow**: one agent plans (e.g. Claude Code), a *different* executor does the work (whatever model/agent/tool you name — Qoder, Qwen, Codex, Gemini…), and they hand a task off through files — never talking directly.

You finish planning and just say *"hand this to Qoder"*. The skill then:

1. Creates a task folder named after that executor,
2. Writes a scoped `PROMPT.md` constraining exactly what it may do,
3. Prints a **ready-to-paste launch prompt** you forward to the executor.

Later you say *"review the Qoder tasks from last night"* and it scans that executor's folders, reads each `DONE.md`, and reports back in one pass.

The executor name is **whatever you say at dispatch time** — the names above are only examples, never a fixed list. Say "hand this to X" and `X` becomes the folder name and the target of the launch prompt.

## Why

If you run a planner/executor split — one agent thinks, another (often cheaper/different) does the work — the handoff has to be reliable without the two agents ever talking:

- The executor must **understand the task with no back-and-forth**.
- It must **stay in its lane** — never touch files outside the assigned scope.
- You must be able to **review a whole batch by executor**, not chase down individual task names.

`task-handoff` is a lightweight convention + skill instructions for exactly this. It is deliberately *not* a full orchestration framework (no CLI, no message bus, no Stage/Task numbering). If you need that, look at [GitHub Spec-Kit](https://github.com/github/spec-kit) (single-agent spec→plan→tasks→implement) or [Agentic Project Management](https://github.com/sdi2200262/agentic-project-management) (multi-agent orchestration with git-worktree isolation). This skill covers the narrow case: **one task, an executor you name, file-driven handoff.**

## Folder layout

```
<project-root>/tasks-for-<executor>-night-<N>/
    task-<seq>_<slug>/
        PROMPT.md                       ← planner writes this (the constraint)
        <executor>-<YYYYMMDD-HHMM>/
            DONE.md                     ← executor writes this (the result)
```

- `<project-root>` is **resolved at runtime** — the git root of the current working directory (`git rev-parse --show-toplevel`), falling back to the cwd if not in a repo. Never hardcoded; the task folders always live inside whatever project you're currently in, the same directory the executor shares.
- `<executor>` is the name you say, slugified.
- `<N>` is the night batch — reused within one dispatch session, incremented for a new night.
- `<seq>` is a globally incrementing task number across all nights.

Grouping by executor + night is what makes *"review last night's Qoder tasks"* a one-line scan.

## Flow

**Dispatch** — you say *"hand this to Qoder"*:
1. Classify the task — `deterministic` (path is clear) or `exploratory` (feasibility unknown, run a minimal demo first).
2. Auto-create the task folder and write `PROMPT.md`: goal, definition of done, scope (allowed paths only), **self-contained** background (no "see the design doc" — the executor can't read your other files), how to verify, and the requirement to write `DONE.md`.
3. Print a launch prompt you paste to the executor — it contains the absolute path to the task folder and the scope rule.

**Execute** — the named executor reads `PROMPT.md`, works only inside `scope`, and writes `DONE.md` into a timestamped subfolder: what changed, how to verify (reproducible steps + actual results), and `status: success | partial | failed`.

**Review** — you say *"review last night's Qoder tasks"*:
1. Scan `tasks-for-qoder-night-*/` for the matching night.
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
