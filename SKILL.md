---
name: task-handoff
description: Standardizes handing off a single task from a planning agent to a separate execution agent (a different model/tool/session sharing the same project directory) via structured PROMPT.md/DONE.md files, with scope constraints and verifiable completion criteria. Use when the user wants to dispatch a task to another agent for execution ("下发任务给执行侧", "交给另一个 agent/模型做", "写个 PROMPT.md 给执行 Agent", "把这个任务交接出去"), when reviewing results an execution agent produced ("验收 DONE.md", "看看执行侧做完了没", "review this handoff"), or when setting up a planner/executor file-driven workflow between two agents that don't talk to each other directly. Not for compacting the current conversation into a session handoff — that is a different, unrelated use case.
---

# Task Handoff

规划侧（本 Agent）与执行侧（另一个 Agent/模型/工具，例如跑在别的会话或工具里的执行者）不直接对话，全靠共享目录里的文件交接一个任务：规划侧写 `PROMPT.md` 下发，执行侧写 `DONE.md` 回写。目标是让执行侧读得懂、跑得动、回得来、不越界，人只读 `DONE.md` 就能验收。

## 硬约束（不可省略）

- **可验证才能下发**：没有「完成定义」和「验证方式」的任务不下发——先补齐再交接。
- **范围隔离**：`PROMPT.md` 的 `scope` 字段列出执行侧允许改动的路径，执行侧只能在这些路径内动手，禁止越界。
- **Self-contained**：执行侧读不到规划侧的其它文档（设计稿、聊天记录等）。凡是执行侧无法从代码库本身发现的背景信息（设计决策、约束、上游任务产出摘要），都要直接摘录进 `PROMPT.md` 正文，不要写"参见 xxx 文档"这种引用——执行侧不会去读。
- **共享目录**：规划侧和执行侧用同一个项目根，`PROMPT.md`/`DONE.md` 放在执行侧能直接读到的路径下，不靠复制粘贴上下文交接。
- **字段固定**：两份文件的结构不可自由发挥，见下方模板，便于机器解析、人一眼扫完。

## 任务目录约定

除非项目里已经有既定约定（先检查是否已存在 `PROMPT.md`/`DONE.md` 的历史任务目录，沿用其结构），默认用：

```
<project-root>/tasks/<slug>/PROMPT.md
<project-root>/tasks/<slug>/DONE.md
```

`<slug>` 是简短的 kebab-case 任务标识。目录不存在则新建。

## 下发（规划侧执行这一步）

1. 判断任务类型：
   - **确定性**：改动路径明确 → 直接下发。
   - **探路 / demo**：不确定能不能做 → 下发时在 `PROMPT.md` 里明确说明"这是探路任务，先跑最小 demo，不要求完整实现"。
2. 复制 `assets/templates/PROMPT.md` 到任务目录，填写：
   - frontmatter：`task`、`type`（`deterministic` 或 `exploratory`）、`scope`（允许改动的路径列表）、`status: pending`
   - 正文：目标 / 完成定义 / 范围约束 / 背景（摘录，不引用路径）/ 验证方式 / 产出要求
   - **探路任务的完成定义和验证方式可以放宽**，但仍要写清楚"验收时看什么"（例如"给出能不能做的结论即可，不要求测试通过"）。
3. 告知用户任务已下发到 `tasks/<slug>/PROMPT.md`，可以触发执行侧去读取。

## 回写（当被要求扮演/模拟执行侧，或代为整理执行侧产出时）

1. 复制 `assets/templates/DONE.md` 到同一任务目录。
2. 填写：
   - frontmatter：`task`、`status`（`success` | `partial` | `failed`，三选一，不用自由文本）
   - 正文：改了什么（文件级摘要）/ 怎么验证（可复现步骤 + 实际结果）/ 结论或下一步
   - 若任务 `type` 是 `exploratory`：结论部分写"能不能做 + 坑在哪 + 建议"，不需要完整实现和测试通过。

## 验收（规划侧读 DONE.md 时）

1. 读取 `DONE.md` 的 `status` 字段，先看结论。
2. 对照对应 `PROMPT.md` 的「验证方式」，核对「怎么验证」部分是否真的按此执行、结果是否吻合。
3. `success` → 把结论回写任务清单/目标，任务关闭。
4. `partial` / `failed` → 看「结论/下一步」判断是需要人拍板还是可以直接追加一轮 follow-up（新开一个 `PROMPT.md`，只写这次要修正什么，不要整份照抄上一版）。
5. 不要仅凭 `status: success` 就无条件通过——如果验证步骤描述含糊或和验证方式对不上，视为 `partial`，要求补充。

## 模板

- [assets/templates/PROMPT.md](assets/templates/PROMPT.md)
- [assets/templates/DONE.md](assets/templates/DONE.md)
