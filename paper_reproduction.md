# RewardHarness 论文复现操作文档

> 目标：在尽量少改代码的前提下，基于当前 `RewardHarness/` 仓库复现论文中 **Qwen 开源模型线** 的一部分可复现实验指标。  
> 约束：本文档只描述复现方案，不修改仓库中的其他文件。  
> 当前日期：2026-06-03

---

## 1. 先说结论：当前仓库里你能复现什么

结合论文正文、README、`scripts/`、`vanilla/`、`OUTPUTS.md` 和当前源码，结论如下：

### 1.1 可以直接复现的内容

1. **Qwen2.5-VL-7B 作为 Sub-Agent 的 RewardHarness 推理与 benchmark 流程**
   - 即：`Router + ChainAnalyzer + Evolver + Qwen2.5-VL-7B(vLLM)` 这一整套系统。
   - 对应脚本：`scripts/run_evolution.py`、`scripts/run_benchmark.py`、`scripts/reproduce.sh`。

2. **EditReward-Bench 上的 K=2 / K=3 / K=4 指标**
   - `scripts/run_benchmark.py` 原生支持。
   - 输出写入 `results/benchmark_results.json`。

3. **Qwen 线的“自进化训练过程”**
   - 用 `AgPerry/EditReward-Data-100` 做 60/40 split。
   - 对应论文 §2.5 的 self-evolution loop。
   - 可拿到 `train_acc`、`val_acc`、各 iteration 的 library checkpoint、best checkpoint。

4. **论文中“冻结 Qwen + 进化 Library 比裸 Qwen 强”的核心结论**
   - 这是当前最现实、最重要的可复现目标。

### 1.2 当前仓库不能完整一键复现的内容

1. **论文 headline 平均分 `45.7`（Qwen）**
   - 论文 Table 1 中这个平均分是：
     - `EditReward-Bench K=2`
     - `EditReward-Bench K=3`
     - `EditReward-Bench K=4`
     - `GenAI-Bench`
     - 这四项的平均值
   - 当前仓库的 `scripts/run_benchmark.py` 只跑前三项。
   - `GenAI-Bench` 没有接入 RewardHarness 主流程，只在 `vanilla/` 中有 baseline 脚本。

2. **论文中 Qwen 线的完全同配置数值**
   - 论文 Orchestrator 写的是 Claude-based。
   - 当前开源仓库 Orchestrator 实现是 Gemini。
   - 因此即使流程一致，也不是论文中完全同一后端。

3. **论文中长程演化后的最终库状态**
   - 论文分析图里有 77 iterations、pruning around iter 50、final selected library。
   - 当前默认配置 `configs/default.yaml` 只有 `max_iterations: 5`。
   - `make evolve` 会覆盖为 `200`，但是否得到接近论文结果仍依赖 API、随机性、运行时环境和模型响应。

4. **论文 §3.2 的 RL / GRPO 下游结果**
   - 当前仓库没有完整的 RL 训练代码栈。
   - 只能复现 RewardHarness 作为 reward evaluator 的部分，不适合在本仓库内直接复现 Table 2。

---

## 2. 推荐的复现目标分级

为了尽量少改代码，我建议把复现目标分为三级。

### 2.1 Level 1：零改动、最快确认主流程

目标：

- 跑通环境检查
- 启动 Qwen vLLM
- 跑一次 `run_benchmark.py`
- 确认默认 `src/library/` 对 Qwen 有有效增益

这一级能验证：

- 仓库环境没问题
- Qwen Sub-Agent 可用
- RewardHarness 主流程可跑
- 你能复现论文的核心思路，而不只是静态看代码

### 2.2 Level 2：零改动、复现 Qwen 自进化 + EditReward-Bench

目标：

- 跑 `run_evolution.py`
- 选择 best checkpoint
- 对 best checkpoint 跑 `run_benchmark.py`
- 得到你自己的 K=2 / K=3 / K=4

这一级是我最推荐的主复现目标，因为：

- 完全依赖当前仓库现有脚本
- 不需要写新代码
- 与论文 Qwen 线的“RewardHarness(Qwen) 在 EditReward-Bench 上提升”最接近

### 2.3 Level 3：尽量逼近论文 Table 1 的 Qwen 平均分

目标：

- 完成 Level 2
- 再补一个 **RewardHarness 风格的 GenAI-Bench 评测脚本**
- 最后把 4 个分数取平均

这是当前仓库下，最接近论文 `45.7` average 的方案。  
但它已经超出“零改动”，因为需要你后续补一个最小脚本，或者手工把 `run_benchmark.py` 改造成同时支持 GenAI-Bench。

