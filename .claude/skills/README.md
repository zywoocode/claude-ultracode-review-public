# 已安装的第三方 Skills

本目录的 skills 于 2026-07 经多智能体搜索 + 全库安全扫描（无危险命令、无注入指令、外部地址均为公开学术/数据库服务）后从 GitHub 引入，仓库内所有会话自动加载。

## 组成

- **K-Dense 科研库**：来自 [K-Dense-AI/scientific-agent-skills](https://github.com/K-Dense-AI/scientific-agent-skills)，覆盖生物信息、单细胞、化学/药物发现、量子计算、统计、机器学习、科研写作/审稿/绘图、临床与实验室集成等。仓库级许可证为 MIT（见 `KDENSE-LICENSE.md`）。

## 2026-07 审计清理（第一批：安全修复）

经多智能体审计后做的无损清理（全部可 git 还原）：

- **删除 6 个**：`token-coach`（依赖未安装的主插件，一运行即报错失效）、`pdf`（与内置同名重复）、`primekg`（硬编码他人电脑路径，必崩）、`dhdna-profiler`（伪科学「认知 DNA」画像 + 隐私风险）、`what-if-oracle`（数字命理当事实）、`autoskill`（读屏守护进程，多数会话失效 + 隐私）。
- **安全修复**：`imaging-data-commons` 原会静默执行 `pip install --break-system-packages` 改环境，已改为仅打印升级提示。
- **删复制粘贴样板**：`markitdown`/`peer-review`/`citation-management`/`scholar-evaluation`/`venue-templates` 里逐字复制的 `generate_schematic*.py`（hash 相同）及相关段落已删（真正的 `scientific-schematics` 不受影响）。
- **精简超长描述 / 收窄误触发**：`experimental-design`、`pathway-enrichment`、`arbor`、`statistical-power`、`bulk-rnaseq`、`pyhealth`、`tamarind`、`paper-lookup`、`markdown-mermaid-writing`、`get-available-resources`、`exa-search`、`optimize-for-gpu` 等的描述已收紧。
- **修坏引用**：`clinical-decision-support`（删不存在的脚本引用）、`literature-review`（未安装的 sibling 改为 `database-lookup`）；`scientific-writing` 的「每篇必须≥20 图」硬配额改为建议。

## 2026-07 审计清理（第二批：删无账号集成 + 合并重叠）

- **删 10 个付费/机构平台集成**（用户确认无账号）：rowan、ginkgo-cloud-lab、benchling-integration、dnanexus-integration、latchbio-integration、labarchive-integration、omero-integration、protocolsio-integration、adaptyv、pacsomatic。
- **合并重叠**：`geomaster` 并入 `geopandas`（核心重复）后删除；文献检索删掉失效的 `bgpt-paper-search`（需 BGPT MCP）、`paperzilla`（需 pz CLI + 账号）。
- **保留说明**：`parallel-web` 被 3 个 skill 引用，删除会产生坏引用，故保留；`citation-management`/`database-lookup`/`scholar-evaluation`/`pyzotero` 各有独立用途（BibTeX 管理 / 通用库查询 / 学术打分 / Zotero），非纯检索重复，保留。

结果：150 → 131 个，描述常驻开销约 16.5k → 13.2k token/会话（累计省约 3.4k）。

## 2026-07 审计清理（第三批：分层加载，最大节省）

**机制选择说明**：Claude Code 只扫描 `.claude/skills/` 一层；skill 的描述会常驻每会话上下文（skill 越多、固定 token 越高）。原本想用 `skillOverrides` 设置开关，但经文档核查确认它在**提交版** `.claude/settings.json` 里无效（[bug #50631]，只有 gitignored 的 `settings.local.json` 生效、不能跨设备同步）。故改用**唯一可靠且能跨设备同步**的办法：**把领域专用 skill 移出 `.claude/skills/`**。

分两层，**文件全部保留**（在 git 里）：

- **常驻核心（35 个，留在 `.claude/skills/`）**：通用写作/统计/绘图/数据/文献/审查类——docx、pptx、xlsx、markitdown、liteparse、exploratory-data-analysis、polars、dask、networkx、sympy、matplotlib、seaborn、scientific-visualization、scikit-learn、statsmodels、statistical-analysis、statistical-power、experimental-design、shap、scientific-writing、peer-review、literature-review、paper-lookup、citation-management、research-lookup、research-grants、scholar-evaluation、scientific-brainstorming、scientific-critical-thinking、markdown-mermaid-writing、scientific-slides、get-available-resources、hypothesis-generation、database-lookup、exa-search。
- **按需库（96 个，移到仓库根 `skills-library/`）**：基因组学/单细胞/化学药物/量子物理/天文/RL/临床影像/实验室平台等。该目录**不被 Claude Code 扫描 = 零 token**，但文件都在，随时可移回启用。

**效果**：每会话描述开销从 ~16,561 → **~3,685 token（省约 78%）**。

### 如何按需激活某个领域 skill

进入某领域任务时，把它从 `skills-library/` 移回 `.claude/skills/` 即可（即时生效）：
```bash
git mv skills-library/scanpy .claude/skills/scanpy   # 例：启用单细胞分析
```
用完想收起来就反向 `git mv` 回 `skills-library/`。激活/收起都提交进 git，两端同步。Claude 在本工作区会**主动**：识别到任务需要某个库中 skill 时，提示或直接 `git mv` 启用它。

> 提示：本用户做放射组学/医学影像研究，可考虑把 `pydicom`、`imaging-data-commons`、`clinical-reports`、`histolab`、`pathml`、`pyhealth`、`nextflow`、`scanpy` 等常用的长期移回核心。

## 分类速览（K-Dense）

- **科研写作 / 审稿 / 表达**：scientific-writing、literature-review、citation-management、peer-review、research-lookup、research-grants、scholar-evaluation、scientific-critical-thinking、venue-templates、markdown-mermaid-writing、scientific-slides、latex-posters、pptx-posters、infographics、scientific-schematics、scientific-visualization、generate-image、market-research-reports
- **统计 / 实验设计**：statistical-analysis、statistical-power、experimental-design、statsmodels、pymc、shap、exploratory-data-analysis
- **机器学习 / 深度学习**：scikit-learn、pytorch-lightning、transformers、torch-geometric、torchdrug、stable-baselines3、pufferlib、aeon、timesfm-forecasting、hugging-science、optimize-for-gpu、arbor
- **生物信息 / 组学**：biopython、bioservices、gget、scanpy、anndata、scvi-tools、scvelo、cellxgene-census、pysam、pydeseq2、bulk-rnaseq、deeptools、nextflow、phylogenetics、etetoolkit、scikit-bio、pathway-enrichment、arboreto、geniml、gtars、polars-bio、tiledbvcf、bids
- **化学 / 药物发现 / 材料**：rdkit、datamol、molfeat、medchem、deepchem、diffdock、pytdc、pyopenms、matchms、molecular-dynamics、pymatgen、glycoengineering、esm
- **量子 / 物理 / 仿真**：qiskit、cirq、pennylane、qutip、fluidsim、simpy
- **临床 / 医疗 / 影像**：clinical-reports、clinical-decision-support、treatment-plans、pyhealth、pydicom、imaging-data-commons、histolab、pathml、neurokit2、neuropixels-analysis、depmap、primekg
- **数据处理 / 数值 / 可视化**：polars、dask、vaex、zarr-python、networkx、sympy、matlab、matplotlib、seaborn、umap-learn、geopandas、geomaster、astropy
- **平台 / 实验室集成**：benchling-integration、dnanexus-integration、latchbio-integration、labarchive-integration、opentrons-integration、protocolsio-integration、omero-integration、ginkgo-cloud-lab、pylabrobot、lamindb、modal、adaptyv、rowan、tamarind
- **文档 / 检索工具**：docx、pdf、pptx、xlsx、markitdown、liteparse、paper-lookup、exa-search、parallel-web、database-lookup、bgpt-paper-search、pyzotero
- **其他**：get-available-resources、hypothesis-generation、hypogenic、scientific-brainstorming、consciousness-council、what-if-oracle 等

## 注意事项

- 许可证：部分 skill 的 `license:` 字段（BSD/Apache/GPL/Proprietary）指其封装的**上游工具**（如 docx/pdf/rowan 及若干 GPL 库）的许可，非 skill 文档本身；商业用途前按需核对具体 skill。
- `autoskill`：需本地 screenpipe 录屏守护进程 + API key 才运行，无守护进程时惰性拒绝，装入仓库不产生任何动作；如不需要可直接删除该目录。
- 多个 skill 的脚本会联网访问公开数据库（PubMed、arXiv、UniProt、ChEMBL 等）——在云端容器可能被网络策略拦截，在本机运行正常。
- **Token 开销**：全量 149 个 skill 的名称与描述常驻每个会话上下文（约一两万 token 固定开销）。若想瘦身，删除对应 `.claude/skills/<name>/` 目录即可，随删随生效。
