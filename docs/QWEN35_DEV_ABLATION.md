# Qwen3.5 9B Complete Dev Quantization Validation Report

> Historical Report: This document records the Hashing main chain prior to integrating Dynamic Context Programming. The current main chain results are governed by `docs/QWEN35_CONTEXT_DEV_ABLATION.md`; data from both sets must not be mixed.

## 1. Audit Conclusion

This round validates only local Ollama `qwen3.5:9b` Top-30 semantic reranking, without adjusting default retrieval weights, enabling Cross-Encoders, switching to BGE, or reading holdout, public, or sealed final hyperparameter tuning configurations.

**Conclusion: Qwen improved MRR on the frozen dev150 but did not improve HR@10; meanwhile, MTTC and runtime latency degraded. Therefore, it does not meet the default enablement threshold of "simultaneous improvement in HR and MRR with acceptable time cost." The default `semantic_ranker_enabled=false` should remain unchanged. No new final split will be generated or run in this round.**

## 2. Strict Pairing Settings

| Item | Qwen-off Baseline | Qwen-on Candidate |
|---|---|---|
| Agent | `solution.agent:Agent` | `solution.ablations:OllamaQwenAgent` |
| Data | `data/splits/dev.jsonl`, 150 samples | Same |
| Dataset SHA256 | `c8955cacd79bb2e2ed6b984979bca91f379fc699648c5bc2e3ac2efb37f8b22a` | Same |
| Catalog SHA256 | `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67` | Same |
| Git commit | `cb15f6dbc3339ee8af6a98d7b44ca6b795d84dd3` | Same |
| Dense | Hashing, 50K catalog | Same |
| Cross-Encoder | Disabled | Disabled |
| Qwen | Disabled | Ollama `qwen3.5:9b`, Top-30 |
| Temperature / seed / thinking | N/A | `0` / `0` / `false` |
| Context / Max Output | N/A | 8192 / 1024 tokens |
| Official evaluator | Unmodified | Unmodified |

The local pre-check environment is Ollama 0.33.2, NVIDIA GeForce RTX 5070 Ti Laptop GPU 12 GB. Real-world smoke tests verified that Buying/Browsing scenarios invoke Qwen, Over-General scenarios skip Qwen, and all recommended IDs originate from the read-only catalog.

## 3. Overall Metrics

| Metric | Qwen-off | Qwen-on | Change | Verdict |
|---|---:|---:|---:|---|
| HR@10 | 0.913333 | 0.913333 | 0.000000 | Flat, no improvement |
| MRR | 0.597302 | 0.611868 | +0.014566 (~+2.44%) | Improved |
| MTTC ↓ | 4.406667 | 4.420000 | +0.013333 | Slight degradation |
| Efficiency | 0.659333 | 0.658000 | -0.001333 | Degraded |
| Technical Score | 0.767724 | 0.771827 | +0.004103 (~+0.53%) | Improved |

Qwen did not expand the Top-10 coverage boundary; gains stem from a few already-hit targets being pushed to earlier positions.

## 4. Scenario-Specific Metrics

| Scenario | Samples | HR@10 Change | MRR: off → on | MTTC: off → on |
|---|---:|---:|---:|---:|
| Buying | 60 | 0 | 0.599107 → 0.623413 | 3.566667 → 3.600000 |
| Browsing | 60 | 0 | 0.641653 → 0.642487 | 4.816667 → 4.816667 |
| Intent Override | 22 | 0 | 0.462428 → 0.493182 | 5.500000 → 5.500000 |
| Boundary | 8 | 0 | 0.622024 → 0.622024 | 4.625000 → 4.625000 |

Major MRR gains come from Buying (+0.024306) and Intent Override (+0.030754). Boundary remains completely unchanged, while Browsing improves only by 0.000834.

## 5. Per-Session Pairing Results

| Result | Sessions |
|---|---:|
| Reciprocal Rank Improved | 8 |
| Reciprocal Rank Degraded | 2 |
| Ranking Unchanged | 140 |
| New Top-10 Hits | 0 |
| Lost Top-10 Hits | 0 |
| Earlier Hit | 1 |
| Later Hit | 2 |
| Hit Turn Unchanged | 147 |

Maximum improvements include:

- `public_0183` (Intent Override): rank 4 → 1;
- `public_0097` (Buying): rank 3 → 1, but hit turn 3 → 4;
- `public_0101` (Buying): rank 3 → 1, but hit turn 1 → 3.

Explicit degradations include:

- `public_0130` (Intent Override): rank 3 → 4;
- `public_0023` (Intent Override): rank 8 → 9.

This indicates that Qwen's ranking gains are highly concentrated: only 10 out of 150 sessions experienced reciprocal rank changes, and the two largest ranking gains were accompanied by later hits, explaining the combination of rising MRR and slight MTTC degradation.

## 6. Call Success Rate, Tokens, and Latency

| Item | Result |
|---|---|
| Actual Sessions | 150 |
| Actual Eval Turns / Qwen Requests | 650 |
| Successfully Applied Qwen Ranking | 554 |
| Safe Fallback | 96 |
| Success Rate | 85.23% |
| Fallback Rate | 14.77% |
| Prompt tokens | 1,763,233 |
| Completion tokens | 215,302 |
| Total reported tokens | 1,978,535 |
| Avg prompt tokens per successful call | ~3,182.7 |
| Avg completion tokens per successful call | ~388.6 |
| Qwen-off Total Duration | 202.41 seconds |
| Qwen-on Total Duration | 4,612.09 seconds (~76 min 52 sec) |
| Total Duration Multiplier | ~22.79× |
| Relative Baseline Incremental Duration | 4,409.69 seconds |
| Estimated Incremental Duration per Request | ~6.78 seconds |

Token statistics include only successfully returned and applied results; safe fallback requests lack reliable server-side token records, so the above total token count cannot be interpreted as the actual consumption of all 650 requests.

The current adapter only stores cumulative request counts, cumulative success counts, and the last error; it does not store error categories or latency sequences for each call. Therefore, this round can precisely confirm 96 safe fallbacks but cannot further decompose them into timeouts, HTTP errors, JSON errors, or candidate permutation errors without re-running the full workload, nor can strict P50/P95 be provided. This observation gap does not affect the official HR/MRR/MTTC results but reduces fault diagnosis capability.

## 7. Threshold Audit

| Enable Condition | This Round Result | Passed? |
|---|---|---|
| HR@10 Improvement | 0.913333 → 0.913333, flat only | No |
| MRR Improvement | +0.014566 | Yes |
| MTTC Not Degraded | +0.013333, slight degradation | No |
| Latency Acceptable | Total duration ~22.79×, estimated ~6.78 sec/request increase | No |
| Call Stability | Success rate 85.23%, fallback rate 14.77% | No |

Final Verdict: **Does not pass default enablement threshold.**

This round does not read or run holdout/final, nor modify default switches. If Qwen research continues in the future, it should first increase per-request error category and P50/P95 observations, then experiment with smaller candidate windows, conditional triggering, or calling only for low-confidence sessions on dev; these belong to subsequent work and are out of scope for this round.

## 8. Raw Artifacts

- Qwen-off: `experiments/runs/step12_hashing_dev/`
- Qwen-on: `experiments/runs/qwen_hashing_dev_full/`
- Qwen-on Raw Results SHA256: `3cae42a31bf5a3dd895183b651d1f247c88f1b02ce4492a5995eeefd56cb4a67`
- Machine-readable Summary: `experiments/analysis/qwen_hashing_dev_ablation.json`
