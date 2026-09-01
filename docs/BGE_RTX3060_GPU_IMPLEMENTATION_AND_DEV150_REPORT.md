# RTX 3060 Full BGE GPU Implementation and dev 150 Pairwise Evaluation Report

## 1. Conclusion

The local machine has completed the full vector construction for `BAAI/bge-small-en-v1.5` with **50,000 products** in the **E: drive project directory**, and performed pairwise evaluation on frozen `dev 150` via a strict CUDA BGE Agent.

- Full index is valid: `50000 × 384`, `float16`, `complete_catalog=true`.
- Both construction and query can utilize `CUDAExecutionProvider`; during evaluation, Python GPU compute processes were observed with approximately 27% GPU utilization and a total VRAM usage of about 1.9 GiB.
- Compared to the current Hashing baseline, BGE achieves parity in overall HR@10, while showing net improvements in MRR, MTTC, Efficiency, and recommendation technical score, without falling below existing overall accuracy.
- BGE does not improve unilaterally in every scenario: Intent Override shows significant improvement, while Browsing HR@10 drops by one sample. Current evidence supports retaining BGE but does not support hard-coding rules for the two samples.
- Exact vector search has been optimized for safety and performance: `float16` is still persisted to disk, but converted once to in-memory `float32` BLAS matrices upon startup; matrix dot product micro-benchmarks for 40 queries show an improvement of approximately 12.31×, with Top-120 ordering remaining completely identical.
- No LLM was truly invoked in this round; token usage in the report is 0; Ollama/Qwen3.5 9B interfaces remain disabled by default, complying with the current constraint that "LLMs are temporarily not needed."

## 2. Dataset Identity and Round Boundaries

### 2.1 Was `public 200` Self-Constructed?

No. The repository's `data/public_set.jsonl` is the original official public development set, containing 200 sessions. Its frozen record is:

- Sample count: 200
- Role: `final_only`
- Current file SHA-256: `571359a8a69014c43fc30d39c996c4a28e875dccc249dffc707358757beb16c0`

A previous workspace generated a different file byte hash due to LF/CRLF line ending differences, but the JSONL semantics and Git content remained consistent. This round has restored the repository's expected line ending format and passed frozen data validation.

### 2.2 Source of `dev 150`

`data/splits/dev.jsonl` is not another official independent test set; rather, it is a frozen tuning subset deterministically sliced from the official `public 200` using a fixed seed `techjam-2026-v1`, stratified by scenario:

- Sample count: 150
- SHA-256: `c8955cacd79bb2e2ed6b984979bca91f379fc699648c5bc2e3ac2efb37f8b22a`
- Buying: 60
- Browsing: 60
- Intent Override: 22
- Boundary: 8
- Role: `tuning`

The remaining 50 samples serve as frozen holdouts and were not read or evaluated in this round. Upon user confirmation to switch to dev 150, the ongoing public 200 run was stopped, its empty incomplete run directory was deleted, and only complete dev 150 metrics were adopted thereafter.

### 2.3 Checkpoints and Partial Data

- The 512/2,048 product smoke indexes were not treated as formal results; these temporary artifacts have been deleted.
- The formal BGE index was constructed from scratch for all 50,000 rows, without resuming from a partial index checkpoint.
- `StrictBgeAgent` rejects artifacts where `complete_catalog=false`, product counts do not match, catalog SHA mismatches, model mismatch occurs, or CUDA is unavailable.
- The evaluator writes and adopts summaries only after all 150 samples are completed; interrupted runs are not treated as metrics.

## 3. Local Environment

| Item | Value |
|---|---|
| GPU | NVIDIA GeForce RTX 3060 Laptop GPU |
| VRAM | 6,144 MiB |
| NVIDIA Driver | 610.47 |
| Python | 3.13.3 |
| FastEmbed | 0.7.3 |
| ONNX Runtime GPU | 1.29.0 |
| NumPy | 2.5.2 |
| ORT device | GPU |
| Available Providers | TensorRT, CUDA, CPU |
| Project/Virtual Environment/Cache/Index | All located in the E: drive project directory |

Under Windows, FastEmbed installs the CPU `onnxruntime` by dependency name; it shares the same Python module path with `onnxruntime-gpu`, which can lead to the CPU wheel overwriting the GPU wheel. This round introduces a new installation script that uninstalls both ORT distribution packages after installing dependencies, then installs the GPU wheel last using `--no-deps --force-reinstall`, and asserts the existence of the CUDA provider.

The warning "some nodes not assigned to preferred provider" appearing during model inference is a standard ONNX Runtime hint placing shape-dependent nodes on CPU; it does not imply the entire model has regressed to CPU. GPU process activity and utilization sampling during evaluation further confirm that CUDA is actively working.

## 4. Full Index Results

| Attribute | Result |
|---|---|
| Model | `BAAI/bge-small-en-v1.5` |
| Rows | 50,000 |
| Dimensions | 384 |
| Disk dtype | `float16` |
| NPY Size | 38,400,128 bytes |
| Model Cache Size | 67,181,178 bytes |
| Build Provider | `CUDAExecutionProvider` |
| Build Duration | ~55.66 seconds |
| catalog SHA-256 | `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67` |
| matrix SHA-256 | `31137495bb7a3d3eddaef04af361556ec848fd39de167a8590968a654e871243` |
| metadata SHA-256 | `42bcc26659cf3b2fa8d96a9072acb75ce1ef717fb21ffbafd905c36f88723686` |

