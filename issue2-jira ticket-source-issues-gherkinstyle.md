# Issue 2 Jira Tickets - Story-Driven Omni-Calc Runtime Performance Plan

> Code branch analyzed: `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`
>
> These Jira stories are based on the Omni-Calc code in that branch only.
>
> Format: Jira-friendly story framing with BDD-style acceptance journeys. Each ticket uses:
>
> - `Feature Promise` for the story outcome.
> - `Starting Context` for the known system state.
> - `Acceptance Journeys` for the Given / When / Then flows QA and engineering can validate.
>
> Important merge decision: Source Issues 11 and 8 are intentionally merged into one Jira story. Source Issue 11 introduces shared Arc-backed column storage, and Source Issue 8 applies that shared storage to remove clone-heavy execution paths. Keeping them separate would split one implementation flow into two overlapping tickets.

## Recommended Implementation Order

1. Jira Ticket 1 - Source Issue 7 - Add `ColumnStore` and indexed lookup.
2. Jira Ticket 2 - Source Issues 11 + 8 - Add shared Arc column storage and reuse it across clone-heavy execution paths.
3. Jira Ticket 3 - Source Issue 19 - Refactor `FormulaEvaluator` shared context.
4. Jira Ticket 4 - Source Issue 13 - Add string interning for selected hot-path keys.
5. Jira Ticket 5 - Source Issue 21 - Introduce structured typed IDs gradually.

## Shared Goal

The current Omni-Calc executor has useful runtime tracing, but the core execution structures still do a lot of repeated scanning, cloning, string parsing, and ownership movement. These stories turn the measured hot paths into focused engineering work.

The roadmap is intentionally staged:

- First make column access indexed and deterministic.
- Then make committed column data shareable through immutable storage.
- Then stop formula evaluation and resolver flows from cloning shared input state.
- Then reduce repeated string lookup overhead.
- Finally move toward semantic typed IDs.

---

## Jira Ticket 1 - Source Issue 7

### Title

Add Indexed `ColumnStore` for Omni-Calc Execution Columns

### Issue Type

Performance / Architecture / Tech Debt

### Priority

Highest

### Story

As an Omni-Calc platform engineer, I want execution columns stored in an ordered indexed container so that the engine can preserve output order while avoiding repeated linear scans during calculation.

### Why This Matters

Today, every wide model makes the executor repeatedly walk through column vectors to answer simple questions like "does this column exist?" and "give me this indicator column." That work is invisible to users, but it compounds across formulas, dependencies, connected dimensions, resolver updates, and final output materialization.

