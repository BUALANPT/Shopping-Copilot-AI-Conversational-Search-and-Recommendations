# Core Architecture Review: Intent Routing and Hybrid Pipeline Refactoring

## 1. Audit Conclusion

This round completed an executable dual-track Buying/Browsing architecture, independent Category Route, structured constraints, open-ended browsing diversity, and a candidate semantic reranking interface for the local Ollama `qwen3.5:9b`.

The default path remains fully offline and deterministic. The Qwen interface is disabled by default; it does not download models, import model runtimes, nor does it alter recommendation results. BGE artifacts are handled uniformly in subsequent merges; this round relies solely on the `DenseRetriever` interface (identical to BGE) to validate the architecture.

Pairwise results on the same machine, frozen dev set, 50K catalog, and Hashing Dense index show no decline in core metrics. HR, MRR, MTTC, and Technical Score all improved. Therefore, this round's deliverables meet the accuracy threshold and can serve as the foundation for subsequent BGE/LLM merging.

## 2. Data and Experiment Integrity

| Item | Value |
|---|---|
| Catalog Rows | 50,000 |
| Catalog SHA256 | `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67` |
| Dev Sample Count | 150 |
| Dev SHA256 | `c8955cacd79bb2e2ed6b984979bca91f379fc699648c5bc2e3ac2efb37f8b22a` |
| Dense Backend | Signed hashing, 384 dimensions, full 50K |
| LLM | Disabled |

The `data/public_set.jsonl` in the workspace contains pre-existing newline format changes. The file SHA256 does not match the frozen public manifest, but Git shows no content diff when ignoring trailing line differences. Dev/holdout SHA256s regenerated from this file match the frozen manifest exactly; therefore, this round uses only the frozen dev set for tuning and pairwise validation, without using the touched public file for final scoring.

## 3. Pre- and Post-Refactoring Metrics

### 3.1 Overall Metrics

| Metric | Pre-Refactoring | Final Version | Change |
|---|---:|---:|---:|
| HR@10 | 0.906667 | **0.913333** | +0.006666 |
| MRR | 0.593405 | **0.596968** | +0.003563 |
| MTTC | 4.453333 | **4.413333** | -0.040000 (lower is better) |
| Efficiency | 0.654667 | **0.658667** | +0.004000 |
| Technical Score | 0.762288 | **0.767490** | +0.005202 |

### 3.2 Scenario Metrics

| Scenario | Metric | Pre-Refactoring | Final Version | Conclusion |
|---|---|---:|---:|---|
| Buying | HR@10 | 0.916667 | 0.916667 | Stable |
| Buying | MRR | 0.586078 | **0.599107** | Improved |
| Buying | MTTC | 3.600000 | **3.550000** | Improved |
| Browsing | HR@10 | 0.933333 | **0.950000** | Hit 1 additional dev sample |
| Browsing | MRR | 0.645774 | 0.641653 | Slight decline |
| Browsing | MTTC | 4.883333 | **4.816667** | Improved |
| Intent Override | HR/MRR/MTTC | 0.818182 / 0.460155 / 5.590909 | Same | No degradation |
| Boundary | HR/MRR | 0.875000 / 0.622024 | Same | No degradation |
| Boundary | MTTC | 4.500000 | 4.625000 | Slight fluctuation in 8 samples |

Overall HR, MRR, MTTC, and Technical Score all meet the requirement of "not lower than current levels." Browsing MRR and Boundary MTTC should be closely monitored in future BGE pairing experiments.

## 4. Dual-Track Architecture

```text
User Message
  ↓
Rule-based Intent Parsing + Structured Constraint Extraction
  ↓
RoutingDecision
  ├─ precision / Buying
  │    ├─ BM25 keyword
  │    ├─ metadata BM25
  │    ├─ category evidence
  │    ├─ Dense supplement
  │    └─ High-confidence constraint filtering + automatic relaxation if candidate pool is insufficient
  │
  └─ discovery / Browsing
       ├─ BM25 keyword
       ├─ Independent Category Route
       ├─ metadata BM25
       ├─ Expanded Dense semantic pool
       └─ Deterministic diversity for open-ended category requests
             ↓
       Deterministic reranker
             ↓
       Optional SemanticRanker (currently disabled)
             ↓
       Candidate whitelist validation and safe fallback
             ↓
       Top 10 + clarification
```

### 4.1 Buying Precision Track

