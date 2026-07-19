# 三方审查协作工作流（Three-Party Review Workflow）

本仓库是一个协作工作区，采用「一个规划核心 + 一个执行方 + 三方审查」的模式。
所有协作都以 **GitHub 仓库（计划文件 + Pull Request）** 为桥梁，各工具之间不需要直接互联。

## 角色分工

| 角色 | 承担者 | 职责 |
|------|--------|------|
| 规划核心 | Claude Code（Ultracode） | 需求分析、总体架构、核心思路、任务拆解，产出 `plans/` 下的计划文件 |
| 执行方 | ChatGPT 桌面 app（Work/Agent 模式） | 按计划文件实现代码，推送分支并开 Pull Request |
| 审查方 A | Claude Code review（`/code-review`，多智能体交叉验证） | 正确性、架构一致性、与计划的偏差 |
| 审查方 B | Codex（GitHub App，`@codex review`） | 独立代码审查 |
| 审查方 C | GPT 桌面端模型审查 | 由用户在桌面 app 中发起，对 PR diff 做第三份独立审查 |

## 标准流程

```
需求 → [1] Claude 出计划 → [2] 执行方实现并开 PR → [3] 三方审查 → [4] 修复 → [5] 合并
```

### 1. 规划（Claude）
- 用户向 Claude 描述需求。
- Claude 产出计划文件 `plans/NNN-<主题>.md`（格式见 `plans/TEMPLATE.md`），提交到仓库。
- 计划中「执行指令」一节是写给执行方的完整、自包含的任务说明。

### 2. 执行（ChatGPT 桌面 app Work 模式）
- 用户把计划文件内容（或仓库链接 + 计划路径）交给执行方。
- 执行方在新分支 `feat/NNN-<主题>` 上实现，推送并开 PR，PR 描述需引用对应计划文件。

### 3. 三方审查（并行，都落在同一个 PR 上）
- **Claude**：用户让 Claude 审查该 PR（重点：是否符合计划、正确性、遗漏项）。
- **Codex**：PR 上评论 `@codex review`（需先安装 Codex GitHub App 并对本仓库授权）。
- **GPT 桌面端**：用户在桌面 app 中让所选模型审查 PR diff，将结论以 PR 评论形式贴回。

### 4. 修复
- 执行方（或 Claude）根据三方审查意见修复，直至三方均无阻塞性意见。

### 5. 合并标准
- 三方审查均已完成且无未解决的阻塞性意见；
- PR 描述中的审查清单（见 PR 模板）全部勾选；
- CI（如有）通过。

## 目录约定

```
plans/           # 计划文件，NNN 递增编号，一任务一文件
  TEMPLATE.md    # 计划模板
WORKFLOW.md      # 本文件
.github/pull_request_template.md   # PR 三方审查清单
```

## 一次性准备事项（由用户完成）

1. 在 GitHub 上为本仓库安装并授权 **Codex GitHub App**（chatgpt.com/codex 设置中连接 GitHub），使 `@codex review` 可用。
2. 在 ChatGPT 桌面 app 中连接本仓库（GitHub 连接器 / Codex），以便 Work 模式直接读取计划文件并推送代码。
3. 之后每个任务，只需向 Claude 说需求即可，其余按上述流程流转。
