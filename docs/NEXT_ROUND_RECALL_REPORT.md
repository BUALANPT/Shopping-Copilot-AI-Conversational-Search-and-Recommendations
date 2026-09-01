# Project 4: Next-Round Recall Enhancement and Risk Audit Report

> Historical Note: This report records the experimental invariants at that time. During the final submission packaging phase, `starter/agent.py` was refactored into a thin adapter for `solution.agent.Agent` to ensure official default commands run the formal solution; the original weak baseline is preserved in `starter/baseline_agent.py`. The official evaluator has never been modified.

## 1. Current Round Conclusions

The default Agent has shifted from "Dense participating simultaneously in RRF and reranking" to "Dense only supplementing candidates, with head ranking primarily driven by sparse retrieval and interpretable features." Subsequently, a small-weight, log-normalized review count soft feature was added to the dev failure boundary to provide deterministic tie-breaking for text-homogeneous products.

| Solution | Data | HR@10 | MRR | MTTC | Tech Score | Total Time/Sec |
|---|---|---:|---:|---:|---:|---:|
| Old hashing hybrid | dev 150 | 0.880000 | 0.561011 | 4.560000 | 0.737103 | See old run |
| Dense only supplement candidates | dev 150 | 0.880000 | 0.583780 | 4.640000 | 0.742334 | 170.08 |
| Dense supplement + review count soft tie-breaking (Current Default) | dev 150 | **0.906667** | **0.593405** | **4.453333** | **0.762288** | 229.47 (before caching) |
| Top-30 cross-encoder | dev 150 | 0.880000 | 0.585672 | 4.626667 | 0.743168 | 396.35 |

The current default solution simultaneously improves HR and MRR. The cross-encoder achieves only a marginal MRR gain but increases runtime by 2.33x, thus failing to advance to the default path.

## 2. Audit of 24 Legacy Failed Sessions

The audit script records BM25, metadata, Dense, sparse RRF, supplemented candidates, and final reranking positions for each round, strictly distinguishing between invalid and valid rounds before and after intent override activation.

- 3 sessions: Target not recalled by BM25, metadata, or Dense.
- 20 sessions: Target entered the final reranking list but consistently fell behind rank 11.
- 1 session: Target entered the fused candidate pool but was removed by constraint filtering.
- Of the 7 override misses, 6 resulted in final ranks 26–117 after coverage activation, and 1 was filtered out by constraints.

Detailed data:

- `experiments/analysis/legacy_24_failures.json`: Complete machine-readable round-by-round records.
- `experiments/analysis/legacy_24_failures.md`: Summary table.
- `experiments/analysis/dense_supplement_dev_failures.json`: Remaining dev misses with the Dense candidate supplement version.

Therefore, this round did not blindly expand Dense candidates but prioritized fixing head ranking.

## 3. Major Code Changes

### `solution/agent.py`

- Dense defaults to only supplementing candidates and no longer adds RRF scores redundantly to existing sparse candidates.
- Added a per-round diagnostic that can be disabled; default is off, which does not affect the official evaluator interface.
- Optional Top-30 cross-encoder runs after standard reranking; default is off.

### `solution/retrieval/fusion.py`

- Added `supplement_with_dense`.
- Dense hits on existing sparse candidates only append Dense evidence without altering sparse RRF scores.
- Dense-only products can enter the candidate pool, but initial scores must not exceed the tail of sparse candidates.

### `solution/ranking/reranker.py` and `solution/config.py`

- All reranking weights centralized into a configuration object for strict ablation.
- Dense head influence reduced from 0.20 to 0.04.
- Added `log1p(rating_number)` soft feature with 0.04 weight; no hard filtering applied, missing values treated as 0.
- Tied results still converge deterministically via ASIN.

### `solution/retrieval/dense.py`

- Continued validation of catalog SHA256, model, matrix row count, and ID count.
- Added complete catalog row count validation; partial smoke-test indexes explicitly disabled.
- Added deterministic LRU query cache; returns new Candidate objects to prevent subsequent reranking from polluting cached objects.

### `solution/ranking/cross_encoder.py`

- Uses FlashRank ONNX cross-encoder, default off.
- Allows only Top 30–50; current ablation is Top 30.
- Single budget 2500 ms; results exceeding budget are discarded and that route is closed.
- Safe degradation on dependency, model, or inference failure, without affecting the default Agent.

### `scripts/build_embeddings.py`

