# Issue 2 Jira Tickets - Story-Driven Omni-Calc Runtime Performance Plan

> Code branch analyzed: `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`
>
> These Jira stories are based on the current Omni-Calc code in that branch only.
>
> Format: Jira-friendly story framing with BDD-style acceptance journeys. Each ticket uses:
>
> - `Feature Promise` for the user/system value.
> - `Starting Context` for the verified current state.
> - `Code Map` for the exact data structures, helper methods, files, and flows involved.
> - `Acceptance Journeys` for Given / When / Then validation paths.
>
> Important merge decision: Source Issues 11 and 8 are intentionally merged into one Jira story. Source Issue 11 creates shared Arc-backed storage, and Source Issue 8 applies that shared storage to remove clone-heavy execution paths. They are one implementation flow.

## Recommended Implementation Order

1. Jira Ticket 1 - Source Issue 7 - Add `ColumnStore` and indexed lookup.
2. Jira Ticket 2 - Source Issues 11 + 8 - Add shared Arc column storage and reuse it across clone-heavy execution paths.
3. Jira Ticket 3 - Source Issue 19 - Refactor `FormulaEvaluator` shared context.
4. Jira Ticket 4 - Source Issue 13 - Add string interning for selected hot-path keys.
5. Jira Ticket 5 - Source Issue 21 - Introduce structured typed IDs gradually.

## Shared Goal

The current Omni-Calc executor now has runtime tracing, but the core execution structures still repeatedly scan vectors, clone full columns, rebuild RecordBatches, parse string IDs, and format business identity into text. These stories convert the measured hot paths into focused Jira tickets.

The roadmap is intentionally staged:

- First make column access indexed and deterministic.
- Then make committed column data shareable through immutable storage.
- Then stop formula evaluation and resolver flows from cloning shared input state.
- Then reduce repeated string-key overhead.
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

Today, every wide model makes the executor repeatedly walk through column vectors to answer simple questions like "does this column exist?", "give me this indicator column", or "is this connected dimension already present?". That work compounds across formula setup, connected dimension preload, dependency resolution, filtering, sequential calculation, cross-object joins, and RecordBatch materialization.