---

## 3. 论文中与 Qwen 复现最相关的指标

论文 Table 1 中，Qwen 线最关键的几组数值如下：

### 3.1 裸 Qwen baseline

- `Qwen2.5-VL-7B`
  - K=2: `52.7`
  - K=3: `24.7`
  - K=4: `3.4`
  - GenAI-Bench: `40.5`
  - Avg: `30.3`

### 3.2 RewardHarness(Qwen)

- `REWARDHARNESS (Qwen)`
  - K=2: `57.9`
  - K=3: `46.7`
  - K=4: `10.8`
  - GenAI-Bench: `67.5`
  - Avg: `45.7`

### 3.3 你在当前仓库里最容易复现的判断结论

即使你暂时不能完整拿到 `45.7` 这个平均值，只要你能复现下面任一事实，这次复现就已经有很高价值：

1. **RewardHarness(Qwen) 的 K=2/K=3/K=4 明显优于裸 Qwen baseline**
2. **进化出的 best checkpoint 明显优于空库或弱库配置**
3. **Qwen 作为冻结 Sub-Agent，仅靠外部 Skills/Tools Library 就能获得显著增益**

---

## 4. 复现设计原则

这次复现我建议遵循三个原则：

1. **先零改动，再最小补丁**
   - 先尽可能用现成脚本拿到可复现结果。
   - 不要一开始就大改 benchmark 流程。

2. **先复现 EditReward-Bench，再考虑 GenAI-Bench**
   - 当前仓库对 EditReward-Bench 是一等支持。
   - 对 GenAI-Bench 只是文档说明和 baseline 参考。

3. **先验证趋势，再追精确数值**
   - 论文的绝对数值依赖 Orchestrator、随机性、模型响应和长程演化。
   - 当前仓库更适合先复现“趋势一致”和“量级接近”。

---

## 5. 环境准备

## 5.1 硬件建议

按 README：

- `make test` / `make check`：CPU 即可
- Qwen 本地推理：
  - 至少 `1 x GPU >= 24GB`
  - 推荐 L40S / A100 / H100
- `make reproduce`：
  - 推荐 `>= 4 GPUs`
  - 约 `4-6 小时`

如果你的目标只是跑 Qwen benchmark，不一定必须 4 卡。  
如果你要做较长 evolution，4 卡更稳。

## 5.2 Python 与依赖

推荐 Python `3.10+`。

安装：

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-vllm.txt
```

说明：

- `requirements.txt` 是核心依赖
- `requirements-vllm.txt` 是本地 Qwen 推理必须项

## 5.3 环境变量

参考 `.env.example`，至少需要：

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/to/your-service-account.json
export GEMINI_PROJECT=your-vertexai-project-id
export GEMINI_LOCATION=global
```

如果要下载 gated benchmark：

```bash
export HF_TOKEN=hf_xxx
```

如果你走本地 Qwen vLLM，通常还会用到：

```bash
export VLLM_MODEL_PATH=Qwen/Qwen2.5-VL-7B-Instruct
export NUM_GPUS=4
export ENDPOINTS_PER_GPU=1
export BASE_PORT=8000
export GPU_MEM=0.85
export MAX_MODEL_LEN=16384
```

如果不是 Bright Cluster Manager/特定集群环境，建议：

```bash
export RH_SKIP_ENV_PIN=1
```

原因是 `scripts/serve_vllm_multi.sh` 默认会注入一套特定集群路径。

## 5.4 endpoints 文件

确认 `configs/endpoints.txt` 里的地址与你启动的 vLLM 端口一致。  
例如：

```text
http://localhost:8000/v1
http://localhost:8001/v1
http://localhost:8002/v1
http://localhost:8003/v1
```

---

## 6. 预检查流程

正式跑之前，先执行：

```bash
python scripts/check_env.py
```

或者：

```bash
make check
```

这个脚本会检查：

1. Python 版本
2. 核心依赖导入
3. `GOOGLE_APPLICATION_CREDENTIALS`
4. `GEMINI_PROJECT`
5. credentials JSON 是否有效
6. `configs/endpoints.txt` 中的 vLLM endpoint 是否可达

如果这里不过，不要直接跑 evolution。

---

## 7. 启动 Qwen2.5-VL-7B Sub-Agent

## 7.1 启动方式

当前仓库推荐：

```bash
bash scripts/serve_vllm_multi.sh
```

该脚本本质上会起多个：

```bash
python -m vllm.entrypoints.openai.api_server
```

