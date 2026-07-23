# skills-library —— 按需领域 skill 库

这里存放 96 个**领域专用** skill（基因组学/单细胞/化学药物/量子物理/天文/RL/临床影像/实验室平台等）。

**为什么在这里**：Claude Code 只扫描 `.claude/skills/`，本目录不被扫描——所以这些 skill 的描述**不进上下文、零 token 开销**，但文件完整保留、随时可用。这样常驻会话只加载 35 个通用核心 skill（约 3,685 token，而非全量约 16,561 token）。

## 启用某个 skill

```bash
git mv skills-library/<name> .claude/skills/<name>
```
即时生效。例：`git mv skills-library/rdkit .claude/skills/rdkit` 启用化学信息学。

## 用完收起

```bash
git mv .claude/skills/<name> skills-library/<name>
```

启用/收起都提交进 git，手机与电脑两端自动同步。分层规则与完整名单见 `../.claude/skills/README.md`。
