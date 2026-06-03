# Final Corrected Source Issue Mapping and Final Correct Implementation Order

Generated: 2026-06-03  
 
Primary source file: `29-april-omni-calc-parallelisation-FINAL.md`  


## Purpose

This file is the corrected final grouped issue roadmap for the Omni-Calc Rust performance and parallelisation work. It consolidates all source issues from `29-april-omni-calc-parallelisation-FINAL.md` into a single corrected grouped issue set.


## Code Areas Reviewed

```text
modelAPI/omni-calc/src/python.rs
modelAPI/omni-calc/src/engine/mod.rs
modelAPI/omni-calc/src/engine/exec/executor.rs
modelAPI/omni-calc/src/engine/exec/context.rs
modelAPI/omni-calc/src/engine/exec/state.rs
modelAPI/omni-calc/src/engine/exec/preload.rs
modelAPI/omni-calc/src/engine/exec/perf.rs
modelAPI/omni-calc/src/engine/exec/formula_eval.rs
modelAPI/omni-calc/src/engine/exec/actuals.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
modelAPI/omni-calc/src/engine/integration/calc_plan.rs
modelAPI/omni-calc/Cargo.toml
modelAPI/omni-calc/benches/*
```

## Code-Backed Findings

- The executor still processes `plan.request.calc_steps` serially and mutates one shared `ExecutionContext`.
- `ExecutionContext` still owns mutable `calc_object_states`, mutable property caches, warnings, and a mutable `CrossObjectResolver`.
- `CalcObjectState` still stores columns as `Vec<(String, Vec<T>)>` and uses linear lookup helpers for number, string, and connected-dimension columns.
- `PreloadedMetadata` exists, but it is stored as `Option<PreloadedMetadata>` on `Engine` and passed as `Option<&PreloadedMetadata>` to `ExecutionContext`; it is not yet a worker-safe shared snapshot such as `Arc<PreloadedMetadata>`.
- `python.rs` performs explicit preload and then calls Rust execution inside `py.allow_threads`, which supports the global rule that worker execution must not call Python.
- `FormulaEvaluator::with_raw_properties()` still clones `EvalContext` and `ActualsContext`, so formula-context clone reduction is a real hot-path issue.
- `CrossObjectResolver` stores `RecordBatch` snapshots and repeatedly extracts/aligns string-keyed dimension and indicator columns.
- `join_path.rs` builds row join keys as pipe-joined `String` values, and `lookup.rs` aggregates with `HashMap<String, f64>`.
- `actuals.rs` repeatedly builds row/entity keys and forecast membership information and still contains audit/debug `eprintln!` paths.
- `perf.rs`, `Cargo.toml`, and `benches/*` show that perf counters, Criterion benches, Rayon, and bench utilities already exist. The profiling issue should therefore complete and formalize baseline/regression gates, not claim instrumentation is entirely absent.

## Global Rule For All Final Issues

After the explicit preload phase, Omni-Calc Rust execution must use Rust-owned data only.

Allowed PyO3 usage:

```text
Python binding boundary
CalcPlan extraction
explicit preload before Rust execution
returning the result back to Python
```

Not allowed inside Rust execution hot paths:

```text
Python<'_>
PyObject
metadata_cache.call_method1(...)
metadata_cache.getattr(...)
lazy metadata callback from node execution
lazy metadata callback from formula evaluation
lazy metadata callback from resolver or join logic
lazy metadata callback from Rayon workers
```

Correct execution direction:

```text
Python boundary
  -> preload required metadata into Rust-owned PreloadedMetadata
  -> run Rust execution under py.allow_threads
  -> use ExecutionSnapshot, PreloadedMetadata, PropertyCacheSnapshot, and shared columns
  -> do not call Python from executor hot paths
```

## Source Issue Inventory

