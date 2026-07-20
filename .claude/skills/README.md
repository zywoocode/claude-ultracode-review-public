# 已安装的第三方 Skills

以下 skills 于 2026-07 经多智能体搜索 + 安全审查（无危险命令、无注入指令、外部地址均为知名学术服务）后从 GitHub 引入，仓库内所有会话自动加载。

## 科研类

| Skill | 用途 | 来源 | 许可证 |
|-------|------|------|--------|
| scientific-writing | 科研论文写作（IMRAD 结构、CONSORT/STROBE/PRISMA 报告规范、多种引用格式） | [K-Dense-AI/claude-scientific-writer](https://github.com/K-Dense-AI/claude-scientific-writer) | MIT |
| literature-review | 系统性文献综述（PubMed、arXiv、Semantic Scholar 等多库检索 + 引用核验） | 同上 | MIT |
| citation-management | 引用管理（元数据提取、DOI→BibTeX、引用校验） | 同上 | MIT |
| peer-review | 同行评审工具箱（方法学、统计、可复现性、伦理、图表完整性）——深度审查科研内容时使用 | 同上 | MIT |
| statistical-analysis | 统计分析向导（检验选择、假设检查、效应量、功效分析、APA 报告） | [K-Dense-AI/scientific-agent-skills](https://github.com/K-Dense-AI/scientific-agent-skills) | MIT |
| scientific-visualization | 期刊级图表（多面板、显著性标注、色盲安全配色、Nature/Science/Cell 格式） | 同上 | MIT |

## 省 Token 类

| Skill | 用途 | 来源 | 许可证 |
|-------|------|------|--------|
| token-coach | 上下文用量教练：分析会话开销与习惯，给出省 token 的配置与工作方式建议 | [alexgreensh/token-optimizer](https://github.com/alexgreensh/token-optimizer) | PolyForm Noncommercial 1.0.0（仅限非商业用途） |

各 skill 目录内保留了原仓库的 LICENSE 文件。需要更多领域专用 skill（如化学 RDKit、生信、药物发现）时，可从 K-Dense scientific-agent-skills（148 个）按需追加。
