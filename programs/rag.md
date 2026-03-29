# RAG Track Program

## Objective
Maximize `total_score` on fixed `eval/evalset_*.jsonl`.

## Allowed knobs
- top_k
- chunk_size
- overlap
- metadata filter
- rerank strategy
- answer/evidence formatting prompt

## Forbidden
- changing evaluation datasets
- changing metrics weights
- editing files outside allowlist

## Promotion
Promote only if:
- score improves by >= 0.01
- evidence_hit_rate does not decrease