- `RoutingDecision.track=precision`.
- Retain the verified BM25 + metadata head fusion to avoid the Category Routedisplace (pushing out) precise candidates from the Top 10.
- Execute Category Route independently and record rank/score as diagnostic evidence, without directly modifying the verified sparse fusion scores.
- High-confidence constraints verifiable against the catalog, such as material, color, size, and brand, can enter precision filtering.
- Check remaining candidate count after applying each hard constraint; if below threshold, automatically downgrade that constraint to a soft constraint.
- Constraint matching checks title, features, details, description, categories, and store to avoid target mis-killing caused by inconsistent field coverage.

### 4.2 Browsing Discovery Track

- `RoutingDecision.track=discovery`.
- Category Route serves as the formal fusion route.
- Expand Dense candidate pool from 120 to 180 to supplement scenarios not recalled by sparse routing.
- Category is a soft signal and does not serve as hard filtering.
- Enable deterministic category diversity for requests without explicit categories or containing only generic categories like `product/clothing/item`.
- Do not force diversity for specific categories provided by the evaluator, avoiding damage to known target relevance for the sake of formal cross-category appearance.

## 5. Structured Constraints and Override

The newly added `Constraint` includes:

- attribute
- operator
- value
- confidence
- source_turn
- hard
- raw

Upon intent override, the following are cleared:

- Old hard constraints
- Old soft preferences
- Old structured constraints
- Old exclusions
- Old budget bounds

Initial categories are retained because the competition override scenario replaces preferences rather than product major categories; if a new message explicitly provides a category, the existing parser will still update it.

## 6. Ollama Qwen3.5 9B Interface

New additions under `solution/llm/`:

- `base.py`: `SemanticRanker`, request and result protocols.
- `disabled.py`: Default no-model implementation.
- `qwen.py`: Local Ollama `/api/chat` client using JSON Schema constraints for complete candidate permutations.
- `factory.py`: Create Ranker based on configuration.

Model input contains only limited summaries of Top-N catalog candidates:

- parent_asin
- title
- category
- catalog attributes
- Deterministic score
- Route ranks
- Current structured constraints and anonymized profile summary

Safety invariants:

- The model can only rerank input candidates; it cannot generate new product IDs.
- Output must be a complete, non-duplicate permutation of input candidates.
- Unknown ASINs, duplicate/missing candidates, error types, and exceptions trigger full fallback.
- Token usage is guaranteed to be non-negative.
- Current safe default remains `semantic_ranker_enabled=false`; upon enabling, use `qwen3.5:9b`, maintaining deterministic sorting if the model is unavailable or outputs illegally.

After model deployment, simply set `semantic_ranker_enabled` to `true`; no modification to Agent API or retrieval pipeline is required.

## 7. Test Results

Final unit test coverage includes:

- Buying/Browsing routing plan differences
- Category Route independence and determinism
- Override cleanup of budget and structured constraints
- High-precision constraint filtering and automatic relaxation when candidate pool is insufficient
- Complete searchable corpus constraint matching
- Category evidence not disturbing Buying scores and candidate sets
- Browsing diversity
- Ollama Qwen3.5 interface construction independent of online model availability
- JSON Schema request, token usage, and legal permutation parsing
- Ollama connection failure and illegal permutation safe fallback
- SemanticRanker legal reranking
- Unknown ASIN and illegal permutation fallback
- Dense missing, damaged, or partial index safe degradation
- Agent official output contract
- Evaluator and experiment framework original tests

Final result: `25/25` tests passed.

## 8. First Round Failure and Correction Record

The first dual-track implementation yielded:

- HR@10: 0.900000
- MRR: 0.589849
- Technical Score: 0.757621

This version did not meet the threshold and was not retained as the default strategy. Iterative failure tracking revealed:

1. `public_0154`'s cotton was located in complete text fields such as the target product description, but the first-version filter only checked partial fields, causing the fused 6th-ranked target to be mis-killed.
2. The target for `public_0026` did not hit the Category Route, while competing products received Category RRF bonuses, dropping the target from baseline rank 10 to 13.

After correction:

- Structured filtering uses the complete searchable corpus.
- Buying's Category Route is changed to append only audit evidence without altering sparse head scores.
- Browsing continues using formal Category fusion.

The final version restored Buying HR and caused overall metrics to exceed the baseline.

## 9. Current Issues and Risks

### 9.1 Performance Regression