| Source Issue | Correct Title / Requirement | Notes |
| --- | --- | --- |
| 1 | Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation | Core serial scheduler foundation. |
| 2 | Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes | Early ready-node parallelism plan. |
| 3 | Parallelize Final RecordBatch Materialization Across Blocks | Safe post-processing parallelism. |
| 4 | Parallelize Connected Dimension Preload Across Blocks | Safe pre-processing parallelism. |
| 5 | Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins | Cross-object join performance. |
| 6 | Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions | Cross-object aggregation performance. |
| 7 | Optimize CalcObjectState Column Storage and Lookup With Ordered Indexed ColumnStore | Indexed ordered column storage. |
| 8 | Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Preloaded Metadata | Clone and shared-data reduction. |
| 9 | Optimize Join Keys and Avoid Repeated String Join Path Allocation | Present in source coverage/final notes; group with cross-object join work. |
| 10 | Expand and Normalize PreloadedMetadata for Worker-Safe Execution | Worker-safe preload foundation. |
| 11 | Introduce Snapshot-Friendly Shared Column Storage Using Arc on Top of ColumnStore | Shared column storage. |
| 12 | Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag | Later ready-layer parallel execution. |
| 13 | Replace Selected Hot-Path String Lookups With Interned IDs | Data-structure/hot-path lookup issue, not benchmarking. |
| 14 | Add Dependency-Aware Resolver Snapshot Refresh | Scheduler/resolver issue, not debug cleanup. |
| 15 | Batch NodeOutput Merge Operations | Scheduler/merge issue, not a vague standalone placeholder. |
| 16 | Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver | Benchmarking/profiling issue. |
| 17 | Add Pre-Execution Validation for Parallel-Safety | Scheduler safety issue, not a vague standalone placeholder. |
| 18 | Optimize Formula Dependency Parsing and Graph Build | Scheduler/dependency graph issue, not observability. |
| 19 | Optimize FormulaEvaluator Context Cloning and Shared Column Access | Formula hot-path clone reduction. |
| 20 | Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse | Actuals/forecast/row key optimization. |
| 21 | Introduce Structured Node, Block, Dimension, Property, and Column IDs to Reduce String Parsing and Lookup Cost | Typed ID foundation. |

## Final Corrected Source Issue Mapping

| Source Issue | Final Issue | Reason |
| --- | --- | --- |
| 1 | Final Issue 3 | Kahn-style serial scheduler foundation belongs with execution graph, snapshots, merge, validation, resolver refresh, and dependency graph build. |
| 2 | Final Issue 7 | Ready-node parallel execution belongs in the final Rayon execution issue. |
| 3 | Final Issue 6 | Final RecordBatch materialization is safe post-execution parallelism. |
| 4 | Final Issue 6 | Connected dimension preload is safe pre-execution parallelism. |
| 5 | Final Issue 5 | Join-path creation and target alignment are cross-object join concerns. |
| 6 | Final Issue 5 | Lookup aggregation is part of cross-object join/lookup optimization. |
| 7 | Final Issue 2 | ColumnStore is the base data-structure optimization. |
| 8 | Final Issue 2 | Shared data and clone reduction depend on the data-structure foundation. |
| 9 | Final Issue 5 | Join-key allocation belongs with join-path and lookup work. |
| 10 | Final Issue 3 | Worker-safe preload is required before scheduler/worker execution. |
| 11 | Final Issue 2 | Arc-backed shared columns belong with ColumnStore and clone reduction. |
| 12 | Final Issue 7 | Feature-flagged ready-layer parallel execution belongs with Rayon execution. |
| 13 | Final Issue 2 | Interned IDs are a hot-path data-structure optimization. |
| 14 | Final Issue 3 | Dependency-aware resolver refresh is part of safe scheduler/merge boundaries. |
| 15 | Final Issue 3 | Batched NodeOutput merge is part of the scheduler foundation. |
| 16 | Final Issue 1 | Profiling and benchmarks are the performance baseline issue. |
| 17 | Final Issue 3 | Parallel-safety validation belongs before scheduler and Rayon execution. |
| 18 | Final Issue 3 | Formula dependency parsing and graph build feeds the execution graph. |
| 19 | Final Issue 2 | FormulaEvaluator clone reduction belongs with shared columns and data structures. |
| 20 | Final Issue 4 | Actuals, forecast indices, and entity key reuse are a focused actuals/forecast issue. |
| 21 | Final Issue 2 | Structured IDs belong with typed data structures and lookup reduction. |

