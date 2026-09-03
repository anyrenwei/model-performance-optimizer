# Benchmark Protocol

Use this protocol to create, audit, or repair measurements before optimization.

## Separate Run Purposes

Do not overload one run with conflicting purposes:

| Run | Purpose | Profiler |
| --- | --- | --- |
| Correctness | Outputs, loss, gradients, task metrics | Off unless diagnosing |
| Steady state | Official latency, throughput, and memory | Off |
| Profiling | Critical-path and resource attribution | On |
| Cold start | Initialization, first compile, and cache creation | Off |

Profiled latency is evidence about causes, not the official unprofiled E2E result.

## Freeze Workload Equivalence

Record and hold constant unless the experiment explicitly targets one field:

- dataset snapshot or deterministic generated input
- preprocessing, tokenizer, prompt template, sampling, and postprocessing
- batch or microbatch size, sequence lengths, image sizes, and shape distribution
- model checkpoint, mode, train/eval state, trainable parameters, and optimizer state
- dtype, autocast, TF32, quantization, accumulation dtype, and backend flags
- random seeds and deterministic settings
- cache state: cold, warm, persistent compile cache, KV cache, and prefix cache
- device identity, topology, clocks/power policy when observable, and software versions

Compare task-equivalent workloads. A faster result from fewer tokens, more padding,
different outputs, or changed trainable parameters is not the same experiment.

## Time Asynchronous Devices Correctly

Select synchronization from the actual configured device, not from whichever runtime
is available first. Synchronize immediately before and after the measured region:

```python
device_sync()
start = time.perf_counter()
result = run_workload()
device_sync()
elapsed = time.perf_counter() - start
```

Do not compute elapsed time before the final synchronization. Keep logging,
serialization, profiler stepping, correctness comparison, and unrelated data loading
outside the timed region unless the user's objective explicitly includes them.

For training, state whether step time covers data loading, host-to-device transfer,
forward, backward, collective communication, optimizer update, and scheduler update.
For serving, define request arrival, queueing, tokenization, time-to-first-token,
inter-token latency, and end-to-end boundaries separately.

## Warmup By State, Not A Magic Count

Warmup is complete only when the target state is ready:

- compilation and intended recompilation have completed
- allocator and target caches have stabilized
- latency over a recent window is stable enough for the requested confidence
- memory is not monotonically growing
- the measured graph and shapes represent the target workload

A fixed warmup count can be a starting point, not proof. Record discarded samples and
the stability rule. Measure cold-start and compile costs separately when they matter.

## Preserve A Distribution

Store raw samples and report at least the primary central and tail metrics appropriate
to the task. Median plus P90/P99 is usually more informative than a single mean.
Repeat across processes or runs when process startup, compilation, thermal state, or
shared-system noise can affect the conclusion.

Choose sample counts from observed variance and the required decision confidence. A
candidate improvement is inconclusive when its confidence interval overlaps the
baseline beyond the configured useful-gain threshold.

For throughput, report work units precisely: samples/s, tokens/s, generated tokens/s,
or training tokens/s. Do not mix latency per batch with latency per sample.

## Pair Comparisons On Shared Accelerators

On a shared accelerator host, acquire an exclusive device lease before loading the
model or creating compile artifacts. Inspect device occupancy before and after the run,
record the physical device identity, and treat unexpected external occupancy as an
infrastructure failure rather than candidate evidence. A scheduler-visible idle device
is not sufficient when another process can allocate it concurrently.

Measure baseline, candidate-local no-op, and promising candidates on the same physical
device. Interleave them when practical so thermal state and background load affect both
sides. If promotion must move to another device or materially different time window,
remeasure the admissible baseline there; do not reuse a cross-device ratio as if it were
paired evidence.

Use a bounded lease and retry policy. Release leases after subprocess failure, and
quarantine samples collected during contamination or occupancy changes. Keep queue and
device-wait time separate from active benchmark time.

## Promote With A Noise Margin

Use a lightweight sample count for search only after a frozen full baseline exists.
Include a no-op replay to estimate local drift. Short-run winners are provisional and
must not satisfy the final target by themselves.

Reserve time for an independent longer promotion. For close decisions, use paired
samples or a suitable resampling/confidence-interval method and require the lower
confidence bound to clear the configured useful-gain threshold. A point estimate that
barely crosses the target is inconclusive when normal drift can reverse it. Persist
both the provisional and promotion results instead of replacing one with the other.

## Prevent False Modes

- Verify that command-line arguments reach the Python entrypoint.
- Record requested mode and effective mode separately.
- Treat compile fallback as failure, even if eager execution succeeds.
- Verify the selected backend, device, precision, and compiled graph count from runtime
  evidence where possible.
- Keep profiler and debug flags out of official E2E runs.
- Do not compare runs with different cache policies without labeling them.

Run backends whose registration depends on import-time environment in separate fresh
processes. Set backend variables before importing the accelerator integration, and
verify generated code or runtime diagnostics rather than trusting the requested flag.

Use an explicit cache policy:

- Cold-cache runs measure initialization and compilation.
- Warm-cache runs measure the intended reuse state.
- Multi-backend comparisons use isolated cache namespaces or safely cleared, explicitly
  resolved per-run cache directories to prevent cross-backend artifact reuse.

Do not hardcode another user's cache path or broadly delete `/tmp`. Resolve and record
the exact cache directory. Cache isolation is not a reason to clear caches between
samples of one declared warm-cache run.

## Dynamic Shapes

Define a representative shape distribution. Before steady-state measurement, exercise
every intended shape bucket needed to trigger expected guards, compilation, and cache
creation. Record compiled graph and recompile counts. Do not describe per-shape static
specialization as one dynamic graph.

For dynamic workloads, report per-shape and aggregate task metrics. Averages can hide a
backend that recompiles or regresses only on an important tail shape.

## Profiler Schema Robustness

Profiler export formats and column positions can vary by framework/backend version.
Inspect headers and parse fields by name with a structured parser where possible. Keep
device compute, vector/cube/core type, memory/copy, communication, and host timing
separate. A summed operator table can double count overlap and must not replace critical
path or unprofiled E2E evidence.

## Baseline Invalidity Checklist

Invalidate the baseline when any material issue remains:

- missing device synchronization around asynchronous work
- different inputs, shapes, effective dtypes, outputs, or trainable parameters
- compilation or cache creation mixed into only one steady-state side
- profiler overhead present on only one side
- silent fallback or requested/effective mode mismatch
- empty or undersized post-warmup samples
- no correctness or business acceptance evidence required by the candidate class
- only summarized metrics with no recoverable raw evidence for a disputed result
- baseline and candidate taken from unmatched devices or contaminated occupancy without
  a paired rebaseline
- source, harness, contract, or environment fingerprint changed after the baseline

Repair the harness, restart the baseline, invalidate its candidate descendants, and
clearly retire earlier claims.
