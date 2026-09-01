# Dynamic Context Programming Implementation and Verification Notes

> Historical Note: The "starter unmodified" description in this report corresponds to the current round's experiment. For the final submission phase, only `starter/agent.py` is changed to serve as an entry adapter for `solution.agent.Agent`; the original weak baseline is preserved in `starter/baseline_agent.py`; the official evaluator remains unmodified.

## 1. Scope Completed in This Round

This round solely completes previously unfinished and partially completed capabilities; it does not redesign the dual-track retrieval, hard constraints, Over-Generality, official metrics, or frozen data flow which have already been fully validated.

Completed:

1. Bounded, typed Personalized Context Distillation;
2. Pure in-memory long-term profile learning requiring an explicit `profile_id`;
3. Two-stage Context Program for Pre-Retrieval / Post-Probe;
4. Strategy Outcome recording and feedback loop for the next round;
5. Qwen Session circuit breaker, error classification, success rate, Tokens, and average/P50/P95 latency observations;
6. Default disabling of the current main chain with complete dev150 pairing validation when Qwen is enabled.

## 2. Current Execution Chain

```text
User Turn
  → update_state
  → profile mutation (explicit profile_id only)
  → ContextDistiller
  → Pre-Retrieval ContextProgram
  → Sparse Probe
  → Post-Probe Program Revision
  → Dynamic Hybrid Pipeline
  → Cross-Encoder / Semantic Ranker (according to plan and circuit breaker status)
  → Novelty / Clarification
  → StrategyOutcome
  → Next round distillation and re-ranking
```

All programs are single-round immutable values; they do not modify the shared `SolutionConfig`. States, profile snapshots, and circuit breakers across different Sessions do not pollute each other.

## 3. File Descriptions

| File | Purpose |
|---|---|
| `solution/context/schemas.py` | Defines `PreferenceEvidence`, `LongTermProfile`, `ProfileMutation`, `DistilledContext` |
| `solution/context/distiller.py` | Compresses messages, slots, Overrides, recommendations, profiles, and Strategy Outcomes into a bounded context |
| `solution/context/profile_store.py` | `ProfileStore` protocol with pure in-memory implementation, supporting cross-session promotion, decay, forgetting, and user isolation |
| `solution/context/policies.py` | Identifies remember, forget, no preference, gift/session-only, and recommendation rejections |
| `solution/orchestration.py` | Defines `ContextProgram`, `StrategyOutcome`, two-stage orchestrator, and novelty penalty |
| `solution/agent.py` | Connects distillation, profiling, orchestration, retrieval, ranking, and Outcome into a closed loop, outputting a complete diagnostic trace |
| `solution/config.py` | Stores all context length limits, profile limits, routing caps, circuit breakers, and orchestration safety boundaries |
| `solution/query_builder.py` | Weakly uses long-term profiles only when no current conflict exists; explicitly prioritizes the current request |
| `solution/state.py` | Records Override rounds; revokes current slots for `no preference`; Overrides clear invalid rejection candidates |
| `solution/pipeline.py` | Accepts single-round dynamic profile weights and diversity intensity without modifying global configuration |
| `solution/llm/qwen.py` | Accumulates success/fallback/error types, Tokens, average/P50/P95 latency, maintaining strict candidate permutation validation |
| `scripts/test_ollama_integration.py` | Continues even if a scenario fails and saves a complete machine-readable audit report |
| `tests/test_context_programming.py` | Covers context boundaries, profile isolation, promotion/revocation, plan revision, feedback, circuit breakers, and Qwen observations |

## 4. Personalized Context Distillation

`DistilledContext` currently includes:

- Current core goal, intent, and confidence;
- Confirmed, tentative, negative, Session temporary, and long-term stable preferences;
- Unresolved attributes;
- Recent Overrides;
- Recommended and rejected candidates;
- Recent Strategy Outcome;
- Bounded summary of the recent message;
- Current profile conflicting attributes;
- Number of rounds without progress and consecutive LLM failures;
- Context revision.

The default limits are 4 recent messages, 10 candidates, 16 preferences, 4 Outcomes, and an 800-character summary. Identical inputs, history, and profile snapshots produce identical results.

Conflict priority is fixed as:

```text
Current round explicit requirement
> Current Session confirmed slot
> Current Session tentative preference
> Long-term profile
> System default
```

If the current request conflicts with the long-term profile on the same attribute, the current round's `profile_weight` automatically becomes 0; long-term preferences cannot override the current task.

## 5. Long-Term Profile Learning and Privacy Boundaries