## Final Correct Implementation Order

1. Final Issue 1 - Add Omni-Calc execution profiling, benchmarks, and regression baseline.
2. Final Issue 2 - Optimize data structures, shared column storage, clone reduction, interned IDs, FormulaEvaluator context, and typed IDs.
3. Final Issue 3 - Refactor executor toward Kahn-style scheduler foundation, worker-safe PreloadedMetadata, dependency-aware resolver refresh, batched NodeOutput merge, parallel-safety validation, and formula dependency graph build.
4. Final Issue 4 - Optimize actuals handling, forecast indices, entity key reuse, and row metadata.
5. Final Issue 5 - Optimize cross-object join path creation, lookup aggregation, and join-key representation.
6. Final Issue 6 - Parallelize safe pre/post execution processing: final RecordBatch materialization and connected dimension preload.
7. Final Issue 7 - Add configurable Rayon-based ready-layer / ready-node parallel execution behind feature flags.

## Priority / Order Validation

The final issue numbers follow implementation priority and dependency order. They are not a simple Jira priority sort, because the final Rayon execution ticket is a P0 product/architecture milestone but must be implemented last after the safety foundations are complete.

| Final Issue | Jira Priority | Implementation Order | Reason |
| --- | --- | --- | --- |
| Final Issue 1 | P0 | 1 | Establishes baseline, profiling, and regression gates before any optimization. |
| Final Issue 2 | P0 | 2 | Data-structure and clone reduction foundation needed before scheduler/parallel work. |
| Final Issue 3 | P0 | 3 | Builds serial Kahn scheduler, snapshots, merge, validation, preload safety, and resolver safety. |
| Final Issue 4 | P1 | 4 | Optimizes actuals/forecast hot paths before broader parallel pressure. |
| Final Issue 5 | P1 | 5 | Optimizes cross-object join and lookup hot paths before worker execution. |
| Final Issue 6 | P1 | 6 | Adds lower-risk pre/post parallelism after foundations are measurable and stable. |
| Final Issue 7 | P0 milestone | 7 | Final ready-node Rayon execution milestone; intentionally last because it depends on Issues 1-6. |

## Why This Order Is Correct

```text
Benchmark and profile first
  -> establish trustworthy before/after measurements

Data structures and clone reduction second
  -> avoid making parallel execution memory-bound or clone-bound

Scheduler foundation third
  -> create deterministic snapshots, NodeOutput, merge, resolver refresh, validation, and dependency graph behavior

Actuals/forecast optimization fourth
  -> remove repeated row/entity-key and forecast membership costs before broader parallelism

Cross-object join optimization fifth
  -> reduce resolver, join-key, and lookup costs before parallel workers increase pressure on them

Safe pre/post parallelism sixth
  -> parallelize naturally independent work with lower semantic risk

Rayon ready-node execution last
  -> only after serial Kahn parity, preload completeness, resolver safety, and validation are proven
```

---

# FINAL ISSUE 1

## Title

Add Omni-Calc Execution Profiling, Benchmarks, and Performance Regression Baseline

## Issue Type

Performance / Observability / Test Infrastructure

## Priority

P0 - Must be implemented before the other performance work.

## Source Issues Merged

```text
Source Issue 16 - Add Execution Profiling and Benchmarks for Scheduler, Snapshot, Merge, and Resolver
```

## Problem Statement

Major executor, data-structure, resolver, actuals, join, and Rayon changes need a reliable baseline. Without a complete benchmark and regression gate, it will be impossible to prove whether later changes improve runtime, reduce memory pressure, preserve output parity, or merely move cost between phases.

## Code Evidence / Current State

