# Dialog Strategy: Multi-Turn Scenario Evolution Refactoring Audit & Handover

## 1. Audit Conclusion

This round completed the implementation of an explicit dynamic dialogue state machine, auditable Slot Store, incremental information accumulation, global and attribute-level Intent Override, candidate overload detection, special-turn retrieval cutoff, and structured proactive clarification.

Normal and information-sufficient requests still execute the complete pipeline determined in Phase 1:

```text
Keyword + Category + Metadata + Vector
→ Hybrid Fusion
→ Deterministic Ranking
→ SemanticRanker
→ Top 10
```

Special cutoff is executed **only** when the request is on the discovery track, has an open category, lacks valid slots, suffers from overly broad queries or sparse candidate overload, and there are still queryable attributes available:

```text
Sparse Probe
→ Over-Generality
→ Skip Dense/Cross Encoder/SemanticRanker
→ Sparse Provisional Top 10
→ Structured Proactive Clarification
```

Once the user supplements a slot, the next turn automatically restores the full Dense/LLM path.

The final frozen dev pair results are no lower than the baseline of the previous phase; HR remains unchanged, while MRR, MTTC, and Technical Score show slight improvements. Therefore, this round's results meet the accuracy threshold and can serve as the foundation for subsequent Qwen/BGE integration and real conversational traffic testing.

## 2. Requirement Completion Status

| Requirement | Status | Implementation |
|---|---|---|
| Dynamic State Machine | Completed | Explicit phases, transition reasons, state versions, and history |
| Information Accumulation | Completed | Structured slot incremental addition, legacy field compatibility synchronization |
| Intent Override | Completed | Three types of rewriting: full, attribute, category |
| Slot Erasure/Rewriting | Completed | Audit records for reset/replace/add |
| Over-Generality Detection | Completed | Query specificity, slot count, candidate count, route saturation |
| Immediate Retrieval Cutoff | Completed | Special turns stop Dense/Semantic stages after Sparse Probe |
| Provisional Top 10 | Completed | Uses sparse candidates; falls back to catalog legal hot items if empty |
| Proactive Structured Guidance | Completed | Single attribute, candidate examples, known condition summary, avoids repetition |
| Slot Reply Context | Completed | Uses pending clarification to explain unknown text like Acme/shoes |
| Normal LLM Pipeline | Maintained | Non-cutoff turns still call SemanticRanker boundaries |

## 3. Data & Experiment Baseline

| Item | Value |
|---|---|
| Catalog Rows | 50,000 |
| Catalog SHA256 | `da979b05a68af864cb0dcf9ee6a81c010c7e66a57978ad286c7a2e005fc69a67` |
| Dev Sample Count | 150 |
| Dev SHA256 | `c8955cacd79bb2e2ed6b984979bca91f379fc699648c5bc2e3ac2efb37f8b22a` |
| Dense Backend | Signed hashing, 50K × 384 |
| LLM | Deferred/disabled; interface calls retained for normal path |
| Control Run | `experiments/runs/post_dual_track_v2` |
| Final Run | `experiments/runs/post_dialog_state_final` |

The file `data/public_set.jsonl` retains its original newline status from the workspace and has not been modified in this round. The newly generated dev SHA256 matches the frozen manifest; this round only performed pairing tuning and validation on the frozen dev set.

## 4. Final Accuracy

### 4.1 Overall Metrics

| Metric | Phase 2 Refactoring Baseline | Final Code | Change |
|---|---:|---:|---:|
| HR@10 | 0.913333 | **0.913333** | Flat |
| MRR | 0.596968 | **0.597302** | +0.000334 |
| MTTC | 4.413333 | **4.406667** | -0.006666 (lower is better) |
| Efficiency | 0.658667 | **0.659333** | +0.000666 |
| Technical Score | 0.767490 | **0.767724** | +0.000234 |

### 4.2 Scenario Metrics

