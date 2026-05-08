# Source Issue 8 Analysis - Reduce Clone-Heavy Execution Paths

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes, but the issue description should be refined for the current branch.

The main point is valid: the current Rust omni-calc execution path still clones large data structures in formula setup, resolver update, RecordBatch materialization, cross-object dependency handling, property loading, warning collection, and final result creation.

However, the current inspected branch does not show a production `PreloadedMetadata` snapshot stored on `ExecutionContext`. Metadata is still accessed through Python `metadata_cache` helpers and PyO3 loaders in active paths. So the Jira issue should say "move toward shared preloaded metadata / immutable metadata snapshots" rather than assuming `preloaded_metadata: Option<PreloadedMetadata>` is already active in this branch.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2053-navbar-dropdown-backend-readiness-persist-model-banner-image-and-provide-lightweight-blocks-listing-for-navigation
```

Clone-heavy state storage:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
```

```rust
pub dim_columns: Vec<(String, Vec<String>)>,
pub number_columns: Vec<(String, Vec<f64>)>,
pub string_columns: Vec<(String, Vec<String>)>,
pub connected_dim_columns: Vec<(String, Vec<String>)>,
```

Resolver construction clones plan maps:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
```

```rust
let resolver = CrossObjectResolver::new(
    plan.request.node_maps.clone(),
    plan.request.variable_filters.clone(),
);
```

RecordBatch materialization clones column vectors:

```rust
arrays.push(Arc::new(StringArray::from(values.clone())));
arrays.push(Arc::new(Float64Array::from(values.clone())));
```

Final result clones warnings:

```rust
for warning in &ctx.warnings {
    result.add_warning(warning.clone());
}
```

Formula setup clones existing number columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
```

```rust
for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}
```

Formula time/dimension setup clones string columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

```rust
self.ctx.dim_string_columns.insert(col_name, values.clone());
time_values = values.clone();
self.ctx.item_date_ranges.insert((dim.id, item.name.clone()), date_range);
```

`with_raw_properties()` clones the full formula context:

```rust
ctx: self.ctx.clone(),
actuals_context: self.actuals_context.clone(),
```

Execution and cross-object paths clone state columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
```

Examples found:

```rust
let mut all_dim_columns = state.dim_columns.clone();
all_dim_columns.push((col_name.clone(), col_values.clone()));
state.number_columns.clone()
state.connected_dim_columns.clone()
let dim_columns_map: HashMap<String, Vec<String>> = state.dim_columns.iter().cloned().collect();
time_values = values.clone()
```

Resolver stores `RecordBatch` snapshots and extracts owned vectors back from Arrow:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

```rust
pub struct BlockData {
    pub batch: RecordBatch,
    pub block_key: String,
}

fn get_indicator_values(&self, batch: &RecordBatch, indicator_id: &str) -> Result<Vec<f64>>
fn get_dimension_columns(&self, batch: &RecordBatch) -> Result<Vec<(String, Vec<String>)>>
fn extract_string_columns_from_batch(&self, batch: Option<&RecordBatch>) -> Vec<(String, Vec<String>)>
fn extract_connected_dim_columns_from_batch(&self, batch: Option<&RecordBatch>) -> Vec<(String, Vec<String>)>
```

Metadata still uses Python cache access in active paths:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/metadata.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
```

Examples:

```rust
pub fn get_cache(py: Python) -> Result<PyObject, String>
pub fn get_dimension_items(py: Python, cache: &PyObject, dim_id: i64) -> Result<Vec<DimensionItem>, String>
pub fn get_item_property_value(...)
pub fn load_property_map(py: Python, metadata_cache: &PyObject, ...)
pub fn load_string_property_map(py: Python, metadata_cache: &PyObject, ...)
```

### Affected Files And Functions

Primary affected files:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/metadata.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
```

Affected operations:

- resolver plan-map setup
- formula evaluator setup
- raw property child evaluator creation
- time/dimension string setup
- connected dimension merging
- property loading and property map creation
- resolver RecordBatch update
- resolver batch extraction
- final RecordBatch build
- warning cloning

### Estimated Impact

Performance impact:

- Medium to high, especially for formula-heavy, cross-object-heavy, property-heavy, and wide models.

Memory impact:

- High on large row-count models because repeated `Vec<T>` clones multiply memory use.

Scalability impact:

- Clone-heavy data movement makes future parallel execution less effective because workers may duplicate data instead of sharing read-only state.

Correctness impact:

- This is mainly performance and scalability work.
- Risk is correctness-sensitive because resolver and formula behavior must remain identical.

## 2. Simple Explanation

The engine currently copies too much data while calculating.

For example, when a formula is calculated, the engine often copies existing columns into the formula evaluator. When a block is updated, it builds a RecordBatch and later extracts data back out into new vectors. These copies do not change the answer, but they cost CPU and memory.

The fix is to share read-only data where possible and only copy when the engine truly needs a new output column.

In simple terms:

```text
Before:
Copy columns into every place that needs them.