关键参数包括：

- `--model "$MODEL"`
- `--served-model-name Qwen2.5-VL-7B-Instruct`
- `--max-model-len "$MAX_MODEL_LEN"`
- `--limit-mm-per-prompt '{"image": 5}'`
- `--dtype bfloat16`

## 7.2 启动后验证

检查 endpoint：

```bash
curl -s http://localhost:8000/v1/models
```

你应该看到模型 id 是：

```text
Qwen2.5-VL-7B-Instruct
```

如果服务端返回的模型名不是这个值，需要设置：

```bash
export REWARDHARNESS_SUBAGENT_MODEL=<your-served-model-id>
```

否则 `SubAgent` 调 `/v1/chat/completions` 时可能会 404。

---

## 8. 推荐复现路径 A：零改动复现 EditReward-Bench

这是我最推荐的主路径。

## 8.1 先跑一个最小 smoke test

```bash
make demo
```

它会：

- 调用 `scripts/run_evolution.py`
- 用 `examples/seed_library`
- 只跑 `1` iteration

目的不是拿论文数值，而是验证：

- Gemini Orchestrator 正常
- Qwen vLLM 正常
- Router / SubAgent / Analyzer / Evolver 全链路正常

## 8.2 跑正式 evolution

建议命令：

```bash
python scripts/run_evolution.py \
  --config configs/default.yaml \
  --results-dir results/my_run \
  --max-iters 200
```

说明：

- `configs/default.yaml` 默认 `max_iterations: 5`
- 但 `run_evolution.py` 支持 `--max-iters`
- 如果你想尽量贴近论文的长程进化，建议直接用 `200`
- 论文 Qwen 线并没有在文中给出完整 iteration 曲线细节，但长程搜索通常比 5 iter 更合理

## 8.3 关注输出

核心输出目录：

```text
results/my_run/
├── evolution_log.json
└── checkpoints/
    ├── iter_0/
    ├── iter_1/
    ├── ...
```

你要重点看：

- `best_val_acc`
- 哪个 iteration 是 best
- 最佳 checkpoint 有多少 skill/tool

脚本结尾会直接打印：

```text
Best iteration: N (val_acc=X)
```

## 8.4 对 best checkpoint 跑 EditReward-Bench

```bash
python scripts/run_benchmark.py \
  --config configs/default.yaml \
  --library-dir results/my_run/checkpoints/iter_N
```

这一步会输出：

- `K=2 accuracy`
- `K=3 accuracy`
- `K=4 accuracy`

并写入：

```text
results/benchmark_results.json
```

## 8.5 你应该如何解读这一步

如果你的目标是复现论文 Qwen 线的主要结论，那么这一步的判断标准不是“必须一模一样”，而是：

1. `K=2/K=3/K=4` 是否处于合理量级
2. 是否显著好于弱库/空库/早期 checkpoint
3. 是否能体现出 evolution 确实让性能提升

---

## 9. 推荐复现路径 B：直接 benchmark 仓库随附库

如果你不想先跑 evolution，可以直接 benchmark 当前仓库自带库：

```bash
python scripts/run_benchmark.py --config configs/default.yaml
```

含义是：

- 直接用 `src/library/`
- 用 Qwen Sub-Agent
- 在 EditReward-Bench 上得到 K=2/K=3/K=4

这个流程的优点：

- 最省时间
- 最适合先确认“当前发布版本能不能跑到一个合理区间”

它的缺点：

- 你无法确认这个库是不是你当前环境下重新进化出来的
- 更像“验证发布产物”而不是“完整复现实验过程”

因此我建议：

- 如果资源紧张，先跑它
- 如果要写严肃复现记录，还是应跑路径 A

---

## 10. 如何尽量逼近论文的 Qwen 平均分 `45.7`

## 10.1 你还缺什么

当前仓库下，要得到论文 Table 1 中 Qwen 线的 `Avg = 45.7`，还需要：

1. 跑完 `run_benchmark.py` 得到：
   - K=2
   - K=3
   - K=4
2. 再额外跑一个 **RewardHarness + Qwen** 的 `GenAI-Bench` 评测
3. 取四项平均

问题在于：

- 当前主流程没有 `RewardHarness` 版的 `GenAI-Bench` 评测脚本
- `vanilla/gemini_bench_genaibench.py` 是 baseline 脚本，不是 RewardHarness 主流程

## 10.2 最小修改设计

如果你后续愿意做最少量代码补充，我建议的最小方案不是重写 benchmark 框架，而是：