This ticket is the foundation for the rest of Issue 2. Shared storage, FormulaEvaluator context sharing, string interning, and typed IDs all become cleaner once column state has one stable indexed container.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/filter_utils.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/property_bridge.rs`

### Code Map - Current Data Structures And Linear Lookup Sites

#### Current Data Structures

Verified in `state.rs`:

- `CalcObjectState::dim_columns: Vec<(String, Vec<String>)>`
- `CalcObjectState::number_columns: Vec<(String, Vec<f64>)>`
- `CalcObjectState::string_columns: Vec<(String, Vec<String>)>`
- `CalcObjectState::connected_dim_columns: Vec<(String, Vec<String>)>`
- `CalcObjectState::node_ids: HashSet<i64>`

These vectors preserve order, but name lookup requires scanning.

#### Current Helper Methods To Replace Or Back With `ColumnStore`

Verified in `state.rs`:

- `CalcObjectState::new_block`
- `CalcObjectState::new_dimension`
- `CalcObjectState::get_number_column`
- `CalcObjectState::get_string_column`
- `CalcObjectState::get_connected_dim_column`
- `CalcObjectState::add_number_column`
- `CalcObjectState::add_string_column`
- `CalcObjectState::add_connected_dim_column`

The current lookup shape is:

```rust
pub fn get_number_column(&self, name: &str) -> Option<&Vec<f64>> {
    self.number_columns
        .iter()
        .find(|(n, _)| n == name)
        .map(|(_, v)| v)
}
```

Similar vector scans exist for string columns and connected dimension columns.

#### Additional Linear Lookup And Duplicate-Check Flows

Verified in `executor.rs`:

- `preload_connected_dimensions`
  - Checks `state.dim_columns.iter().any(...)`.
  - Checks `state.connected_dim_columns.iter().any(...)`.
  - Checks queued connected columns with `connected_dims_to_add.iter().any(...)`.
  - Finds source dimension values with `state.dim_columns.iter().find(...)`.
- `process_calculation_step`
  - Checks existing cross-object and dimension-property columns with `state.number_columns.iter().any(...)`.
  - Checks connected dimension additions with `state.connected_dim_columns.iter().any(...)`.
  - Builds `dim_columns_map` from `state.dim_columns.iter().cloned().collect()`.
  - Finds time values with `state.dim_columns.iter().find(...)`.
- `process_sequential_step`
  - Reuses direct dimension, number, string, and connected columns during sequential evaluation.
  - Checks existing number columns before adding dependencies.
- `collect_cross_object_columns`
  - Checks `existing_columns.iter().any(...)`.
  - Checks `result.iter().any(...)`.
- `collect_dimension_property_columns`
  - Checks `existing_columns.iter().any(...)`.
  - Checks `result.iter().any(...)`.
  - Reads dimension state number columns for property references.
- `collect_connected_dim_columns`
  - Checks `existing_connected_dims.iter().any(...)`.
  - Checks `result.iter().any(...)`.
  - Calls `dim_state.get_connected_dim_column(...)`.
- `align_dimension_property_to_target` and string alignment helpers
  - Search source and target dimension columns by name.

Verified in `context.rs`:

- `build_record_batch`
- `build_record_batch_with_timing`
  - Iterates over all column families to materialize output.
  - Builds duplicate field detection from final fields.
  - Tracks `duplicate_check_ms`, `column_lookup_ms`, clone counts, and estimated clone bytes.

Verified in `filter_utils.rs`:

- `FilterContext::get_dimension_column`
  - Searches `dim_columns`, then `connected_dim_columns`.
- `FilterContext::get_property_column`
  - Searches `string_columns`.
- `apply_property_filter`
  - Searches connected dimension columns and direct dimension columns for dimension-type filters.

Verified in `resolver.rs`:

- `CrossObjectResolver::get_node_map`
  - Builds `NodeMapKey::from_parts(...)` and looks up in `node_maps`.
- `CrossObjectResolver::resolve_reference_with_timing`
  - Checks whether connected-dimension join columns are already present in `source_dim_columns`.
  - Extracts source dimensions, property columns, string columns, and connected dimension columns from RecordBatch snapshots.
- `CrossObjectResolver::apply_node_map`
  - Filters `merge_on` using `source_dim_columns.iter().any(...)`.
  - Builds source and target join paths.
  - Creates lookup maps for aggregation.

Verified in `node_alignment/property_bridge.rs`:

- Time-column and property-column lookup use `.iter().find(...)`.
- `get_connected_dimension_column` searches `connected_dim_columns`.

Verified in `node_alignment/lookup.rs`:

- `create_lookup_map`
- `aggregate_to_target`
- `join_property_map_to_rows`
  - Search dimension columns and build lookup maps from string join paths.

### Acceptance Storyline

#### Feature Promise

Fast indexed column access for the Rust Omni-Calc executor, while preserving the exact column order and output shape users already rely on.

#### Starting Context

- Given the Rust Omni-Calc executor is running on branch `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`.
- And `CalcObjectState` currently stores dynamic columns as ordered vectors.
- And helper methods and several execution flows perform linear name lookup over those vectors.
- And existing output schema order must remain stable.

#### Acceptance Journeys

**Journey 1: Find any execution column without walking the whole list**

- Given a block state contains many dimension, numeric, string, and connected dimension columns.
- When the executor requests a column by name.
- Then the column is resolved through a maintained index.
- And the returned values match the current Vec-based implementation.

**Journey 2: Keep output order exactly as users see it today**

- Given columns are inserted in calculation order.
- When the final RecordBatch is materialized.
- Then ordered iteration returns columns in the same order as before.
- And external RecordBatch field names are unchanged.

**Journey 3: Replace a calculated column without duplicate output fields**

- Given a calculated output column already exists in the store.
- When a later step replaces that column.
- Then the store updates the existing column slot.
- And no duplicate field appears in final output.

**Journey 4: Centralize duplicate and contains behavior**

- Given calculation, sequential, resolver, and filter flows currently perform local `.any(...)` and `.find(...)` checks.
- When those flows migrate to `ColumnStore`.
- Then duplicate prevention and contains checks use the shared container API.
- And local ad-hoc scans are removed where the store owns the data.

**Journey 5: Prove lookup and duplicate-check cost changed**

- Given runtime tracing is available in the BLOX-2143 branch.
- When a model is executed before and after `ColumnStore` migration.
- Then `column_lookup_ms` and `duplicate_check_ms` can be compared.
- And any neutral or improved result is documented.

### Definition Of Done

- `ColumnStore<T>` exists with ordered storage plus an index.
- `CalcObjectState` uses `ColumnStore` for number, string, dimension, and connected dimension columns.
- The methods listed in the Code Map are migrated or delegated to `ColumnStore`.
- Lookup, insertion, replacement, contains checks, duplicate checks, and ordered iteration are covered by tests.
- RecordBatch schema order remains unchanged.
- Serial Rust output remains identical to the current implementation.
- Runtime benchmark notes include lookup and duplicate-check impact.

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

The runtime tracing branch can now show where time is spent, but many expensive paths still move data by cloning full vectors. This is especially risky for future Kahn-style scheduling and Rayon execution: if every worker snapshot copies every column, parallelism can lose before it begins.

This ticket turns committed columns and preloaded data into reusable read-only execution inputs. Newly calculated outputs remain owned while being produced, then become shared only when merged into committed state.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/mod.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/mod.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/preload.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs`