This ticket creates the foundation that later performance tickets build on. Without it, shared storage, snapshots, typed IDs, and scheduler work all sit on top of a container that still behaves like a list lookup in hot paths.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/filter_utils.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`

### Acceptance Storyline

#### Feature Promise

Fast indexed column access for the Rust Omni-Calc executor, while preserving the exact column order and output shape users already rely on.

#### Starting Context

- Given the Rust Omni-Calc executor is running on branch `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`.
- And `CalcObjectState` currently stores dynamic columns as ordered vectors.
- And existing output schema order must remain stable.

#### Acceptance Journeys

**Journey 1: Find a numeric column without walking the whole list**

- Given a block state contains many numeric columns.
- When the executor requests a numeric column by name.
- Then the column is resolved through a maintained index.
- And the lookup returns the same values as the current Vec-based implementation.

**Journey 2: Keep output order exactly as users see it today**

- Given columns are inserted in calculation order.
- When the final RecordBatch is materialized.
- Then ordered iteration returns columns in the same order as before.
- And the external RecordBatch field names are unchanged.

**Journey 3: Replace a calculated column without duplicate output fields**

- Given a calculated output column already exists in the store.
- When a later step replaces that column.
- Then the store updates the existing column slot.
- And the index points to the replacement column.
- And no duplicate field appears in final output.

**Journey 4: Cover every execution column family**

- Given the executor stores dimension, numeric, string, and connected dimension columns.
- When `CalcObjectState` is migrated to `ColumnStore`.
- Then each column family supports get, contains, insert, replace, and ordered iteration.
- And existing serial execution results remain identical.

**Journey 5: Prove the hot-path lookup cost changed**

- Given runtime tracing is available in the BLOX-2143 branch.
- When a model is executed before and after `ColumnStore` migration.
- Then column lookup timing and duplicate-check timing can be compared.
- And any neutral or improved result is documented.

### Definition Of Done

- `ColumnStore<T>` exists with ordered storage plus an index.
- `CalcObjectState` uses `ColumnStore` for number, string, dimension, and connected dimension columns.
- Lookup, insertion, replacement, contains checks, and ordered iteration are covered by tests.
- RecordBatch schema order remains unchanged.
- Serial Rust output remains identical to the current implementation.
- Runtime benchmark notes include column lookup and duplicate-check impact.

### Dependencies / Risks

- No prerequisite source issue.
- Required before Jira Tickets 2, 3, 4, and 5.
- Main risk: changing output column order accidentally.

---

## Jira Ticket 2 - Source Issues 11 + 8

### Title

Introduce Shared Arc Column Storage and Reuse It Across Clone-Heavy Execution Paths

### Issue Type

Performance / Architecture / Memory / Tech Debt

### Priority

High

### Story

As an Omni-Calc performance engineer, I want committed execution data stored as immutable shared references so that snapshots, formulas, resolver flows, and future parallel workers can reuse the same data without deep-cloning large columns.

### Why This Matters

The runtime tracing branch can now show where time is spent, but many expensive paths still move data by cloning full vectors. This becomes especially dangerous for future Kahn-style scheduling and Rayon execution: if every worker snapshot copies every column, parallelism can lose before it begins.

This ticket turns columns and preloaded data into reusable read-only execution inputs. Newly calculated outputs remain owned while being produced, then become shared only when merged into committed state.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/mod.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/preload.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`

### Acceptance Storyline

#### Feature Promise

Cheap read-only sharing for committed columns and preloaded metadata, so future execution snapshots and current hot paths stop paying deep-clone costs.

#### Starting Context

- Given `ColumnStore` from Jira Ticket 1 is available.
- And committed execution columns are read often and replaced explicitly.
- And newly calculated node outputs are still produced as owned vectors.

#### Acceptance Journeys

**Journey 1: Share committed columns instead of copying them**

- Given a block has committed numeric, string, dimension, and connected dimension columns.
- When a snapshot or downstream execution phase reads those columns.
- Then it receives shared immutable column references.
- And the original column vectors are not deep-cloned for read-only access.

**Journey 2: Keep new calculation results owned until they are ready**

- Given a calculation node produces a new numeric output.
- When the formula evaluation finishes.
- Then the output remains an owned `Vec` while it is being produced.
- And it is converted into shared column storage only when merged into block state.

**Journey 3: Make future scheduler snapshots cheap to clone**

- Given an execution snapshot contains block columns and preloaded metadata.
- When the snapshot is cloned for a future worker-safe execution path.
- Then the clone increments shared references instead of copying column values.
- And snapshot clone behavior is covered by tests.

**Journey 4: Share preloaded metadata without reopening Python-backed paths**

- Given `PreloadedMetadata` is available for dimensions and properties.
- When execution phases need metadata during calculation.
- Then metadata is read through shared immutable references where practical.
- And optimized hot paths do not introduce new Python or PyO3 callbacks.

**Journey 5: Keep unavoidable Arrow copies at the final boundary**

- Given final API output still requires RecordBatch materialization.
- When the result is built.
- Then any Arrow conversion copy happens at the RecordBatch boundary.
- And earlier formula, resolver, and snapshot paths do not clone just to prepare for final output.

**Journey 6: Show the clone reduction with runtime evidence**

- Given runtime counters track clone counts or estimated clone bytes.
- When a model is benchmarked before and after shared storage migration.
- Then number column clone, string column clone, formula context clone, or RecordBatch materialization counts are lower.
- Or the remaining clone boundaries are explicitly documented.

### Definition Of Done