### 方案 1：仿照 `scripts/run_benchmark.py` 新增一个 `scripts/run_genaibench.py`

设计原则：

- 复用 `Library`
- 复用 `Router`
- 复用 `SubAgent`
- 只改 dataset 读取和 label 解析逻辑

这会是最干净、最合理的最小扩展。

你需要补的内容：

1. 加载 `TIGER-Lab/GenAI-Bench`
2. 把单个样本整理成：
   - source image
   - candidate_1
   - candidate_2
   - instruction
   - ground truth preference
3. 调用现有 `evaluate_pair()` 风格流程
4. 计算 accuracy
5. 写入一个 `genai_bench_results.json`
6. 最后再写一个小脚本或 `jq` 命令合并平均值

### 方案 2：扩展 `scripts/run_benchmark.py`

不推荐作为第一选择，因为：

- 会把当前只针对 EditReward-Bench 的脚本逻辑混杂进另一套 dataset schema
- 更容易把已有可用脚本改坏

所以从“最小改动风险”角度，我更推荐 **新增脚本而不是修改原脚本**。

---

## 11. 我建议你后续需要完成的代码/脚本工作

如果你的目标是“尽量少改动但尽量接近论文 Qwen 指标”，建议按以下顺序推进。

### 11.1 第一阶段：不写任何代码

先完成：

1. `make check`
2. 启动 Qwen vLLM
3. `make demo`
4. `run_evolution.py --max-iters 200`
5. `run_benchmark.py --library-dir <best_checkpoint>`

这一步已经能产出一份有意义的复现记录。

### 11.2 第二阶段：只新增，不改旧逻辑

新增一个：

```text
scripts/run_genaibench.py
```

同时可选新增一个：

```text
scripts/merge_benchmark_results.py
```

职责：

- `run_genaibench.py`
  - 只负责 RewardHarness 流程下的 GenAI-Bench accuracy
- `merge_benchmark_results.py`
  - 读取 `benchmark_results.json`
  - 读取 `genai_bench_results.json`
  - 计算 paper-style average

这样做的优点：

1. 对现有主流程侵入最小
2. 风险最小
3. 结构清晰
4. 便于单独调试

### 11.3 第三阶段：可选 wrapper

最后再新增一个 wrapper，例如：

```text
scripts/reproduce_qwen_table1.sh
```

它把下面几步串起来：

1. 环境检查
2. 启动 vLLM
3. 跑 evolution
4. 跑 EditReward-Bench
5. 跑 GenAI-Bench
6. 计算 average

这是锦上添花，不是第一优先级。

---

## 12. 具体执行顺序

下面给出一份我建议你直接照着跑的顺序。

## 12.1 安装依赖

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-vllm.txt
```

## 12.2 配置环境变量

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
export GEMINI_PROJECT=your-project-id
export GEMINI_LOCATION=global
export HF_TOKEN=hf_xxx
export RH_SKIP_ENV_PIN=1
export NUM_GPUS=4
export ENDPOINTS_PER_GPU=1
export BASE_PORT=8000
export GPU_MEM=0.85
export MAX_MODEL_LEN=16384
export VLLM_MODEL_PATH=Qwen/Qwen2.5-VL-7B-Instruct
```

## 12.3 启动服务

```bash
bash scripts/serve_vllm_multi.sh
```

另开一个终端检查：

```bash
python scripts/check_env.py
```

## 12.4 最小验证

```bash
make demo
```

## 12.5 正式进化

```bash
python scripts/run_evolution.py \
  --config configs/default.yaml \
  --results-dir results/qwen_repro \
  --max-iters 200
```

如果中断：

```bash
python scripts/run_evolution.py \
  --config configs/default.yaml \
  --results-dir results/qwen_repro \
  --max-iters 200 \
  --resume
```

## 12.6 对最优 checkpoint 跑 EditReward-Bench

```bash
python scripts/run_benchmark.py \
  --config configs/default.yaml \
  --library-dir results/qwen_repro/checkpoints/iter_N
```

其中 `iter_N` 由 evolution 结束时打印的 best iteration 决定。

## 12.7 保存你的主复现结论

至少记录：

1. 使用的 commit / 仓库版本
2. 使用的 Qwen served model id
3. 使用的 iteration 数
4. best iteration
5. K=2/K=3/K=4
6. evolution 中的 best validation accuracy
7. 最终 library 的 skill/tool 数量

---

## 13. 结果判定标准

## 13.1 什么算复现成功

对当前仓库而言，我建议把“成功”定义为三档。

### 档位 A：流程成功

满足：

- environment OK
- evolution OK
- benchmark OK

