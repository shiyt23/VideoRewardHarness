# RewardHarness 代码库阅读报告

> 论文：`Papers/RewardHarness Self-Evolving Agentic Post-Training.pdf`  
> 代码目录：`Video-Reward-Harness/RewardHarness/`  
> 审视结论：原报告总体把握了“冻结 VLM + 自进化 Skills/Tools Library”的主线，但有几处容易误导的地方：把论文设定与当前开源实现混在一起、把论文中的 1-5 标度写成实现标度、对默认库和复现实验路径的描述不够精确。本文已按论文文本和当前源码重新校正。

---

## 1. 仓库结构总览

```text
RewardHarness/
├── src/
│   ├── pipeline.py               # 自进化主循环：评估、链分析、更新、验证、剪枝、checkpoint
│   ├── router.py                 # Orchestrator 的推理期 Router：从 Library 中选择 Skills/Tools
│   ├── chain_analyzer.py         # Orchestrator 的进化期分析器：从推理链产生更新信号
│   ├── evolver.py                # 应用 skill/tool 更新，支持 snapshot/restore 和 tool prompt 校验
│   ├── sub_agent.py              # 冻结 Sub-Agent VLM 的多轮推理、工具调用、答案解析
│   ├── library/__init__.py       # Library 数据结构：Skill/Tool 的读写、注册表、工具执行
│   ├── library/                  # 当前默认随仓库提供的 evolved library
│   ├── library_gemini_v*/        # 历史/实验库快照，运行时不会自动使用
│   ├── evaluator.py              # A/B/tie 正确性与 K=2/3/4 group accuracy
│   ├── gemini_client.py          # Vertex AI Gemini 调用封装
│   └── endpoint_pool.py          # vLLM OpenAI-compatible endpoint 轮询
├── scripts/
│   ├── run_evolution.py          # 自进化入口
│   ├── run_benchmark.py          # EditReward-Bench 只读评测入口
│   ├── reproduce.sh              # 复现脚本
│   ├── serve_vllm_multi.sh       # 本地多 endpoint vLLM 启动脚本
│   └── check_env.py              # 环境检查
├── configs/
│   ├── default.yaml              # Gemini model、evolution、benchmark 配置
│   └── endpoints.txt             # Sub-Agent vLLM endpoint 列表
├── score-guidelines/             # Sub-Agent 每次推理固定加载的 1-4 分模板
├── examples/                     # 单样例打分、库检查、示例输入输出
├── vanilla/                      # 非 RewardHarness 的基线 benchmark 脚本
├── tests/                        # pytest 测试，主要使用 mock，不依赖 GPU/网络
├── data/                         # 数据缓存占位；真实数据从 HuggingFace 加载
└── README.md / WALKTHROUGH.md / OUTPUTS.md / Makefile 等文档与入口
```

需要特别注意默认库状态：论文 §2.2 说 Library 在自进化开始时为空；当前仓库的 `src/library/` 已经包含若干 Skill/Tool，作为随代码发布的默认库。真正的 iteration 0 baseline 是“指定一个空库或在 pipeline 开始时使用空库状态”时的行为，而不是当前 checkout 下 `src/library/` 的自然状态。

---

## 2. 论文算法核心与当前实现差异

论文把 RewardHarness 定义为一种上下文进化范式：不训练奖励模型参数，而是从约 100 条偏好示例中迭代维护一个外部 Skills/Tools Library。推理时，Orchestrator 选择相关库条目形成上下文 `C`，冻结 Sub-Agent VLM 基于源图、候选编辑图、指令和上下文输出分数与偏好排序。进化时，系统比较预测与人类标签，分析正确/错误推理链，并对库执行新增、修改或删除。

当前代码基本实现了这条主线，但有几个实现层面的差异：

- 论文 §2.1 描述的是 K 个候选图、1-5 离散分数、按分数排序得到 ranking；当前 `src/sub_agent.py` 的输出 schema 是 A/B pairwise comparison，并要求 `score_*` 为 1-4。K=3/4 benchmark 是由 `scripts/run_benchmark.py` 对多个 pair 结果做 group accuracy 聚合，而不是一次性把 K 个候选同时送入 Sub-Agent 排序。
- 论文 §2.3 写 Orchestrator 是 Claude-based；当前开源代码的 Router、ChainAnalyzer 和部分 Evolver prompt refinement 使用 `src/gemini_client.py` 调 Gemini，默认配置是 `gemini-3.1-pro-preview`。
- 论文 §2.5 的 validation gating 表述为验证准确率超过当前最好结果才接受；当前代码使用 `explore_margin`，条件是 `val_acc >= prev_val_acc - explore_margin`，允许小幅回落，同时 `prev_val_acc` 仍记录历史最好值。
- 论文 Figure 2 中 Router 可查看编辑指令、源图和候选图；当前 `Router.prepare_context(prompt)` 只把编辑指令文本传给 Gemini 做库条目选择，没有把图像输入传入 Router。
- 论文实验提到 77 iterations、iteration 69 最终库、3 Skills + 4 Tools；当前 `configs/default.yaml` 默认 `max_iterations: 5`，`make evolve` 覆盖为 200，`make reproduce` 走脚本流程。是否得到论文最终库取决于运行配置、模型和数据访问，不应把默认 5 iter 等同于论文完整实验。

