# Ascend NPU And DVM Protocol

Use this reference for Ascend NPU multi-backend measurements and DVM graph/fusion work.
Backend internals are version-sensitive: inspect the installed `torch_npu` source and
generated artifacts before relying on a file name, API list, or historical workaround.

## Prove The Effective Backend

DVM backend registration can occur when `torch_npu` is imported. Set the backend before
Python imports the integration:

```bash
TORCHINDUCTOR_NPU_BACKEND=dvm python model.py
```

Do not assume that changing an Inductor config after import changes the registered
scheduler. Use a fresh process per backend. In a diagnostic run, enable compile debug
output and verify DVM-generated functions or the current version's equivalent signature
in generated code. Keep debug mode off for official E2E.

For multi-backend comparisons:

- freeze identical workload, precision, dynamic-shape, and cache policies
- isolate backend processes and compile caches
- record requested and verified effective backend
- treat backend fallback as failure for that backend
- verify any backend-specific compile-thread requirement against the installed version

Cross-backend cache reuse can make one backend execute another backend's artifact. Use
explicit per-run cache namespaces when supported, or safely clear only the exact resolved
cache directory before a declared cold/backend-isolation run. Do not clear caches within
the steady-state samples of a warm-cache experiment.

## Phase 1: Maximize Graph Capture

First determine how many Dynamo/compile subgraphs exist and why boundaries occur. Use
graph-break logs, compile debug directories, readable FX graphs, and graph/recompile
counters. Record the source line and mechanism for every material break.

Common causes include:

- `.item()`, `.numpy()`, printing, or other device-to-host synchronization
- Tensor-dependent Python control flow
- data-dependent slicing or shape construction
- dynamic Python containers and unsupported third-party calls
- mutation, alias, custom operators, or unsupported autograd behavior

Prefer a semantics-preserving Tensor/batched rewrite when it actually reduces exposed
time. Scalar-capture flags or fixed shapes are candidates only when their guards,
dynamic behavior, and workload coverage are valid. Accept a graph break when removing it
would execute substantially more work or change behavior.

Re-run graph diagnostics after each accepted capture change and report subgraph and
recompile deltas. No fusion can cross a remaining compiled subgraph boundary.

## Phase 2: Maximize In-Graph DVM Coverage

For each hot captured subgraph, triangulate:

1. readable FX graph for op context, dtype, shape, layout, and neighbors
2. pre-fusion IR for scheduler nodes and ExternKernel boundaries
3. generated code for DVM fusion, external calls, and backend fallback
4. NPU profiler output for critical-path and operator cost

Names such as `fx_graph_readable.py`, `ir_pre_fusion.txt`, `output_code.py`,
`dvm_fused`, `ExternKernelSchedulerNode`, or `import_fx` are useful known signatures,
not permanent API contracts. Discover the current equivalents in the generated debug
tree.

Group ExternKernels by operator type and root cause, then prioritize exposed total time
and blocked fusion chains rather than raw count. A cheap view/cat boundary may be less
valuable than one repeated reduction blocking a large chain.

## Diagnose An ExternKernel

Use this root-cause ladder against the installed source:

```text
Did the registration/rule reject the observed dtype, shape, or layout?
  -> fix an overly narrow rule only when the operation is legal

Is the op absent from the scheduler's DVM generation/fusion candidate set?
  -> add it only when emitter and semantic support exist

Is registration or an emitter/lowering missing or incorrect?
  -> implement and test the mapping and fallback

Is the op composite but decomposable into supported DVM operations?
  -> add a numerically sound decomposition and validate the fused result

Does the operation require another engine, unsupported data construction, or opaque
custom code?
  -> keep the boundary or implement the correct owning backend support
```

Known source areas in some `torch_npu` versions include DVM op emitters/rules,
scheduler fusion candidate lists, graph builders, and decomposition registries under
`torch_npu/_inductor/dvm/`. Search the installed tree instead of assuming exact paths.
Inspect the runtime DVM Kernel API rather than copying a stale method list.

Matrix multiplication may belong to a cube/matrix engine while DVM targets vector work;
that boundary is not automatically a bug. Optimize heterogeneous engine overlap and
fusion legality rather than forcing every operator into DVM. Do not switch precision to
make an op eligible unless the separate precision contract permits it.

## Dynamic-Shape DVM Work

Warm every representative target shape outside the measurement region, then verify
graph and recompile counts. Test the same shape sequence across eager and all backends.
Inspect whether a first-seen shape biases generated schedules or causes later fallback.

