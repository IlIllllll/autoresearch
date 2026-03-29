# QLoRA Track Program

## Objective
Maximize `post_train_total_score - base_total_score`.

## Allowed knobs
- lora_r, lora_alpha, lora_dropout
- learning_rate, warmup_ratio
- max_seq_length (<= 1024)
- gradient_accumulation_steps

## Hard constraints
- Base model fixed: `Qwen/Qwen2.5-7B-Instruct`
- Single RTX 4090
- No full-parameter finetuning
- Train stage <= 15 minutes

## Promotion
Promote adapter only if score increases by >= 0.01 with no safety regression.