The matrix SHA in the historical report from a colleague's RTX 5070 machine is `7f3bd95e...`, which differs from this local machine's matrix SHA. However, metadata SHA, catalog SHA, shape, dtype, row order, and integrity are consistent. The most likely cause is low-bit numerical differences arising from different GPU/CUDA graph executions; equality of matrix byte hashes across GPUs cannot be the sole condition for correctness. When merging, a single standard artifact should still be explicitly selected and published, along with its SHA.

## 5. Code Changes

### 5.1 E: Drive GPU Environment

- Added `scripts/setup_gpu_environment.ps1`: Creates the `.venv` in the E: drive, moves pip cache and temporary directories to the E: drive, corrects CPU/GPU ONNX wheel overwrite issues, and asserts CUDA provider existence.
- Updated `requirements-gpu.txt`: Records FastEmbed/ORT Windows conflicts and the correct installation entry point.
- Updated `.gitignore`: Ignores rebuildable pip caches and temporary directories within the E: drive project.

### 5.2 Full Construction and Offline Loading

- `scripts/build_embeddings.py` added `--cache-dir`; model cache defaults to `artifacts/fastembed_cache`.
- Added `--local-files-only`; after the first download, subsequent constructions no longer access the network.
- `SolutionConfig` added `dense_cache_dir`.
- `Agent` parses project-relative cache directories and passes them to `DenseRetriever`.
- `DenseRetriever` enforces `local_files_only=True` at runtime to avoid online requests and cache drift.

### 5.3 Exact Search Performance Optimization

The original implementation directly executed:

```python
scores = float16_memmap @ float32_query
```

NumPy's CPU `float16` matrix-vector path is noticeably slower on this local machine. The current implementation retains the compact `float16` disk file, then establishes a `float32` in-memory search matrix once integrity verification is complete before executing exact inner products.

Fixed random seed micro-benchmark for 50,000×384 with 40 queries:

| Metric | Original Path | Optimized Path |
|---|---:|---:|
| 40 Matrix Dot Products | 1.9913 seconds | 0.1617 seconds |
| One-time Conversion Cost | - | 0.0418 seconds |
| Dot Product Speedup | - | 12.31× |
| 40 Groups Top-120 Intersection | - | All 120/120 |
| 40 Groups Top-120 Ordered Results | - | All Completely Identical |
| Max Score Difference | - | 0.0 |

End-to-end Hashing dev 150 duration dropped from 306.818 seconds (with identical metrics in previous runs) to 295.450 seconds, a reduction of approximately 3.71%. The end-to-end gain is smaller than the dot product micro-benchmark, indicating that BM25, dialogue state, constraint handling, and reranking still account for the majority of runtime.

## 6. dev 150 Strict Pairwise Results

Two runs used the same catalog, same dev SHA, same weights, same candidate count, same state machine, same clarification strategy, and disabled LLM; the only core variable was the Dense backend.

| Scheme | HR@10 | MRR | MTTC ↓ | Efficiency | Tech Score | Duration |
|---|---:|---:|---:|---:|---:|---:|
| Current Hashing | 0.913333 | 0.597302 | 4.406667 | 0.659333 | 0.767724 | 295.450s |
| Full BGE/CUDA | 0.913333 | 0.603272 | 4.373333 | 0.662667 | 0.770181 | 332.911s |
| BGE Delta | 0.000000 | +0.005970 | +0.033334 Improvement | +0.003334 | +0.002457 | +37.461s |

Conclusion: Overall hit rate did not decrease; MRR and technical score improved; average hit rounds slightly advanced; the cost is an increase in full evaluation duration of approximately 12.68%.

### 6.1 Per-Scenario Results

| Scenario | Scheme | HR@10 | MRR | MTTC ↓ |
|---|---|---:|---:|---:|
| Boundary | Hashing | 0.875000 | 0.622024 | 4.625000 |
| Boundary | BGE | 0.875000 | 0.684524 | 4.625000 |
| Browsing | Hashing | 0.950000 | 0.641653 | 4.816667 |
| Browsing | BGE | 0.933333 | 0.649372 | 4.850000 |
| Buying | Hashing | 0.916667 | 0.599107 | 3.566667 |
| Buying | BGE | 0.916667 | 0.588724 | 3.533333 |
| Intent Override | Hashing | 0.818182 | 0.462428 | 5.500000 |
| Intent Override | BGE | 0.863636 | 0.487680 | 5.272727 |

Key Differences:

- BGE rescues `public_0198` in Intent Override, improving HR by 4.55 percentage points in this scenario.
- BGE loses `public_0076` in Browsing, dropping HR by 1.67 percentage points in this scenario.
- Buying hit count is flat; MTTC is slightly better, but MRR is slightly lower.
- Boundary hit count and MTTC remain unchanged; MRR shows marked improvement; this group contains only 8 samples, so interpretation should be cautious.
- First-hit rounds or best ranks changed for 35/150 sessions; BGE's impact does not stem solely from the two flipped hit samples.

