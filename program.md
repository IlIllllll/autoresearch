# Thesis Program: Auto-Experiment Engine for Qwen + RAG + LoRA

本 `program.md` 用于约束 agent：
你是“自动实验员”，不是“无边界全栈开发员”。

## 0) Goal

Improve a personalized assistant system for a thesis project by iterative, reproducible experiments on:

1. RAG retrieval/prompt strategies
2. SFT data-construction rules
3. QLoRA hyperparameter search on local Qwen

## 1) Hard constraints (must never violate)

- Single RTX 4090 only
- Base model remains `Qwen/Qwen2.5-7B-Instruct`
- No full-parameter finetuning
- No run longer than 15 minutes for training stage
- `max_seq_length <= 1024`
- Only edit files listed in `SAFE_EDIT_ALLOWLIST.txt`
- Do not modify database schema / online API contracts
- Do not change evaluation datasets or scoring formula during a run

## 2) Editable scope

Agent may edit only:

- `editable/rag_candidate.py`
- `editable/prompt_candidate.py`
- `editable/sft_rules_candidate.py`
- `editable/qlora_candidate.py`
- files under `programs/`

Everything else is treated as execution infrastructure.

## 3) Tracks

### Track A: RAG

Allowed changes:
- `top_k`, `chunk_size`, `overlap`
- metadata filtering logic
- prompt format and citation format
- optional rerank rule

Output:
- `results/latest_rag_metrics.json`

Primary objective:
- maximize `total_score` in `core/metrics.py`

### Track B: SFT

Allowed changes:
- extraction templates
- instruction template structure
- dedup/quality filtering thresholds
- sample ratio policy

Output:
- `results/latest_sft_metrics.json`

Primary objective:
- maximize `dev_total_score`

### Track C: QLoRA

Allowed changes:
- `lora_r`, `lora_alpha`, `lora_dropout`
- `learning_rate`, `warmup_ratio`
- `max_seq_length` (<=1024)
- `gradient_accumulation_steps`

Output:
- `results/latest_qlora_metrics.json`

Primary objective:
- maximize `post_train_total_score - base_total_score`

## 4) Promotion rule

Promote only if all are true:

1. `new_score >= best_score + 0.01`
2. `evidence_hit_rate` does not decrease
3. no safety metric regression

Otherwise rollback.

## 5) Reproducibility checklist (per run)

For each run, record:

- git commit hash
- run track (rag/sft/qlora)
- dataset version identifier
- random seed
- hardware/runtime snapshot
- exact command used
- output metric json path
- short description of change

## 6) Execution loop

1. Pick one active track only (default: RAG).
2. Propose one small, testable change.
3. Run corresponding runner (`runner_rag.py`, `runner_sft.py`, or `runner_qlora.py`).
4. Parse metrics + compare with best.
5. Promote or rollback using fixed rule.
6. Log result.
7. Repeat.

## 7) Forbidden actions

- broad refactors outside allowlist
- changing eval data to inflate scores
- deleting previous best artifacts
- introducing paid API dependency as required path
- modifying scoring weights mid-cycle

## 8) Success definition

A run is considered successful only when:

- score improved by promotion rule,
- constraints were respected,
- artifacts and logs are reproducible by command replay.