- Cross-session writes are allowed only when an external profile contains a non-empty `profile_id`;
- Without a `profile_id`, incoming profiles and current Session state can be used, but persistence is prohibited;
- Ordinary single-round preferences are retained only in short-term state;
- A preference promoted to long-term status only after being confirmed in two different Sessions;
- `remember ...` can explicitly promote directly;
- `forget ...`, `don't remember ...`, and `no preference for ...` can revoke;
- Confidence decay is executed upon profile updates;
- Requests of the type "gift / for someone else" do not write to the long-term profile;
- The Store saves only minimal structured evidence, not the complete original conversation;
- The initial version uses process memory, complying with the competition's assumption of no external heavy databases and single-user Sessions.

## 6. Adaptive Orchestration and Feedback

Each round's Context Program saves: route, route limits, hard filters, Dense mode, diversity, profile weight, whether Qwen is called, Top-N, clarification mode, novelty penalty, fallback policy, context revision, and reason.

Integrated adaptive rules include:

- Over-General: Immediately stop Dense/LLM and switch to active clarification;
- Too few candidates: Expand Dense within configuration limits;
- Multiple rounds of genuine lack of progress: Expand recall and increase diversity/novelty;
- User indicates no preference after valid clarification response: Do not count as lack of progress;
- User rejects recommendation: Penalize previous round's candidates in the next round;
- Current request conflicts with long-term profile: Profile weight becomes zero;
- Intent Override: Protect current precision plan, preventing old recommendation feedback from overriding new intent;
- Two consecutive LLM failures in the same Session: Open Session circuit breaker and restore deterministic ranking;
- All dynamic candidate counts, Dense, Semantic Top-N, and diversity cannot exceed `SolutionConfig` limits.

Strategy Outcome saves route candidate counts, unique candidate count, candidate reduction amount, constraint application/relaxation, recommendation repetition rate, whether clarification answered, whether a slot was obtained, user rejection, Override, LLM latency/failure, and fallback. The next round's distillation consumes these results.

## 7. Default Main Chain dev150 Regression

The comparison object is the old Hashing default Agent using the same catalog, same frozen dev SHA, and same commit. Both Qwen and Cross-Encoder are disabled.

| Metric | Old Default Agent | Context Programming Agent | Change |
|---|---:|---:|---:|
| HR@10 | 0.913333 | 0.920000 | +0.006667 |
| MRR | 0.597302 | 0.607254 | +0.009952 |
| MTTC ↓ | 4.406667 | 4.360000 | Improved by 0.046667 |
| Efficiency | 0.659333 | 0.664000 | +0.004667 |
| Technical Score | 0.767724 | 0.774976 | +0.007252 |
| Total Time | 202.41 seconds | 196.13 seconds | No increase |

Scenario results: Browsing HR/MRR/MTTC improved; Buying MRR/MTTC improved; Intent Override and Boundary are fully consistent with the old baseline, without masking key scenario regressions with overall gains.

## 8. Current Main Chain Qwen Full Pairing

| Metric | Context-only | Context + Qwen | Change |
|---|---:|---:|---:|
| HR@10 | 0.920000 | 0.920000 | Flat |
| MRR | 0.607254 | 0.620931 | +0.013677 |
| MTTC ↓ | 4.360000 | 4.380000 | Degraded by 0.020000 |
| Efficiency | 0.664000 | 0.662000 | -0.002000 |
| Technical Score | 0.774976 | 0.778679 | +0.003703 |
| Total Time | 196.13 seconds | 4,800.70 seconds | ~24.48× |

Qwen Call Audit:

- Actual requests: 623; successfully applied: 546;
- 77 were entirely `invalid_candidate_permutation`, safely fallbacked;
- Success rate 87.64%, fallback rate 12.36%;
- Average latency 7.40 seconds, P50 7.77 seconds, P95 8.28 seconds;
- Reported prompt tokens: 1,737,271; completion tokens: 212,169; total: 1,949,440;
- Per-session reciprocal rank: 6 improved, 2 degraded, 142 unchanged; no new or lost Top-10 hits; 2 sessions hit slower.

Therefore, the implementation of Qwen, real connection, full quantification, safe fallback, observations, and Session circuit breakers are complete; however, the default release threshold has not been met, so `semantic_ranker_enabled=false` must remain. Feature completion does not equal default enablement.

## 9. Verification and Invariants

- All 49 tests passed;
- `compileall`, strict `ResourceWarning`, and `git diff --check` passed;
- Frozen data SHA and sample counts passed;
- Official `evaluator/local_evaluator.py` and `starter/agent.py` unmodified;
- Catalog SHA unchanged;
- Holdout, public, or sealed final not run;
- Default retrieval weights not modified;
- Default Qwen remains disabled.