- `modelAPI/omni-calc/src/engine/exec/perf.rs` already defines `ExecutionTiming` and `ModelShape`.
- `modelAPI/omni-calc/Cargo.toml` already includes Criterion and benchmark entries.
- `modelAPI/omni-calc/benches/*` exists for full execution, formula evaluation, join alignment, RecordBatch, actuals, and real model scenarios.
- `python.rs` supports `OMNI_CALC_PERF_TRACE` and `OMNI_CALC_LOG_HOT_PATH_DETAILS`.
- The current instrumentation is useful but should be formalized into required baselines, thresholds, and CI/regression gates.

## Correct Scope

This issue is not Source Issue 13, 14, or 18. Those belong to interned IDs, resolver refresh, and formula dependency graph work. This issue only covers profiling, benchmark completeness, performance reporting, and regression gating.

## Proposed Implementation

- Define a standard benchmark suite for small, medium, wide-column, cross-object, actuals-heavy, sequential-heavy, and real-model workloads.
- Capture phase timings for preload, connected dimension preload, input, calculation, sequential, formula setup/eval, resolver update/extract, join path creation, lookup aggregation, target alignment, RecordBatch materialization, and final result build.
- Capture clone counters and estimated clone bytes where currently tracked.
- Add benchmark result reporting that can compare current branch against baseline.
- Add CI or pre-merge regression thresholds for runtime and output parity.
- Keep debug-hot-path metrics measurable but disabled unless explicitly requested.

## Acceptance Criteria

- Benchmark suite runs successfully with `cargo bench` for the listed workloads.
- Benchmarks cover scheduler, snapshot, merge, resolver, formula evaluation, actuals, joins, and RecordBatch materialization.
- Baseline data is documented before implementing Final Issues 2-7.
- Performance trace output is stable enough to compare runs.
- Regression thresholds are documented and enforceable.
- Output parity tests are part of benchmark validation where applicable.

## Dependencies

None. This is the first implementation step.

## QA / Testing Notes

Run benchmarks with perf trace off and on, because instrumentation overhead itself must be understood.

---

# FINAL ISSUE 2

## Title

Optimize Data Structures, Shared Column Storage, Clone Reduction, Interned IDs, FormulaEvaluator Context, and Typed IDs

## Issue Type

Performance / Refactor

## Priority

P0 - Required before safe parallel execution can produce useful speedups.

## Source Issues Merged

```text
Source Issue 7  - Optimize CalcObjectState Column Storage and Lookup With Ordered Indexed ColumnStore
Source Issue 8  - Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Preloaded Metadata
Source Issue 11 - Introduce Snapshot-Friendly Shared Column Storage Using Arc on Top of ColumnStore
Source Issue 13 - Replace Selected Hot-Path String Lookups With Interned IDs
Source Issue 19 - Optimize FormulaEvaluator Context Cloning and Shared Column Access
Source Issue 21 - Introduce Structured Node, Block, Dimension, Property, and Column IDs to Reduce String Parsing and Lookup Cost
```

## Problem Statement

The current execution hot path is still string-keyed, vector-scanned, and clone-heavy. Adding Rayon before this foundation is cleaned up risks amplifying memory pressure and lock/contention costs instead of improving runtime.

## Code Evidence / Current State

- `CalcObjectState` in `state.rs` stores `dim_columns`, `number_columns`, `string_columns`, and `connected_dim_columns` as `Vec<(String, Vec<T>)>`.
- Lookup helpers in `state.rs` use `.iter().find(...)`.
- `ExecutionContext::new` clones `node_maps` and `variable_filters` into `CrossObjectResolver`.
- `Engine` stores `Option<PreloadedMetadata>`, not `Arc<PreloadedMetadata>`.
- `FormulaEvaluator::with_raw_properties()` clones `EvalContext` and `ActualsContext`.
- `EvalContext` stores columns as `HashMap<String, Vec<f64>>`.
- `resolver.rs`, `join_path.rs`, and `lookup.rs` use string-heavy keys and repeated string parsing/allocation.

## Correct Scope

This issue groups the foundation data-structure work. It should not implement parallel ready-node execution. It should make later snapshots and worker execution cheap enough to be worth parallelizing.