### Code Map - Current Clone-Heavy Data Structures And Methods

#### Current Owned State

Verified in `state.rs`:

- `CalcObjectState` stores committed data as `Vec<(String, Vec<f64>)>` and `Vec<(String, Vec<String>)>`.
- There is no production shared column alias such as `Arc<[f64]>` or `Arc<[String]>`.

Verified in `steps/mod.rs`:

- Step results are owned output vectors:
  - `StepResult::columns: Vec<(String, Vec<f64>)>`
  - `StepResult::connected_dim_columns: Vec<(String, Vec<String>)>`
  - `StepResult::string_columns: Vec<(String, Vec<String>)>`

This owned shape is correct for newly calculated outputs, but committed state should become shareable after merge.

#### Metadata Sharing Gap

Verified in `engine/mod.rs`:

- `Engine::preloaded_metadata: Option<PreloadedMetadata>`
- `Engine::set_preloaded_metadata`
- `Engine::preloaded_metadata() -> Option<&PreloadedMetadata>`

Verified in `preload.rs`:

- `PreloadedMetadata::dimension_items: HashMap<i64, Vec<DimensionItem>>`
- `PreloadedMetadata::property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>`
- `PreloadedMetadata::get_property_value`
- `PreloadedMetadata::get_property_map`
- `collect_metadata_needs`
- `preload_metadata_with_timing`
- `extract_dimension_items_bulk`
- `extract_property_rows`

The data is borrowed during execution, but not yet shaped as an immutable `Arc<PreloadedMetadata>` execution snapshot.

#### Clone-Heavy Execution Methods

Verified in `context.rs`:

- `ExecutionContext::new`
  - Clones `plan.request.node_maps`.
  - Clones `plan.request.variable_filters`.
  - Holds mutable `string_property_map_cache` and `numeric_property_map_cache`.
- `ExecutionContext::update_resolver_with_timing`
  - Calls `build_record_batch_with_timing` and pushes a new RecordBatch into the resolver.
- `build_record_batch_with_timing`
  - Clones values into Arrow arrays for `dim_columns`, `connected_dim_columns`, `string_columns`, and `number_columns`.
  - Increments `string_column_clone_count`, `number_column_clone_count`, `recordbatch_clone_count`, and `estimated_clone_bytes`.
- `build_execution_result`
  - Builds RecordBatches for every block.
  - Clones warnings into the final result.

Verified in `executor.rs`:

