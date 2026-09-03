# Autonomous Candidate Search

Use autonomous search to discover and test opportunities that cannot be fully enumerated
in a static catalog. Search remains subordinate to the frozen workload and gates.

## Discovery Sources

Search in this order as evidence requires:

1. local model, harness, profiler output, compiler diagnostics, and existing tests
2. installed framework/backend source and version-specific configuration
3. current official documentation, release notes, upstream code, issues, and merged PRs
4. model or hardware primary implementations and papers
5. broader case studies only as hypotheses requiring local verification

When external information may have changed, verify it at search time and cite the
source in the experiment ledger. Record version, device, dtype, shape, and workload
assumptions; another system's reported speedup is not local evidence.

## Candidate Card

Every discovered candidate needs:

- hypothesis and mechanism
- source or local evidence
- owning layer and exact proposed change
- applicable versions, devices, dtypes, shapes, and modes
- estimated exposed/recoverable time and coverage
- semantic class and required gates
- compile, memory, stability, and maintenance risks
- expected experiment cost and rollback path
- origin and prior evidence when inherited from another task

Deduplicate candidates by mechanism, not wording. Rank cards for experiment priority;
do not treat expected speedup as a result.

## Freeze Search Before Execution

Freeze:

- original baseline artifact and parent relation
- immutable source snapshot and baseline fingerprint
- workload and environment manifest
- correctness and business gates
- primary metric, statistic, unit, direction, comparison formula, and target semantics
- raw result schema and profiler policy
- allowed files/change layers and precision policy
- experiment budget, budget clock, waiting policy, promotion reserve, and stop rules

The verifier must reject invalid correctness before returning a rankable performance
result. Never let a scalar score compensate for a failed gate.

## Keep Orchestration Replaceable

Define search through an orchestration-neutral interface: immutable source snapshot,
candidate patch or revision, verifier command or API, structured result, promotion
result, and accepted patch. The same contract must be runnable sequentially, manually,
or by an external search runtime. Do not make correctness, ranking, or provenance depend
on a particular product's mutable session, global draft, worker, or reviewer state.

An external runtime may add concurrency, scheduling, usage accounting, semantic review,
or final checking, but those are optional extensions. It may not weaken the frozen
verifier or become the sole source of baseline identity.

## Requalify Historical Evidence

Before generating new candidates, inspect applicable prior ledgers and implementations.
Re-run their correctness and performance gates against the current contract and
environment. Classify each as unchanged-and-revalidated, adapted, rejected, or
inapplicable. Preserve origin attribution; remeasurement does not make an inherited
mechanism a current-task discovery.

Persist accepted and rejected mechanisms in a compact, versioned ledger keyed by
backend, device class, software version, dtype, mode, shape, and mechanism. Use it to
avoid repeating known no-op, unsafe, or low-headroom directions. Historical speedups
remain hypotheses until current requalification.

## Isolate Concurrent Runs

Create each run from its declared immutable source snapshot. Keep candidate code,
compile caches, logs, and generated outputs in run-scoped locations. Do not let a
candidate write temporary files into another run's source tree or verifier staging
area. Delay shared-branch integration until active runs that depend on that snapshot
finish, or integrate only from the exact recorded best revision and reverify it.

If source or artifact provenance changes unexpectedly, classify the attempt as a
workspace-integrity failure. Recover from the recorded revision instead of treating the
candidate as a correctness or performance failure.

## Search Loop

Each candidate loop follows:

```text
read parent evidence
-> select an evidence-supported hypothesis
-> implement one primary mechanism
-> run correctness gates
-> run lightweight performance verifier
-> fully profile pattern/kernel or new-best candidates
-> accept, reject, retain as enabler, or mark inconclusive
-> persist evidence and continue or stop
```

Candidate branches may start from different high-confidence mechanisms. Preserve parent
lineage; do not combine changes from separate branches without remeasuring the combined
candidate. A worse candidate does not erase the last verifier-backed best.

Keep the execution bundle compact: frozen spec and fingerprints, relevant source,
baseline/profile summary, current parent, applicable prior mechanisms, and known
rejections. Do not repeatedly attach unrelated repository history or other models'
transcripts. Prefer deterministic extraction for routine verifier outcomes; reserve
semantic review for ambiguous failures, new best candidates, high-risk changes, and
final promotion.

Classify failed attempts as candidate, infrastructure, workspace-integrity, verifier,
budget, policy, or compiler failures. Retry only retryable classes within a bounded
retry budget. Infrastructure failures do not poison mechanism deduplication.

## Ranking And Scoring

Use pre-experiment priority to decide what to try:

```text
priority ~ recoverable_time * evidence_confidence * coverage / cost_and_risk
```

After measurement, use hard gates followed by the frozen primary metric. Use tail
latency, memory, compilation cost, and maintenance risk as constraints or tie breakers.
When a search runtime requires one finite scalar, derive it only from a gate-passing
primary metric in the declared direction; represent gate failure as candidate failure,
not a merely lower but selectable score.

## Search Stop Rules

Stop starting exploratory candidates when only the frozen promotion reserve remains.
Stop the whole search when the target is formally promoted, the active-work budget is
exhausted, the verifier or baseline becomes invalid, remaining headroom is below
uncertainty/useful gain, the user stops, or all remaining candidates require an
unapproved semantic class. Queue or device waiting counts only if the frozen budget
contract says it does. Do not stop a valid candidate loop solely because another branch
currently ranks higher.

Persist rejected hypotheses and bounded upper estimates so later agents do not repeat
low-value or unsafe experiments.