| Scenario | HR@10 | MRR | MTTC | Difference vs Control |
|---|---:|---:|---:|---|
| Buying | 0.916667 | 0.599107 | 3.566667 | HR/MRR identical, MTTC +0.016667 |
| Browsing | 0.950000 | 0.641653 | 4.816667 | Identical |
| Intent Override | 0.818182 | **0.462428** | **5.500000** | MRR +0.002273, MTTC -0.090909 |
| Boundary | 0.875000 | 0.622024 | 4.625000 | Identical |

Overall accuracy passes the "no lower than current level" threshold. There is a minor sample-level fluctuation in Buying MTTC, but Intent Override improvements raise overall MTTC and Technical Score.

## 5. Dynamic State Machine

New phases:

```text
new
discovery
clarifying
constrained
ready
overloaded
rewriting
```

Typical transitions:

```text
new
  └─ Vague request → discovery → overloaded → clarifying
                                      ↓ User supplements slot
                                constrained → ready

constrained
  └─ Full override → rewriting → constrained/ready

constrained
  └─ Attribute rewrite → rewriting → constrained/ready
```

Each phase change records:

- turn
- from_phase
- to_phase
- reason

Repeated calculations within the same phase do not generate meaningless duplicate transitions.

## 6. Slot Store & Information Accumulation

`SessionState` additions:

```text
slot_store
slot_history
dialogue_phase
transition_history
state_revision
last_processed_turn
last_processed_message
pending_clarification
over_generality
```

Each active slot continues to reuse `Constraint`:

- attribute
- operator
- value
- confidence
- source_turn
- hard
- raw

Every change is audited using `SlotMutation`:

- add
- replace
- replace_value
- reset

Records old values, new values, turn, and reason.

### 6.1 Incremental Information

Normal multi-turn replies append non-duplicate slots. For example:

```text
material=cotton
→ use_case=hiking
→ budget<=100
```

Existing `hard_constraints`, `soft_preferences`, and budget fields are retained for compatibility with QueryBuilder and Reranker; the current `slot_store` is the new structured view. Legacy state was not deleted in one go in this round.

### 6.2 Previous Turn Question Context

When the user answers only:

```text
Acme
hiking boots
medium
```

The vocabulary itself may not determine the attribute. The system reads `pending_clarification.attribute` to interpret the plain answer as the brand/category/size asked in the previous turn.

If the user uses an explicit `material: leather` or issues an intent override, explicit content takes precedence and is not restricted by the previous question type.

## 7. Intent Override Semantics

### 7.1 Full Override

Trigger forms include:

- ignore my earlier preference
- ignore everything
- forget everything
- start over

Cleanup actions:

- hard constraints
- soft preferences
- structured constraints
- exclusions
- budget min/max

Preserved items: session ID, history, asked attributes, and original category; if the new message explicitly provides a category, it is updated.

### 7.2 Attribute Override

For example:

```text
Actually, blue instead of black.
Change the size to large.
```

Only the corresponding attribute is replaced; other slots are retained.

### 7.3 Category Override

For example:

```text
I'm looking for hiking boots instead of running shoes.
```

Replaces the category and clears category/size/style slots heavily dependent on the old category; general conditions like budget are retained.

### 7.4 Turn Protection

- Same turn, same input: Returns cached response; does not re-execute Dense/LLM or modify state.
- Same turn, different input: Rejected.
- Old turn overwriting new state: Rejected.

## 8. Over-Generality Detection

Added `OverGeneralityDecision`:

```text
overloaded
confidence
reasons
unique_candidate_count
saturated_routes
query_term_count
active_slot_count
```

Detection signals:

1. Current track must be discovery.
2. Category must be empty or belong to broad categories like product/clothing/item.
3. Active slot count must not exceed the configured threshold.
4. Query tokens are too few, unique candidate count exceeds limit, or multiple sparse routes are saturated.

Default configuration:

```text
over_generality_cutoff_enabled=true
over_generality_max_query_terms=4
over_generality_max_active_slots=0
over_generality_min_unique_candidates=160
over_generality_min_saturated_routes=2
over_generality_ask_until_turn=9
```

