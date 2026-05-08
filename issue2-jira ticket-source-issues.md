# Issue 2 Jira Tickets - Code-Verified Omni-Calc Source Issues

## Recommended Implementation Order

1. Source Issue 11 - Snapshot-friendly shared Arc column storage.
2. Source Issue 8 - Reduce clone-heavy execution paths using shared storage and metadata snapshots.
3. Source Issue 19 - Refactor FormulaEvaluator shared context.
4. Source Issue 13 - Add string interning for selected hot-path keys.
5. Source Issue 21 - Introduce structured typed IDs gradually.

Note: Source Issue 7 remains the prerequisite foundation for these tickets because it introduces indexed `ColumnStore` behavior.

---

## Jira Ticket 1

### Title

Introduce Snapshot-Friendly Shared Column Storage Using Arc on Top of ColumnStore

### Issue Type

Performance / Architecture / Tech Debt

### Priority

High

### Background / Problem

The current Rust omni-calc execution state stores columns as owned `Vec<T>` values inside `CalcObjectState`. Future Kahn-style scheduling and Rayon execution need read-only snapshots. If those snapshots deep-clone every column, snapshot creation can become slower and more memory-heavy than the current serial executor.

### Current Behavior

`CalcObjectState` stores:

```rust
Vec<(String, Vec<f64>)>
Vec<(String, Vec<String>)>
```

RecordBatch materialization, formula setup, resolver update, and future snapshot paths can clone large vectors.

### Proposed Solution

After Source Issue 7 introduces indexed `ColumnStore`, evolve column storage to use immutable shared data:

```rust
pub type SharedNumberColumn = Arc<[f64]>;
pub type SharedStringColumn = Arc<[String]>;
```

Store committed columns as shared `Arc<[T]>`. Keep newly calculated node outputs as owned `Vec<T>` until merge, then convert them into shared columns.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`

### Acceptance Criteria

- Shared numeric and string column aliases exist.
- `ColumnStore` stores or exposes shared immutable column data.
- Snapshot-style structures can clone column references without deep-cloning vectors.
- Newly calculated outputs remain owned until merge.
- RecordBatch field order and schema remain unchanged.
- Serial executor output remains identical.
- Tests prove shared columns preserve lookup/order behavior and cheap snapshot cloning.
- Benchmarks or counters show reduced clone overhead or document remaining boundaries.

### Dependencies / Risks

Depends on Source Issue 7. Enables Source Issue 8 and Source Issue 19. Arrow conversion may still copy at materialization boundaries, which is acceptable for this ticket.

---

## Jira Ticket 2

### Title

Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Metadata Snapshots

### Issue Type

Performance / Memory / Tech Debt

### Priority

High

### Background / Problem

The Rust omni-calc executor still deep-clones large columns and metadata-derived data across formula setup, resolver updates, cross-object dependency handling, property loading, RecordBatch materialization, warning collection, and final result creation.

### Current Behavior

Examples in current code:

- `ExecutionContext::new` clones node maps and variable filters into the resolver.
- formula setup clones existing numeric columns.
- time/dimension setup clones string columns.
- resolver stores RecordBatch snapshots and extracts owned `Vec<T>` values back from them.
- final result creation clones warnings.
- active metadata access still flows through Python `metadata_cache` / PyO3 helper paths.

### Proposed Solution

Use shared column storage from Source Issues 7 and 11. Introduce immutable Rust-side metadata/property snapshots where practical. Reduce resolver RecordBatch roundtrips by moving toward shared `BlockSnapshot` data. Add clone counters for remaining unavoidable boundaries.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/metadata.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`

### Acceptance Criteria

- Major clone-heavy paths are documented and measured.
- Formula input setup uses shared columns where available.
- Time and dimension string setup uses shared data where available.
- Metadata/property maps are represented as immutable Rust-side snapshots where practical.
- Resolver update/materialization avoids unnecessary full data cloning where possible.
- RecordBatch materialization remains correct and isolated to needed boundaries.
- Existing output schema and ordering remain unchanged.
- Serial executor output matches current behavior.
- Clone counters or benchmarks show reduced cloning or document remaining clone boundaries.

### Dependencies / Risks

Depends on Source Issues 7 and 11. Resolver and metadata paths are correctness-sensitive and need parity tests against existing behavior.

---

## Jira Ticket 3

### Title

Optimize FormulaEvaluator Context Cloning and Shared Column Access

### Issue Type

Performance / Formula Engine / Tech Debt

### Priority

High

### Background / Problem