After:
Share finished columns and copy only newly calculated results or final API output.
```

## 3. Technical Explanation

### Current Behavior

The executor uses owned vectors as the common data shape. That makes many functions simple because they can own their inputs, but it causes clone-heavy data movement:

- `ExecutionContext::new` clones node maps and variable filters into the resolver.
- formula setup clones every existing numeric column for the block.
- time and dimension strings are cloned into `EvalContext`.
- `with_raw_properties()` clones the full `EvalContext`.
- resolver update creates RecordBatch snapshots from state.
- resolver reads RecordBatch arrays back into owned `Vec<T>`.
- final output rebuilds RecordBatch from state.

### Root Cause

The engine does not yet have a shared immutable execution data model. Most APIs between components accept or return owned vectors:

```rust
Vec<(String, Vec<f64>)>
Vec<(String, Vec<String>)>
HashMap<String, Vec<f64>>
HashMap<String, Vec<String>>
```

Because these types imply ownership, callers clone frequently to satisfy function signatures or avoid lifetime problems.

### Why This Is A Problem

This creates avoidable CPU and memory overhead. It also makes future parallel scheduling harder because worker-safe snapshots need cheap shared references, not deep copies of the whole block state.

## 4. Proposed Fix

### Recommended Implementation Approach

This should be implemented after Source Issue 7 and Source Issue 11:

- Source Issue 7: add indexed `ColumnStore`.
- Source Issue 11: upgrade `ColumnStore` to shared `Arc<[T]>` storage.
- Source Issue 8: remove clone-heavy call-site behavior using shared storage and immutable snapshots.

### Code-Level Changes Needed

Use shared column storage in executor state:

```rust
pub type SharedNumberColumn = Arc<[f64]>;
pub type SharedStringColumn = Arc<[String]>;
```

Move formula setup from owned clones to shared input views:

```rust
// Current style
for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}

// Target style
for (name, values) in existing_columns.iter_shared() {
    evaluator.add_column_shared(name.to_string(), Arc::clone(values));
}
```

Store metadata in an immutable execution snapshot. In this branch, that first requires introducing a Rust-side preloaded metadata shape rather than relying on PyO3 cache access in hot paths:

```rust
pub struct PreloadedMetadata {
    pub dimension_items: HashMap<i64, Arc<[DimensionItem]>>,
    pub string_property_maps: HashMap<(i64, i64, i64), Arc<HashMap<i64, String>>>,
    pub numeric_property_maps: HashMap<(i64, i64, i64), Arc<HashMap<i64, f64>>>,
}

pub struct ExecutionDataSnapshot {
    pub metadata: Arc<PreloadedMetadata>,
    pub blocks: HashMap<String, Arc<BlockSnapshot>>,
}
```

Introduce immutable property cache snapshots:

```rust
pub struct PropertyCacheSnapshot {
    pub string_maps: HashMap<(i64, i64, i64), Arc<StringPropertyMap>>,
    pub numeric_maps: HashMap<(i64, i64, i64), Arc<PropertyMap>>,
}
```

Move resolver away from RecordBatch as the main intermediate storage:

```rust
pub struct BlockSnapshot {
    pub dim_columns: Arc<ColumnStore<String>>,
    pub connected_dim_columns: Arc<ColumnStore<String>>,
    pub string_columns: Arc<ColumnStore<String>>,
    pub number_columns: Arc<ColumnStore<f64>>,
}

pub struct CrossObjectResolver {
    calculated_blocks: HashMap<String, Arc<BlockSnapshot>>,
    node_maps: Arc<[PlannedNodeMap]>,
    variable_filters: Arc<HashMap<String, VariableFilter>>,
}
```

Keep RecordBatch creation at output or compatibility boundaries:

```rust
// Allowed boundary copy:
final output -> Arrow RecordBatch