## Proposed Implementation

- Add ordered indexed `ColumnStore<T>` with stable iteration order and O(1)-style lookup by key.
- Migrate `CalcObjectState` dynamic columns to `ColumnStore`.
- Add `Arc`-backed shared column storage once the ordered store is stable.
- Move immutable preloaded maps, property maps, formula input columns, time values, connected dimension columns, and resolver source columns toward shared references.
- Split immutable FormulaEvaluator context from mutable evaluator state.
- Reduce or eliminate `FormulaEvaluator::with_raw_properties()` context cloning.
- Add interned ID support for selected narrow hot paths while preserving external string names.
- Add typed IDs for node, block, dimension, property, indicator, and column identity.
- Preserve external compatibility, output field names, and RecordBatch schema order.

## Acceptance Criteria

- `ColumnStore<T>` exists and preserves deterministic insertion/output order.
- Dynamic number, string, and connected-dimension columns use indexed lookup.
- Shared column storage is available for snapshots/evaluators/resolver where safe.
- Major clone boundaries are measured and reduced.
- FormulaEvaluator raw-property handling no longer clones full context for common hot paths.
- Interned IDs are used only in safe internal hot paths and external names remain available.
- Typed IDs are introduced incrementally with compatibility helpers.
- Serial output matches current behavior.
- No Python/PyO3 callback is introduced in execution hot paths.

## Dependencies

Final Issue 1 should be complete first so improvements are measurable.

## QA / Testing Notes

Test wide-column models, cross-object formulas, dimension properties, connected dimensions, time functions, sequential functions, actuals, and RecordBatch schema order.

---

# FINAL ISSUE 3

## Title

Refactor Executor Toward Kahn-Style Scheduler Foundation, Worker-Safe PreloadedMetadata, Dependency-Aware Resolver Refresh, Batched NodeOutput Merge, Parallel-Safety Validation, and Formula Dependency Graph Build

## Issue Type

Architecture / Refactor / Correctness Foundation

## Priority

P0 - Required before Rayon ready-node execution.

## Source Issues Merged

```text
Source Issue 1  - Refactor Rust Omni-Calc Executor Toward Safe Kahn-Style DAG Scheduling Foundation
Source Issue 10 - Expand and Normalize PreloadedMetadata for Worker-Safe Execution
Source Issue 14 - Add Dependency-Aware Resolver Snapshot Refresh
Source Issue 15 - Batch NodeOutput Merge Operations
Source Issue 17 - Add Pre-Execution Validation for Parallel-Safety
Source Issue 18 - Optimize Formula Dependency Parsing and Graph Build
```

## Problem Statement

The current executor is still serial and mutable-state-centric. It processes Python-provided `calc_steps` in order and directly mutates shared `ExecutionContext`. That is safe for the current serial model, but it is not a safe foundation for deterministic worker execution.

## Code Evidence / Current State

- `executor.rs` loops over `plan.request.calc_steps` and calls `process_input_step`, `process_calculation_step`, and `process_sequential_step` with `&mut ctx`.
- `ExecutionContext` owns mutable `calc_object_states`, mutable property caches, warnings, and resolver state.
- `update_resolver_with_timing` rebuilds a RecordBatch from current block state and adds it to the resolver.
- `PreloadedMetadata` is available but not normalized into a complete worker-safe execution snapshot.
- Formula dependencies are parsed/handled during execution paths; dependency metadata is not yet a stable pre-execution graph artifact.

## Correct Scope

This issue should build the deterministic serial Kahn-style foundation and worker-safe data model. It should not enable multi-threaded ready-node execution yet. That belongs to Final Issue 7.

## Proposed Implementation