`FormulaEvaluator` is a core hot path. The current `EvalContext` owns cloned input columns and `with_raw_properties()` clones the full context for raw-property child evaluation in functions such as `max`, `min`, `mod`, `if`, and `ifelse`.

### Current Behavior

`EvalContext` stores:

```rust
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
Vec<String>
```

Formula setup clones existing columns. Time and dimension values are cloned. Child evaluators clone the entire context.

### Proposed Solution

Split immutable formula input data from evaluator-local mutable state. Store input context in `Arc<EvalContext>` and share numeric columns, dimension strings, time values, and item date ranges. Keep warnings, property filter mode, actuals pointer, prior state, and integer-result state local to each evaluator.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`

### Acceptance Criteria

- `FormulaEvaluator` input context can be shared by `Arc`.
- `with_raw_properties()` does not clone the full `EvalContext`.
- Existing numeric input columns are not cloned into evaluator setup where shared storage is available.
- Dimension/time string columns are not cloned into evaluator setup where shared storage is available.
- Formula outputs remain owned where necessary.
- Property filtering behavior remains unchanged.
- Raw-property behavior inside `IF`, `MAX`, `MIN`, and `MOD` remains unchanged.
- Existing formula, calculation, and sequential tests pass.
- Benchmarks or counters show reduced formula context clone cost.

### Dependencies / Risks

Works best after Source Issues 7, 11, and 8. The main risk is accidentally changing property filtering, actuals, warning, integer division, or sequential function behavior.

---

## Jira Ticket 4

### Title

Replace Selected Hot-Path String Lookups With Interned IDs

### Issue Type

Performance / Tech Debt

### Priority

Medium

### Background / Problem

The Rust omni-calc executor repeatedly clones, hashes, compares, formats, and parses string identifiers such as block keys, node IDs, column names, variable names, cross-object references, and join paths.

### Current Behavior

There is no production `StringInterner` or `InternedId`. Existing wrappers still store `String`. Hot paths use `HashMap<String, ...>`, string-based resolver keys, string column names, and string join paths.

### Proposed Solution

Add an execution-scoped `StringInterner` and `InternedId`. Apply interning to one measured hot path first, such as `ColumnStore` lookup or resolver `NodeMapKey`. Preserve original strings for RecordBatch fields, API output, warnings, errors, and debug logs.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`

### Acceptance Criteria

- `StringInterner` and `InternedId` exist.
- Tests cover intern, resolve, duplicate intern, missing resolve, and stable IDs.
- Interning is applied to at least one measured hot path.
- Original string names remain unchanged for output/debug/error boundaries.
- Tests confirm output column names remain unchanged.
- Benchmarks or runtime metrics show improvement or document neutral result.

### Dependencies / Risks

Should follow Source Issue 7. This is a bridge optimization, not a replacement for Source Issue 21 typed IDs. Avoid over-interning.

---

## Jira Ticket 5

### Title

Introduce Structured Node, Block, Dimension, Property, and Column IDs for Gradual Execution Migration

### Issue Type

Architecture / Performance / Tech Debt

### Priority

Medium

### Background / Problem

The Rust omni-calc engine encodes semantic identity into strings such as `ind123`, `prop456`, `b789`, `block789___ind123`, and `dim111___prop222`. This causes repeated parsing, formatting, hashing, and comparison across executor, resolver, formula handling, joins, dependency planning, and future graph construction.

### Current Behavior

Existing wrappers such as `BlockId(pub String)` and `ColumnId(pub String)` still store strings. Many paths use ad-hoc `format!`, `strip_prefix`, `split`, and `starts_with` logic.

### Proposed Solution

Add semantic typed IDs for block, indicator, dimension, property, node, scenario, and item IDs. Add structured `ColumnId`. Centralize parsing and external-name conversion helpers. Migrate one narrow internal path first and document the broader migration plan.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/types.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Acceptance Criteria

- Semantic typed IDs exist for block, indicator, dimension, property, node, scenario, and item IDs.
- Structured `ColumnId` exists for indicator, dimension, property, cross-object indicator, and connected dimension identity.
- Parsing helpers centralize current string parsing behavior.
- External-name conversion helpers preserve current output names exactly.
- Unit tests cover parse/to-external-name roundtrips.
- At least one internal path uses typed IDs instead of ad-hoc parsing.
- ColumnStore and resolver typed-ID migration paths are documented.
- Debug/error/warning messages remain readable.
- Existing serial outputs remain unchanged.

### Dependencies / Risks

Should follow Source Issue 7. Can follow Source Issue 13 if interning is used first. This must be gradual because broad migration can introduce subtle correctness bugs.