说明你已经可以稳定跑 RewardHarness(Qwen)。

### 档位 B：趋势成功

满足：

- evolved checkpoint 明显优于弱库或 baseline
- K=2/K=3/K=4 处于论文量级附近

这已经是很有说服力的实验复现。

### 档位 C：指标接近论文

满足：

- K=2/K=3/K=4 接近论文 Qwen 数值
- 若补跑了 GenAI-Bench，则 average 接近 `45.7`

这才是最强意义上的数值复现。

## 13.2 什么不应视为失败

以下情况不应立刻判定为复现失败：

1. 数值与论文不完全一致
   - 因为当前开源 Orchestrator 是 Gemini，不是论文中的 Claude
2. 5 iteration 跑不出论文量级
   - 默认 5 iter 只是 demo/短跑配置
3. 某些 iteration 有 rollback
   - 这是系统设计的一部分，不是异常

---

## 14. 风险与注意事项

## 14.1 最大的不确定性来源

1. **Orchestrator 不一致**
   - 论文写 Claude-based
   - 开源实现是 Gemini

2. **GenAI-Bench 主流程未接入**
   - 所以当前不能零改动拿到完整 `45.7`

3. **长程 evolution 的随机性和服务端波动**
   - 包括 Gemini 响应差异
   - vLLM 负载状态
   - 数据访问和 endpoint 稳定性

4. **默认配置偏短**
   - `default.yaml` 的 `5 iter` 不是论文完整实验配置

## 14.2 运行层面的常见坑

1. `serve_vllm_multi.sh` 默认假设多 GPU
   - 资源不够时需要调整 `NUM_GPUS`

2. endpoint model id 不匹配
   - 需要设置 `REWARDHARNESS_SUBAGENT_MODEL`

3. HuggingFace gated dataset 无法下载
   - 需要 `HF_TOKEN` 或 `huggingface-cli login`

4. Gemini credentials 不完整
   - `GOOGLE_APPLICATION_CREDENTIALS` 和 `GEMINI_PROJECT` 缺一不可

---

## 15. 我对“最小改动复现设计”的最终建议

如果你的目标是严肃、可落地、改动尽量少，我建议你采用下面的路线：

### 路线 A：本次就能做

1. 只使用现有代码
2. 跑 Qwen evolution
3. 跑 EditReward-Bench K=2/K=3/K=4
4. 写出对论文 Qwen 线核心结论的复现报告

这是当前最合理的第一阶段目标。

### 路线 B：下一步最小扩展

新增：

- `scripts/run_genaibench.py`
- 可选 `scripts/merge_benchmark_results.py`

然后再追论文 `45.7` average。

我不建议你一开始就重构 benchmark 框架，也不建议先碰 RL/GRPO 复现，因为那会把任务范围迅速扩大，而且当前仓库并没有完整支持。

---

## 16. 一份最实用的命令清单

```bash
# 1. 安装
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-vllm.txt

# 2. 环境变量
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
export GEMINI_PROJECT=your-project-id
export GEMINI_LOCATION=global
export HF_TOKEN=hf_xxx
export RH_SKIP_ENV_PIN=1
export NUM_GPUS=4
export ENDPOINTS_PER_GPU=1
export BASE_PORT=8000
export GPU_MEM=0.85
export MAX_MODEL_LEN=16384
export VLLM_MODEL_PATH=Qwen/Qwen2.5-VL-7B-Instruct

# 3. 启动 vLLM
bash scripts/serve_vllm_multi.sh

# 4. 预检查
python scripts/check_env.py

# 5. 最小验证
make demo

# 6. 正式 evolution
python scripts/run_evolution.py \
  --config configs/default.yaml \
  --results-dir results/qwen_repro \
  --max-iters 200

# 7. 对最优 checkpoint 跑 benchmark
python scripts/run_benchmark.py \
  --config configs/default.yaml \
  --library-dir results/qwen_repro/checkpoints/iter_N
```

---

## 17. 总结

如果只依赖当前仓库、且坚持尽量少改动，那么最合理的论文复现目标不是一步到位追 `45.7`，而是：

1. **先完整复现 Qwen 版 RewardHarness 的 evolution + EditReward-Bench**
2. **确认它相对 baseline 的性能提升趋势**
3. **再用一个最小新增脚本补上 GenAI-Bench**

这条路径工程上最稳，结论上也最可信。

当前仓库已经足够支持你完成前两步；第三步只缺一个很薄的 RewardHarness 版 `GenAI-Bench` 评测脚本，而不需要重构整套系统。
