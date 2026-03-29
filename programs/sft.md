# SFT Track Program

## Objective
Maximize `dev_total_score` for SFT data construction quality.

## Allowed knobs
- extraction templates
- sample generation templates
- dedup rules
- quality filter thresholds
- category ratio (preference/procedure/fact)

## Forbidden
- manual edits to eval labels
- expanding training duration beyond budget

## Promotion
Promote only if dev score improves and data quality checks pass.