Measure per-shape latency and aggregate by the frozen production distribution. A DVM
dynamic-shape claim requires one compatible compiled strategy or clearly reported
specializations, not compilation mixed into timed iterations.

## NPU Profiling

Use `torch_npu.profiler` in a separate profiling run after warmup. Export the richest
stable format supported by the installed version, but parse tables by header names
because column order can change. Separate AI Core/cube, AI Vector Core, memory/copy,
communication, and host costs.

Check that the active profiler window excludes compilation and intended cache creation.
Outlier maximum operator times or summed operator time exceeding plausible step time can
indicate compilation capture, repeated windows, or overlap/double counting. Re-profile
before making an optimization claim.

## Run A Profile-Triggered DVM Tuning Loop

Do not treat DVM tuning as a flat environment-variable sweep. Classify the exposed
cost first, then search only variables that can change that cost. Keep graph-boundary,
fusion-boundary, kernel-codegen, compiler-autotune, and runtime-wrapper experiments as
separate mechanisms so their gains remain attributable.

Normalize each steady-state profile into at least:

- task E2E and device critical-path span
- exposed host time and inter-kernel gaps
- launch count and short-kernel distribution
- hot DVM kernels, hot ACLNN/external kernels, and their call counts
- layout copies, transpose/contiguous work, and allocator activity
- compiled graph, fallback, dynamic-shape, and recompile counts

Use raw trace events when aggregate tables cannot distinguish serialized gaps from
overlapped host work. Re-profile the current best after a material fusion or lowering
change because the next bottleneck may move.

### Reuse Versioned Empirical Seeds

Inspect current local ledgers before broad flag or model-code search. Store successful
and rejected mechanisms with the NPU model, `torch_npu`/PyTorch/CANN/DVM versions,
dtype, mode, shape, graph signature, and measured evidence. Requalify them against the
current contract; do not treat a historical speedup as portable proof.

Repeatedly useful DVM candidate families include:

| Evidence | High-priority seed | Required guard |
| --- | --- | --- |
| Frozen eval convolution followed by normalization | Fold normalization into convolution; remove provably fixed pooling | Eval-only state, frozen parameters, full-output tolerance |
| Attention dominated by external calls, masks, or layout copies | Active-subclass BSND attention, QKV packing, compact masks, matmul fusion | Exact mask/causal/dropout semantics and FP32/HF32 audit |
| Dense connectivity with repeated prefix concatenation | Cumulative or native cat, pooled transitions, layout propagation | Shape/stride coverage and reordered-FP correctness class |
| Many small biased projections or poor DVM matmul integration | Compare native NPU Linear/grouped matmul with DVM template | Wrapper, allocation, launch, and full E2E accounting |
| Fixed output spatial reduction | Equivalent spatial mean or fixed-pool elimination | Frozen shape or a guarded general implementation |
| Layout-copy or allocation pressure | Metadata permutation, view policy, or buffer reuse | Artifact delta, alias safety, and per-model promotion |

These are hypotheses, not defaults. In particular, buffer reuse, metadata permutation,
view policies, and native-operation boundaries can be wins, no-ops, or regressions on
different graphs. The artifact-effectiveness and unprofiled E2E gates still decide.

When one mechanism succeeds across several models, consider moving it into a shared
harness, canonicalization, pattern, or backend lowering. Validate negative models so a
portfolio-level rollout does not broaden matching unsafely.

### Map Evidence To Candidate Families

| Profile or artifact evidence | Search next | Avoid assuming |
| --- | --- | --- |
| Exposed host gaps or many repeated short kernels | graph capture, vertical/horizontal fusion, fusion-size policy, direct DVM lowering, output preallocation | a lower summed device time will improve E2E |
| Long fused kernel with low compute and bandwidth utilization | shrink or split the fusion, change reduction edges, kernel type, tiling, layout, or lowering | larger fusion is always better |
| Hot ACLNN/external operator blocks a large DVM chain | supported decomposition, DVM emitter/lowering, canonicalization, or a native DVM template | every external boundary should be decomposed |
| Hot DVM primitive chain is slower than a native operator | retain/native fallback, a different decomposition, or a specialized lowering | decomposition is always faster because it enables fusion |
| Repeated transpose, contiguous, copy, or as-strided work | view-load policy, metadata permutation, transpose propagation, layout specialization | toggling a view flag changes arbitrary external-op boundaries |
| Dynamic tiling overhead, recompiles, or shape-dependent regressions | static specialization, representative shape buckets, dynamic-kernel policy | one shape's winner generalizes to the production distribution |
| Device kernel improves but E2E is flat or worse | wrapper dispatch, `torch.ops` fallback, allocation, tuple outputs, synchronization, and launch count | a named fused/custom op is an integrated fusion win |

