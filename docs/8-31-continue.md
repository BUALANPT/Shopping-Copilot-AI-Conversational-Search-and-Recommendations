# 8-31 Summary of Follow-up Work and Implementation Plan

> Historical Plan Note: Sections 4–7 of this document record the gaps and design prior to implementation. Personalized Context Distillation, Memory Profile Store, Adaptive Context Program, Strategy Outcome closed-loop, and the current main chain Qwen full pairing are now implemented; the latest status is governed by `docs/DYNAMIC_CONTEXT_PROGRAMMING.md` and `docs/COMPETITION_REQUIREMENTS_STATUS.md`. The current testing baseline is 49/49.

## 1. Document Purpose

This document synthesizes the following three phase reports and the results of the third-phase Self-Evolution code audit to uniformly explain the project's current status, remaining gaps, subsequent implementation order, testing thresholds, and items that should not be executed at this time.

- [Core Architecture: Intent Routing & Hybrid Pipeline Refactoring Audit](CORE_ARCHITECTURE_DUAL_TRACK_REVIEW.md)
- [Dialog Strategy: Multi-Turn Scenario Evolution Refactoring Audit & Handover](DIALOG_STRATEGY_MULTI_TURN_REVIEW.md)
- [RTX 3060 Full BGE GPU Implementation and Dev 150 Pairing Evaluation Report](BGE_RTX3060_GPU_IMPLEMENTATION_AND_DEV150_REPORT.md)

The three reports have a chronological relationship:

- The "BGE not yet verified" in the Core Architecture report has been updated by the latest BGE report.
- The "Qwen/BGE pending merge" in the Dialog Strategy report currently only includes Qwen; BGE has completed full construction and pairing evaluation on this machine.
- The current full testing baseline is 36/36, not the earlier reports' 25/25 or 35/35.
- The latest performance and accuracy baselines are governed by the BGE RTX 3060 report.

## 2. Current Overall Status

| Module | Current Status | Subsequent Focus |
|---|---|---|
| I. Intent Routing & Hybrid Pipeline | Mainly Complete | Integrate BGE from experimental Agent into formal deployment configuration; Qwen not yet tested |
| II. Multi-Turn Scenario Evolution | Mainly Complete | Supplement real Over-General data, free language parsing, and cache lifecycle |
| III. Dynamic Context Programming | Partially Complete | Main development task for the next phase |
| Full BGE/GPU | RTX 3060 native verification complete | Unify artifacts, deployment entry points, performance, and final set validation |
| Qwen3.5 9B | Interface complete, model not enabled | Shadow first, then controlled enablement |

The current system possesses:

- Dual-track routing for Buying/Precision and Browsing/Discovery;
- Multi-path recall via Keyword, Category, Metadata, and Dense;
- High-confidence hard constraint filtering and safety relaxation;
- Cross-category diversity in Browsing;
- Multi-turn Slot accumulation;
- Three types of Intent Override: Full, Attribute, and Category;
- Explicit dialogue state phases and migration history;
- Over-Generality detection;
- Provisional Top 10 skipping Dense/LLM for special turns;
- Active clarification driven by candidate information gain;
- Ollama Qwen3.5 9B candidate semantic reranking interface and safety fallback;
- Full BGE index of 50,000 products on RTX 3060;
- Optimized precise vector search `float16` disk and `float32` memory.

The current system does not yet fully possess:

- Unified distillation from dialogue history to personalized context;
- Learning of long-term user profiles, confidence levels, revocation, and cross-session storage;
- Generation of per-turn workflow plans based on historical strategy effects;
- The `History → Distillation → Orchestration → Outcome → Strategy Revision` closed-loop;
- Real model operation of Qwen3.5 9B with accuracy, latency, and VRAM validation.

## 3. Current Baseline to Freeze

Subsequent modifications must be paired compared against results from the following split.

| Dense Backend | HR@10 | MRR | MTTC ↓ | Efficiency | Technical Score | Time |
|---|---:|---:|---:|---:|---:|---:|
| Hashing | 0.913333 | 0.597302 | 4.406667 | 0.659333 | 0.767724 | 295.450s |
| BGE/CUDA | 0.913333 | 0.603272 | 4.373333 | 0.662667 | 0.770181 | 332.911s |

