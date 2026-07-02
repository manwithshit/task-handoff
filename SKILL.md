---
name: task-handoff
description: When the user finishes planning and says to hand a task off to another model/agent for execution ("把这个丢给千问干", "交给 codex 做", "让 gemini 去实现", "dispatch this to another model"), this skill auto-creates a per-executor task folder, writes a scoped PROMPT.md constraining what that model may do, and outputs a ready-to-paste launch prompt the user forwards to the executor. Also handles next-day batch review by executor ("验收昨天交给千问的任务", "看看 codex 那边做完没", "千问那些任务咋样了") — scanning that executor's recent task folders and reading each DONE.md. The executor is a separate model/session that shares the project directory; the two agents never talk directly. Not for compacting the current conversation into a session handoff.
---

# Task Handoff

规划由你（本 Agent）完成，执行交给另一个模型（千问/codex/gemini…）。两边不对话，靠共享目录里的文件交接。你负责：**下发时建好文件+吐转交 prompt**，**验收时按执行者批量看结果**。

## 下发：用户说「把这个丢给 X 干」

一句话触发（X = 千问/qwen、codex、gemini 等）。自动做三件事：

1. **建文件夹**（不存在则新建）：
   ```
   <project-root>/handoffs/<executor>/<YYMMDD>_<slug>/
   ```
   - `<project-root>`：**运行时解析当前项目根**，不要写死任何固定路径。取当前工作目录所在的 git 仓库根（`git rev-parse --show-toplevel`）；不在 git 仓库内则退回当前工作目录。执行侧与规划侧共享的就是这个根，`handoffs/` 是它下面的子目录。
   - `<executor>`：从话里提取（"千问"→`qwen`，"codex"→`codex`）
   - `<YYMMDD>`：今天日期
   - `<slug>`：简短 kebab-case 任务名

2. **写 `PROMPT.md`** 到该文件夹（模板见 [assets/templates/PROMPT.md](assets/templates/PROMPT.md)），约束执行侧：
   - frontmatter：`task` / `executor` / `dispatched`（日期）/ `type`（`deterministic` 或 `exploratory`）/ `scope`（允许改动的路径）/ `status: pending`
   - 正文：目标 / 完成定义 / 范围约束 / 背景（**摘录进正文，不引用外部文档**——执行侧读不到）/ 验证方式 / 产出要求（写回 DONE.md）
   - 硬约束：没有「完成定义+验证方式」不下发；`scope` 外的路径禁止动。

3. **吐一段转交 prompt** 直接输出在对话里，让用户复制粘贴给执行模型。格式：
   ```
   你有一个新任务。请先读取并严格遵守：
   <绝对路径>/handoffs/<executor>/<YYMMDD>_<slug>/PROMPT.md
   只允许改动 PROMPT.md 中 scope 列出的路径，禁止越界。
   完成后在同一文件夹写 DONE.md（含 status: success|partial|failed、改了什么、怎么验证、结论）。
   ```
   用绝对路径，执行模型才能直接定位。

> 若用户说的执行方是本环境能直接调用的工具（如 codex skill），可以直接调它执行，而不必让用户手动转交——按当前可用工具判断。默认（千问等无接口的模型）走"吐转交 prompt 让用户手动丢过去"。

## 验收：用户说「验收昨天交给千问的任务」

或任何提到某个非当前执行模型在处理的事项（"千问那些咋样了"、"codex 做完没"）。自动：

1. **定位执行者**：从话里提取（千问→`qwen`）。
2. **扫目录**：列出 `handoffs/<executor>/` 下匹配日期的所有任务文件夹（"昨天"→昨天日期前缀；没说日期就看最近未验收的）。
3. **逐个读**：每个文件夹读 `DONE.md`。
   - 有 `DONE.md` → 看 `status`，对照同目录 `PROMPT.md` 的「验证方式」核对「怎么验证」是否真做了、结果对得上。
   - 没 `DONE.md` → 标记为"未回/未完成"。
4. **汇总回报**：一次性给出——哪些 `success`、哪些 `partial`/`failed`（卡在哪、要不要你拍板）、哪些没回。逐条不用你再点名。
5. 不要只看 `status: success` 就放行——验证描述含糊或和 PROMPT 的验证方式对不上，按 `partial` 处理并说明。

`partial`/`failed` 且需要重做时：在同文件夹新写一版 `PROMPT.md`（只写这次要修正什么，不照抄原版），再吐一段转交 prompt。

## 模板

- [assets/templates/PROMPT.md](assets/templates/PROMPT.md) — 下发约束文件
- [assets/templates/DONE.md](assets/templates/DONE.md) — 执行侧回写文件（放模板供参考，执行侧照此产出）
