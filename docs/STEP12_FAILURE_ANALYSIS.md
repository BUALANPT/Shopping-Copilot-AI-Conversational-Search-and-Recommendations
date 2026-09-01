# Project 4 Step 12: Systematic Failure Analysis and Audit Conclusions

> Historical Note: This report documents the state of files at the time of Step 12. During the final packaging phase, `starter/agent.py` was replaced with a thin adapter for the official Agent; the original weak baseline was moved to `starter/baseline_agent.py`, while the official evaluator remains unmodified.

## 1. Scope of This Round

This round re-runs the current default Agent exclusively on the frozen `dev 150` dataset, performing a round-by-round replay for all misses. The objective is to identify general failure patterns in recall, fusion, constraint filtering, final ranking, and prompting strategies. No modifications to ranking weights are made in this round.

Strict Boundaries:

- No reading of `holdout.jsonl`, `public_set.jsonl`, or any final-only split;
- Catalog SHA256 hashes remain consistent before and after;
- No modifications to `evaluator/local_evaluator.py` or `starter/agent.py`;
- Each round returns a maximum of 10 recommended IDs for audit, all belonging to the read-only catalog;
- No special rules are written based on sample IDs, target ASINs, or individual failure cases;
- Default `semantic_ranker_enabled=false`; this round is not an experiment on Qwen accuracy.

## 2. Frozen Dev Results

The run configuration uses Hashing Dense with Cross-Encoder and Qwen disabled. The full 150 samples took 202.41 seconds to process.

| Metric | Result |
|---|---:|
| HR@10 | 0.913333 |
| MRR | 0.597302 |
| MTTC | 4.406667 |
| Efficiency | 0.659333 |
| TechnicalScore | 0.767724 |

| Scenario | Samples | Hits | Misses | Miss Rate | MRR | MTTC |
|---|---:|---:|---:|---:|---:|---:|
| Buying | 60 | 55 | 5 | 8.33% | 0.599107 | 3.566667 |
| Browsing | 60 | 57 | 3 | 5.00% | 0.641653 | 4.816667 |
| Intent Override | 22 | 18 | 4 | 18.18% | 0.462428 | 5.500000 |
| Boundary | 8 | 7 | 1 | 12.50% | 0.622024 | 4.625000 |

Intent Override is currently the most obvious shortcoming; however, all 4 override misses entered the final list after the new intent took effect. The issue lies primarily in top-ranking rather than state not being cleared.

## 3. Root Causes of 13 Failed Sessions

| Root Cause | Count | Evidence |
|---|---:|---|
| Not recalled by any router | 2 | Target absent from BM25, Category, Metadata, Hashing Dense, fusion, and final list |
| Final rank > 10 | 11 | Target entered the full final list but best position was still rank 12–100 |
| Lost during fusion | 0 | Recalled targets did not disappear during fusion |
| Incorrectly deleted by constraint filtering | 0 | No targets entered fusion were completely filtered out in this round |

Router Coverage: BM25 recalled 10/13, Metadata recalled 9/13, Category recalled 5/13, Hashing Dense recalled 3/13; 11/13 visible in the final full list.

Distribution of Best Final Ranks:

- Completely missing: 2;
- Rank 11–20: 2;
- Rank 21–50: 8;
- Rank 51 and beyond: 1.

This indicates that the next round cannot simply expand all candidate pools indiscriminately. First, address the two cases not recalled by any router, then use general ranking evidence to validate the two near-misses in the Top 20, before considering candidates at deeper positions.

## 4. Dialogue and Efficiency Audit

- The 13 failed sessions raised a total of 91 questions, averaging 7 per session;
- Failed sessions with repeated `ask_attribute` calls: 0;
- Sessions stopped asking further questions in rounds 8–10, consistent with the current question window;
- 0 failed sessions triggered Over-General truncation, as the official initial message already included specific coarse categories;
- 9 failed sessions applied hard constraints, and 1 experienced safety relaxation;
- Qwen active rounds: 0, Token usage: 0, consistent with the default disabled baseline.

The current issue is not "asking the same attribute repeatedly," but rather some sessions continuously receiving "no additional preferences" while still utilizing all 7 different attributes. The next round can study early stopping or strategy switching based on consecutive uninformative answers, but this must be validated using an independent dialogue-specific dataset and cannot rely solely on adjustments from these 13 misses.

## 5. Suggested Order for Next Round

1. **Recall Missing Analysis**: Compare target text with the generic query builder for the 2 `not_recalled` sessions to test whether category normalization, field queries, and BGE supplementation can improve recall across samples.
2. **Top 20 Near-Miss Analysis**: First conduct competitive feature comparisons for candidates at final ranks 12 and 19; Qwen can only re-rank the frozen Top 30, so complete candidate ordering must be preserved to validate with the switch disabled.
3. **Override Analysis**: Continue using the eligible rank after override takes effect; verify new intent queries and generic reranker; do not treat the appearance of a covered target before the cutoff as a valid hit.
4. **Boundary Analysis**: Investigate why Category/Dense evidence was pushed to rank 100 in final sorting while maintaining "no preference" without generating pseudo-constraints.
5. **Questioning Efficiency Analysis**: Test stopping further questioning or switching to scenario questions for sessions with two consecutive turns yielding no new slots; evaluate MTTC, slot acquisition, and duplicate recommendation rates.
6. **Strict Pairing**: Any changes must be paired with this report on the same dev SHA, catalog SHA, and commit; only after passing can a single final-only confirmation be discussed.

## 6. Relationship Between Qwen and Step 12

Step 11 completed local Qwen connection and security smoke tests, but the default official path still keeps LLMs disabled. Consequently, the baseline report for Step 12 does not mix Qwen smoke test results with Hashing dev metrics.

Subsequent Qwen accuracy experiments must satisfy:

- The same frozen dev split;
- The same retrieval backend and weights;
- Paired results with Qwen disabled/enabled;
- Recording HR@10, MRR, MTTC, total duration, P50/P95, Tokens, and fallbacks;
- Only if both dev metrics and latency thresholds pass is a single confirmation without using the final split allowed.

## 7. Reproduction Commands

```powershell
python -m experiments.verify_frozen_data

python -m experiments.run_experiment `
  --config experiments/configs/hybrid_steps_3_10.json `
  --name step12_hashing_dev `
  --run-dir experiments/runs/step12_hashing_dev `
  --overwrite

python -m experiments.trace_failures `
  --results experiments/runs/step12_hashing_dev/results.json `
  --dataset data/splits/dev.jsonl `
  --catalog data/catalog.jsonl `
  --agent solution.agent:Agent `
  --output experiments/analysis/step12_hashing_dev_failures.json `
  --review-limit 20

python -m experiments.finalize_failure_report `
  --input experiments/analysis/step12_hashing_dev_failures.json
```

Detailed round-by-round reports are located at `experiments/analysis/step12_hashing_dev_failures.md`; the JSON file of the same name saves the complete state, queries, questions, router ranks, Top 10, constraints, and LLM status.
