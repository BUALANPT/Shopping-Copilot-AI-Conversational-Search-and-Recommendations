# 50K BGE GPU Current Results and LLM Integration Plan

## Achievements Saved in This Round

- **GPU**: NVIDIA GeForce RTX 5070 series, 12 GB VRAM.
- **Runtime**: ONNX Runtime GPU 1.29.0, CUDAExecutionProvider, CUDA 13/cuDNN 9.
- **Model**: `BAAI/bge-small-en-v1.5`.
- **Index**: 50,000 rows × 384 dimensions, `float16`, approximately 38.4 MB.
- **catalog SHA256**: `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67`.
- **BGE matrix SHA256**: `7f3bd95ecb598191969893c0682305346cb8776dcd012416efaa367aaee7f49a`.
- **metadata SHA256**: `42bcc26659cf3b2fa8d96a9072acb75ce1ef717fb21ffbafd905c36f88723686`.
- **Metadata Verification**: `complete_catalog=true`, `build_provider=CUDAExecutionProvider`.

## Strict Ablation with dev Split

Weights, candidate count, RRF, reranker, dialogue state, and clarification strategies are all frozen. The only variable is the Dense backend.

| Scheme | HR@10 | MRR | MTTC | Tech Score | Time/Sec |
|---|---:|---:|---:|---:|---:|
| hashing paired dev | 0.906667 | 0.593405 | 4.453333 | 0.762288 | 139.352 |
| 50K BGE GPU dev | **0.920000** | **0.616587** | **4.353333** | **0.777909** | **137.163** |

BGE improves HR, MRR, MTTC, and the technical score simultaneously; paired latency does not increase, thus passing the dev promotion threshold.

## Sealed Final State

A 200-item catalog-synthetic sealed final with zero overlap to public target ASINs has been generated:

- **Path**: `data/splits/final_bge_200.jsonl`
- **SHA256**: `993bed4bd40ce400a6158094d81cb14ea57561a6c4ba5f1aa56dac9115524d8c`
- **Scenario**: 80 buying, 80 browsing, 30 intent override, 10 boundary
- **public target overlap**: 0
- **Intent card**: Not pre-generated

Per the user's request to "end as soon as possible," this final round stopped before completion; `results.json` was not generated. Consequently, there are no valid final metrics, and no hyperparameter tuning was performed based on them. This data remains sealed and can serve as the sole confirmation task for the next round. It represents additional synthetic generalization evidence and cannot replace the official private 800.

## Next Task Sequence

1. Complete the single BGE run for the sealed final when resources permit; seal results upon completion without further weight tuning.
2. Migrate vector search from NumPy 50K brute-force multiplication to FAISS GPU/CPU IndexFlatIP, requiring Top-120 consistency with current exact retrieval per query.
3. Add a checkpoint to the evaluator that writes progress only and does not expose labels, preventing full recalculation after long final interruptions.
4. Do not submit BGE artifacts to Git; publish SHA256-verified build products via GitHub Release or object storage.
5. When integrating LLMs, initially adopt shadow mode to compare structured intent parsing without directly altering recommendation lists.

## Specific Plan for LLM Integration

### 1. Interface Layer

Add the following files:

- `solution/llm/base.py`: Defines the `IntentExtractor` protocol.
- `solution/llm/openai_compatible.py`: OpenAI-compatible HTTP client.
- `solution/llm/schemas.py`: Strict JSON Schema and server-side validation.
- `solution/llm/fallback.py`: Wraps the current rule-based parser; immediately falls back on any failure.

Environment variables are read only at runtime:

```text
TECHJAM_LLM_ENABLED=0
TECHJAM_LLM_BASE_URL=https://<provider>/v1
TECHJAM_LLM_API_KEY=<secret>
TECHJAM_LLM_MODEL=<model-name>
TECHJAM_LLM_TIMEOUT_MS=1500
TECHJAM_LLM_MAX_TOKENS=300
```

API keys must not be written to configuration, logs, result files, or Git.

### 2. Model Outputs Structured Intent Only

Phase one allows the model to output only:

```json
{
  "intent": "buying|browsing|override",
  "buying_probability": 0.0,
  "category": "string|null",
  "constraints": ["string"],
  "exclusions": ["string"],
  "override": false,
  "no_preference_attribute": "string|null"
}
```

The model must not receive ground truth, the evaluator hidden intent card, or the full catalog, nor can it directly generate ASINs. Final recommended IDs must still come exclusively from BM25/metadata/BGE candidates and the catalog dictionary.

### 3. Safety and Determinism

- `temperature=0`, fixed system prompts, and JSON Schema versions.
- Requests include only current messages, confirmed constraints, and de-identified aggregated profiles.
- Fall back to the current rule parser on timeout, rate limiting, invalid JSON, schema validation failure, or network errors.
- Cache results based on normalized input and state hash; cache records must include model, prompt version, and response SHA256.
- Trigger a circuit breaker on consecutive failures; subsequent calls in this session use the rule parser exclusively.
- Logs must not record API keys, full user profiles, or raw free-text inputs.

### 4. Shadow Mode

Default `TECHJAM_LLM_ENABLED=0`. Initial experiments set to shadow mode:

- Rule parser continues to drive formal recommendations.
- LLM parsing results are written only to independent diagnostic logs.
- Compare differences in category, constraints, override, and ask_attribute.
- Do not read holdout data; perform failure audits only on dev.

### 5. Promotion Thresholds

LLMs may enter active ablation only if they simultaneously meet the following conditions:

- dev HR@10 ≥ 0.920000;
- dev MRR ≥ 0.616587;
- MTTC degradation ≤ 0.10 rounds;
- P95 LLM latency ≤ 1.5 seconds;
- Maximum 3 calls per session;
- Fallback success rate 100%, zero invalid ASINs;
- Record token and cost limits every 200 sessions.

### 6. Recommended Model Sequence

1. First, connect to an OpenAI-compatible API model supporting JSON Schema to validate interfaces, fallback mechanisms, and costs.
2. Next, test local 7B/8B instruct models (Ollama or vLLM), maintaining the same schema and test set.
3. Compare intent parsing capabilities only; product knowledge remains provided by the 50K catalog+BGE.
4. If active dev LLM does not simultaneously improve HR/MRR, retain shadow mode and do not proceed to the final Agent.

## Reproduction Commands

```powershell
# GPU dependencies
.\.venv\Scripts\python -m pip install -r requirements-gpu.txt

# Full BGE; append --resume after interruption
.\.venv\Scripts\python scripts\build_embeddings.py `
  --backend fastembed --provider cuda --batch-size 512 --checkpoint-every 2048

# Same dev split BGE ablation
.\.venv\Scripts\python -m experiments.run_experiment `
  --config experiments\configs\bge_dense_dev.json

# Next round's sole sealed final
.\.venv\Scripts\python -m experiments.run_experiment `
  --config experiments\configs\bge_dense_dev.json `
  --dataset data\splits\final_bge_200.jsonl --final-eval
```