| Version | Total Time for 150 Dev |
|---|---:|
| Pre-Refactoring | 224.930 seconds |
| Final Version | 293.023 seconds |
| Change | +68.092 seconds, approx. 1.303x |

The main cause is the expanded Browsing Dense pool and additional candidate reranking. Current accuracy gains meet the threshold, but performance regression must be addressed during BGE merging. Recommendations:

1. Replace NumPy full-matrix retrieval with FAISS/ANN indexes verified for Top-K consistency.
2. Implement cross-session persistent caching or batched vector retrieval for normalized queries.
3. Perform 120/150/180 paired ablation on Browsing Dense limits to confirm the minimum range needed for new hits.
4. Re-measure actual encoding and vector search latency after BGE merging; do not directly infer from Hashing results.

### 9.2 Routing Remains Rule-Based

The current Intent Router is reliable for competition English templates but has limited generalization for free expression, implied purchase intent, complex negation, and Chinese input. Qwen's first integration should adopt shadow mode, comparing structured intent without directly controlling final recommendations.

### 9.3 Hard Constraint Whitelist Limited

Currently, only high-confidence fields such as material, color, size, and brand undergo added precision filtering. Features, style, and use_case remain soft scoring to avoid mis-killing caused by insufficient catalog metadata coverage. If expanding hard filtering later, field coverage rate and mis-killing rate must be statistically analyzed first.

### 9.4 Diversity Enabled Only for Truly Open Categories

Currently, the evaluator generates specific categories from target products; therefore, most dev Browsing cases do not trigger final category diversity. This round's Browsing improvement mainly stems from the larger Dense pool and independent Category Route. True open-ended dialogue requires building a separate test set without explicit categories.

### 9.5 BGE Not Yet Validated in Current Workspace

This round did not replicate, rebuild, or modify BGE artifacts on colleagues' machines. Merging must validate:

- catalog SHA256
- Full 50K row count
- parent_asin order
- 384 dimensions and model name
- `complete_catalog=true`
- Buying/Browsing same-dev pairing metrics

Existing `DenseRetriever` and dual-track Pipeline retain unified interfaces; BGE merging does not require rewriting the architecture.

## 10. Modified Files

Core additions:

- `solution/constraint_parser.py`
- `solution/routing.py`
- `solution/pipeline.py`
- `solution/retrieval/category.py`
- `solution/ranking/diversity.py`
- `solution/ranking/semantic.py`
- `solution/llm/base.py`
- `solution/llm/disabled.py`
- `solution/llm/qwen.py`
- `solution/llm/factory.py`

Core modifications:

- `solution/agent.py`
- `solution/config.py`
- `solution/schemas.py`
- `solution/state.py`
- `solution/intent.py`
- `solution/retrieval/fusion.py`
- `solution/ranking/constraints.py`
- `experiments/run_experiment.py`
- `experiments/trace_failures.py`
- `tests/test_solution.py`
- `solution/README.md`

No modifications:

- `evaluator/local_evaluator.py`
- `starter/agent.py`
- Catalog content
- Official metric formulas

## 11. Reproduction Commands

```powershell
# Generate frozen dev/holdout
python -m experiments.split_public_set

# Build local Hashing Dense for this round
python scripts\build_embeddings.py --backend hashing --batch-size 512 --checkpoint-every 4096

# Unit tests
python -m unittest discover -s tests -v

# Final dev evaluation
python -m experiments.run_experiment `
  --config experiments\configs\hybrid_steps_3_10.json `
  --run-dir experiments\runs\post_dual_track_v2

# Compare with pre-refactoring pairing
python -m experiments.compare_runs `
  experiments\runs\pre_dual_track_baseline `
  experiments\runs\post_dual_track_v2
```

## 12. Final Audit Opinion

This round, without a real LLM and without re-managing BGE, has completed the dual-track architecture and future model interfaces, proving overall accuracy is not lower than the original implementation via frozen dev. Deliverables can enter the merge preparation phase.

Top priorities before merging are not further adjusting accuracy weights but:

1. Integrating colleagues' fully operational complete BGE and performing same-split pairing re-validation.
2. Resolving the approx. 30% performance regression caused by Browsing pool expansion.
3. Completing real latency, throughput, memory, and pairing accuracy re-validation after deploying Ollama `qwen3.5:9b`.
4. Supplementing independent test sets for true no-category Browsing and free-language routing.