BGE Current Conclusions:

- Overall HR@10 is on par with Hashing;
- MRR, MTTC, Efficiency, and Technical Score show net improvement;
- Intent Override shows significant improvement;
- Browsing HR misses one dev sample;
- Buying MRR slightly decreases;
- Do not hard-code rules for a small number of flipped samples.

### 3.1 Data Usage Boundaries

- `data/splits/dev.jsonl`: 150 items, the unique tuning set.
- `data/splits/holdout.jsonl`: 50 items, used only for final validation.
- `data/public_set.jsonl`: Official public 200, not repeatedly run during development.
- `dev 150` is a frozen tuning subset generated from the official public 200 using a fixed seed.
- Do not use interrupted, partial, or checkpoint results as accuracy conclusions.
- Do not write special logic for sample IDs like `public_0076`, `public_0198`, or target products.

## 4. Third-Phase Requirement Gaps

### 4.1 Runtime Adaptation

Existing short-term state capabilities are basically complete:

- `SessionState` saves current intent, slots, budget, exclusions, message history, slot change history, state migration history, and previous recommendations;
- `update_state()` supports incremental information, attribute override, category override, full Override, and pending clarification answer explanations per turn;
- Queries and sorting weakly use externally passed `user_profile`.

Main Gaps:

1. `message_history`, `previous_recommendations`, `slot_history`, `transition_history`, `last_routing` are mostly written to but do not participate in next-turn strategy decisions.
2. No independent, typed, length-limited `DistilledContext`.
3. No distinction between confirmed preferences, tentative preferences, negative preferences, temporary preferences, and long-term stable preferences.
4. `user_profile` is injected only at `reset()`; it does not learn, merge, decay, or revoke thereafter.
5. No Profile Store and no reliable cross-session user identification.

### 4.2 Adaptive Orchestration

Existing dynamic capabilities include:

- Buying/Browsing dual-track selection;
- Provisional/full pipeline switching during over-general scenarios;
- Candidate-driven clarification attribute selection;
- Dense/LLM special turn skipping and recovery.

Main Gaps:

1. Execution order within the Agent remains fixed.
2. `RoutingDecision` merely selects parameters from two sets of static configurations.
3. Does not replan based on recommendation repetition, user negation, insufficient candidates, constraint relaxation, clarification effectiveness, or model latency.
4. No per-turn generated typed Context Program.
5. No Strategy Outcome and no strategy feedback closed-loop.
6. Currently belongs to "Dynamic State Machine + Static Heuristic Branching," not yet complete Dynamic Context Programming.

## 5. Target Architecture

```text
Current User Message
    ↓
State Update
    ↓
Context Distillation
    ├─ Short-Term Context
    ├─ Long-Term Profile Snapshot
    └─ Strategy/Outcome Memory
    ↓
Pre-Retrieval Context Program
    ↓
Sparse Probe
    ↓
Post-Probe Program Revision
    ↓
Dynamic Pipeline Execution
    ↓
Ranking / Guidance
    ↓
Outcome Recording
    ↓
Update Short-Term Context and Eligible Long-Term Preferences
```

The goal is to achieve constrained, auditable, and rollbackable runtime strategy adaptation, rather than letting the model modify source code or global configurations itself.

## 6. Subsequent Implementation Plan

### Phase 0: Freeze Existing I, II, and BGE Achievements

Goal: Establish a stable reference before third-phase refactoring to avoid breaking existing capabilities.

Work Content:

1. Fix current Hashing/BGE configuration, dev SHA, and catalog SHA.
2. Establish a standard artifact manifest for full BGE, recording:
   - Model name;
   - Catalog SHA;
   - 50,000 rows;
   - 384 dimensions;
   - Parent ASIN order;
   - Metadata SHA;
   - Standard matrix SHA or numerical tolerance rules.
3. Add formal BGE deployment configuration, no longer relying solely on the `StrictBgeAgent` experimental entry.
4. Retain three-level degradation:

```text
BGE/CUDA → Hashing Dense → Sparse-only
```

