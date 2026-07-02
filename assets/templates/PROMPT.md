---
task: <slug>
type: deterministic
scope:
  - <path-1>
status: pending
---

# <Task Title>

## 目标
<一句话说清要解决什么>

## 完成定义
<做到什么算完成，必须可验证>

## 范围约束
只允许改动 frontmatter `scope` 列出的路径，禁止修改其它文件/模块。
需要的背景信息已摘录在下方，不要去读规划文档或其它未列出的文件。

## 背景（如有必要）
<执行侧无法从代码库本身发现的上下文：设计决策、约束、依赖任务的产出摘要等。
只嵌入内容，不要引用文档路径。>

## 验证方式
<怎么自测：命令 / 步骤 / 预期结果>

## 产出要求
完成后在本任务目录下创建 `DONE.md`（模板见 assets/templates/DONE.md），status 只能三选一：
`success` | `partial` | `failed`。
