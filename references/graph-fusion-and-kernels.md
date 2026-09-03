# Graph, Fusion, Lowering, And Kernel Optimization

Treat compiler and backend implementation as writable optimization surfaces when the
user's scope permits it. Use profile and graph evidence to choose the owning layer.

## Correlate Evidence Across Layers

Map hot profiler regions to eager operators, FX/export graphs, compiler IR, generated
code, and device kernels. Determine whether time comes from graph boundaries, launch
overhead, intermediate memory traffic, an inefficient fused kernel, or unsupported
fallback.

Before writing a pattern, ask why an existing fusion did not match:

- graph break or unsupported operator
- noncanonical operator order or decomposition
- dtype, layout, broadcast, alias, mutation, or autograd constraint
- dynamic-shape guard or recompilation behavior
- pattern matcher coverage
- missing backend lowering or schedule
- fusion is legal but not profitable for the observed shapes

## Escalation Order

1. Enable an existing supported pass or backend option.
2. Canonicalize model code or decomposition so an existing pattern matches.
3. Add or modify a reusable pattern matcher and replacement/lowering.
4. Improve the fused lowering, tiling, scheduling, vectorization, or memory layout.
5. Implement a custom kernel only when existing compiler expression is insufficient.

Use the narrowest owning layer that solves the observed workload without duplicating a
general backend concern in every model. A model-only rewrite can be appropriate for a
unique pattern; recurring patterns across models belong closer to the compiler/backend.

## Candidate Pattern Families

Search profile-supported instances of:

- matmul plus bias, activation, residual, or normalization
- Q/K/V projection, rotary position processing, attention, and output projection
- scale, mask, softmax, dropout, and attention value multiplication
- convolution, normalization, activation, and residual chains
- residual plus normalization and normalization plus activation
- vertical pointwise chains and horizontal operations sharing inputs
- cast, transpose, reshape, contiguous, and layout-conversion chains
- optimizer update, gradient scaling, clipping, and zeroing chains
- repeated small reductions, gathers, scatters, or expert-routing operations

This is a seed list. Do not write a fusion because adjacent nodes look familiar; prove
that the region is legal, frequent, exposed, and recoverable.

## Pattern Change Contract

Record:

- source and replacement graph
- supported devices, dtypes, shapes, strides, layouts, and modes
- alias/mutation, autograd, dynamic-shape, and fallback constraints
- why the previous graph missed or performed poorly
- expected eliminated launches, bytes, or compute
- negative cases that must not match

Add focused tests for positive matching, negative matching, numerical behavior,
representative shapes/dtypes, dynamic shapes when supported, forward/backward paths,
fallback, and recompilation behavior.

## Kernel Change Contract

For a lowering, schedule, or custom kernel, compare against a trusted reference and
record tile choices, parallelism/occupancy, vectorization, accumulation dtype, layout,
temporary memory, shape coverage, and fallback. Benchmark both isolated kernel behavior
and task-level E2E; an isolated microbenchmark win can lose after integration.

## Acceptance Evidence

Kernel count reduction alone is not success. Measure:

- unprofiled task metric
- device compute and critical-path time
- launch count and exposed gaps
- intermediate memory traffic and copies
- peak and temporary memory
- compile/recompile cost
- numerical and business gates
- coverage over target shape/dtype distribution

Reject a fusion that reduces graph nodes without materially improving the frozen task
objective, unless it is explicitly retained as a neutral enabler for a dependent,
measured candidate.

For DVM-specific graph capture, ExternKernel, registration, and code-generation
diagnosis, use `ascend-npu-dvm.md` in addition to this generic protocol.