5. Whether BGE is enabled is decided by deployment configuration, not hard-coded for the native environment.
6. Retain current 36 tests and Hashing/BGE dev results as regression baselines.

Acceptance Criteria:

- Standard Agent can select BGE via configuration;
- Partial indexes, error catalogs, error models, or no CUDA environment cannot masquerade as valid BGE;
- Hashing and BGE baselines are reproducible.

### Phase 1: Implement Personalized Context Distillation

Suggested additions:

```text
solution/context/
├─ schemas.py
├─ distiller.py
├─ profile_store.py
└─ policies.py
```

`DistilledContext` must at least include:

- Current core objective;
- Current intent and confidence;
- Confirmed preferences;
- Tentative preferences;
- Negative preferences;
- Current session temporary preferences;
- Long-term stable preferences;
- Unresolved attributes;
- Recent Override;
- Recommended, negated, or invalidated candidates;
- Recent strategy and effects;
- Limited-length context summary;
- Context revision.

Each preference evidence must include:

- Attribute/value;
- Positive or negative;
- Explicit or inferred;
- Confidence;
- Source turn/session;
- Session-only or durable;
- Update time and version.

Conflict priority is fixed as:

```text
Current Turn Explicit Requirement
> Current Session Confirmed Slot
> Current Session Inferred Preference
> Long-Term User Profile
> System Default Strategy
```

Acceptance Criteria:

- Current requests are not covered by long-term profiles;
- After Override, invalid short-term evidence is no longer used;
- Context length has a fixed upper limit;
- Same input and history produce identical distillation results.

### Phase 2: Implement Long-Term User Profiles

First implement protocol and pure memory version:

```python
class ProfileStore(Protocol):
    def load(profile_id: str) -> LongTermProfile: ...
    def update(profile_id: str, mutations: ...) -> LongTermProfile: ...
    def delete(profile_id: str) -> None: ...
```

Suggested rules:

- Single-turn ordinary preferences enter only short-term state;
- User explicit request to remember allows direct promotion;
- Promotion allowed after repeated confirmation across multiple sessions;
- `forget`, `don't remember`, `no preference` can delete or downgrade weight;
- Intent Override defaults to covering only the current session;
- Do not save complete original dialogue, only save minimized structured evidence.

Recommended upper-level profile format:

```json
{
  "profile_id": "Stable anonymous ID provided by external system",
  "summary": "...",
  "preference_tags": []
}
```

Without `profile_id`:

- Allow using passed-in profiles;
- Allow updates within the current session;
- Prohibit cross-session writes to prevent user profile mixing.

Before starting implementation, confirm:

1. Which external layer provides `profile_id`;
2. Whether the first version uses only memory storage;
3. What conditions allow short-term preferences to be promoted to long-term preferences.

Recommended first version: Explicit `profile_id`, pure memory, default no persistence.

### Phase 3: Implement Adaptive Context Program

Suggested additions:

- `solution/orchestration.py`
- `ContextProgram`
- `AdaptiveOrchestrator`

Generate immutable execution plans per turn:

```text
ContextProgram
- track
- active_routes
- route_limits
- route_weights
- hard_filtering
- constraint_relaxation_policy
- profile_weight
- dense_mode
- diversity_strength
- semantic_ranker_enabled
- semantic_ranker_top_n
- clarification_mode
- novelty_penalty
- fallback_policy
- context_revision
- reasons
```

Execution divided into two planning stages:

1. Pre-Retrieval Plan: Select initial routes based on distilled context.
2. Post-Probe Revision: Constrained adjustment based on candidate count, route saturation, and constraint conditions.

Allow runtime adjustments:

- Toggle BM25, Category, Metadata, Dense;
- Adjust candidate counts per route;
- Dynamically adjust profile weights;
- Adjust diversity and repetition recommendation penalties;
- Select hard filtering or soft constraints;
- Decide whether to call SemanticRanker;
- Early cutoff due to overload;
- Expand recall due to consecutive failures;
- Session-level circuit breaker due to model timeout.

`SolutionConfig` continues to save safety upper/lower limits; Context Program can only select parameters within configuration allowed ranges, not modify shared global configurations.

