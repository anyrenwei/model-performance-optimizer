# Optimization Specification And Report Schema

Use structured records so optimization claims remain reproducible and attributable.
Adapt fields to the repository rather than creating empty ceremony.

## OptimizationSpec

Freeze a task contract before modifying performance-sensitive behavior:

```yaml
task:
  name: model-workload
  mode: inference
  entrypoint: run_model.py
  device: npu
  contract_id: sha256-of-frozen-contract
workload:
  dataset: validation.jsonl
  batch_sizes: [1, 4]
  sequence_lengths: [512, 2048]
  cache_policy: warm
  random_seed: 3407
objective:
  primary_metric:
    name: steady_state_latency
    statistic: p90
    unit: ms
    direction: minimize
  comparison:
    name: throughput_improvement
    formula: baseline / candidate - 1
    threshold: 0.20
constraints:
  business_metrics:
    - name: task_accuracy
      max_regression: null
  peak_memory_mb: null
  compile_time_ms: null
  no_new_nan_inf: true
precision:
  baseline: bf16
  candidate_must_match: true
  allow_precision_search: false
allowed_changes:
  model_script: true
  fixed_shape_specialization: false
  model_source_specialization: false
  compiler_config: true
  decomposition: true
  fusion_pattern: true
  lowering: true
  custom_kernel: true
  algorithm_classes: [A, B]
  approximate_algorithm: false
search:
  max_experiments: 30
  optimization_budget_seconds: 14400
  budget_clock: active_item_wall
  queue_wait_counts: false
  device_wait_counts: false
  promotion_reserve_seconds: 1200
profiling:
  baseline: full
  every_candidate: lightweight
  new_best: full
  final: full
```

Null business thresholds do not authorize regression; they exclude candidates that
need those thresholds. User/project values override illustrative fields.

## Metric And Contract Semantics

Name the comparison formula rather than using a bare `improvement` field. For a
latency baseline `b` and candidate `c`:

```text
latency_reduction = 1 - c / b
equivalent_throughput_improvement = b / c - 1
```

These values are not interchangeable. A 20% throughput improvement is only a 16.67%
latency reduction. The contract freezes the metric, statistic, unit, direction,
formula, threshold, and denominator before search starts.

Derive `contract_id` from the workload, precision, backend, cache, allowed-change,
measurement, correctness, and objective fields that determine comparability. A change
to any material field creates a new contract instead of silently mutating the old one.

## Candidate Result

Persist a record equivalent to:

```json
{
  "candidate_id": "fusion-007",
  "parent_id": "compile-003",
  "contract_id": "sha256-of-frozen-contract",
  "source_snapshot": "git-or-content-hash",
  "baseline_fingerprint": "sha256-of-baseline-provenance",
  "origin": "discovered_current_task",
  "hypothesis": "Fuse the exposed residual-normalization chain",
  "effective_mode": "compile",
  "effective_precision": "bf16",
  "device_id": "npu:2",
  "gates": {
    "execution": "pass",
    "correctness": "pass",
    "business_metric": "pass",
    "nan_inf": "pass",
    "stability": "pass"
  },
  "performance": {
    "e2e_median_ms": 19.82,
    "e2e_p90_ms": 20.31,
    "throughput": 128.6,
    "peak_memory_mb": 18120,
    "compile_time_ms": 38210
  },
  "profiling": {
    "device_compute_ms": 16.91,
    "exposed_host_ms": 1.42,
    "exposed_communication_ms": 0.0,
    "device_starvation_ms": 0.63,
    "kernel_launch_count": 314,
    "graph_breaks": 0,
    "recompiles": 0
  },
  "comparison": {
    "formula": "baseline / candidate - 1",
    "incremental_speedup": 1.074,
    "cumulative_speedup": 1.463,
    "business_metric_delta": 0.0
  },
  "decision": "accept",
  "failure_class": null
}
```

Store raw timing samples, logs, profiles, environment manifests, commands, and code
revision references outside the compact record but link them from it.

## Decision Vocabulary

- `accept`: all gates pass and measured improvement is material.
- `reject`: a gate or frozen constraint fails.
- `inconclusive`: evidence does not separate the result from noise or is incomplete.
- `enabler`: gate-passing, neutral standalone change needed for a dependent candidate.
- `invalid`: benchmark, verifier, effective mode, or environment evidence is unusable.
- `blocked`: the frozen contract has no admissible baseline or executable route.

Classify attempt failures independently as candidate, infrastructure, workspace
integrity, verifier, budget, policy, or compiler failures. Infrastructure and workspace
failures are not evidence that the candidate mechanism is slow or incorrect.

## Report Contents

Lead with the final outcome and whether the original baseline was trustworthy. Include:

1. frozen objective and formula, workload, precision policy, environment, and gates
2. eager reference, current compile baseline, distribution, memory, cold start,
   profile attribution, and headroom when each is in scope
3. accepted candidate chain with exact changes, immutable provenance, and evidence
4. inherited-and-revalidated, modified, newly discovered, rejected, inconclusive, and
   enabler experiments with reasons
5. final unprofiled E2E plus final profile attribution
6. remaining bounded headroom and untested high-risk directions
7. reproduction commands, raw artifact locations, resource-usage coverage, and
   limitations

For every accepted item report parent and original-baseline comparisons:

```text
incremental_speedup = parent_latency / candidate_latency
cumulative_speedup = original_latency / candidate_latency
```

For maximize metrics:

```text
incremental_speedup = candidate_throughput / parent_throughput
cumulative_speedup = candidate_throughput / original_throughput
```

Do not multiply isolated branch speedups. Measure the actual combined candidate.

Use a table such as:

| Candidate | Mechanism | Precision | Gate | Parent metric | Candidate metric | Incremental | Cumulative | Memory | Decision |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

Distinguish facts from estimates. Predicted recoverable time and externally reported
speedups remain hypotheses until verified locally.

For multi-model reports, use
[portfolio-optimization.md](portfolio-optimization.md) for contract cohorts,
denominators, per-item terminal states, resource accounting, and final consistency
checks.