- `process_calculation_step`
  - Clones cross-object and property columns into state.
  - Clones time values for `PropertyFilterContext`.
  - Builds `dim_columns_map` by cloning dimension columns.
- `collect_dimension_property_columns`
  - Resolves and returns owned numeric property vectors.
- `collect_connected_dim_columns`
  - Resolves and returns owned string connected dimension vectors.
- `build_execution_result_with_timing`
  - Materializes output RecordBatches and tracks clone counters.

Verified in `steps/calculation.rs`:

- `CalculationStepHandler::process_with_dim_columns`
- `CalculationStepHandler::process_with_filter_context`
- `CalculationStepHandler::process_with_filter_context_timed`
  - Call `evaluator.add_column(name.clone(), values.clone())`.
  - Clone actual values in actuals handling.
  - Re-add calculated outputs into the evaluator as owned vectors.

Verified in `formula_eval.rs`:

- `EvalContext` owns cloned input maps and time values.
- `FormulaEvaluator::with_raw_properties` clones the whole `EvalContext`.
- `FormulaEvaluator::init_time_functions` clones dimension string columns and time values.
- `FormulaEvaluator::get_non_time_dim_columns` clones string dimension columns for grouped functions.

Verified in `resolver.rs`:

- `CrossObjectResolver::add_block`
  - Stores a RecordBatch snapshot.
- `get_indicator_values`
- `get_dimension_columns`
- `get_property_column`
- `extract_string_columns_from_batch`
- `extract_connected_dim_columns_from_batch`
  - Extract RecordBatch columns back into owned `Vec` values.
- `apply_filters_with_context`
  - Returns owned filtered values.
- `apply_node_map`
  - Builds join paths and lookup maps as owned data.

#### Runtime Counters Already Available

Verified in `perf.rs` and call sites:

- `number_column_clone_count`
- `string_column_clone_count`
- `recordbatch_clone_count`
- `formula_context_clone_count`
- `estimated_clone_bytes`
- `resolver_batch_extract_ms`
- `recordbatch_materialization_ms`
- `time_values_clone_ms`

### Acceptance Storyline

#### Feature Promise

Cheap read-only sharing for committed columns and preloaded metadata, so future execution snapshots and current hot paths stop paying deep-clone costs.

#### Starting Context

- Given `ColumnStore` from Jira Ticket 1 is available.
- And committed execution columns are read often and replaced explicitly.
- And newly calculated node outputs are still produced as owned vectors.
- And BLOX-2143 already tracks clone counts and estimated clone bytes.

#### Acceptance Journeys

**Journey 1: Share committed columns instead of copying them**

- Given a block has committed numeric, string, dimension, and connected dimension columns.
- When a snapshot or downstream execution phase reads those columns.
- Then it receives shared immutable column references.
- And the original column vectors are not deep-cloned for read-only access.

**Journey 2: Keep new calculation results owned until they are ready**

- Given a calculation node produces a new numeric output.
- When formula evaluation finishes.
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

**Journey 6: Show clone reduction with runtime evidence**

- Given runtime counters track clone counts and estimated clone bytes.
- When a model is benchmarked before and after shared storage migration.
- Then number column clone, string column clone, formula context clone, or RecordBatch materialization counts are lower.
- Or the remaining clone boundaries are explicitly documented.

### Definition Of Done

- Shared column aliases exist, preferably `Arc<[f64]>` and `Arc<[String]>`.
- `ColumnStore` stores or exposes immutable shared column data.
- Snapshot-style structures can cheaply clone column and metadata references.
- Newly calculated outputs remain owned until merge.
- The clone-heavy methods listed in the Code Map are migrated, isolated, or explicitly documented as remaining clone boundaries.
- `PreloadedMetadata` is shared by reference or `Arc` and not repeatedly cloned.
- Property maps are built from preloaded metadata and exposed through immutable snapshots where practical.
- RecordBatch schema and field order remain unchanged.
- Existing serial executor output remains identical.
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

