# Model Code And Algorithm Optimization

Inspect the model and execution script before assuming the compiler or device kernel is
the only owner. These changes can remove work that a compiler cannot legally infer.

## Model-Script Opportunities

Use local evidence to search for:

- masks, position ids, rotary tables, constants, and preprocessing rebuilt each step
- unused attentions, hidden states, auxiliary outputs, losses, or cache results
- `.item()`, scalar Tensor branches, logging, serialization, or copies that synchronize
- repeated `.to()`, cast, transpose, contiguous, clone, or device detection
- excessive padding, unstable shapes, avoidable recompilation, or tiny batches
- Python loops and data structures that break or fragment compilation
- allocations that can be safely reused or moved outside a hot loop
- incorrect train/eval, grad/no-grad/inference-mode, or cache configuration
- tokenizer, image/audio preprocessing, collation, prefetch, and transfer bottlenecks
- optimizer, gradient zeroing, clipping, scheduler, and checkpointing overhead

Cache only values whose invalidation rules are understood. Training state, input
content, dtype, device, shape, and model configuration can invalidate apparently
constant values.

## Compiler-Friendly Refactoring

When graph evidence supports it, stabilize return structures, isolate unsupported
regions, remove Tensor-dependent Python control flow, reduce mutation/alias ambiguity,
and express computations in canonical framework operations. Do not alter masks,
broadcasting, randomness, update order, or cache semantics merely to force a match.

## Algorithm Candidate Classes

Use the semantic classes from `acceptance-gates.md`:

- Class A: behavior-preserving caching, batching, packing, and equivalent algorithms.
- Class B: mathematically equivalent kernels or reductions with floating-point reorder.
- Class C: approximate algorithms, changed decoding, reduced precision, or model change.

Default autonomous search includes A and B only when their gates exist. Class C needs
explicit approval and a task-quality contract.

## Evidence-Driven Algorithm Seeds

| Workload evidence | Candidate algorithms |
| --- | --- |
| Attention dominates with large intermediates | SDPA/Flash-style attention, exact chunking |
| Decode recomputes historical keys/values | Correct KV caching, paged cache/layout |
| Repeated prefixes dominate | Prefix cache with isolation/invalidation checks |
| Padding ratio is high | Length buckets or sequence packing with correct masks |
| Many small MoE expert GEMMs | Token grouping, grouped GEMM, expert/communication overlap |
| Embedding gather is memory-bound | EmbeddingBag/grouping, cache, sharding, layout |
| Convolution/layout conversions dominate | Appropriate convolution/layout algorithm |
| Optimizer/backward dominates | Fused optimizer, selective recompute, distributed sharding |
| Communication is exposed | Parallelism strategy, bucketization, overlap, collective choice |
| Diffusion loop dominates | Cache/reuse; scheduler or fewer steps are Class C |
| Serving has variable arrivals/shapes | Continuous batching, prefill/decode separation |
| Autoregressive decode is dominant | Speculative decoding with distribution/quality validation |

Do not assume a named implementation is equivalent. Verify masks, causal behavior,
dropout, scaling, accumulation, cache positions, distributed semantics, and training
backward behavior.

## Trigger For Algorithm Work

Enter algorithm optimization when the hotspot is on the critical path, script and
measurement problems cannot explain the remaining headroom, the current algorithm is
far enough from the attainable task objective, the candidate covers the target
distribution, and its validation contract exists.

Do not accept an algorithm from a single-batch allclose result. Exercise representative
shapes, sequence lengths, data distributions, cache states, and relevant training or
generation behavior.