- Shared column aliases exist, preferably `Arc<[f64]>` and `Arc<[String]>`.
- `ColumnStore` stores or exposes immutable shared column data.
- Snapshot-style structures can cheaply clone column and metadata references.
- Newly calculated outputs remain owned until merge.
- Formula setup, resolver update, connected dimension paths, and sequential paths use shared columns where practical.
- `PreloadedMetadata` is shared by reference or `Arc` and not repeatedly cloned.
- Property maps are built from preloaded metadata and exposed through immutable snapshots where practical.
- RecordBatch schema and field order remain unchanged.
- Existing serial executor output remains identical.
- Tests cover shared column lookup/order and cheap snapshot clone behavior.
- Benchmarks or runtime counters document clone reduction or remaining clone boundaries.

### Dependencies / Risks

- Depends on Jira Ticket 1.
- Enables Jira Ticket 3.
- Resolver and metadata behavior are correctness-sensitive.
- Arrow materialization may still copy; full Arrow zero-copy is out of scope.
- `Arc<[T]>` is cleaner for immutable slices, while `Arc<Vec<T>>` may be easier as an intermediate step.

---

## Jira Ticket 3 - Source Issue 19

### Title

Refactor `FormulaEvaluator` to Share Immutable Input Context

### Issue Type

Performance / Formula Engine / Tech Debt

### Priority

High

### Story

As an Omni-Calc formula engineer, I want formula evaluators to share immutable input context so that raw-property functions and formula setup do not repeatedly clone the same source columns.

### Why This Matters

Formula evaluation sits in the center of calculation performance. The BLOX-2143 branch already tracks formula parse, setup, evaluation, and raw-property context clone counts, which confirms that this area is important enough to measure. The next step is to make those measured clone-heavy paths cheaper without changing formula behavior.