Formula evaluation sits in the center of calculation performance. The BLOX-2143 branch already tracks formula parse, setup, evaluation, and raw-property context clone counts. The next step is to make those measured clone-heavy paths cheaper without changing formula behavior.

The goal is not to change formula outputs. The goal is to make every formula read from shared input context while keeping warnings, property filter mode, actuals behavior, prior state, and integer-result state local to the evaluator instance.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/balance.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/change.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/defer.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/rampup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/rolling_sum.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/function_impl/shift.rs`

### Code Map - Formula Context Data And Clone Methods

#### Current Evaluator Data Structures

Verified in `formula_eval.rs`:

- `EvalContext`
  - `columns: HashMap<String, Vec<f64>>`
  - `dim_string_columns: HashMap<String, Vec<String>>`
  - `item_date_ranges: HashMap<(i64, String), ItemDateRange>`
  - `time_values: Vec<String>`
  - `time_bounds: (String, String)`
  - `integer_columns: HashSet<String>`
  - derives `Clone`
- `FormulaEvaluator`
  - `ctx: EvalContext`
  - `warnings: Vec<EvalWarning>`
  - `actuals_context: Option<ActualsContext>`
  - `property_filter_context: PropertyFilterContext`
  - `prior_called`
  - `last_result_is_integer`
  - `raw_property_context_clone_count`

#### Methods That Currently Own Or Clone Input Data

Verified in `formula_eval.rs`:

- `EvalContext::add_column`
- `EvalContext::get_column`
- `EvalContext::add_dim_string_column`
- `EvalContext::get_dim_string_column`
- `EvalContext::set_time_data`
- `EvalContext::set_item_date_ranges`
- `EvalContext::get_item_date_ranges_for_dim`
- `FormulaEvaluator::new`
- `FormulaEvaluator::new_with_filter_context`
- `FormulaEvaluator::with_raw_properties`
  - Clones `self.ctx`.
  - Clones `self.actuals_context`.
  - Increments `raw_property_context_clone_count`.
- `FormulaEvaluator::init_time_functions`
  - Clones dimension string columns into `ctx.dim_string_columns`.
  - Clones time values into `ctx.time_values`.
- `FormulaEvaluator::add_column`
- `FormulaEvaluator::add_column_with_filter`
- `FormulaEvaluator::apply_filters`
  - Creates owned filtered vectors.
- `FormulaEvaluator::get_non_time_dim_columns`
  - Clones grouped dimension columns for balance/change/shift/rampup/defer/rolling functions.
- `FormulaEvaluator::evaluate_with_timing`
  - Adds `raw_property_context_clone_count` into `formula_context_clone_count`.

Verified in `steps/calculation.rs`:

- `CalculationStepHandler::process_with_dim_columns`
  - Calls `evaluator.init_time_functions`.
  - Calls `evaluator.add_column(name.clone(), values.clone())`.
  - Adds calculated output columns back into evaluator with cloned values.
- `CalculationStepHandler::process_with_filter_context`
- `CalculationStepHandler::process_with_filter_context_timed`
  - Same setup pattern with property filter context.
  - Clones actual values during actuals-aware handling.

#### Function Implementations That Consume Cloned Dimension Groups

Verified in `function_impl` modules:

- `balance.rs`
- `change.rs`
- `defer.rs`
- `rampup.rs`
- `rolling_sum.rs`
- `shift.rs`

These grouped functions currently receive `&[(String, Vec<String>)]`, so the evaluator builds owned grouped dimension vectors before calling them.

### Acceptance Storyline

#### Feature Promise

Shared formula input context that reduces setup and raw-property clone overhead while preserving every existing formula result.

#### Starting Context

- Given shared column storage from Jira Ticket 2 is available.
- And `FormulaEvaluator` currently owns numeric columns, dimension strings, time values, properties, and actuals context.
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
- Then the child evaluator shares immutable `EvalContext`.
- And the child evaluator receives fresh local warnings and local evaluator state.

**Journey 3: Preserve property filtering behavior**

- Given a formula uses property filters.
- When formula evaluation runs with shared context.
- Then property filter behavior remains unchanged.
- And raw property behavior remains unchanged.

**Journey 4: Protect time, actuals, and sequential semantics**

- Given formulas use time functions, actuals context, `prior`, `days`, `weekdays`, balance, change, rampup, defer, rolling, or shift behavior.
- When the shared-context evaluator is used.
- Then those formulas return the same values and warnings as before.

**Journey 5: Prove formula cloning got cheaper**

- Given runtime tracing tracks formula setup and raw-property clone count.
- When the same model is run before and after the `FormulaEvaluator` refactor.
- Then `formula_context_clone_count` is reduced.
- And formula output parity remains unchanged.

### Definition Of Done

- `FormulaEvaluator` uses an `Arc<EvalContext>` or equivalent immutable shared context.
- `with_raw_properties()` no longer deep-clones full `EvalContext`.
- The methods listed in the Code Map are migrated, adapted to shared storage, or explicitly documented as remaining clone boundaries.
- Existing numeric input columns are not cloned into evaluator setup where shared storage is available.
- Dimension and time string columns are not cloned into evaluator setup where shared storage is available.
- Formula outputs remain owned where necessary.
- Property filtering, raw property behavior, actuals behavior, warnings, and integer-column handling remain unchanged.
- Existing formula, calculation, and sequential tests pass.
- Runtime counters or benchmarks show reduced formula context clone cost.

### Dependencies / Risks

- Depends on Jira Ticket 1 and Jira Ticket 2.
- Main risk: accidentally placing mutable evaluator state into shared context.
- Formula behavior is correctness-sensitive, especially raw properties, actuals, prior, warning collection, and integer handling.

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

The executor repeatedly handles names like `b123`, `ind456`, `_789`, `prop123`, `block123___ind456`, and `dim123___prop456`. These strings are meaningful at system boundaries, but inside hot execution paths they behave like repeated work.

This ticket introduces a narrow interning layer for selected high-volume keys. It is not a broad typed-ID rewrite. It is an incremental bridge that reduces string overhead while preserving readable external names.

### Affected Areas

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/filter_utils.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/mod.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs`