- Add `ExecutionSnapshot` for immutable per-run shared state.
- Add `NodeOutput` so node execution returns owned output instead of directly mutating shared context.
- Add central batched merge for node outputs.
- Add an explicit Rust-side `ExecutionGraph` from formula dependencies and source plan metadata.
- Add a single-threaded Kahn scheduler for output parity before parallelism.
- Normalize `PreloadedMetadata` into a complete worker-safe shape.
- Add dependency-aware resolver snapshot refresh boundaries.
- Add pre-execution validation for parallel safety, missing preload data, unsupported sequential groups, resolver hazards, cycles, and unsafe Python callback paths.
- Optimize formula dependency parsing and graph build so it is done once before execution, not repeatedly in hot paths.

## Acceptance Criteria

- Serial Kahn scheduler produces the same outputs as the current serial executor.
- Node execution can return `NodeOutput` without mutating shared state directly.
- Batched merge applies node outputs deterministically.
- Resolver refresh happens at dependency-aware safe boundaries.
- Worker-safe preload validation fails early when required metadata is missing.
- Formula dependency graph is built once and reused by scheduler planning.
- Sequential groups remain serial unless explicitly proven safe.
- No global `Arc<Mutex<ExecutionContext>>` scheduler design is introduced.

## Dependencies

Final Issue 1 and Final Issue 2 should happen first.

## QA / Testing Notes

Use output parity tests across input, calculation, sequential, cross-object, property reference, actuals, connected dimension, and formula dependency scenarios.

---

# FINAL ISSUE 4

## Title

Optimize Actuals Handling, Forecast Indices, Entity Key Reuse, and Row Metadata

## Issue Type

Performance / Focused Refactor

## Priority

P1 - Important before broad parallel execution, especially for actuals-heavy models.

## Source Issues Merged

```text
Source Issue 20 - Optimize Actuals Handling, Forecast Indices, and Entity Key Reuse
```

## Problem Statement

Actuals and forecast handling repeatedly compute row membership, row dimension maps, dimension keys, and entity keys. In actuals-heavy or sequential-heavy models this can be a meaningful runtime cost and can distort later scheduler/parallelism benchmarks.

## Code Evidence / Current State

- `actuals.rs` computes forecast row indices by scanning time values.
- `actuals.rs` builds row dimension maps and actual maps for matching.
- `ActualsContext::is_forecast()` checks `forecast_indices.contains(&row_idx)`, which is linear in the forecast index vector.
- Audit/debug `eprintln!` calls still exist in actuals mismatch handling.
- Sequential functions rely on last actual values per entity and forecast membership.

## Correct Scope

This issue is standalone because it is focused on actuals/forecast row metadata and can be implemented without changing scheduler behavior.

## Proposed Implementation

- Precompute forecast masks or bitsets per block/time dimension.
- Precompute row dimension keys once per block where possible.
- Precompute row entity keys for non-time dimensions.
- Replace linear forecast membership checks with indexed masks or sets.
- Reuse actual maps and row metadata across indicators where safe.
- Remove or gate `eprintln!` audit output from production hot paths.
- Keep current actuals semantics and Python parity.

## Acceptance Criteria

- Actuals outputs match existing behavior.
- Forecast membership checks avoid repeated linear scans.
- Row/entity key construction is reused where possible.
- Production hot paths do not emit unconditional `eprintln!`.
- Benchmarks show improved or neutral actuals-heavy runtime.

## Dependencies

Final Issue 1 should be complete first. Final Issue 2 helps if shared row metadata is introduced.

## QA / Testing Notes

Test indicators with and without actuals, stale dimension-key actuals, monthly and yearly time dimensions, sequential `balance`/`change`, and multi-entity models.

---

# FINAL ISSUE 5

## Title

Optimize Cross-Object Join Path Creation, Lookup Aggregation, and Join-Key Representation

## Issue Type

Performance / Cross-Object Resolver Optimization

## Priority

P1 - Important for large cross-object models.

## Source Issues Merged

```text
Source Issue 5 - Parallelize Join-Path Creation and Target Alignment for Large Cross-Object Joins
Source Issue 6 - Evaluate Parallel Lookup-Map Aggregation with Deterministic Reductions
Source Issue 9 - Optimize Join Keys and Avoid Repeated String Join Path Allocation
```

## Problem Statement