If all attributes have been asked or are unavailable, or the cutoff question window is exceeded, cutoff is no longer applied; full retrieval resumes to avoid the system being unable to converge.

## 9. Two-Stage Retrieval

### 9.1 Probe

`HybridPipeline.probe()` executes:

- BM25 keyword
- Category Route
- metadata BM25
- sparse fusion
- unique candidate count
- route saturation

The normal path reuses Probe results and does not re-execute sparse queries.

### 9.2 Full Path

Non-over-general requests continue to execute:

- Dense
- fusion/supplement
- hard constraint policy
- deterministic rerank
- optional cross encoder
- SemanticRanker

### 9.3 Cutoff Path

Overloaded turns execute `HybridPipeline.provisional()`:

- Does not call Dense
- Does not call Cross Encoder
- Does not call SemanticRanker
- Performs lightweight deterministic sorting on sparse candidates
- Returns provisional Top 10
- If sparse candidates are empty, only uses legal hot items within the catalog to fill

SemanticRanker diagnostic reason is explicitly recorded as:

```text
over_generality_cutoff
```

It is not confused with `ranker_disabled` or model exception fallbacks.

## 10. Proactive Clarification

`choose_clarification()` returns a structured decision:

```text
attribute
score
reason
coverage
entropy
expected_reduction
example_values
prompt
```

Selection remains based on candidate coverage, attribute entropy, expected candidate reduction, and strategy priors, while retaining:

- Ask only one legal attribute per turn
- Do not repeat questions
- Do not ask attributes the user has declared no preference for
- Do not ask category if it is already explicit

Over-General prompts explain that candidates are too broad and include as much as possible:

- Currently confirmed conditions
- One explicit attribute question
- 2–3 example values appearing in current candidates

## 11. Diagnostics & Failure Tracking

Agent trace additions:

- dialogue phase
- state revision
- slot store
- Probe candidate count
- saturated routes
- over-generality reasons/confidence
- retrieval cutoff
- clarification score/entropy/examples
- semantic skip reason

`experiments/trace_failures.py` has been synchronized to export these fields for round-by-round review upon real Qwen integration:

- Why the model was skipped
- Which slot triggered restoration of full retrieval
- Which old values were deleted by override
- Which clarification question was selected

## 12. Test Results

Current full regression is `35/35` tests passed. Dialog strategy additions cover:

- Global override slot reset
- Attribute-level black → blue rewrite
- Category override while retaining budget
- Override unaffected by previous turn question error type
- Slot Store incremental accumulation
- Pending clarification type inference
- Stale/conflicting turn rejection
- Duplicate response returns cache and does not re-call Dense/Ranker
- Candidate count and multi-route saturation detection
- Over-General first-turn skip of Dense/Ranker
- Cutoff still returns legal provisional recommendations
- Proactive prompts and `ask_attribute`
- Restoration of Dense/Ranker after user supplements a slot
- Existing retrieval, ranking, Dense degradation, and Agent contract regression

## 13. Performance

| Version | 150 Dev Duration |
|---|---:|
| Phase 2 Control | 293.023 seconds |
| Final State Machine Version | 306.818 seconds |
| Change | +13.796 seconds, approx 1.047x |

This difference may include machine runtime fluctuations, but there is currently no evidence that the state machine incurs zero overhead. The normal path Probe reuses sparse results; truly Over-General turns skip Dense/LLM, which should theoretically be significantly faster. However, the official dev set lacks non-category overload samples, so overall savings cannot be verified with this collection.

## 14. Current Issues & Risks

### 14.1 Public Dev Does Not Cover True Over-General First Turn

The evaluator generates specific categories based on target products; therefore, Browsing initial messages in dev are usually not fully open requests. Cutoff correctness is verified by dedicated synthetic tests, but true trigger rates, thresholds, and user convergence rates require independent datasets or online shadow logs.

