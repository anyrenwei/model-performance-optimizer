# Acceptance Gates

Correctness is a hard constraint. Define gates before measuring candidate speed.

## Classify Semantic Risk

| Class | Meaning | Typical examples | Default policy |
| --- | --- | --- | --- |
| A | Same model semantics | caching constants, exact KV cache, equivalent batching | Allowed with correctness gates |
| B | Same mathematical intent with floating-point reordering | fusion, alternate reductions, Flash/SDPA kernels | Allowed with dtype-aware gates |
| C | Approximate or behavior-changing | reduced precision, quantization, pruning, sparse attention, fewer diffusion steps | Explicit approval and business contract |

Do not classify a candidate from its marketing label. Inspect its actual dtype,
accumulation, approximation, randomness, and output behavior.

## Gate Order

1. execution and effective-mode verification
2. finite-value and structural checks
3. deterministic/repeatability checks where expected
4. eager/reference versus candidate numerical checks
5. task or business metric checks
6. stability and resource constraints
7. performance ranking

A failed earlier gate cannot be offset by a later speedup.

## Same-Precision Compiler And Kernel Work

Hold effective parameter, input, autocast, and accumulation precision constant. Compare
representative outputs using model- and dtype-appropriate tolerances. Check output
structure, shape, dtype, device, finite values, maximum/relative errors, and relevant
task summaries.

Do not allow `equal_nan` to turn newly produced or shared NaNs into a pass. Report NaN
and Inf counts independently before any closeness comparison.

Use FP64 or FP32 references when meaningful and feasible, but record reference failure
and any fallback such as cosine similarity. A compiler-equivalence pass at FP16/BF16
does not prove that lowering from FP32 preserved business accuracy.

## Training Gates

For deterministic short-step checks, compare as applicable:

- forward outputs and component losses
- gradients for representative parameters or statistically selected leaves
- optimizer-updated parameters and optimizer state
- skipped/overflowed steps and loss-scaling behavior
- loss trajectory over fixed inputs and seeds

For an accepted training optimization, also validate the project-defined downstream
metric or convergence proxy when the blast radius warrants it. Single-batch allclose
is insufficient for a changed optimizer, precision, sampling distribution, or training
algorithm.

## Inference And Generation Gates

For deterministic inference, use fixed representative inputs and compare task outputs.
For generation, record prompts, seeds, decoding parameters, maximum lengths, stop
conditions, tokenizer version, and cache behavior. Token equality may be required for
semantics-preserving candidates; probabilistic or approximate candidates need a task
quality distribution, not one generated sample.

Serving optimizations must also preserve request isolation, ordering, cancellation,
cache invalidation, and configured latency/quality policies.

## Reduced Precision And Approximate Algorithms

Do not introduce FP16, BF16, TF32, FP8, INT8/INT4, pruning, distillation, early exit,
token dropping, approximate attention, or reduced sampling steps unless:

- the user explicitly permits that search space
- a representative validation set exists
- the baseline business metric is recorded
- the allowed regression threshold comes from the user or project
- long-tail or safety-sensitive cases are included when applicable
- the effective dtype and accumulation path are audited

Do not invent an acceptable accuracy loss. Random weights and random tensors can test
operator consistency, not model quality.

## Candidate Outcomes

- `accept`: all gates pass and the improvement is material.
- `reject`: a gate fails or a constraint regresses beyond its threshold.
- `inconclusive`: evidence cannot distinguish the candidate from noise or is incomplete.
- `enabler`: no standalone claim, but a gate-passing change is necessary to test a
  dependent candidate. Report the combined result and the enabler's neutral cost.
- `blocked`: the frozen workload has no admissible reference, compiled path, or allowed
  next step. Do not invent a relative performance result.
- `invalid`: the benchmark, verifier, effective mode, provenance, or environment
  evidence is unusable.

Persist the failing gate and evidence so autonomous search does not repeat unsafe paths.
Classify transient infrastructure and workspace-integrity failures separately from the
candidate decision so they can be retried without becoming negative model evidence.
