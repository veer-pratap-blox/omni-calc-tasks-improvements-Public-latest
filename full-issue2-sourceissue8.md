# Source Issue 8 Analysis - Reduce Clone-Heavy Execution Paths

> Code branch analyzed: `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`
>
> This document is based on the Omni-Calc implementation in that branch only. This branch already contains runtime performance tracing and a Rust `PreloadedMetadata` path, so the analysis reflects those branch-specific facts.

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes. The issue is valid for the BLOX-2143 branch.

The branch has already added important preload and timing infrastructure, but several clone-heavy execution paths still remain. The correct framing is not "add preloaded metadata from scratch." The correct framing is:

```text
Use the existing preloaded metadata and future shared column storage as immutable shared execution data, then remove or isolate remaining deep clones.
```

Branch-specific facts:

- `PreloadedMetadata` exists in `modelAPI/omni-calc/src/engine/exec/preload.rs`.
- `Engine` stores `preloaded_metadata: Option<PreloadedMetadata>`.
- `ExecutionContext` receives `Option<&PreloadedMetadata>`.
- property loading can use snapshot loaders instead of Python callbacks.
- `ExecutionContext` still owns mutable property-map caches.
- column state still stores `Vec<(String, Vec<T>)>`.
- formula setup, RecordBatch materialization, resolver extraction, and warning output still clone.
- BLOX-2143 measures clone boundaries, but does not remove them.

This issue is strong enough to become a Jira issue, but it depends on Source Issue 7 and Source Issue 11. Source Issue 7 gives indexed column storage. Source Issue 11 makes that storage shareable with `Arc`. Source Issue 8 is the usage/migration story that applies shared storage across clone-heavy paths.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline
```

Owned column storage remains in:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
```

```rust
pub dim_columns: Vec<(String, Vec<String>)>,
pub number_columns: Vec<(String, Vec<f64>)>,
pub string_columns: Vec<(String, Vec<String>)>,
pub connected_dim_columns: Vec<(String, Vec<String>)>,
```

`PreloadedMetadata` exists, but it is not an `Arc` snapshot:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/preload.rs
```

```rust
#[derive(Clone, Debug, Default)]
pub struct PreloadedMetadata {
    pub dimension_items: HashMap<i64, Vec<DimensionItem>>,
    pub property_maps: HashMap<(i64, i64, i64), HashMap<i64, String>>,
}
```

The engine owns it directly:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/mod.rs
```

```rust
preloaded_metadata: Option<PreloadedMetadata>

pub fn preloaded_metadata(&self) -> Option<&PreloadedMetadata> {
    self.preloaded_metadata.as_ref()
}
```

`ExecutionContext` borrows preloaded metadata and owns mutable caches:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
```

```rust
pub preloaded_metadata: Option<&'a PreloadedMetadata>,
pub string_property_map_cache: HashMap<(i64, i64, i64), StringPropertyMap>,
pub numeric_property_map_cache: HashMap<(i64, i64, i64), (PropertyMap, Vec<String>)>,
```

Resolver setup still clones plan maps:

```rust
let resolver = CrossObjectResolver::new(
    plan.request.node_maps.clone(),
    plan.request.variable_filters.clone(),
);
```

RecordBatch materialization still clones vectors into Arrow arrays, even though BLOX-2143 now counts those clone boundaries:

```rust
arrays.push(Arc::new(StringArray::from(values.clone())));
arrays.push(Arc::new(Float64Array::from(values.clone())));
t.string_column_clone_count += 1;
t.number_column_clone_count += 1;
t.recordbatch_clone_count += 1;
```

Final result still clones warnings:

```rust
for warning in &ctx.warnings {
    result.add_warning(warning.clone());
}
```

Formula setup still clones existing columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
```

```rust
for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}
```

Formula time/dimension setup still clones string columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

```rust
self.ctx.dim_string_columns.insert(col_name, values.clone());
time_values = values.clone();
self.ctx.item_date_ranges.insert((dim.id, item.name.clone()), date_range);
```

`with_raw_properties()` still clones the full `EvalContext`, while BLOX-2143 counts it:

```rust
self.raw_property_context_clone_count += 1;
ctx: self.ctx.clone(),
actuals_context: self.actuals_context.clone(),
```

Execution paths clone state columns:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
```

Examples:

```rust
let dim_columns_map: HashMap<String, Vec<String>> =
    state.dim_columns.iter().cloned().collect();

let time_values = ... .map(|(_, values)| values.clone());

let (dim_columns, row_count) = match ctx.calc_object_states.get(&dim_key) {
    Some(state) => (state.dim_columns.clone(), state.row_count),
    None => return,
};
```

Resolver still stores RecordBatch data and extracts owned vectors:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

```rust
pub struct BlockData {
    pub batch: RecordBatch,
    pub block_key: String,
}
```

The resolver has extraction paths that return owned vectors, such as indicator values, dimension columns, string columns, and connected dimension columns.

Snapshot property loaders exist and are a good foundation:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
```

```rust
pub fn load_string_property_map_from_snapshot(...)
pub fn load_property_map_from_snapshot(...)
```