Ratios or thresholds are workload-specific. Rank candidates by exposed recoverable
milliseconds, evidence confidence, workload coverage, and uncertainty rather than by
operator count alone.

### Search DVM Variables By Layer

Discover the controls and legal values in the installed `torch_npu`, PyTorch Inductor,
and compiler versions. The names below are known examples, not a stable API.

1. **Graph capture and specialization:** static versus dynamic shapes, representative
   shape buckets, scalar capture/specialization, graph breaks, input layout guards, and
   recompile policy. Optimize capture before asking fusion to cross a graph boundary.
2. **Fusion boundaries:** view-load level, metadata permutation, vertical and horizontal
   fusion, pre-reduction and post-reduction edges, maximum fusion size, aggressive
   fusion, and any available DVM graph-partition pass. Prefer per-edge profitability to
   a global reduce-fusion switch when backend changes are allowed. Known control seeds
   include `view_fusion_level={0,1,2}`, `metadata_permute`,
   `disable_post_reduce_fusion`, Inductor `max_fusion_size`, and
   `aggressive_fusion`.
3. **Matmul templates:** template enablement, pointwise epilogue depth, legal broadcast,
   bias canonicalization, transpose propagation, and prologue support. Compare the DVM
   template with the native matrix-engine path for the actual shapes; do not force all
   matrix work into vector fusion. Check `enable_matmul_fusion` together with Inductor
   `epilogue_fusion`; either control can prevent the intended epilogue fusion.
4. **Decomposition and lowering:** choose per hot operator among a DVM-supported
   decomposition, a native DVM lowering/template, an ACLNN/external fallback, or a
   specialized composite lowering. Typical decision points include normalization,
   activation, softmax, reduction, and layout-sensitive operators.
5. **Kernel form:** legal DVM kernel types such as vector, specialized-reduction,
   graph-split, or mixed matrix/vector forms; reduction unroll/split policy; static or
   dynamic codegen. Treat forced defaults as compatibility constraints until focused
   tests prove alternative modes legal.
6. **Compiler kernel tuning:** confirm whether tiling, operation reorder, and automatic
   multi-buffer choices are already autotuned. Extend search to block/grid dimensions
   or tuning ranges only for attributable hot kernels, and keep compile and launch
   parameters consistent. In known versions `AUTOTUNE` controls an inner search over
   tiling size, operation reorder, and automatic multi-buffer; inspect the installed
   compiler rather than duplicating a presumed grid.
7. **Memory and runtime integration:** buffer reuse, memory planning, in-place or donated
   outputs, contiguous materialization, preallocated output buffers, direct kernel
   launch, Python/custom-op dispatch, and dynamic tiling host work. Known DVM versions
   expose `allow_buffer_reuse`, while alignment assertions are diagnostic rather than a
   positive optimization candidate.

Some versions expose these as Python config fields, some as environment-backed fields,
and some only through a workload harness or source patch. Resolve the actual setting
site and apply it before importing or registering the DVM backend. Use a fresh process
for every candidate whose configuration is consumed at import time.

Generic Inductor controls such as epilogue fusion, maximum fusion size, or aggressive
fusion count only when the installed DVM scheduler inherits and observes them. Prove
that by inspecting generated artifacts instead of trusting the requested config value.

### Apply An Artifact-Effectiveness Gate

Run each compiler configuration in a fresh process with an isolated or explicitly
declared cache policy. Before spending the long-run benchmark budget, compare the
candidate with its parent using the current version's equivalents of:

- FX/readable graph and compiled-subgraph count
- pre-fusion scheduler IR and fusion groups
- generated wrapper and DVM source
- DVM/external kernel signatures and counts
- fallback and contiguous-materialization sites

Hash or otherwise normalize generated artifacts when practical. If the target graph,
fusion signature, wrapper path, and relevant kernel set are unchanged, mark the
candidate `no-op` and stop it. A changed flag value is not evidence that a pass ran.
When artifacts change, verify that the intended mechanism changed rather than an
unrelated cache, shape, or fallback path.