- Hashing and BGE use different default filenames to avoid experimental cross-contamination.
- Batched writing to NumPy memmap supports progress checkpoints and `--resume`.
- Metadata includes `catalog_row_count`, `indexed_row_count`, and `complete_catalog`.
- Local machine completed BGE 32-item and 1000-item smoke tests; 1000 items took ~162 seconds, extrapolating to over 2 hours for full 50K on local CPU. Therefore, partial BGE was not treated as a formal metric this round.

### `experiments/trace_failures.py`

- Extracts misses from existing `results.json`.
- Reproduces evaluator initial messages, clarification responses, and override timing sequences.
- Outputs per-round route rank, query, status, question, and final Top 10.

### `experiments/frozen_datasets.json`, `experiments/frozen.py`

- Freeze SHA256 and sample counts for dev, holdout, and public.
- Tuning configurations running on holdout/public must explicitly pass `--final-eval`.
- Validate current data files before execution; silent use of modified data is prohibited.

### `solution/ablations.py`

- `LegacyDoubleDenseAgent`: Used only to reproduce legacy double-Dense scoring.
- `CrossEncoderTop30Agent`: Used only for cross-encoder dev experiments.
- `StrictBgeAgent`: Directly rejects experiments if BGE files are incomplete, mismatched, or fallback occurs, preventing sparse/hashing results from being mislabeled as BGE.

## 4. Risk Mitigation Status

| Risk | Handling This Round | Status |
|---|---|---|
| BGE 50K incomplete | Backend independent file, resume from breakpoint, integrity validation, strict ablation; completed 1000-item smoke test | Reduced, still requires appropriate hardware for full build |
| public 200 susceptible to overfitting | Weights adjusted only on dev; holdout/public include frozen hash and final switch | Controlled; private 800 remains the final basis |
| 375s runtime | Dense/BM25 deterministic LRU cache; cross-encoder rejected due to 2.33x runtime increase | Partially resolved; FAISS can be tested subsequently |
| Low price coverage | Retain "conservative filtering only if value exists, missing does not reject"; no new hard price rules added | Controlled |
| No cross-encoder evidence | Completed Top-30 dev ablation, budget, and disable switch | Completed; currently not advancing |
| holdout leakage | Files frozen; weights will no longer be tuned based on holdout results thereafter | Implemented |

## 5. Reproduction Commands

```powershell
# Verify frozen data
.\.venv\Scripts\python -m experiments.verify_frozen_data

# Default hashing dense dev
.\.venv\Scripts\python -m experiments.run_experiment --config experiments\configs\hybrid_steps_3_10.json

# Per-round audit of failed sessions
.\.venv\Scripts\python -m experiments.trace_failures `
  --results experiments\runs\<run>\results.json `
  --dataset data\splits\dev.jsonl `
  --output experiments\analysis\dev_failures.json

# Build full BGE on appropriate hardware; add --resume after interruption
.\.venv\Scripts\python scripts\build_embeddings.py --backend fastembed --batch-size 256 --checkpoint-every 1000

# Strict BGE same-split ablation; some indexes will error directly
.\.venv\Scripts\python -m experiments.run_experiment --config experiments\configs\bge_dense_dev.json

# Optional cross-encoder Top-30 dev ablation
.\.venv\Scripts\python -m experiments.run_experiment --config experiments\configs\cross_encoder_top30_dev.json

# Final evaluation only after candidates are fully frozen
.\.venv\Scripts\python -m experiments.run_experiment --config experiments\configs\hybrid_steps_3_10.json `
  --dataset data\splits\holdout.jsonl --final-eval
```

## 6. Next Round Order

1. Current default weights are frozen; no longer reading holdout for hyperparameter tuning.
2. Complete 50K BGE build using breakpoint builder on GPU/high-core CPU machines, and run `StrictBgeAgent` same-dev split ablation.
3. Only if BGE improves HR/MRR on dev simultaneously with acceptable runtime, generate a brand new confirmation run on an unused final split.
4. Cross-encoder currently not advancing; only smaller models, smaller Top-N, or caching that reduces extra runtime within budget will reopen experiments.
5. If continuing to improve override, prioritize fixing low-recall queries and constraint states after coverage activation, without increasing Dense head weights.

## 7. Invariant Review

- `evaluator/local_evaluator.py` and `starter/agent.py` remain unmodified.
- Catalog is read-only; recommended IDs come only from catalog documents and retrieval candidates.
- Disable if Dense catalog SHA256/row count/model mismatch; strict BGE experiments fail directly.
- Default Agent safely degrades if missing Dense or cross-encoder dependencies.
- Default retrieval, fusion, reranking, questioning, and ASIN tie-breaking are deterministic.
- Current tests, `compileall`, ResourceWarning strict mode, and `git diff --check` must be executed again before delivery.
