# autoresearch (Thesis Adaptation Blueprint)

![teaser](progress.png)

本仓库当前代码仍是极简 `prepare.py + train.py + program.md` 研究循环；本文档将其重定义为：

> 一个用于毕业设计的 **自动实验引擎（Auto-Experiment Engine）**，用于围绕本地 Qwen 主模型、外部 AI 协同、个性化知识库/RAG、LoRA 自我迭代和评测脚本进行可复现实验。

---

## 1. Thesis-oriented target

你要自动化的不是“整个系统从零开发”，而是以下可量化研究任务：

1. **RAG 自动优化**：检索参数、提示词、证据格式自动搜索。
2. **SFT 数据构造优化**：知识抽取模板、过滤规则、样本配比自动迭代。
3. **QLoRA 短时自迭代**：在单卡（RTX 4090）约束下自动寻找更好的 adapter 配置。
4. **统一评测与晋级**：固定评测集 + 固定评分函数 + 晋级/回滚机制。
5. **可复现实验配置**：固定随机种子、固定数据切分、固定运行时预算。

---

## 2. Recommended repository layout

```text
programs/
  rag.md
  sft.md
  qlora.md

editable/
  rag_candidate.py
  prompt_candidate.py
  sft_rules_candidate.py
  qlora_candidate.py

core/
  runner_rag.py
  runner_sft.py
  runner_qlora.py
  metrics.py
  promote.py
  logger.py

knowledge/
  personal_kb/              # 本地个性化知识（结构化/非结构化）

models/
  base/                     # Qwen 基座模型（只读）
  adapters/                 # LoRA adapters

eval/
  evalset_personal.jsonl
  evalset_project.jsonl
  evalset_procedure.jsonl

artifacts/
logs/
results/
SAFE_EDIT_ALLOWLIST.txt
```

---

## 3. Hard constraints (must enforce)

- 单 GPU（RTX 4090）
- 主模型固定：`Qwen/Qwen2.5-7B-Instruct`
- 禁止全参数微调（仅 LoRA/QLoRA）
- 单次训练实验时长 `<= 15 min`
- `max_seq_length <= 1024`
- 仅允许编辑白名单文件（`SAFE_EDIT_ALLOWLIST.txt`）
- 不改数据库 schema、不改线上 API 协议

建议白名单：

```text
editable/rag_candidate.py
editable/prompt_candidate.py
editable/sft_rules_candidate.py
editable/qlora_candidate.py
programs/
```

---

## 4. Multi-model collaboration strategy

### 4.1 Local primary model (must)

- 本地主模型：Qwen（负责推理、RAG 回答、SFT/LoRA 的目标模型）。

### 4.2 External AI collaborators (optional but controlled)

外部 AI 仅用于“辅助信号”，不直接替代主模型决策：

- 辅助重写 query（query rewrite）
- 候选答案对比投票（judge/rerank）
- 数据清洗建议（规则候选）

要求：

- 每次调用需记录 provider/model/temperature/token 用量到 `logs/`。
- 外部 AI 输出必须可追溯，不可直接覆盖 gold 评测集。

---

## 5. Personalized KB & RAG pipeline

1. 用户资料、项目资料、流程资料进入 `knowledge/personal_kb/`。
2. 固定切分/索引参数建立向量库（建议 FAISS + metadata）。
3. `editable/rag_candidate.py` 决定候选配置：
   - `top_k`
   - `chunk_size`
   - `overlap`
   - metadata filter
   - rerank 开关
4. `editable/prompt_candidate.py` 负责回答格式和证据引用模板。
5. `core/runner_rag.py` 在固定评测集上输出 `results/latest_rag_metrics.json`。

---

## 6. LoRA self-iteration policy

在 `editable/qlora_candidate.py` 内限制搜索空间：

```python
BASE_MODEL = "Qwen/Qwen2.5-7B-Instruct"
SEARCH_SPACE = {
    "lora_r": [8, 16, 32],
    "lora_alpha": [16, 32, 64],
    "lora_dropout": [0.0, 0.05, 0.1],
    "learning_rate": [1e-4, 2e-4, 3e-4],
    "max_seq_length": [512, 1024],
    "gradient_accumulation_steps": [8, 16, 32],
}
```

禁止项：

- 切换到 14B+
- 全参数微调
- `max_seq_length > 1024`
- 删除历史最优 adapter

`core/runner_qlora.py` 每次只保存 adapter，并记录：

- 训练配置哈希
- 训练/验证分数
- 显存峰值
- 训练时长

---

## 7. Unified metrics and promotion gate

建议在 `core/metrics.py` 定义统一总分：

```python
def total_score(metrics):
    return (
        0.40 * metrics["answer_correctness"]
        + 0.25 * metrics["evidence_hit_rate"]
        + 0.20 * metrics["preference_consistency"]
        + 0.10 * metrics["format_pass_rate"]
        - 0.05 * metrics["latency_penalty"]
    )
```

`core/promote.py` 规则：

```python
if new_score > best_score + 0.01 and metrics["evidence_hit_rate"] >= best_evidence:
    promote()
else:
    rollback()
```

并限制：仅保留最近 N=20 次实验快照。

---

## 8. Reproducible experiment protocol

每次实验必须固化：

- git commit hash
- 数据集版本号/切分版本号
- 随机种子
- 依赖版本（`uv.lock`）
- GPU/驱动/CUDA 信息
- 运行命令
- 指标 JSON 与日志路径

建议追加 `results.tsv` 字段：

```text
commit	run_type	score	main_metric	memory_gb	seconds	status	description
```

---

## 9. Migration path (3 phases)

### Phase 1: RAG 自动实验（推荐先做）

- 新增 `SAFE_EDIT_ALLOWLIST.txt`
- 建立 `programs/rag.md`
- 实现 `runner_rag.py + metrics.py`
- 跑 20~50 轮候选组合

### Phase 2: SFT 数据自动实验

- 新增 `programs/sft.md`
- 实现 `runner_sft.py`
- 自动筛选样本构造规则

### Phase 3: QLoRA 自动实验

- 新增 `programs/qlora.md`
- 实现 `runner_qlora.py`
- 自动晋级最佳 adapter

---

## 10. What this framework is / is not

**是**：自动实验员（bounded autonomous researcher）
**不是**：自动全栈开发员（unbounded auto-builder）

当你给它：清晰目标 + 明确边界 + 固定评测，它会很强；
当你给它：开放式“帮我做完整系统”，它会失控且不可复现。

---

## License

MIT