Change one primary mechanism at a time. After selecting a winner from one layer,
re-profile it before composing a second layer. Confirm small gains with more samples
on the same device because compiler autotuning and shared-device noise can reverse
sub-percent results.

### Evaluate Decomposition And Custom Fusion End To End

For a decomposition, native fallback, pattern pass, or custom operator, estimate:

```text
net_recoverable_ms = eliminated_device_ms
                     - added_device_ms
                     - added_exposed_host_ms
                     - added_allocation_or_copy_ms
```

A native large operator may add a kernel while splitting an existing DVM reduction
chain. Conversely, a primitive decomposition may reduce boundaries but produce an
oversized, inefficient kernel. Use the profile and generated fusion groups to decide;
do not rank either representation by kernel count alone.

A named composite/custom operator is not automatically a fusion. Verify:

- whether the parent primitives were already one DVM kernel
- whether launch count actually falls
- whether generated code uses a direct DVM launcher with preallocated outputs or a
  `torch.ops`/external dispatcher path
- whether tuple outputs, allocation, synchronization, or layout conversion were added
- whether isolated device-kernel savings survive unprofiled task E2E

Prefer a real backend lowering or the backend's direct graph-codegen mechanism over a
Python monkey patch that only changes FX names. Retain a manual pattern only when it
closes a demonstrated fusion boundary, selects a measurably better schedule, or enables
a dependent measured optimization. Include negative pattern cases so unrelated graphs
cannot match.

### Treat Compiler Autotuning As A Nested Search

Separate outer graph/fusion search from inner kernel compilation tuning. Inspect the
installed compiler to learn whether it already searches tiling, operation reorder,
multi-buffer, or similar parameters. Do not duplicate that grid in the outer search.

Verify that autotuning is enabled when steady-state kernel performance is the objective,
that the selected best-kernel cache belongs to the current graph and device, and that
determinism or fallback did not silently disable tuning. If the built-in tuner uses a
small warmup or repetition count, re-benchmark the winning configurations of important
hot kernels before accepting close results. Report compile latency and cache footprint
separately from warm steady-state latency.

Source-level search over block dimension, kernel type, unroll threshold, or tuning range
needs focused correctness and compiler-stability tests. Do not expose an arbitrary
global switch until the legal shape, dtype, layout, dynamic-shape, and hardware ranges
are known.

### Exclude Diagnostic And Contract-Changing Controls

Do not rank debug dumps, alignment assertions, forced fallback modes, compile-process
parallelism, or error-diagnostic switches as warm steady-state optimizations. They may
be useful to collect evidence or alter cold compile time, but belong in a separate
measurement objective. Treat compiler optimization-off modes as ablations, not likely
deployment candidates.

Precision promotion, BF16 retention, reduced precision, approximate formulas, or model
rewrites belong to a separate approved precision or behavior contract. They must not
enter a strict same-precision DVM search merely because they make a fusion legal.

### Record DVM Candidate Evidence

For every candidate, add these DVM-specific fields to the experiment ledger:

```yaml
trigger: exposed-host-gap | fragmented-kernels | anomalous-fused-kernel | layout-copy | external-hotspot | dynamic-shape
requested_variables: {}
effective_backend: dvm
artifact_delta:
  graph: changed | unchanged
  fusion_signature: changed | unchanged
  wrapper_path: direct-dvm | torch-ops | external | unchanged
  dvm_kernel_delta: 0
  external_kernel_delta: 0
profile_delta:
  device_critical_path_ms: 0.0
  exposed_host_ms: 0.0
  launch_count: 0
  layout_copy_ms: 0.0
task_delta:
  steady_state_metric: 0.0
  compile_time_s: 0.0
  peak_memory_bytes: 0
decision: accept | reject | no-op | inconclusive | enabler
```

The artifact delta explains whether the intended DVM mechanism ran; the profile delta
explains why performance moved; only the unprofiled task metric and acceptance gates
decide whether the candidate is retained.

## DVM Change Verification

After every DVM rule, candidate-list, emitter, decomposition, or pattern change:

1. run focused numerical and pattern tests
2. regenerate FX, pre-fusion IR, and generated code
3. confirm the target boundary or fallback changed as intended
4. record DVM fused-kernel and ExternKernel deltas as diagnostics
5. run full NPU profiling for critical-path attribution
6. run unprofiled steady-state E2E and task acceptance gates

Stop the DVM route when remaining boundaries are legal/irreducible, their recoverable
time is below uncertainty, or changes no longer improve the frozen task objective.