// Avoid repeated internal loop:
state -> RecordBatch -> resolver -> Vec<T>
```

Add clone counters:

```rust
pub struct CloneMetrics {
    pub number_column_clone_count: u64,
    pub string_column_clone_count: u64,
    pub recordbatch_materialization_count: u64,
    pub formula_context_clone_count: u64,
    pub warning_clone_count: u64,
    pub estimated_clone_bytes: u64,
}
```

### Recommended Implementation Order

1. Add clone metrics around current known boundaries.
2. Convert `ExecutionContext` plan maps to borrowed or shared ownership.
3. Move formula setup to shared column input.
4. Introduce Rust-side preloaded metadata snapshot if not already present on the target branch.
5. Move property map loading from repeated PyO3 access toward immutable Rust maps.
6. Replace resolver RecordBatch snapshots with `BlockSnapshot` where possible.
7. Keep final Arrow materialization unchanged and isolated.
8. Compare outputs with existing correctness tests.

### Risks, Tradeoffs, And Alternatives

Risks:

- Resolver behavior is correctness-sensitive.
- Moving metadata out of Python callbacks may expose missing preload cases.
- Some clone boundaries are required because formula outputs and final API materialization need owned data.

Tradeoffs:

- `Arc` improves sharing but adds explicit immutability and replacement semantics.
- Full Arrow zero-copy would be larger and should stay out of scope.

Alternative:

- Only add metrics and remove a few obvious clones. This is safer but does not fully prepare for parallel execution.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

This issue reduces wasted copying inside the Rust calculation engine. It should make larger models use less memory and prepares the engine for future parallel execution.

The outputs should not change. The goal is to move the same data around more efficiently.

### Developer-Friendly Explanation

The executor currently uses owned vectors as the boundary between state, formula evaluation, resolver, and RecordBatch output. That creates repeated deep clones. After `ColumnStore` and Arc-backed storage exist, we should migrate call sites to shared read-only columns, convert metadata into immutable snapshots, and reduce internal RecordBatch roundtrips.

### Suggested Meeting Talking Points

- This is separate from Source Issue 11: Issue 11 adds shared storage; Issue 8 removes clone-heavy usage.
- Current branch still relies on Python metadata cache helpers, so the metadata snapshot work needs to be explicit.
- Keep output schema stable and measure clone reduction.
- Do not combine this with formula semantic changes or scheduler changes.

## 6. Suggested Diagrams And Documents

### Diagrams

Create a clone-boundary map:

```text
CalcObjectState
  -> formula setup clone
  -> resolver RecordBatch clone
  -> resolver extraction clone
  -> final RecordBatch clone
  -> warning clone
```

Create an after-state sharing diagram:

```text
Shared ColumnStore + Metadata Snapshot
  -> FormulaEvaluator reads shared slices
  -> Resolver reads BlockSnapshot
  -> Final RecordBatch materializes once
```

### Supporting Documents

Prepare:

- list of required clone boundaries versus avoidable clone boundaries
- benchmark output before and after
- correctness comparison plan against Python engine
- remaining clone boundary documentation

## 7. Jira-Ready Issue

### Title

Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Metadata Snapshots

### Background / Problem Statement

The Rust omni-calc executor still performs repeated deep clones of large column vectors and metadata-derived maps. These clones occur during formula setup, resolver updates, RecordBatch materialization, cross-object dependency handling, warning collection, and final output creation.

### Current Behavior

Column data is passed between components as owned `Vec<T>` values. Formula evaluator setup clones existing columns. Resolver update rebuilds RecordBatch snapshots and later extracts owned vectors back from them. Metadata access still goes through Python `metadata_cache` and PyO3 helpers in active paths.

### Expected Improvement

The executor should share immutable column and metadata data wherever possible, and clone only when producing newly calculated outputs or final API/Arrow materialization.

### Proposed Solution

After Source Issues 7 and 11 introduce indexed shared column storage, migrate clone-heavy call sites to shared column references. Add immutable metadata/property snapshots, reduce resolver RecordBatch roundtrips, and track remaining clone boundaries with counters or benchmarks.

### Impact

- Lower CPU overhead from repeated vector clones.
- Lower memory pressure for large and wide models.
- Better readiness for Kahn-style scheduling and Rayon worker snapshots.
- Clear measurement of remaining clone boundaries.

### Acceptance Criteria

- Major clone-heavy paths are documented and measured.
- Formula input setup uses shared columns where available.
- Time and dimension string columns are shared where available.
- Metadata/property maps are represented as immutable Rust-side snapshots where practical.
- Resolver update/materialization avoids unnecessary full data cloning where possible.
- RecordBatch materialization remains correct and isolated to required boundaries.
- Existing output schema and ordering remain unchanged.
- Serial executor output matches current behavior.
- No new PyO3/Python callbacks are introduced in optimized hot paths.
- Clone counters or benchmarks show reduced cloning or clearly document remaining clone boundaries.

### Notes / Dependencies / Risks

Dependencies:

- Depends on Source Issue 7 `ColumnStore`.
- Depends on Source Issue 11 shared Arc column storage.
- Feeds Source Issue 19 FormulaEvaluator shared context.

Risks:

- Resolver behavior is sensitive and must be validated carefully.
- Some clone boundaries are legitimate and should be documented, not removed blindly.
- Current branch may need Rust-side preloaded metadata work before metadata can be fully shared.