### 14.2 Intent/Slot Parser Remains English Rule-Based

Currently suitable for competition templates and common rewrite sentence patterns, but free language, Chinese, complex negations, and multiple alternative options may still lead to misjudgment. When integrating Qwen, it is recommended to first shadow output TurnEvent/SlotMutation without immediately controlling the formal state.

### 14.3 Context Slot Typing May Over-Trust Previous Question

Plain replies without prefixes are default interpreted as the attribute asked in the previous turn. If the user ignores the question and actively answers another attribute, misclassification may occur. Explicit `attribute: value` and overrides are handled with priority; future improvements can use model confidence or confirmation mechanisms.

### 14.4 Provisional Results Lack Dense Cross-Category Capability

This is the design purpose of cutoff: converge first before performing expensive semantic retrieval. The next turn restores Dense/LLM after the user supplements a slot. If product requirements mandate displaying semantically diverse results in the first turn, cutoff can be configured off, but this would contradict the active convergence goal of this module.

### 14.5 State Has Duplicate Representations

`slot_store/structured_constraints` and legacy hard/soft/budget fields currently coexist to reduce migration risk but increase maintenance costs. Data sources need unification after complete BGE/Qwen metrics stabilize.

### 14.6 Session/Response Cache Lacks Cross-Session Reclamation Strategy

Competition sessions are limited to 10 turns; current risk is controllable. For long-term service deployment, session TTL, capacity limits, and explicit close/reset cleanup should be added.

### 14.7 Qwen/BGE Still Pending Merge

This round did not download or manage models. After real model deployment, re-validation is needed:

- Normal path Ranker call rate
- Cutoff skip rate
- Slot convergence turns
- P95 latency
- Token/cost
- Fallback rate
- HR/MRR/MTTC

## 15. Modified Files

Added:

- `solution/state_machine.py`
- `solution/generality.py`

Modified:

- `solution/schemas.py`
- `solution/state.py`
- `solution/constraint_parser.py`
- `solution/intent.py`
- `solution/clarification.py`
- `solution/pipeline.py`
- `solution/agent.py`
- `solution/config.py`
- `solution/README.md`
- `experiments/trace_failures.py`
- `tests/test_solution.py`

Unmodified:

- `evaluator/local_evaluator.py`
- `starter/agent.py`
- Catalog content
- Official metric formulas

## 16. Reproduction Commands

```powershell
# Unit and regression tests
python -m unittest discover -s tests -v

# Final frozen dev evaluation
python -m experiments.run_experiment `
  --config experiments\configs\hybrid_steps_3_10.json `
  --run-dir experiments\runs\post_dialog_state_final

# Comparison with Phase 2 refactoring baseline
python -m experiments.compare_runs `
  experiments\runs\post_dual_track_v2 `
  experiments\runs\post_dialog_state_final
```

## 17. Follow-up Work Recommendations

1. Construct a dialogue set of 100–200 truly non-categorized, short queries with candidate overload to calibrate cutoff thresholds.
2. Integrate Qwen shadow parser to compare TurnEvent and SlotMutation without immediately controlling the formal state.
3. Re-test normal path and cutoff restoration paths after BGE merge.
4. Add confidence scores and "Do you mean brand Acme?" confirmation strategies for Context Slot Typing.
5. Add TTL/capacity strategies for session state, response cache, and retrieval cache.
6. Statistically analyze the real conversion funnel: clarification attribute → slot acquisition → next-turn candidate reduction.

## 18. Final Audit Opinion

This round has met the code requirements for Dynamic State Machine, Information Accumulation, Intent Override, Over-Generality Cutoff, and Proactive Guidance, while maintaining the structure of the Phase 1 normal LLM Pipeline unchanged.

Frozen dev proves accuracy did not decline; the Over-General special branch is proven by dedicated tests not to call Dense/Ranker and to restore the full pipeline after the user supplements a slot. Results can proceed to the next module or Qwen/BGE merge preparation phase.