---

## 3. 论文算法到代码的具体映射

### 3.1 Skills & Tools Library：`src/library/`

论文 §2.2 将 Library 分成两类条目：

- Skill：声明式 Markdown 评分指南，包含名称、描述、rubric、示例或判断规则。
- Tool：过程式视觉分析规格，包含名称、用途、输入输出 schema、调用条件、执行协议和 system prompt。

代码中由 `src/library/__init__.py` 的 `Library` 类实现：

- `add_skill()` / `update_skill()` / `delete_skill()` 管理 `skills/<name>/SKILL.md`。
- `add_tool()` / `update_tool()` / `delete_tool()` 管理 `tools/<name>/SKILL.md`。
- `registry.json` 维护 `name -> {type, description, path}`，供 Router 做 L1 summary 路由。
- `get_all_summaries()` 返回 name/description，对应论文 progressive disclosure 的 Level 1。
- `get_full_content(name)` 返回 Markdown body，对应 Level 2 注入 Sub-Agent 的全文内容。
- `call_tool(name, args, endpoint_pool)` 用 Tool frontmatter 中的 `system_prompt` 再调用一次 vLLM，让通用 VLM 临时扮演 OCR、对象验证、布局分析等专门工具。
- `snapshot()` / `restore()` 支持回滚；`Evolver` 也有一套 checkpoint 用 snapshot 格式。

当前默认库 `src/library/` 已包含例如 `score-calibration`、`verifying-multi-attribute-objects`、`fine-grained-object-verifier`、`visual-text-and-layout-analyzer` 等条目。`src/library_gemini_v*` 是历史库目录，不会被默认加载，除非通过 `--library-dir` 指定。

### 3.2 Orchestrator Router：`src/router.py`

论文 §2.3 中 Orchestrator 在推理阶段负责选择相关 Skills/Tools 并组装上下文。当前实现是 `Router.prepare_context(prompt)`：

1. 调用 `library.get_all_summaries()` 获取所有 Skill/Tool 的 name 和 description。
2. 将编辑指令和 L1 summaries 填入 `ROUTING_PROMPT`。
3. 调用 Gemini，要求返回 `{"skills": [...], "tools": [...]}`。
4. `_assemble_context()` 对选中条目调用 `get_full_content()`，拼成 `# EVALUATION SKILLS` 和 `# AVAILABLE TOOLS` 两段上下文。
5. 如果库为空，直接返回空字符串；如果 Gemini API 最终失败，代码会 fallback 为使用所有库条目。

因此，代码确实体现了论文的 progressive disclosure，但当前 Router 的选择依据只有 prompt 文本，不包含源图和候选图视觉内容。

### 3.3 Sub-Agent 推理与工具调用：`src/sub_agent.py`

论文 §2.4 将 Sub-Agent 分为 rubric application、tool-guided analysis、aggregation and ranking 三步。当前实现对应如下：

- `evaluate()` 每次读取 `score-guidelines/template1_instruction_following.md` 和 `template2_visual_quality.md`，形成固定的 instruction following / visual quality 双维度评分骨架。
- Router 产生的 `skill_context` 会追加到 system prompt 中；如果其中含 `# AVAILABLE TOOLS`，再追加工具调用协议。
- 用户消息包含 source image、edited image A、edited image B 和 editing instruction。
- 多轮循环最多允许 `MAX_TOOL_CALLS = 5` 次工具交互。模型输出 `<tool>{...}</tool>` 时，代码调用 `library.call_tool()`，并把结果包成 `<obs>...</obs>` 回灌给 Sub-Agent。
- 模型输出 `<answer>...</answer>` 时，`_parse_answer()` 解析 JSON，要求字段包含 `preference`、四个 1-4 分数和 reasoning。
- 若解析失败或调用失败，返回 fallback：`tie` 和四个中性 2 分。

这里最重要的校正是分数标度：论文说实验实现为 1-5 分，但当前开源代码和 `score-guidelines/` 明确是 1-4 分。报告中如果写“论文公式对应代码 1-4”需要说明这是开源实现差异，而不是论文原文。

### 3.4 自进化主循环：`src/pipeline.py`