### Code Map - Current String-Key Hot Paths

#### Missing Interner

Verified in the branch:

- No production `StringInterner` exists.
- No production `InternedId` exists.
- `calc_buffers/columns.rs` currently has `ColumnId(pub String)`, which labels strings but does not remove string allocation, hashing, comparison, parsing, or formatting.

#### Parser And Reference String Methods

Verified in `get_source_data/parser.rs`:

- `is_cross_object_reference`
- `is_dim_prop_reference`
- `parse_cross_object_ref`
  - Splits on `___`.
  - Strips `block`.
  - Formats `b{id}`.
  - Returns string fields.
- `parse_dim_prop_ref`
  - Splits on `___`.
  - Strips `dim`.
  - Parses dimension ID.
  - Keeps property column and full column name as strings.
- `extract_cross_object_refs`
- `extract_dim_prop_refs`

Verified in `function_impl/lookup.rs`:

- `parse_lookup_token`
- `parse_dim_prop_reference`
  - Split and parse lookup tokens such as `block456___ind123` and `dim1___prop2`.

#### Resolver String-Key Methods

Verified in `resolver.rs`:

- `NodeMapKey`
  - `source_block_key: String`
  - `target_block_key: String`
  - `variable_name: String`
- `NodeMapKey::new`
  - Clones strings from `PlannedNodeMap`.
- `NodeMapKey::from_parts`
  - Allocates strings during lookup.
- `CrossObjectResolver::new`
  - Builds `HashMap<NodeMapKey, PlannedNodeMap>`.
- `CrossObjectResolver::parse_reference`
  - Returns `(String, String)`.
- `CrossObjectResolver::get_node_map`
  - Builds a string-backed lookup key.
- `CrossObjectResolver::resolve_reference_with_timing`
  - Uses string reference, block key, indicator ID, and property join column names.
- `CrossObjectResolver::apply_node_map`
  - Uses string `merge_on` keys and string join paths.

