# Ollama Qwen3.5 9B Local Semantic Ranking Integration Guide

## Current Status

The code has been integrated with the official Ollama local Chat API, using the model tag `qwen3.5:9b`. To ensure baseline stability before the model is deployed, `semantic_ranker_enabled` defaults to `false`.

The normal retrieval path passes Top-N candidates to Qwen3.5 after keyword, category, vector recall, and deterministic ranking. Active clarification truncation for overly broad queries will still skip Dense and LLM; the full path resumes once the user supplements slots.

Official references:

- [Ollama qwen3.5:9b model](https://ollama.com/library/qwen3.5:9b)
- [Ollama Chat API](https://docs.ollama.com/api/chat)
- [Ollama Structured Outputs](https://docs.ollama.com/capabilities/structured-outputs)

## Deployment and Enablement

```powershell
ollama pull qwen3.5:9b
ollama run qwen3.5:9b
```

Ollama defaults to listening on `http://127.0.0.1:11434`. To explicitly enable it in the code:

```python
from solution.agent import Agent
from solution.config import SolutionConfig

config = SolutionConfig(semantic_ranker_enabled=True)
agent = Agent(config=config, diagnostics=True)
```

If a different service address is used, configure accordingly:

```python
config = SolutionConfig(
    semantic_ranker_enabled=True,
    semantic_ranker_base_url="http://127.0.0.1:11434",
)
```

## Default Parameters

| Parameter | Default Value | Description |
|---|---:|---|
| `semantic_ranker_backend` | `ollama` | Local Ollama backend |
| `semantic_ranker_model` | `qwen3.5:9b` | Official 9B model tag |
| `semantic_ranker_top_n` | `30` | Number of candidates passed to the model for re-ranking |
| `semantic_ranker_timeout_ms` | `120000` | Timeout including the initial model load request |
| `semantic_ranker_keep_alive` | `10m` | Ollama model residency duration |
| `semantic_ranker_temperature` | `0` | Reduces sorting randomness |
| `semantic_ranker_num_ctx` | `8192` | Current ranking request context window |
| `semantic_ranker_num_predict` | `1024` | Maximum output token budget |

Requests fixedly use `stream:false`, `think:false`, and JSON Schema. The schema requires returning each input `parent_asin` exactly once.

## Security and Degradation

- Only sends a limited catalog summary of the current Top-N products, not the full directory.
- Catalog text is treated as untrusted input; system prompts prohibit executing instructions contained within product text.
- The model can only re-rank candidates; it cannot add, delete, duplicate, or modify product IDs.
- Failures due to connection issues, timeouts, HTTP errors, responses exceeding 1 MiB, invalid JSON, unknown IDs, missing IDs, or duplicate IDs all fallback to the original deterministic ranking.
- Agent initialization does not probe or pull models; therefore, default configurations are unaffected when Ollama is offline.

## Post-Merge Real Model Acceptance

1. Confirm `qwen3.5:9b` exists in `ollama list`.
2. Use a Buying and a Browsing session to confirm diagnostics show `backend=ollama` and `applied=true`.
3. Check `prompt_tokens`, `completion_tokens`, `last_latency_ms`, and `last_error`.
4. Run LLM-off and LLM-on versions separately on frozen development sets, comparing HR@10, MRR, MTTC, and total latency.
5. Verify that the first round for overly broad queries remains `over_generality_cutoff`, with Ollama invoked only in the subsequent round after the user supplements slots.

Current automated tests use simulated Ollama responses and do not require downloading the model; real model quality and performance must be re-validated on the deployment machine.

## Local Real Model Smoke Test

The repository provides a dedicated test entry point. It first checks for the local model tag, then runs a Buying, a Browsing, and an Over-General session: the first two must genuinely invoke Qwen, while the third must skip LLM at the active clarification truncation point. All recommended IDs are re-validated as members of the current read-only catalog, and results can be saved as JSON.

```powershell
python scripts/test_ollama_integration.py `
  --catalog data/catalog.jsonl `
  --output experiments/runs/ollama_qwen35_live_smoke.json
```

If the catalog is located in another local directory, pass the absolute path via `--catalog`. This test covers connectivity, structured output, security boundaries, fallback paths, and latency logging; it does not replace accuracy paired evaluations on frozen dev.

The entry point for complete paired experiments on frozen dev is:

```powershell
python -m experiments.run_experiment `
  --config experiments/configs/qwen_hashing_dev.json `
  --name qwen_hashing_dev
```

This experiment must be compared against the LLM-off version using the same commit, same catalog, and same dev split; it must not read holdout tuning weights.

## Local Acceptance Results on 2026-08-31

- Environment: Ollama `0.33.2`, model `qwen3.5:9b`, RTX 5070 Ti Laptop 12 GB; Ollama reports the model occupies ~5.7 GB with 100% GPU usage, while retrieval uses Hashing Dense on a 50K catalog.
- Buying: `applied=true`, Prompt 2815 tokens, Completion 387 tokens, model invocation ~6.67 seconds.
- Browsing: `applied=true`, Prompt 2990 tokens, Completion 385 tokens, model invocation ~6.27 seconds.
- Over-General: `retrieval_cutoff=true`, Ollama request increment is 0, usage is 0, due to `over_generality_cutoff`.
- Total duration for the three sessions was approximately 46.25 seconds; recommendations were all 10 non-duplicate catalog IDs; no new, unknown, or duplicate ASINs were found.

The above constitutes a smoke acceptance of connectivity and security boundaries, not accuracy conclusions. Default weights and `semantic_ranker_enabled=false` remain unchanged; holdout was not read, and the final split was not executed.

## Final dev150 Results After Merging Dynamic Context Programming

The primary chain has been re-run with strict pairing and cannot be mixed with the earlier Hashing report above:

- Context-only: HR@10 0.920000, MRR 0.607254, MTTC 4.360000, 196.13 seconds;
- Context + Qwen: HR@10 0.920000, MRR 0.620931, MTTC 4.380000, 4,800.70 seconds;
- Of 623 requests, 546 succeeded, and 77 resulted in `invalid_candidate_permutation` safe fallbacks;
- Success rate 87.64%, fallback rate 12.36%; average 7.40 seconds, P50 7.77 seconds, P95 8.28 seconds;
- HR showed no improvement, MTTC slightly degraded, latency increased by approximately 24.48 times; the default enablement threshold was not met;
- `semantic_ranker_enabled=false` remains unchanged; holdout/final were not run.

The current report is available at `docs/QWEN35_CONTEXT_DEV_ABLATION.md`. The smoke tool now continues to check remaining scenarios and saves JSON even if a single scenario fails; it still reports failure explicitly with a non-zero exit code.