Cross-object references build string join paths, create lookup maps, aggregate duplicates, and align source values to target rows. This work is currently string-heavy and sequential, and it can become expensive for large cross-object joins.

## Code Evidence / Current State

- `join_path.rs` builds join paths by collecting dimension values and joining them with `|`.
- `lookup.rs` stores lookup maps as `HashMap<String, f64>`.
- `lookup.rs` performs aggregation with sequential `HashMap` updates for `sum`, `mean`, `first`, and `last`.
- `resolver.rs` repeatedly extracts dimension and indicator values from RecordBatches and applies join/lookup logic.
- Source Issue 9 is not a full standalone source ticket section, but the source file explicitly requires avoiding repeated string allocation/cloning for join keys.

## Correct Scope

This issue covers cross-object join and lookup performance. It should not change cross-object semantics, output parity, or aggregation behavior.

## Proposed Implementation

- Add threshold-based parallel join-path creation for large row counts.
- Add threshold-based parallel target alignment.
- Evaluate deterministic per-worker lookup maps and ordered reduction.
- Preserve aggregation semantics for `sum`, `mean`, `first`, and `last`.
- Replace repeated join-path strings with structured/interned/encoded join-key representation where safe.
- Keep serial fallback for small inputs and for cases where parallel overhead is not justified.

## Acceptance Criteria

- Serial and optimized paths produce equivalent outputs.
- `first` and `last` preserve original row-order semantics.
- Floating-point tolerance for parallel `sum`/`mean` is documented.
- Threshold fallback exists.
- Join-key allocation is reduced.
- Benchmarks prove the optimized path is faster for large joins and not worse for small joins.

## Dependencies

Final Issue 1 first. Final Issue 2 is recommended before broad join-key representation changes.

## QA / Testing Notes

Test duplicate join keys, missing join keys, one-to-one joins, many-to-one joins, property joins, filters, grouped and non-grouped references, and repeated deterministic runs.

---

# FINAL ISSUE 6

## Title

Parallelize Safe Pre/Post Execution Processing: Final RecordBatch Materialization and Connected Dimension Preload

## Issue Type

Performance / Low-Risk Parallelism

## Priority

P1 - Useful before ready-node parallelism but after baseline and data-structure work.

## Source Issues Merged

```text
Source Issue 3 - Parallelize Final RecordBatch Materialization Across Blocks
Source Issue 4 - Parallelize Connected Dimension Preload Across Blocks
```

## Problem Statement

Some work is naturally independent by block and can be parallelized with less scheduler risk than node execution. Final RecordBatch materialization and connected dimension preload are good candidates, but they still need deterministic merge/order behavior and output parity.

## Code Evidence / Current State

- `build_execution_result_with_timing` builds final block results after execution.
- `build_record_batch_with_timing` materializes Arrow arrays from state columns.
- `preload_connected_dimensions` iterates blocks and dimensions inside `executor.rs`, collecting connected dimension columns.
- Connected dimension preload currently checks existing columns with linear scans and performs per-block work serially.

## Correct Scope

This issue should parallelize safe pre/post work only. It should not parallelize calculation node execution. That belongs to Final Issue 7.

## Proposed Implementation

- Split connected dimension preload into local per-block compute and deterministic central merge.
- Parallelize connected dimension preload across blocks when row/model size exceeds a threshold.
- Parallelize final RecordBatch materialization across blocks.
- Preserve RecordBatch field order and block result order.
- Keep serial fallback for small models and debugging.

## Acceptance Criteria

- Parallel and serial paths produce identical RecordBatch schemas and values.
- Connected dimension columns are added deterministically.
- No shared mutable `ExecutionContext` is mutated by workers.
- Threshold and feature/config controls exist.
- Benchmarks show measurable benefit for large models.

## Dependencies

Final Issue 1 and Final Issue 2 are recommended. This can happen before Final Issue 3 if implemented with local compute and deterministic merge, but it is cleaner after shared data structures exist.

## QA / Testing Notes

