---
name: model-performance-optimizer
description: Analyze and optimize model training, inference, or serving performance with trustworthy timing, device-headroom analysis, correctness gates, controlled code changes, compiler and fusion work, autonomous candidate search, and attributable reporting. Use for PyTorch, torch.compile, GPU/NPU, profiling, model-script, algorithm, backend, kernel, memory, or distributed performance work; do not use for generic refactoring without a performance objective.
---

# Model Performance Optimizer

Optimize the user's task-level metric without silently changing model behavior. A
faster result is not successful until its measurement is trustworthy and its
acceptance gates pass.

## Respect The Requested Mode

- For analysis, review, or diagnosis requests, inspect and report without editing.
- For optimization requests, make scoped changes, validate them, and retain only
  evidence-backed candidates.
- Do not broaden a model optimization request into an unapproved precision,
  quantization, pruning, architecture, dataset, or serving-policy change.
- Prefer the repository's harness, profiler, tests, and backend conventions. Repair
  them when necessary for trustworthy evidence and within the user's requested
  scope.

## Freeze The Objective

Before changing code, record or infer an `OptimizationSpec` containing the workload,
primary metric, metric statistic and unit, comparison formula, target semantics,
constraints, precision policy, allowed change layers, cache policy, and budget. Give
the frozen contract a stable fingerprint. Use a strict same-precision policy by
default. Missing business-metric thresholds block approximate or reduced-precision
candidates, but do not block semantics-preserving work.

Never use an ambiguous `improvement` field. For example, latency reduction
`1 - candidate / baseline` and equivalent-throughput improvement
`baseline / candidate - 1` are different percentages. Record the chosen formula and
report both when readers may otherwise confuse them.

Read [references/report-schema.md](references/report-schema.md) when creating the
spec or result ledger.

## Establish A Trustworthy Baseline

Audit timing, synchronization, workload equivalence, warmup, compile fallback,
precision, randomness, cache state, source identity, and result persistence before
trusting existing numbers. If the baseline is invalid, fix or replace the measurement
first, invalidate its descendants, and mark earlier speedup claims unusable.

Use separate correctness, steady-state, profiling, and cold-start runs. Official
steady-state latency must be measured without profiler overhead.

Read [references/benchmark-protocol.md](references/benchmark-protocol.md) before
creating or repairing a benchmark.

## Define Acceptance Gates

Run correctness and business gates before performance ranking. Reject new NaN or
Inf values. Compiler, backend, fusion, and kernel candidates normally keep the same
effective precision. Precision and approximate-algorithm candidates require an
explicit, representative business-accuracy contract.

Read [references/acceptance-gates.md](references/acceptance-gates.md) whenever a
candidate can affect numerical or model behavior.

## Estimate Headroom And Choose The Layer

Use synchronized E2E measurements, profiler critical paths, and device-calibrated
compute and bandwidth ceilings. Optimize exposed time, not work already hidden by
device execution or communication. Estimate recoverable milliseconds and the
task-level Amdahl bound before implementing a candidate.

Classify the bottleneck as host, launch, compute, memory, communication, capacity,
or mixed. Stop investigating an opportunity when its recoverable time is not
materially larger than measurement uncertainty.

Read [references/device-headroom-and-triggers.md](references/device-headroom-and-triggers.md)
for the decision protocol.

## Generate And Implement Candidates

Treat the built-in optimization catalog as seeds, not a complete list. Search local
code and current primary sources when model-, version-, backend-, or hardware-specific
evidence is needed. Record the mechanism, applicability, expected recoverable time,
risk, coverage, and validation plan for each candidate.

Escalate only as evidence requires:

1. measurement and workload corrections
2. runtime, data path, batching, shape, and memory behavior
3. model-script and semantics-preserving algorithm changes
4. graph capture, compilation, decomposition, and canonicalization
5. fusion pattern, lowering, scheduling, and custom kernel changes
6. approximate algorithms or reduced precision under a separate approved contract

Read [references/model-code-and-algorithms.md](references/model-code-and-algorithms.md)
for model and algorithm work. Read
[references/graph-fusion-and-kernels.md](references/graph-fusion-and-kernels.md) before
editing compiler patterns, lowerings, schedules, or kernels.

For Ascend NPU DVM work, also read
[references/ascend-npu-dvm.md](references/ascend-npu-dvm.md). Verify the effective
backend before analysis and separate graph-capture problems from in-graph fusion
problems. Use its profile-triggered tuning loop instead of blindly enumerating DVM
flags. Require an observable generated-artifact or fusion-signature change before a
long benchmark, and account for wrapper dispatch, output allocation, and launch count
when evaluating a fused or custom operator.

## Run Controlled Experiments

- Give every candidate a parent and change one primary mechanism at a time.
- Keep exact code/config diffs, raw samples, effective precision, and environment
  provenance.
- Bind baseline and candidate evidence to an immutable contract, source snapshot, and
  baseline fingerprint. Mutable global or session state is not evidence provenance.
- Warm compilation and target caches outside steady-state measurement.
- A compile failure that falls back to eager is a failed compile candidate, never a
  compile result.
- Roll back rejected candidates to their last verifier-backed parent.
- A performance-neutral change may be retained only as an explicit `enabler` for a
  later candidate; do not advertise it as a speedup.
- Fully profile the baseline, pattern/kernel candidates, new best candidates, and
  final candidate. Re-run final E2E with profiling disabled.

For broad autonomous exploration, read
[references/autonomous-search.md](references/autonomous-search.md). Freeze the verifier
and search budget before launching candidate loops. Keep the verifier and candidate
contract runnable without any particular orchestration product; external search
runtimes are replaceable adapters.

For a multi-model or multi-workload optimization campaign, read
[references/portfolio-optimization.md](references/portfolio-optimization.md). Partition
results by contract, define the per-item budget clock, and reserve enough budget to
promote promising candidates before starting parallel search.

## Decide And Stop

Use hard gates followed by task-metric ranking. Do not let a weighted performance
score compensate for correctness failure. Prefer, in order: the primary metric,
tail behavior, memory constraints, compilation cost, and lower maintenance risk.

Stop when the explicit target is reached under its required final gate, the budget is
exhausted, the verifier is invalid, remaining recoverable time is below uncertainty or
the minimum useful gain, or every remaining direction needs unapproved behavioral
change.

## Report Evidence

Report the accepted chain, rejected and inconclusive experiments, raw metric summary,
profiling attribution, remaining headroom, limitations, and reproduction commands.
For compile optimization, report eager reference, current compile baseline, and tuned
compile when each is admissible. Distinguish inherited-and-revalidated work from code
changed or discovered in the current task.

For each accepted item report both:

```text
incremental_speedup = parent_metric / candidate_metric
cumulative_speedup = original_baseline_metric / candidate_metric
```

Use the inverse ratios for maximize objectives such as throughput. Never multiply
isolated branch speedups to claim cumulative gain.

Read [references/report-schema.md](references/report-schema.md) before finalizing the
optimization report. Write the report in the user's language.