## 7. Testing and Evidence

- Frozen data validation: Passed; sample counts and hashes for dev/public/holdout match `experiments/frozen_datasets.json`.
- Unit tests: 36/36 passed.
- New regression test verification: Disk `float16` indexes are promoted to in-memory `float32` search matrices with unchanged shape.
- Overall metrics before and after Hashing optimization are item-by-item completely identical.
- BGE strict run status: `dense.enabled=true`, `dense.backend=fastembed`.
- LLM status: disabled; prompt/completion tokens are both 0.
- Pairwise difference file: `experiments/runs/gpu_round_bge_dev/comparison_vs_hashing.json`.

The experiment run directory belongs to rebuildable artifacts and is ignored by `.gitignore`, but it is retained in the current E: drive workspace for local review.

## 8. Current Issues and Risks

1. **BGE did not win in every scenario.** Overall technical score shows net improvement, but Browsing HR and Buying MRR show regression. Future validation should occur on independent holdout/final sets; tuning should not continue based on two flipped samples in dev.
2. **BGE query encoding has end-to-end cost.** Current BGE dev 150 is approximately 37.46 seconds slower than Hashing. Exact matrix search has been optimized; the next hotspot should be confirmed via profiler, and one cannot assume FAISS must be introduced directly.
3. **Cross-GPU artifact byte inconsistency.** Metadata and structure are consistent, but matrix SHA differs; when publishing/merging, a single standard build artifact and SHA must be selected, or an allowed numerical tolerance validation process must be defined.
4. **ORT package metadata conflicts.** To ensure the CUDA module takes effect finally, the GPU wheel is installed last with `--no-deps`; `pip check` may still report FastEmbed's declared CPU `onnxruntime` distribution as missing. Runtime CUDA provider assertion is more reliable than package name checks, but team documentation must retain this explanation.
5. **Limited 6 GiB GPU headroom.** Current BGE-small works normally; when Qwen3.5 9B resides simultaneously in the future, it likely cannot fit entirely on this GPU, requiring Ollama layered unloading, quantization, CPU offload, or independent device planning.
6. **LLM has not yet entered this round's accuracy.** Existing Ollama interfaces and safety fallbacks are ready, but Qwen3.5 9B is not downloaded or running; one cannot claim to have verified the final "BGE + LLM" effect.
7. **Official public 200 was not completed in this round.** This was actively stopped per user request to avoid tuning on the final-only set; this report represents dev 150 tuning evidence only, not final official scores.

## 9. Recommended Follow-up Order

1. Unify BGE artifact sources when merging; verify catalog SHA, metadata SHA, row count, and provider; do not submit `.npy` to Git; publish via Release/Object Storage instead.
2. Freeze current code and weights; run BGE validation only once on independent holdout or prescribed final sets; after review, do not return to dev weight tuning.
3. Use profiler to split BM25, BGE query encode, matrix dot product, constraints, and reranker runtime, then decide whether to add FAISS IndexFlatIP. If added, verify Top-120 per query against the current exact implementation.
4. After deploying Ollama/Qwen3.5 9B, perform shadow evaluation first; enable `semantic_ranker_enabled` only when HR/MRR do not drop on the same frozen set and timeouts/format errors allow safe fallback.
5. Conduct semantic failure analysis for `public_0076` and `public_0198`, but extract only generalizable mechanisms, avoiding hard-coding sample IDs, target products, or answer features.

## 10. Reproduction Commands

First installation and first model download:

```powershell
.\scripts\setup_gpu_environment.ps1
.\.venv\Scripts\python.exe scripts\build_embeddings.py `
  --backend fastembed `
  --provider cuda `
  --batch-size 512 `
  --checkpoint-every 50000 `
  --cache-dir artifacts\fastembed_cache
```

Offline full reconstruction after cache is ready:

```powershell
.\.venv\Scripts\python.exe scripts\build_embeddings.py `
  --backend fastembed `
  --provider cuda `
  --batch-size 512 `
  --checkpoint-every 50000 `
  --cache-dir artifacts\fastembed_cache `
  --local-files-only
```

Frozen validation and pairwise evaluation:

```powershell
.\.venv\Scripts\python.exe -m experiments.verify_frozen_data

.\.venv\Scripts\python.exe -m experiments.run_experiment `
  --config experiments\configs\hybrid_steps_3_10.json `
  --name gpu_round_hashing_dev `
  --dataset data\splits\dev.jsonl `
  --run-dir experiments\runs\gpu_round_hashing_dev

.\.venv\Scripts\python.exe -m experiments.run_experiment `
  --config experiments\configs\bge_dense_dev.json `
  --name gpu_round_bge_dev `
  --dataset data\splits\dev.jsonl `
  --run-dir experiments\runs\gpu_round_bge_dev

.\.venv\Scripts\python.exe -m experiments.compare_runs `
  experiments\runs\gpu_round_hashing_dev `
  experiments\runs\gpu_round_bge_dev
```