The goal is not to change formula outputs. The goal is to make every formula read from shared input context while keeping warnings, property filter mode, actuals behavior, and integer-result state local to the evaluator instance.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`

### Acceptance Storyline

#### Feature Promise

Shared formula input context that reduces setup and raw-property clone overhead while preserving every existing formula result.

#### Starting Context

- Given shared column storage from Jira Ticket 2 is available.
- And `FormulaEvaluator` currently reads numeric columns, dimension strings, time values, properties, and actuals context.
- And formula outputs must remain identical to the current serial executor.

#### Acceptance Journeys

**Journey 1: Build an evaluator without cloning the input world**

- Given a calculation step is preparing to evaluate a formula.
- When the evaluator is built.
- Then numeric input columns are added as shared immutable references where available.
- And dimension and time string columns are added as shared immutable references where available.

**Journey 2: Run raw-property child evaluation without copying the full context**

- Given a formula calls a raw-property path through `IF`, `MAX`, `MIN`, `MOD`, or `IFELSE` behavior.
- When `with_raw_properties` creates a child evaluator.
- Then the child evaluator shares the immutable `EvalContext`.
- And the child evaluator receives fresh local warnings and local evaluator state.

**Journey 3: Preserve property filtering behavior**

- Given a formula uses property filters.
- When formula evaluation runs with the shared context.
- Then property filter behavior remains unchanged.
- And raw property behavior remains unchanged.

**Journey 4: Protect time, actuals, and sequential semantics**

- Given formulas use time functions, actuals context, `prior`, `days`, `weekdays`, or sequential behavior.
- When the shared-context evaluator is used.
- Then those formulas return the same values and warnings as before.

**Journey 5: Prove formula cloning got cheaper**

- Given runtime tracing tracks formula setup and raw-property clone count.
- When the same model is run before and after the `FormulaEvaluator` refactor.
- Then formula context clone count is reduced.
- And formula output parity remains unchanged.

### Definition Of Done

- `FormulaEvaluator` uses an `Arc<EvalContext>` or equivalent immutable shared context.
- `with_raw_properties()` no longer deep-clones full `EvalContext`.
- Existing numeric input columns are not cloned into formula setup where shared storage is available.
- Dimension and time string columns are not cloned into formula setup where shared storage is available.
- Formula outputs remain owned where necessary.
- Property filtering, raw property behavior, actuals behavior, warnings, and integer-column handling remain unchanged.
- Existing formula, calculation, and sequential tests pass.
- Runtime counters or benchmarks show reduced formula context clone cost.

### Dependencies / Risks

- Depends on Jira Ticket 1 and Jira Ticket 2.
- Main risk: moving mutable evaluator state into shared context by accident.
- Formula behavior is correctness-sensitive, especially raw properties, actuals, prior, and warning collection.

---

## Jira Ticket 4 - Source Issue 13

### Title

Add String Interning for Selected Hot-Path Execution Keys

### Issue Type

Performance / Tech Debt

### Priority

Medium

### Story

As an Omni-Calc runtime engineer, I want repeated hot-path string keys interned once per execution so that the engine spends less time cloning, hashing, comparing, formatting, and parsing the same identifiers.

### Why This Matters

The executor repeatedly handles names like `b123`, `ind456`, `_789`, `prop123`, `block123___ind456`, and `dim123___prop456`. These strings are meaningful at system boundaries, but inside hot execution paths they behave like repeated work: cloned, hashed, compared, formatted, split, and parsed again and again.

This ticket introduces a narrow interning layer for selected high-volume keys. It is not a broad typed-ID rewrite. It is an incremental bridge that reduces string overhead while preserving readable external names.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Acceptance Storyline

#### Feature Promise

Intern repeated execution keys once per run, then use stable IDs in selected hot paths while keeping every external name readable and unchanged.

#### Starting Context

- Given the executor uses repeated string identifiers for blocks, nodes, columns, variables, and cross-object references.
- And external output must continue to use the original string names.

#### Acceptance Journeys

**Journey 1: Turn duplicate execution strings into one stable internal ID**

- Given the same column or node key appears many times in an execution plan.
- When the plan is prepared for execution.
- Then the key is interned once.
- And repeated internal references can use the same `InternedId`.

**Journey 2: Start with one measured hot path**

- Given `ColumnStore` lookup, resolver calculated block lookup, or node-map lookup is selected as the first migration path.
- When the hot path is migrated.
- Then it can resolve values by interned ID.
- And a compatibility path still resolves the original string for output, warnings, and debug logs.

**Journey 3: Keep external names clean and familiar**

- Given RecordBatch output, Python API output, warning messages, and debug messages require readable names.
- When interned IDs are used internally.
- Then external field names and user-facing messages remain unchanged.

**Journey 4: Validate the interner itself**

- Given a `StringInterner` is used during an execution run.
- When strings are interned, resolved, duplicated, or missing.
- Then tests cover stable IDs, duplicate intern behavior, resolve behavior, and missing resolve behavior.

**Journey 5: Prove whether string overhead improved**

- Given runtime tracing can measure column lookup, resolver lookup, or string clone counts.
- When a benchmark is run before and after interning.
- Then the improvement or neutral result is documented.
- And no interned ID leaks into external output.

### Definition Of Done

- `InternedId` and `StringInterner` exist.
- Interning is applied to at least one measured hot path.
- Original string names remain available at RecordBatch, Python/API, warning, error, and debug boundaries.
- Tests cover intern, resolve, duplicate intern, missing resolve, and stable IDs.
- Tests confirm external output names remain unchanged.
- Benchmarks or runtime metrics show improvement or document neutral result.
- Migration notes identify remaining string-heavy paths for typed-ID migration.

### Dependencies / Risks

- Should follow Jira Ticket 1.
- Works better after Jira Ticket 2.
- Must not leak interned IDs into user-visible output.
- Multiple interner instances must be scoped carefully so IDs are not compared across incompatible contexts.

---

## Jira Ticket 5 - Source Issue 21

### Title

Introduce Structured Typed IDs for Omni-Calc Execution Identity

### Issue Type

Performance / Architecture / Maintainability / Tech Debt

### Priority

Medium

### Story

As an Omni-Calc architecture engineer, I want semantic typed IDs for blocks, indicators, dimensions, properties, nodes, scenarios, items, and columns so that execution code can stop repeatedly encoding and decoding business identity through strings.

### Why This Matters

The current executor often treats identity as formatted text. A block can be `b123`, an indicator can be `ind456`, a dimension can be `_789`, and a cross-object reference can be `block123___ind456`. Those names must remain at external boundaries, but inside the engine they create repeated parsing and formatting across resolver logic, formula dependencies, joins, column lookup, and future graph construction.

This ticket introduces semantic IDs gradually. It is not a big-bang rewrite. The goal is to centralize parsing and start moving selected internal paths from string identity to typed identity while keeping every external result unchanged.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/types.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Acceptance Storyline

#### Feature Promise

Move internal execution identity from encoded strings toward semantic typed IDs, while preserving every existing external name and formula contract.

#### Starting Context

- Given the executor currently encodes semantic identity in strings such as `ind123`, `_456`, `prop789`, `b123`, and `block123___ind456`.
- And external output names, formula syntax, warning text, and API response shapes must remain unchanged.

#### Acceptance Journeys

**Journey 1: Parse today's external names into typed execution IDs**

- Given an existing external name such as `b123`, `ind456`, `_789`, `dim123___prop456`, or `block123___ind456`.
- When the central parsing helper receives the name.
- Then it returns the correct semantic typed ID.
- And invalid names return a controlled parse error.

**Journey 2: Convert typed IDs back to the exact names users already see**

- Given a typed block, indicator, dimension, property, or cross-object column identity.
- When the executor needs a RecordBatch field name, warning, debug message, or Python/API output.
- Then the typed ID converts back to the exact external string format currently used.

**Journey 3: Remove ad-hoc parsing from one real hot path**

- Given a resolver lookup, `ColumnStore` lookup, or dependency planning path repeatedly parses string identity.
- When that path is migrated to typed IDs.
- Then ad-hoc strip, split, and format logic is removed from that path.
- And behavior remains identical for existing models.

**Journey 4: Align execution identity with PreloadedMetadata**

- Given `PreloadedMetadata` already uses numeric IDs internally.
- When new typed-ID code reads metadata.
- Then it can use typed numeric IDs directly.
- And it does not convert numeric IDs into strings and back again just to perform a lookup.

**Journey 5: Keep every external contract unchanged**

- Given existing tests assert output field names and formula behavior.
- When typed IDs are introduced internally.
- Then RecordBatch schemas, Python result shapes, formula syntax, warnings, errors, and API output names remain unchanged.

### Definition Of Done

- Typed IDs exist for block, indicator, dimension, property, node, scenario, item, and column identity.
- Central parse helpers exist for current string formats.
- External conversion helpers preserve current output names exactly.
- At least one internal path uses typed IDs instead of ad-hoc string parsing.
- `ColumnStore` has a documented path to typed `ColumnId` lookup.
- Resolver and node-map lookup have a documented or partial typed-ID migration path.
- PreloadedMetadata lookup can use typed numeric IDs directly in new code.
- Debug, error, and warning messages remain readable.
- Existing serial output remains unchanged.
- Tests cover parse and external-name roundtrips for supported formats.
- No Python or PyO3 callbacks are introduced.

### Dependencies / Risks

- Should follow Jira Ticket 1.
- Can follow Jira Ticket 4, but does not need to wait for every interned-key migration.
- Broad migration is risky if attempted all at once; the first implementation should migrate one narrow measured path.
- Formula parser output may remain string-based initially.
- Typed IDs and interned IDs must have clear responsibilities.

---

## Rollup Notes For Planning

### What To Build First

Start with Jira Ticket 1. It is the smallest foundation with the largest downstream value. It changes the container shape while preserving order and output behavior.

### What To Merge Together

Keep Source Issues 11 and 8 merged as Jira Ticket 2. Shared storage and clone reduction are one implementation story: first create the shared storage, then use it in clone-heavy paths.

### What To Keep Separate

Keep Jira Ticket 3 separate because `FormulaEvaluator` has unique correctness risk around raw properties, `IF` / `MAX` / `MIN` / `MOD` behavior, actuals, time functions, warnings, and integer handling.

Keep Jira Ticket 4 separate because string interning is a tactical optimization and should be limited to measured hot paths.

Keep Jira Ticket 5 separate because typed IDs are a broader architecture migration and should be introduced gradually.

### Overall Definition Of Done For Issue 2

- Each ticket has a code-verified implementation boundary.
- Each ticket preserves current serial Rust output behavior.
- Each ticket includes tests or benchmark evidence.
- Runtime tracing from `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline` is used to document before/after impact where practical.
- No external API output, RecordBatch schema, warning text, or formula syntax changes unless a future ticket explicitly scopes that change.
