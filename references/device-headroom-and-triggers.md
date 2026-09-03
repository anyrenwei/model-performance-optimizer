# Device Headroom And Optimization Triggers

Use this protocol to decide whether an observed cost is actionable and which layer owns
it. Optimize exposed critical-path time, not merely large profiler totals.

## Normalize The Timeline

Derive or estimate these steady-state quantities from E2E and profiler evidence:

- device compute, memory, copy, and idle intervals
- host preprocessing, enqueue, synchronization, and exposed host intervals
- collective communication and its exposed portion
- kernel launch count, short-kernel distribution, and inter-kernel gaps
- graph breaks, compiled graph count, recompilations, and fallback regions
- peak memory, allocator activity, intermediate bytes, and layout conversions

When trace overlap is available, compute:

```text
host_overlap_ratio = host_work_overlapped_by_device / total_host_work
device_starvation_ratio = device_idle_waiting_for_host / steady_state_step
exposed_host_ratio = critical_path_host_time / steady_state_step
```

High host activity that is fully hidden is not a task-level bottleneck. Low host
overlap is actionable only when it exposes time or starves the device.

## Calibrate Attainable Ceilings

Prefer same-device, same-runtime, same-dtype, and shape-relevant microbenchmarks over
datasheet peaks. Calibrate compute, memory bandwidth, representative GEMM/attention or
convolution, and communication when those resources dominate.

```text
compute_utilization = achieved_work / calibrated_compute_ceiling
bandwidth_utilization = achieved_bytes_per_second / calibrated_bandwidth_ceiling
```

Use these as diagnostic ranges, not universal pass/fail percentages. Small and
dependency-heavy kernels may never approach a large GEMM ceiling.

## Estimate Recoverable Task Time

For each opportunity estimate:

```text
recoverable_ms = exposed_ms * removable_fraction * workload_coverage
predicted_speedup = baseline_ms / (baseline_ms - recoverable_ms)
```

Account for overlap: do not add host, device, and communication totals that run in
parallel. Compare recoverable time with measurement uncertainty and the minimum useful
gain before implementing a candidate.

## Trigger Matrix

| Evidence | Likely bound | Candidate families |
| --- | --- | --- |
| Low compute and bandwidth, many short kernels and gaps | Launch/scheduling | Fusion, graph capture, batching, fewer syncs |
| Device gaps correlate with CPU enqueue or preprocessing | Host | Remove Python/sync work, pipeline, async preprocessing |
| Bandwidth near attainable ceiling, low arithmetic intensity | Memory | Fusion, layout, reuse, fewer intermediates/copies |
| Compute near attainable ceiling | Compute | Better algorithm/kernel; precision only under separate gate |
| Long kernel with low compute and bandwidth | Kernel efficiency | Tiling, occupancy, vectorization, lowering, layout |
| Graph breaks or recompiles on the critical path | Compiler coverage | Canonicalization, decomposition, shape policy, pattern work |
| Exposed collectives or serialized communication | Communication | Overlap, bucket/layout changes, collective or parallelism strategy |
| Memory capacity limits useful batch/shape | Capacity | Recompute, sharding, offload, allocator/cache design |
| High device utilization but poor E2E | Mixed/outer pipeline | Queueing, preprocessing, communication, dependency analysis |

## Opportunity Gate

Do not implement a candidate until the evidence answers:

1. Is the target cost on the critical path?
2. Is it exposed rather than hidden by overlap?
3. Is recoverable time larger than uncertainty?
4. Does the pattern cover enough target workload and shapes?
5. Is the mechanism permitted by the frozen spec?
6. Can benefit plausibly amortize compile, memory, and maintenance costs?
7. Are the required correctness gates available?

Record opportunities in a ledger such as:

```yaml
id: residual-norm-fusion
category: fusion-pattern
evidence:
  critical_path_ms: 8.4
  frequency_per_step: 32
  intermediate_bytes: 134217728
headroom:
  removable_fraction: 0.55
  recoverable_ms: 4.62
  predicted_speedup: 1.08
coverage:
  workload_ratio: 0.91
risk:
  precision: low
  dynamic_shape: medium
action_layer: compiler-pattern
status: proposed
```

Rank untested opportunities by recoverable time, evidence confidence, workload
coverage, implementation cost, and regression risk. This priority is for deciding
what to test; only measured verifier results decide what to accept.

## Stop On Bounded Headroom

After every new best candidate, re-profile and recalculate remaining exposed host,
fusion, kernel, communication, and memory opportunity. Stop when the remaining Amdahl
bound is below uncertainty or the configured useful gain, rather than searching an
unbounded catalog indefinitely.