Acceptance Criteria:

- Buying, Browsing, Over-General, Override produce different and interpretable plans;
- Trace saves plans and generation reasons;
- Plans of different sessions do not pollute each other;
- Context Program cannot exceed candidate counts, latency, and model call limits.

### Phase 4: Establish Strategy Outcome Feedback Closed-Loop

Add `StrategyOutcome`:

```text
- context_program
- candidate_counts
- applied_constraints
- relaxed_constraints
- recommendation_repeat_rate
- clarification_attribute
- clarification_answered
- slot_acquired
- candidate_reduction
- user_rejection
- override_detected
- llm_latency
- llm_failure
- fallback_used
```

Next-turn orchestration uses these results:

- High recommendation repetition: Increase novelty/diversity;
- Consecutive no candidates: Expand Dense or relax profile;
- User negated attribute: Record negative preference and switch clarification dimension;
- User consecutively not answering questions: Stop repeated questioning, switch to structured options;
- Ollama consecutive failures: Temporarily disable LLM for current session;
- Long-term profile conflict with current request: Current task profile weight set to 0;
- Multi-turn non-convergence: Allow returning from precision to discovery or changing scenario questions.

After completion of this phase, form:

```text
History → Distillation → Orchestration → Outcome → Strategy Revision
```

### Phase 5: Supplement Specialized Test Sets

Existing dev is insufficient for complete third-phase validation and does not cover true category-less Over-General.

#### 5.1 Context Evolution Set

Covers:

- Multi-turn preference accumulation;
- Conflict between temporary preferences and long-term profiles;
- Buying gifts for others without polluting own profile;
- User revoking long-term preferences;
- Repeated attribute correction;
- Cross-session repeated preference promotion;
- Different user profile isolation;
- Prohibit persistence when no `profile_id`.

#### 5.2 Open Browsing / Orchestration Set

Suggested construction of 100–200 items, covering:

- Completely category-less short queries;
- Candidate pool overload;
- Consecutive recommendation repetition;
- User not answering clarification;
- User consecutively negating;
- Dense pool expansion and contraction;
- LLM timeout or exception;
- Recovery to full path after Sparse provisional.

New process metrics:

- Profile precision;
- Profile leakage;
- Slot acquisition rate;
- Clarification answer rate;
- Candidate reduction;
- Repeated recommendation rate;
- Cutoff rate;
- Cutoff recovery rate;
- Context Program branch coverage;
- LLM fallback rate;
- P50/P95 latency.

### Phase 6: Performance Analysis and BGE Productization

Conduct after third-phase stabilization; not recommended to directly migrate FAISS now.

Execution order:

1. Profile current end-to-end latency.
2. Decompose BM25, Category, BGE query encode, matrix dot product, constraint filtering, reranker, context distillation, and orchestration latency.
3. Select query cache, route cache, batch encoding, FAISS IndexFlatIP, or ANN based on evidence.

If introducing FAISS:

- First use exact `IndexFlatIP`;
- Compare current Top-120 for each test query;
- Require consistent candidate sets and order, or define strict numerical tolerance;
- Do not trade unverified accuracy loss for performance.

When serving, simultaneously add:

- Session TTL;
- Response Cache capacity;
- Retrieval Cache capacity;
- Profile Store TTL;
- Explicit close/cleanup;
- GPU/CPU memory monitoring.

### Phase 7: Qwen3.5 9B Shadow Access

Qwen should not directly control formal recommendations first.

Recommended order:

1. Turn/Intent/Slot shadow parser;
2. Context Distillation shadow;
3. Context Program shadow proposal;
4. Candidate semantic reranking;
5. After passing thresholds, only allow low-risk strategies to be controlled and effective.

Qwen output must pass through:

- JSON Schema;
- Route whitelist;
- Parameter ranges;
- Current state version;
- Hard constraint protection;
- Profile conflict protection;
- Complete candidate ID permutation;
- Timeout and error fallback.

Prohibit model from:

- Modifying source code;
- Modifying global configurations;
- Writing unconfirmed long-term profiles;
- Generating new product IDs;
- Arbitrarily skipping hard constraints;
- Deciding cross-user persistence itself.