Runtime metrics already exist:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/perf.rs
```

```rust
pub number_column_clone_count: u64,
pub string_column_clone_count: u64,
pub recordbatch_clone_count: u64,
pub formula_context_clone_count: u64,
pub warning_clone_count: u64,
pub estimated_clone_bytes: u64,
```

### Affected Files And Functions

Primary affected files:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/mod.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/preload.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/perf.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/dim_loader.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

Affected operations:

- plan map construction in `ExecutionContext::new`
- formula evaluator setup
- raw-property child evaluator creation
- time/dimension string setup
- property map caching
- resolver RecordBatch update and extraction
- connected dimension and property column merging
- final RecordBatch build
- warning cloning

### Estimated Impact

Performance impact:

- Medium to high for formula-heavy, cross-object-heavy, property-heavy, wide, and large row-count models.

Memory impact:

- High potential reduction because repeated `Vec<T>` clones multiply memory use.

Scalability impact:

- Important for future scheduler/Rayon work because worker snapshots should clone references, not column payloads.

Maintainability impact:

- Improves ownership boundaries by separating immutable shared input from owned node outputs.

Correctness impact:

- No intended calculation behavior change.
- Risk is medium/high because resolver, property filtering, actuals, and formula behavior are correctness-sensitive.

## 2. Simple Explanation

The branch already measures where clones happen, and it already preloads metadata into Rust. But the engine still copies a lot of large column data while calculating.

This issue is about using shared read-only data instead of copying the same columns over and over.

In simple terms:

```text
Before:
Column data is copied into formula setup, resolver snapshots, RecordBatches, and property flows.

After:
Finished columns and metadata are shared through immutable references.
Only newly calculated outputs are owned until they are merged.
```

## 3. Technical Explanation

### Current Behavior

BLOX-2143 introduced:

- `PreloadedMetadata`
- preload timing
- bulk preload vs fallback timing
- clone counters
- formula context clone counters

But the active execution state still uses owned vectors and mutable caches. The key remaining problem is that APIs between executor, formula evaluator, resolver, and output materialization still pass or return owned vectors.

### Root Cause

The engine does not yet have one immutable execution snapshot shape. The current state mixes:

- mutable execution state
- owned column payloads
- borrowed preloaded metadata
- mutable property-map caches
- RecordBatch resolver snapshots
- owned formula contexts

Because the common shape is still `Vec<T>`, many call sites clone to satisfy ownership.

### Why This Is A Problem

The clone counters in BLOX-2143 make the issue visible. They do not remove the underlying copy cost. If future scheduling creates snapshots from this state, each snapshot can copy column data and hide or erase the benefit of parallelism.

## 4. Proposed Fix

### Recommended Implementation Approach

Implement this after Source Issue 7 and Source Issue 11.

Recommended strategy:

1. Use shared `ColumnStore` from Source Issue 11.
2. Convert committed columns to `Arc<[T]>`.
3. Keep `StepResult` and node outputs as owned `Vec<T>`.
4. Convert owned outputs into shared columns only at merge time.
5. Wrap preloaded metadata in `Arc`.
6. Convert mutable property caches into immutable snapshot-style maps where practical.
7. Move resolver toward shared `BlockSnapshot` instead of RecordBatch as the internal store.
8. Keep final Arrow materialization as an allowed boundary copy.

### Code-Level Changes Needed

Shared preload shape:

```rust
use std::sync::Arc;

pub struct SharedPreloadedMetadata {
    pub dimension_items: HashMap<i64, Arc<[DimensionItem]>>,
    pub property_maps: HashMap<(i64, i64, i64), Arc<HashMap<i64, String>>>,
}
```

Engine ownership:

```rust
pub struct Engine {
    pub config: EngineConfig,
    preloaded_metadata: Option<Arc<SharedPreloadedMetadata>>,
}