#### Executor String Formatting And Parsing Methods

Verified in `executor.rs`:

- Repeated `format!("_{}", id)` for dimension and connected dimension columns.
- Repeated `format!("b{}", id)` and `format!("ind{}", id)` style construction.
- Repeated `strip_prefix("ind")` for node ID parsing.
- Regex-based dimension property extraction with `dim(\d+)___prop(\d+)`.
- `parse_dim_prop_reference`
- `extract_dimension_property_references`
- `extract_cross_object_references`
- `collect_dimension_property_columns`
- `collect_connected_dim_columns`
- `resolve_cross_object_references`

#### Formula And Filter String Methods

Verified in `formula_eval.rs`:

- `EvalContext` key maps are `HashMap<String, Vec<f64>>` and `HashMap<String, Vec<String>>`.
- `FormulaEvaluator::is_property_column` checks names like `dim{id}___prop{id}`.
- `FormulaEvaluator::add_column` and `EvalContext::add_column` store string keys.
- `FormulaEvaluator::init_time_functions` formats dimension column names as `_{dim_id}`.

Verified in `filter_utils.rs`:

- `FilterContext::get_dimension_column` formats `_{dim_id}`.
- `FilterContext::get_property_column` formats `prop{property_id}`.
- Dimension-type property filters format `_{linked_dim_id}`.

#### Join And Lookup String Paths

Verified in `node_alignment/join_path.rs` and `node_alignment/lookup.rs`:

- Join paths are string paths.
- Lookup maps are built using string join keys.
- `lookup::create_lookup_map`
- `lookup::aggregate_to_target`
- `lookup::join_property_map_to_rows`

#### Plan Structures Holding String Identity

Verified in `integration/calc_plan.rs`:

- `PlannedNodeMap::source_block_key: String`
- `PlannedNodeMap::target_block_key: String`
- `PlannedNodeMap::variable_name: String`
- `PlannedNodeMap::source_dims_that_map: Vec<String>`
- `PlannedNodeMap::additional_columns_for_join: Vec<String>`
- `PlannedNodeMap::merge_on: Vec<String>`
- `PropertyJoinColumn::source_column: String`
- `VariableFilter::variable_name: String`
- `VariableFilter::source_block_key: String`
- `VariableFilter::source_node_id: String`

### Acceptance Storyline

#### Feature Promise

Intern repeated execution keys once per run, then use stable IDs in selected hot paths while keeping every external name readable and unchanged.

#### Starting Context

- Given the executor uses repeated string identifiers for blocks, nodes, columns, variables, join paths, filters, and cross-object references.
- And external output must continue to use the original string names.
- And this is a narrow hot-path optimization, not the full typed-ID migration.

#### Acceptance Journeys

**Journey 1: Turn duplicate execution strings into one stable internal ID**

- Given the same column or node key appears many times in an execution plan.
- When the plan is prepared for execution.
- Then the key is interned once.
- And repeated internal references can use the same `InternedId`.

**Journey 2: Start with one measured hot path**

- Given `ColumnStore` lookup, resolver calculated block lookup, node-map lookup, or formula existing-column lookup is selected as the first migration path.
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
- Interning is applied to at least one measured hot path from the Code Map.
- Original string names remain available at RecordBatch, Python/API, warning, error, and debug boundaries.
- Tests cover intern, resolve, duplicate intern, missing resolve, and stable IDs.
- Tests confirm external output names remain unchanged.
- Benchmarks or runtime metrics show improvement or document neutral result.
- Migration notes identify remaining string-heavy paths for Jira Ticket 5.

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

- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/filter_utils.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/preload.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`
- `/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs`

### Code Map - Current String Identity And Typed-ID Gaps

#### Current Wrappers Are Still String-Based

Verified in `calc_buffers/columns.rs`:

- `ColumnId(pub String)`
- `ColumnKey { id: ColumnId, version: u64 }`
- `ColumnMeta`

This gives a label but not semantic identity such as `BlockId(i64)`, `IndicatorId(i64)`, `DimensionId(i64)`, `PropertyId(i64)`, or structured `ColumnId`.

#### Resolver Identity Still Uses Strings

Verified in `resolver.rs`:

- `NodeMapKey`
  - `source_block_key: String`
  - `target_block_key: String`
  - `variable_name: String`
- `BlockData`
  - `block_key: String`
- `CrossObjectResolver::calculated_blocks: HashMap<String, BlockData>`
- `CrossObjectResolver::node_maps: HashMap<NodeMapKey, PlannedNodeMap>`
- `CrossObjectResolver::variable_filters: HashMap<String, VariableFilter>`

These should eventually move toward typed block, node, and column identity while keeping compatibility with current string inputs.

#### Parser And Formula Identity Still Use String Formats

Verified in `get_source_data/parser.rs`:

- `CrossObjectRef`
  - `block_key: String`
  - `indicator_col: String`
  - `variable_name: String`
- `DimPropRef`
  - `dim_id: i64`
  - `property_col: String`
  - `full_col_name: String`
- `parse_cross_object_ref`
- `parse_dim_prop_ref`
- `extract_cross_object_refs`
- `extract_dim_prop_refs`

Verified in `formula_eval.rs`:

- Formula columns are looked up by string name.
- Property columns are recognized by string pattern `dim{id}___prop{id}`.
- Time and dimension columns are formatted as `_{dim_id}`.

#### Executor Identity Conversion Is Scattered

Verified in `executor.rs`:

- `strip_prefix("ind")` is used to parse indicator/node IDs in multiple flows.
- `format!("_{}", dim_id)` is used for direct dimensions and connected dimensions.
- `format!("b{}", block_id)` is used for object keys.
- `format!("dim{}___prop{}", dim_id, prop_id)` is used for property references.
- `parse_dim_prop_reference` and regex extraction duplicate format knowledge.
- `collect_dimension_property_columns`, `collect_connected_dim_columns`, `resolve_cross_object_references`, and preload/alignment helpers all depend on string identity.

#### Plan Structures Already Contain Numeric IDs But Also Carry String Identity

Verified in `integration/calc_plan.rs`:

- Numeric IDs exist in plan specs:
  - block IDs
  - indicator IDs
  - dimension IDs
  - property IDs
  - scenario IDs
  - item IDs
- But `PlannedNodeMap`, `VariableFilter`, and property-join structures still carry string column and variable names for execution.

#### Metadata Already Uses Numeric Keys

Verified in `preload.rs`:

- `PreloadedMetadata::dimension_items: HashMap<i64, Vec<DimensionItem>>`
- `PreloadedMetadata::property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>`
- `PreloadedMetadata::get_property_value(dimension_id, property_id, scenario_id, item_id)`
- `PreloadedMetadata::get_property_map(dimension_id, property_id, scenario_id)`

Typed IDs should let execution use these numeric identities directly instead of repeatedly converting into string column names and back again.

### Acceptance Storyline

#### Feature Promise

Move internal execution identity from encoded strings toward semantic typed IDs, while preserving every existing external name and formula contract.

#### Starting Context

- Given the executor currently encodes semantic identity in strings such as `ind123`, `_456`, `prop789`, `b123`, `block123___ind456`, and `dim123___prop456`.
- And external output names, formula syntax, warning text, and API response shapes must remain unchanged.
- And this migration must be gradual.

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

- Given a resolver lookup, `ColumnStore` lookup, formula dependency path, or dependency planning path repeatedly parses string identity.
- When that path is migrated to typed IDs.
- Then ad-hoc strip, split, regex, and format logic is removed from that path.
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
- At least one internal path from the Code Map uses typed IDs instead of ad-hoc string parsing.
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
- Each ticket lists the relevant data structures, helper methods, files, and flows.
- Each ticket preserves current serial Rust output behavior.
- Each ticket includes tests or benchmark evidence.
- Runtime tracing from `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline` is used to document before/after impact where practical.
- No external API output, RecordBatch schema, warning text, or formula syntax changes unless a future ticket explicitly scopes that change.