Test blocks with no dimensions, direct dimensions, connected dimension properties, string and numeric properties, duplicate linked dimensions, and large multi-block outputs.

---

# FINAL ISSUE 7

## Title

Add Configurable Rayon-Based Ready-Layer / Ready-Node Parallel Execution

## Issue Type

Parallel Execution / Feature-Flagged Architecture

## Priority

P0 for the final parallelisation milestone, but last in implementation order.

## Source Issues Merged

```text
Source Issue 2  - Add Configurable Rayon-Based Parallel Execution for Independent Ready Nodes
Source Issue 12 - Add Rayon-Based Ready-Layer Parallel Execution Behind a Feature Flag
```

## Problem Statement

The project goal is real Rust-side parallel execution of independent ready nodes. However, this should only happen after the serial Kahn scheduler, worker-safe snapshots, preload completeness, resolver refresh boundaries, validation, shared columns, and benchmarks are in place.

## Code Evidence / Current State

- `Cargo.toml` already includes `rayon = "1.10"`.
- `python.rs` runs Rust execution under `py.allow_threads`.
- Current executor does not yet use a ready queue or Rayon workers for node execution.
- Current node execution still mutates shared `ExecutionContext`, so parallel workers would be unsafe before Final Issue 3.

## Correct Scope

This issue implements controlled parallel ready-node execution. It should not use `Arc<Mutex<ExecutionContext>>` as the main design. Workers should consume immutable snapshots and return `NodeOutput` for central deterministic merge.

## Proposed Implementation

- Add feature flag / runtime config for ready-layer parallelism.
- Execute only nodes proven independent and safe by the pre-execution validator.
- Keep sequential groups serial initially.
- Use immutable snapshots and shared preloaded data in workers.
- Return `NodeOutput` from workers and merge centrally.
- Preserve serial fallback.
- Add thresholds to avoid Rayon overhead on small ready sets.

## Acceptance Criteria

- Parallel execution is disabled by default or guarded by explicit config until proven stable.
- Serial Kahn and parallel Kahn outputs match.
- Workers never call Python/PyO3.
- Workers do not mutate shared `ExecutionContext`.
- Sequential groups remain serial unless explicitly safe.
- Parallel runs are deterministic across repeated executions.
- Benchmarks demonstrate speedup on models with sufficient independent ready-node width.

## Dependencies

Final Issues 1, 2, and 3 are required. Final Issues 4, 5, and 6 are recommended before enabling broadly.

## QA / Testing Notes

Test small ready sets, wide independent ready sets, cross-object dependencies, sequential groups, actuals-heavy models, missing preload validation, error propagation, and deterministic repeated runs.

---

# Coverage Verification

## Every Source Issue Is Accounted For

```text
Final Issue 1 covers: 16
Final Issue 2 covers: 7, 8, 11, 13, 19, 21
Final Issue 3 covers: 1, 10, 14, 15, 17, 18
Final Issue 4 covers: 20
Final Issue 5 covers: 5, 6, 9
Final Issue 6 covers: 3, 4
Final Issue 7 covers: 2, 12
```

Expanded source coverage:

```text
1  -> Final Issue 3
2  -> Final Issue 7
3  -> Final Issue 6
4  -> Final Issue 6
5  -> Final Issue 5
6  -> Final Issue 5
7  -> Final Issue 2
8  -> Final Issue 2
9  -> Final Issue 5
10 -> Final Issue 3
11 -> Final Issue 2
12 -> Final Issue 7
13 -> Final Issue 2
14 -> Final Issue 3
15 -> Final Issue 3
16 -> Final Issue 1
17 -> Final Issue 3
18 -> Final Issue 3
19 -> Final Issue 2
20 -> Final Issue 4
21 -> Final Issue 2
```

No source issue from the source planning file is intentionally left ungrouped. Source Issue 9 is treated as a real requirement even though it appears in the source file's coverage/final-note section rather than as a full standalone ticket body.

## Final File Status

This final grouped file should supersede both draft grouped files for planning and Jira creation. The original three source/draft files were not edited.