pub fn preloaded_metadata(&self) -> Option<Arc<SharedPreloadedMetadata>> {
    self.preloaded_metadata.as_ref().map(Arc::clone)
}
```

Immutable property cache snapshot:

```rust
pub struct PropertyCacheSnapshot {
    pub string_maps: HashMap<(i64, i64, i64), Arc<StringPropertyMap>>,
    pub numeric_maps: HashMap<(i64, i64, i64), Arc<PropertyMap>>,
}
```

Shared block snapshot for resolver:

```rust
pub struct BlockSnapshot {
    pub dim_columns: Arc<ColumnStore<String>>,
    pub connected_dim_columns: Arc<ColumnStore<String>>,
    pub string_columns: Arc<ColumnStore<String>>,
    pub number_columns: Arc<ColumnStore<f64>>,
}
```

Formula setup target:

```rust
for (name, values) in state.number_columns.iter_shared() {
    evaluator.add_column_shared(name.to_string(), Arc::clone(values));
}
```

Allowed boundary copy:

```text
Final result / Arrow RecordBatch materialization may still copy.
Internal formula/resolver/snapshot paths should not copy unless ownership is required.
```

### Risks, Tradeoffs, And Alternatives

Risks:

- Resolver behavior must remain identical.
- Property maps must preserve parsing and warning behavior.
- Some copies are still required for new outputs and Arrow boundaries.

Tradeoffs:

- `Arc<[T]>` is cleaner but requires more conversion work.
- `Arc<Vec<T>>` is easier but less semantically strict.

Alternative:

- Only keep clone counters and defer clone reduction. That is safer short term but does not prepare the engine for scheduler snapshots.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

The branch already tells us where the engine spends time and where data is cloned. This issue uses that measurement to reduce unnecessary copying so larger models use less memory and future parallel execution has a better chance of being faster.

### Developer-Friendly Explanation

BLOX-2143 introduced `PreloadedMetadata` and clone counters, but `CalcObjectState`, `FormulaEvaluator`, and resolver boundaries still use owned `Vec<T>`. After `ColumnStore` and Arc-backed columns exist, migrate call sites to shared immutable columns, wrap preload data in `Arc`, and replace mutable property-map caches with snapshot-style structures where possible.

### Suggested Meeting Talking Points

- This issue uses BLOX-2143 metrics as proof points.
- It should be merged with Source Issue 11 in the Jira rollup if we want one story for "make shared storage and use it."
- It should not change output schema or formulas.
- The final Arrow boundary may still copy.
- Resolver and metadata changes need parity checks.

## 6. Suggested Diagrams And Documents

### Diagrams

Create a before/after clone boundary diagram:

```text
Before:
CalcObjectState Vec<T>
  -> FormulaEvaluator clone
  -> Resolver RecordBatch clone
  -> Resolver extraction clone
  -> Final RecordBatch clone

After:
Shared ColumnStore Arc<[T]>
  -> FormulaEvaluator shared read
  -> Resolver BlockSnapshot shared read
  -> Final RecordBatch boundary copy
```

Create a metadata ownership diagram:

```text
Current BLOX-2143:
Engine owns PreloadedMetadata
ExecutionContext borrows &PreloadedMetadata
ExecutionContext owns mutable property caches

Target:
Arc<PreloadedMetadataSnapshot>
Arc<PropertyCacheSnapshot>
ExecutionSnapshot shares both cheaply
```

### Supporting Documents

Prepare:

- clone boundary list from BLOX-2143 counters
- before/after benchmark report
- property-map cache migration notes
- resolver BlockSnapshot migration plan
- correctness comparison plan against current serial output

## 7. Jira-Ready Issue

### Title

Reduce Clone-Heavy Execution Paths by Reusing Shared Columns, Resolver Data, and Preloaded Metadata

### Background / Problem Statement

BLOX-2143 added runtime tracing, clone counters, and Rust-side `PreloadedMetadata`. However, the active executor still moves large column and metadata-derived data through owned vectors, mutable caches, formula context clones, RecordBatch snapshots, and resolver extraction paths.

### Current Behavior

`CalcObjectState` stores columns as `Vec<(String, Vec<T>)>`. Formula setup clones existing columns. `with_raw_properties()` clones `EvalContext`. RecordBatch materialization clones column vectors. Resolver stores RecordBatches and extracts owned vectors. `PreloadedMetadata` is owned by `Engine` and borrowed by `ExecutionContext`, while property-map caches are mutable `HashMap`s.

### Expected Improvement

Committed columns and preloaded metadata should be shared through immutable references. Clone-heavy paths should reuse shared columns and snapshots where possible. New node outputs should remain owned until merged into shared state. Final RecordBatch materialization may remain an isolated copy boundary.

### Proposed Solution

Use shared column storage from Source Issues 7 and 11. Wrap preloaded metadata and property maps in immutable shared snapshot structures. Move formula setup, resolver source data, connected dimension data, and property cache access toward shared reads. Use existing BLOX-2143 clone counters to prove reduction and document remaining boundaries.

### Impact

- Reduces CPU and memory overhead from repeated clones.
- Makes scheduler snapshots cheaper.
- Improves readiness for Rayon execution.
- Keeps existing output shape and calculation behavior unchanged.

### Acceptance Criteria

- Major clone-heavy paths are documented and measured.
- `PreloadedMetadata` is shared by reference/`Arc` and not repeatedly cloned.
- Property maps are built from `PreloadedMetadata` and exposed as immutable snapshots where practical.
- Formula input setup uses shared columns where possible.
- Resolver update/materialization avoids unnecessary full data cloning where possible.
- RecordBatch materialization remains correct and isolated to needed boundaries.
- Existing output schema and ordering remain unchanged.
- Serial executor output matches current behavior.
- No Python/PyO3 callbacks are introduced in execution hot paths.
- Clone counters or benchmarks show reduced cloning, or remaining clone boundaries are explicitly documented.

### Notes / Dependencies / Risks

Dependencies:

- Depends on Source Issue 7.
- Depends on Source Issue 11.
- Feeds Source Issue 19 FormulaEvaluator refactor.

Risks:

- Resolver behavior is correctness-sensitive.
- Some clones are required for newly calculated outputs and Arrow materialization.
- Moving mutable caches into immutable snapshots may expose missing preload gaps.