论文 §2.5 的五步循环在 `SelfEvolutionPipeline.evolve()` 中实现：

1. Evaluation：`run_iteration()` 先对每个样例并行调用 Router，再用 `sub_agent.batch_evaluate()` 做 A/B 预测。
2. Scoring：`evaluate_prediction()` 比较 `prediction` 与 `gt`，`compute_kpair_accuracy(..., k=2)` 计算 train/val accuracy。
3. Chain Analysis：`ChainAnalyzer.analyze(train_results, current_lib)` 将所有正确和错误样例的 reasoning chain 发给 Gemini，要求输出 `skill_updates`、`tool_updates` 和 `analysis_summary`。
4. Library Update：`Evolver.apply_signals()` 根据 action 执行 add/update/delete。
5. Validation & Gating：Phase A 先只应用 skill updates 并跑 val，Phase B 再应用 tool updates 并跑 val；如果低于 `prev_val_acc - explore_margin` 则 `restore()` 回滚。

此外，代码还实现了两个重要工程扩展：

- `_augment_with_swaps()` 会把训练样例 A/B 对调并翻转标签，用来约束位置不变性；这和 `chain_analyzer.py` prompt 中禁止 Skill 依赖 “Image A/B” 的要求一致。
- `_prune_library()` 做 leave-one-out pruning：逐个临时删除 skill/tool 跑 val，如果删除后 accuracy 不低于 baseline，就认为条目冗余或有害并永久删除。默认 `prune_every_n: 50`。

需要指出一个实现细节：`pipeline._checkpoint()` 使用的是 `Evolver.snapshot()` 产生的 `skills_content/tools_content` 格式，而 `Library.snapshot()` 返回的是 `files` 格式。两者都叫 snapshot，但结构不同；报告中不要混称。

### 3.5 Chain Analyzer：`src/chain_analyzer.py`

论文 §2.5 Step 3 要求从正确和错误推理链中提取改进信号。当前 `ANALYSIS_PROMPT` 明确要求：

- 从正确预测中抽取可复用推理模式或 heuristic。
- 从错误预测中判断是否缺少 Skill、Skill 需要修改，或需要新 Tool。
- 支持 `add/update/delete` 三类 action。
- 强制 position-invariant，避免写出依赖 A/B 位置的 Skill。
- 鼓励在工具少于 3 个时提出 Tool，并说明 Tool 的调用协议。

`analyze()` 会校验 Gemini 返回 JSON 的基本结构，丢弃缺字段的 malformed update，避免 Evolver 直接写入不可用条目。

### 3.6 Evolver：`src/evolver.py`

论文 §2.5 Step 4 的库更新由 `Evolver.apply_signals()` 落地：

- Skill add/update/delete 分别调用 `Library.add_skill()`、`update_skill()`、`delete_skill()`。
- Tool add/update/delete 分别调用 `Library.add_tool()`、`update_tool()`、`delete_tool()`。
- 新增 Tool 时，如果存在 endpoint pool，会用 `_validate_tool_prompt()` 对 system prompt 做 dummy JSON 输出校验；成功率低于 80% 时尝试让 Gemini 改写 prompt，最多 3 轮。

这部分是开源实现的工程增强，论文正文只描述“创建、修改、废弃”和 pruning，没有展开这种 dummy validation。

### 3.7 Benchmark 与指标：`src/evaluator.py`、`scripts/run_benchmark.py`

`src/evaluator.py` 提供两个函数：

- `evaluate_prediction(prediction, ground_truth)`：只判断 A/B/tie 是否相等，`gap` 字段目前保留为 0，没有实现论文中“score gap 用于诊断分析”的真实数值计算。
- `compute_kpair_accuracy(pair_results, k)`：按 `group_id` 分组，一组内所有 pair 都正确才算 K=3/K=4 group 正确。

`scripts/run_benchmark.py` 只读评测 `TIGER-Lab/EditReward-Bench`：

- 将 benchmark row 中的 `ranking` 和 `comparison_type` 解析为 A/B/tie。
- 每个 pair 调用 Router + SubAgent，不更新 Library。
- 分 K=2/3/4 保存到 `results/benchmark_results.json`。

README 明确说明 GenAI-Bench 不由 `scripts/run_benchmark.py` 消费；论文 headline average 需要额外的 GenAI-Bench 评测结果合并统计。`vanilla/` 中有基线相关脚本，但并不是完整的 RewardHarness GenAI-Bench 主流程实现。

---

## 4. 配置与运行入口

`configs/default.yaml` 中与论文/实现关系最密切的字段：

