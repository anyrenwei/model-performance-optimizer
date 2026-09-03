# Portfolio Optimization Protocol

Use this protocol when one task covers several models, workloads, shapes, backends, or
deployment variants. The portfolio is a collection of independently auditable
experiments, not one large benchmark.

## Partition By Contract

Give every item an immutable contract fingerprint derived from at least:

- model and workload identity, mode, checkpoint, input source, batch, and shapes
- effective precision, accumulation policy, backend, device class, and cache policy
- benchmark and correctness protocol versions
- allowed change layers, specialization policy, primary metric, formula, and target

Aggregate performance only within the same contract cohort. Training and inference,
strict FP32 and reduced precision, or general and fixed-shape-specialized models are
different cohorts even when they share a model name. Report cross-cohort results as
separate outcomes, not wins or losses.

## Track An Item State Machine

Persist one state per item rather than inferring completion from files or a global
task state:

```text
unstarted -> baseline_running
baseline_running -> baseline_valid | baseline_blocked | baseline_invalid
baseline_valid -> requalifying | searching
requalifying | searching -> promotion_pending | retained_baseline | exhausted
promotion_pending -> accepted | valid_below_target | retained_baseline
```

An item is formally accepted only after its configured promotion gate. A short search
result is provisional. A portfolio may finish with a mix of terminal item states;
report that mix instead of treating every item as successful or the whole campaign as
failed.

Use distinct attempt failure classes:

- `candidate_failure`: the proposed mechanism fails a semantic or performance gate
- `infrastructure_failure`: device contention, startup failure, or transient service
  failure; retry only within its retry budget
- `workspace_integrity_failure`: source or artifact provenance does not match
- `verifier_failure`: the measurement implementation is invalid
- `budget_timeout`: the configured item budget is exhausted
- `policy_blocked`: the next step requires an unapproved behavior or change layer
- `compiler_blocked`: the frozen workload has no admissible compiled path

Infrastructure and integrity failures are not negative performance evidence and must
not poison candidate deduplication.

## Define Budget Clocks

Do not start every item's deadline when a batch is created. Freeze these fields:

```yaml
budget:
  baseline_budget_seconds: 1200
  optimization_budget_seconds: 7200
  clock: active_item_wall
  queue_wait_counts: false
  device_wait_counts: false
  infrastructure_retry_seconds: 600
  promotion_reserve_seconds: 1200
```

The numbers are illustrative. Derive them from startup, compile, verifier, and
promotion costs for the current workload.

`active_item_wall` is the union of intervals in which an optimizer or verifier is
actively working on that item; overlapping work counts once. Record agent work,
verifier execution, device waiting, queue waiting, and natural elapsed time separately.
If another clock is chosen, name it precisely.

Activate an item's optimization clock only when it receives an execution slot. Stop
starting exploratory candidates when the remaining budget equals the promotion
reserve. Do not let queued items consume their budget before search begins.

## Triage Before Broad Search

Run a cheap first pass across the portfolio:

1. establish or validate eager and current-compile references
2. classify compile blockers separately from performance opportunities
3. inspect prior ledgers and source for applicable accepted and rejected mechanisms
4. requalify inherited candidates against the current contract and environment
5. estimate headroom and apply low-risk, high-confidence recipes
6. allocate autonomous search to unresolved items with material headroom

Stop early for an item that reaches the target or has bounded remaining headroom.
Reallocate available execution slots to other items without changing their individual
caps. Reserve device and verifier capacity for promotion so provisional winners do not
expire in a queue.

## Reuse Evidence Without Inheriting Claims

Historical implementations and ledgers are candidate seeds, not current results.
Record their origin and current disposition:

```yaml
provenance:
  origin: inherited | adapted | discovered_current_task
  source_snapshot: <content hash>
  prior_evidence: <artifact reference or null>
  changed_in_current_task: true
  revalidated_current_contract: true
```

Re-run current correctness and performance gates before accepting inherited code.
Report how many final implementations were unchanged-and-revalidated, modified, newly
discovered, rejected, or retained as the current compile baseline. Do not attribute an
inherited speedup to the current search merely because it was remeasured there.

When the same mechanism succeeds across multiple models, evaluate whether it belongs
in a shared harness, compiler pass, or backend implementation. Validate the shared
form against positive and negative models before replacing model-local versions.

## Handle Compile Enablement Separately

Keep two objectives distinct:

- performance improvement relative to an admissible current compile baseline
- compile coverage for a workload with no current compile baseline

A semantics-preserving compile enabler may report correctness, eager latency, and its
absolute compiled latency, but it has no relative compile speedup. A fixed-shape or
model-source specialization must be permitted by the contract and must state its
generalization boundary. Precision-changing enablement belongs to a new contract
cohort.

Compiler blockers and enablers do not enter the denominator for relative performance
statistics. Report compile coverage as a separate KPI.

## Aggregate Without Hiding Failures

For every contract cohort report:

- listed, baseline-valid, promoted, target-met, below-target, retained, exhausted, and
  blocked counts
- target-met rate with an explicit denominator
- median normalized improvement and geometric mean speedup across admissible results
- per-item eager, current compile, and tuned compile metrics when available
- baseline drift or paired-control diagnostics across repeated campaigns
- provisional results separately from formal promotions

Do not sum unrelated model latencies. Do not aggregate ambiguous `improvement_pct`
values; name and compute the frozen formula. Include both normalized change and
absolute final latency when comparing campaigns because baseline drift can reverse the
apparent winner.

## Account For Resources

When the runtime exposes usage, record per item and in aggregate:

- active portfolio wall time as an interval union
- active item time under the declared budget clock
- optimizer, candidate-search, verifier, and semantic-review activity
- device, queue, and infrastructure waiting time
- token and cost totals with role and coverage
- whether cost is recorded, estimated, a lower bound, or missing

Do not add overlapping activity durations and call the sum wall time. Do not report a
missing cost as zero. Keep orchestration-specific identifiers in optional extensions;
the optimization result schema must remain usable by sequential, manual, or external
search execution.

## Final Consistency Audit

For a broad portfolio or a shared compiler/backend change, perform an independent
consistency audit when available and proportionate to risk. Check formulas, cohort
membership, denominators, workload and precision equality, baseline fingerprints,
source provenance, item budgets, promotion status, and raw artifact links. The audit
validates the report and evidence; it does not replace the acceptance gates.