RTX 3060 has only 6 GiB VRAM; simultaneous residency of BGE and Qwen3.5 9B poses risks. Before deployment, verify:

- Quantization version;
- GPU layers;
- CPU offload;
- Whether BGE and Ollama rotate residency;
- P95 latency;
- Peak VRAM;
- Timeout and circuit breakers.

### Phase 8: Freeze and Final Validation

After third-phase and Qwen tuning completion:

1. Freeze code, configurations, model versions, and artifacts.
2. Run full unit tests and specialized tests.
3. Perform final pairing confirmation on dev 150.
4. Run independent holdout/final only once.
5. After viewing final results, do not return to dev for targeted hyperparameter tuning.
6. Output final merged report including:
   - I, II, III requirement mapping;
   - Hashing/BGE/Qwen ablation;
   - Overall and scenario metrics;
   - Performance, VRAM, and latency;
   - Fallback situations;
   - Artifact SHA;
   - Known risks.

## 7. Unified Testing and Entry Thresholds

### 7.1 Functional Thresholds

- Current explicit requirements always higher than long-term profiles;
- Intent Override does not retain invalid short-term conditions;
- No cross-session writes when no stable `profile_id`;
- Profile leakage between different users is 0;
- Context Program has version, reasons, and trace;
- Illegal Context Program falls back to deterministic plans;
- Maintain legal candidates and complete degradation when Qwen unavailable;
- Same input, context, and artifact results are reproducible.

### 7.2 Accuracy Thresholds

- Continue using frozen dev 150 for tuning only;
- Hashing mode HR@10 not lower than 0.913333;
- Hashing mode technical score not lower than 0.767724;
- BGE mode HR@10 not lower than 0.913333;
- BGE mode technical score not lower than 0.770181;
- Intent Override should not degrade;
- Long-term profile conflict scenarios must not be lower than control without profiles;
- After Qwen enablement, results must be paired with same backend, same split, same configuration but Qwen disabled.

### 7.3 Performance Thresholds

- Record end-to-end elapsed and P50/P95;
- Record latency for each route, distillation, orchestration, and LLM;
- Context/Session/Profile Cache must have capacity and TTL;
- New adaptive logic must not cause unbounded memory growth;
- GPU OOM, Ollama timeout, and CUDA unavailability must be safely degraded.

## 8. Recommended Execution Priority

```text
P0  Freeze existing baseline and BGE formal deployment entry
 ↓
P1  Distilled Context
 ↓
P2  Profile Store
 ↓
P3  Context Program / Adaptive Orchestrator
 ↓
P4  Strategy Outcome Feedback Closed-Loop
 ↓
P5  Context/Open-Browsing Specialized Testing
 ↓
P6  Performance Optimization and Cache Lifecycle
 ↓
P7  Qwen Shadow and Controlled Enablement
 ↓
P8  Independent Final Validation
```

## 9. Items Not to Execute Temporarily

- Do not continue tuning weights for individual failed samples in dev.
- Do not reuse public 200 for development tuning.
- Do not implement cross-user long-term profiles without stable `profile_id`.
- Do not let LLM directly modify global strategies or write long-term profiles.
- Do not migrate FAISS/ANN directly before profiler.
- Do not enable Qwen SemanticRanker by default.
- Do not immediately delete legacy state; first ensure compatibility migration and complete pairing validation.
- Do not judge index errors solely based on cross-GPU matrix SHA differences.
- Do not replace official final evaluation with specialized synthetic data.

## 10. Next Steps Suggestions

The most reasonable specific task for the next step is:

1. Confirm `profile_id` source and profile storage boundaries;
2. Confirm business rules for promoting short-term preferences to long-term preferences;
3. First implement without relying on LLM:

```text
DistilledContext
+ InMemoryProfileStore
+ ContextProgram
+ AdaptiveOrchestrator basic skeleton
```

4. Supplement unit tests and specialized synthetic dialogues for the above basic capabilities;
5. Run Hashing/BGE pairing regression separately on frozen dev 150;
6. Enter Strategy Outcome and Qwen Shadow phases only after metrics pass.
