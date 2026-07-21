# 计划 002：修复 habitat-radiomics-platform 的 acceptance CI（pyradiomics 依赖安装）

- 状态：已确认
- 计划人：Claude Code（深度审查方）
- 目标仓库：`zywoocode/habitat-radiomics-platform`
- 执行分支：`fix/acceptance-ci-pyradiomics-20260721`（新分支，勿复用已合并的 PR #2）
- 关联 PR：（开新 PR 后回填；**不要重开已合并的 #2**）

## 1. 目标

让 `main` 上的 `.github/workflows/acceptance.yml`（分层验收 CI）三个套件 `quick / public-data / full` 全部转绿。目前它们在装依赖阶段就全部失败，主干 CI 长期红。

## 2. 背景与根因（已由 Claude 深度审查定位，证据充分）

- PR #2「Restore layered acceptance CI」已合并进 `main`，但合并时 CI 三套件全红，且根因未修。
- 三个套件都**卡在 `python -m pip install -r requirements.txt`，还没跑到测试**。日志铁证：
  ```
  File "<string>", line 7, in <module>
  ModuleNotFoundError: No module named 'numpy'
  ERROR: Failed to build 'pyradiomics' when getting requirements to build wheel
  ```
- 机制：`pyradiomics` 的 `setup.py` 在构建期 `import numpy`，却未在 `build-system.requires` 声明，PEP 517 隔离构建环境无 numpy → 构建失败。pip 还先因版本元数据不一致丢弃 `pyradiomics 3.1.0`（expected 3.1.0 / metadata 3.0.1a1），回退 3.0.1 触发此错。
- **pyradiomics 是三套件都硬需要的，无法用「精简依赖」绕过**：`core/feature_extraction.py:19` 是**裸顶层 `import radiomics`**，而 `test_feature_extraction_protocol_gate.py`（quick 套件）顶层 `from core.feature_extraction import ...`；`test_public_radiomics_feature_readout.py`（public-data 套件）同理。任一套件缺 pyradiomics 都会在收集阶段崩。
- workflow 的加固（`push` 限 `main`、`permissions: contents: read`、`concurrency`、`timeout-minutes: 45`、`shell: pwsh`、pip 缓存）在 commit `9ed66ff` 已到位，**无需再改**；唯一要动的是依赖安装。

## 3. 核心思路

修复点只有一个：让 pyradiomics 在 windows-latest + Python 3.11 上**装成功**。采用标准绕过——先装构建依赖（numpy），再对 pyradiomics 单独关闭构建隔离编译，最后装完整 requirements。

## 4. 执行指令（交给执行方的完整任务书）

> **执行要求：ChatGPT 桌面 app Work 模式 / Codex，模型必须选用 GPT-5.6 最高推理档位（extended/max reasoning），不得降档。**
>
> 本节自包含：执行方读本节 + 仓库代码即可开工。

1. 从最新 `main` 新建分支 `fix/acceptance-ci-pyradiomics-20260721`（**不要重开已合并的 PR #2**）。
2. 编辑 `.github/workflows/acceptance.yml`，把两个 job（`layered` 与 `manual`）里唯一的安装步骤
   ```yaml
   - run: python -m pip install -r requirements.txt
   ```
   替换为（候选方案，需按第 4.3 步实测校准）：
   ```yaml
   - run: python -m pip install --upgrade pip setuptools wheel
   - run: python -m pip install "numpy<2"
   - run: python -m pip install --no-build-isolation "pyradiomics==3.0.1"
   - run: python -m pip install -r requirements.txt
   ```
   说明：`numpy<2` 是因为 pyradiomics 3.x 的 C 扩展针对 numpy 1.x C-API；`--no-build-isolation` 让 pyradiomics 构建时能看到已装的 numpy。
3. **本地针对性验证（硬性，不得跳过）**：在真实 Windows + Python 3.11 环境（或等价 CI 自测）里，实际执行上面的安装序列，确认：
   - pyradiomics 能编译安装成功（这是最大不确定点，务必亲测）；
   - `python scripts\run_acceptance_matrix.py --suite quick -- -q`、`--suite public-data -- -q`、`--suite full -- -q` 三条**都真正跑通（不是只装上）**。
   - torch（`full` 套件需要）能装上；如 CUDA 轮子过大/超时，改用 CPU 轮子：`pip install torch --index-url https://download.pytorch.org/whl/cpu`。
4. 若 pyradiomics 3.0.1 在 py3.11 仍编译失败，**不要盲目重试同一方案**（遵守「三次失败即根因升级」）。按优先级换思路：
   a. 换 pyradiomics 版本 / 找可用预编译 wheel；
   b. 为 CI 固定 numpy 到 pyradiomics 3.0.1 兼容的具体版本；
   c. 评估 CI 是否降到 Python 3.9（pyradiomics 有 3.9 wheel）；
   d. 末选：把 `core/feature_extraction.py` 等的 `import radiomics` 改为惰性/可选（try/except），让无 pyradiomics 时相关测试 `importorskip`——**此为代码行为变更，需单独说明并走三方复审**。
5. 推送分支、开**一个新 Draft PR**，PR 描述引用本计划（`plans/002`），并说明实测验证结果（贴关键日志：pyradiomics 装成功 + 三套件绿）。

## 5. 验收标准

- [ ] PR 上 `layered (quick)`、`layered (public-data)`、`layered (full)` 三个 check 全绿；
- [ ] 关键日志显示 pyradiomics 成功安装、`core.feature_extraction` 可正常 import；
- [ ] 未改动已到位的 workflow 加固项（push 过滤 / permissions / concurrency / timeout）；
- [ ] 一个工作流只对应这一个新主 PR（遵守单一主 PR）。

## 6. 审查要点（给三方审查用）

- 最易翻车处：pyradiomics 在 win/py3.11 的源码编译；PR 必须附「装成功」的实测证据，不能只贴 workflow 改动。
- 边界：`full` 套件的 torch 安装耗时/体积；确认没把 CUDA 大包拖进 CI 导致超时。
- 证据边界：绿色 CI 只证明软件机制可运行，不构成临床/监管/发表/外部验证证据。

## 7. 明确不做（Out of Scope）

- 不重开或复用已合并的 PR #2；不改与依赖安装无关的 workflow 加固项。
- 不在本 PR 里顺带改动业务逻辑或测试断言（除非第 4.4.d 末选方案被采纳，且单独说明）。

## 8. 闭环要求

按本工作区规则（与 Codex 全局 AGENTS.md 对称）：三方回执（Claude + Codex + GPT-5.6 Pro）、修订、针对性验证、增量复审四项齐全前，不得标「完成」。Claude 侧负责增量复审：PR 一开/更新，Claude 复查 CI 与 diff 并落 PR 评论。