| 字段 | 含义 |
| --- | --- |
| `gemini.model` | 当前 Orchestrator 使用的 Gemini 模型 |
| `evolution.train_dataset` | 自进化使用的 100 条偏好数据来源 |
| `evolution.train_n: 60` | train split 大小，对应论文 60 examples |
| `evolution.val_n: 40` | held-out validation 大小，对应论文 40 examples |
| `evolution.max_iterations: 5` | 配置默认短跑，不等同论文 77 iterations |
| `evolution.batch_concurrent` | Sub-Agent batch 并发度 |
| `evolution.explore_margin: 0.075` | 当前实现的宽松 gating 容忍度 |
| `evolution.augment_swap: true` | A/B 交换增强 |
| `evolution.prune_every_n: 50` | 周期性 leave-one-out pruning |
| `benchmark.dataset` | EditReward-Bench 数据集 |

README 与 `configs/default.yaml` 都说明 `model:` block 主要是文档用途；实际 vLLM serving 参数来自脚本和环境变量，Sub-Agent 请求中的模型名来自 `REWARDHARNESS_SUBAGENT_MODEL`，默认 `Qwen2.5-VL-7B-Instruct`。

常用入口：

- `python scripts/run_evolution.py --config configs/default.yaml --results-dir results/my_run --max-iters N`
- `python scripts/run_evolution.py --config configs/default.yaml --resume`
- `python scripts/run_benchmark.py --config configs/default.yaml --library-dir <library_or_checkpoint_dir>`
- `python examples/score_pair.py`
- `make test`
- `make evolve`，当前 Makefile 覆盖为 `--max-iters 200`
- `make reproduce`，调用 `scripts/reproduce.sh`

---

## 5. 原报告中应修正的关键问题

1. 原报告说“论文公式写 1-5，默认 K=2，Sub-Agent 输出 1-4”时没有明确区分论文和代码。准确说法是：论文问题形式化是 K candidates + 1-5 scores；当前开源实现主要做 A/B pairwise + 1-4 双维度分数，再用 pairwise 结果聚合 K=3/4 benchmark。
2. 原报告把 `src/library/` 描述成“默认 Library 目录”是对的，但容易让人以为默认就是空库 baseline。当前 `src/library/` 已包含发布库；论文的空库 baseline 是进化起点概念，不是当前目录内容。
3. 原报告写“论文实验描述为 Claude-based，仓库 Gemini 实现”是正确的，但应进一步说明这不只是默认实现差异，也影响复现实验的严格一致性。
4. 原报告说 Router 体现论文 Level 1/Level 2 是对的，但遗漏了当前 Router 只看 prompt、不看图像；论文 Orchestrator 描述中包含 source image 和 candidate edits。
5. 原报告对 validation gating 的“论文 strictly improve / 实现 explore_margin”方向正确，但建议改为“论文接受验证准确率超过当前最好；代码允许不低于历史最好减 margin，并单独维护 best_val_acc”。
6. 原报告说 `results/<run>/checkpoints/iter_N/` 是库快照正确，但未说明 checkpoint snapshot 格式来自 `Evolver.snapshot()`，不同于 `Library.snapshot()`。
7. 原报告提到“GenAI-Bench 基线在 vanilla/ 目录”容易误读为 RewardHarness 的 GenAI-Bench 评测也已完整接入；当前 README 明确 `run_benchmark.py` 不消费 GenAI-Bench，需要额外评测并合并。

---

## 6. 推荐阅读顺序

1. `src/library/__init__.py`：先理解 Skill/Tool 文件格式、registry 和 tool execution。
2. `src/router.py`：理解推理期如何选库并拼上下文。
3. `src/sub_agent.py` 与 `score-guidelines/`：理解固定评分模板、多轮 tool call 和 `<answer>` 解析。
4. `src/pipeline.py`：理解 evolution 主循环、gating、checkpoint、pruning。
5. `src/chain_analyzer.py` 与 `src/evolver.py`：理解更新信号如何生成和落地。
6. `src/evaluator.py` 与 `scripts/run_benchmark.py`：理解 pairwise accuracy 和 K=2/3/4 group accuracy。

---

## 7. 总结

这个代码库确实实现了论文最核心的算法思想：RewardHarness 不训练奖励模型权重，而是让 Orchestrator 从少量偏好样本中迭代维护外部 Skills/Tools Library，推理时由冻结 Sub-Agent VLM 读取相关上下文并输出可解释的偏好判断。

但当前开源实现不是论文正文的逐字复刻：它将 Orchestrator 换成 Gemini，实现层面采用 A/B pairwise + 1-4 分模板，Router 只基于文本 prompt 路由，并通过 `explore_margin` 放宽验证门控。这些差异不影响理解代码主线，但在写报告或复现实验时必须显式说明，否则容易把论文设定、README 复现说明和当前源码行为混为一谈。
