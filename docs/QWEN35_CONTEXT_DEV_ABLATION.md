# Current Context Programming Main Chain Qwen3.5 dev150 Validation

## Conclusion

Qwen3.5 9B functionality, live connectivity, full dev quantization, error distribution, tokens, average/P50/P95 latency, safety fallback, and session circuit breakers have all been completed.

**Release threshold not met: Keep `semantic_ranker_enabled=false` and do not run final.**

## Pairwise Metrics

| Metric | Context-only | Context + Qwen | Change |
|---|---:|---:|---:|
| HR@10 | 0.920000 | 0.920000 | 0 |
| MRR | 0.607254 | 0.620931 | +0.013677 |
| MTTC ↓ | 4.360000 | 4.380000 | Degradation 0.020000 |
| Efficiency | 0.664000 | 0.662000 | -0.002000 |
| Technical Score | 0.774976 | 0.778679 | +0.003703 |
| Total Duration | 196.13 seconds | 4,800.70 seconds | ~24.48× |

By scenario: Buying MRR +0.022917, MTTC degradation 0.05; Intent Override MRR +0.030754; Browsing and Boundary remain unchanged.

Per session: 6 reciprocal rank improvements, 2 degradations, 142 unchanged; no new or lost Top-10 hits; 2 hits became slower.

## Model Invocation Audit

| Item | Result |
|---|---:|
| Qwen Requests | 623 |
| Successfully Applied | 546 |
| Safety Fallback | 77 |
| Success Rate | 87.64% |
| Fallback Rate | 12.36% |
| Errors | 77 instances of `invalid_candidate_permutation` |
| Average Latency | 7.40 seconds |
| P50 | 7.77 seconds |
| P95 | 8.28 seconds |
| Prompt Tokens | 1,737,271 |
| Completion Tokens | 212,169 |
| Total Tokens | 1,949,440 |

Diagnosis confirms that Ollama may repeat a valid ID while omitting another while satisfying the length constraint, without actually enforcing JSON Schema's `uniqueItems`. The system does not accept or automatically patch such output but maintains deterministic ordering. Circuit breaking occurs only within the current session after two consecutive failures; new sessions are unaffected.

## Integrity

- Agent: `solution.ablations:OllamaQwenAgent`;
- Model: Local Ollama `qwen3.5:9b`; Top-30; temperature 0; seed 0; `think:false`;
- Dataset SHA256: `c8955cacd79bb2e2ed6b984979bca91f379fc699648c5bc2e3ac2efb37f8b22a`;
- Catalog SHA256: `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67`;
- Qwen-on results SHA256: `95463f74d33cfdcebf86f36fd68cc6911d8e6e9e33f05e539c6bdd3227d9c0c5`;
- Official evaluator/starter not modified;
- Holdout, public, or sealed final runs not executed;
- Machine-readable report: `experiments/analysis/qwen_context_dev_ablation.json`.

Threshold determination: HR did not improve, MTTC degraded, and latency is unacceptable; therefore, "functionality complete" does not equal "enabled by default."
