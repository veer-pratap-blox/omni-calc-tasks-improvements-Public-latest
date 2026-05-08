# Source Issue 11 Analysis - Snapshot-Friendly Shared Arc Column Storage

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes. The issue is valid for the current Rust omni-calc execution path.

The active executor still stores column values directly as owned vectors inside `CalcObjectState`. There is no production shared column storage, no `Arc<[T]>` column alias, and no `ExecutionSnapshot`-style structure that can be cheaply cloned for future Kahn-style scheduling or Rayon worker execution.

The issue should be kept as a separate Jira issue, but it should be positioned after Source Issue 7. Source Issue 7 creates the indexed `ColumnStore` foundation. Source Issue 11 should then upgrade that storage model so column data can be shared cheaply instead of deep-cloned.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2053-navbar-dropdown-backend-readiness-persist-model-banner-image-and-provide-lightweight-blocks-listing-for-navigation
```

Primary code evidence:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
```

`CalcObjectState` owns all column values directly:

```rust
pub dim_columns: Vec<(String, Vec<String>)>,
pub number_columns: Vec<(String, Vec<f64>)>,
pub string_columns: Vec<(String, Vec<String>)>,
pub connected_dim_columns: Vec<(String, Vec<String>)>,
```

The lookup helpers expose references to owned `Vec<T>` values:

```rust
pub fn get_number_column(&self, name: &str) -> Option<&Vec<f64>>
pub fn get_string_column(&self, name: &str) -> Option<&Vec<String>>
pub fn get_connected_dim_column(&self, name: &str) -> Option<&Vec<String>>
```

RecordBatch creation clones those owned vectors:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
```

```rust
arrays.push(Arc::new(StringArray::from(values.clone())));
arrays.push(Arc::new(Float64Array::from(values.clone())));
```

Calculation setup clones existing columns into the formula evaluator:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
```

```rust
for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}
```

Cross-object resolver materializes calculated blocks as Arrow `RecordBatch` values and extracts owned vectors from them again:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

Examples:

```rust
fn get_indicator_values(&self, batch: &RecordBatch, indicator_id: &str) -> Result<Vec<f64>>
fn get_dimension_columns(&self, batch: &RecordBatch) -> Result<Vec<(String, Vec<String>)>>
fn get_property_column(&self, batch: &RecordBatch, column_name: &str) -> Option<Vec<String>>
fn extract_string_columns_from_batch(&self, batch: Option<&RecordBatch>) -> Vec<(String, Vec<String>)>
fn extract_connected_dim_columns_from_batch(&self, batch: Option<&RecordBatch>) -> Vec<(String, Vec<String>)>
```

There are early column identity types in:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs
```

But they are not a production shared column store. `ColumnId` is still just a string wrapper:

```rust
pub struct ColumnId(pub String);
```

Search evidence:

```text
No production SharedNumberColumn / SharedStringColumn aliases found.
No Arc<[T]> column storage found in CalcObjectState.
No ExecutionSnapshot structure found.
No shared ColumnStore wired into the active executor state.
```

### Affected Files And Functions

Primary affected files:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/context.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

Affected areas:

- `CalcObjectState` column ownership.
- `build_record_batch`.
- `ExecutionContext::update_resolver`.
- calculation evaluator setup.
- cross-object resolver snapshots.
- final execution result materialization.
- future scheduler snapshot shape.

### Estimated Impact

Performance impact:

- Medium to high for large row-count models, wide blocks, cross-object-heavy models, and formula-heavy models.
- Biggest impact appears when snapshots or repeated resolver updates clone full columns.

Memory impact:

- High potential memory pressure when the same large columns are copied into formula contexts, resolver batches, final batches, and future worker snapshots.

Scalability impact:

- Current owned-vector storage is fine for serial execution but does not scale cleanly into read-only snapshots for parallel scheduling.

Correctness impact:

- This is not a confirmed correctness bug.
- Main risk is performance and memory overhead.
- Implementation must preserve RecordBatch field order and output names exactly.

## 2. Simple Explanation

Today the Rust engine keeps each column as its own owned list of values.

When another part of the engine needs a snapshot of the data, it often copies the whole list. That is okay for small data, but with large models it can become expensive.

The proposed fix is to store finished columns in shared read-only containers. Then a snapshot can point to the same column data instead of copying it.

In simple terms:

```text
Before:
Snapshot = copy all column values

After:
Snapshot = share references to existing read-only columns
```

New calculated columns can still be normal owned vectors while they are being computed. Once the node is finished, the output vector can be converted into shared storage and merged into the state.

## 3. Technical Explanation

### Current Behavior

The main execution state owns all columns as `Vec<T>`.

This makes mutation simple, but it makes cheap snapshots difficult. If future scheduling creates an immutable snapshot per ready-node batch, every snapshot risks copying:

- number columns
- dimension columns
- string property columns
- connected dimension columns
- time values
- resolver input columns

This is especially important because future Kahn-style scheduling and Rayon execution need a read-only view of state while nodes execute.

### Root Cause

The executor currently has one state shape that is used for:

- active mutable execution state
- formula input setup
- resolver materialization
- final result materialization
- future snapshot candidates

Because the state owns plain `Vec<T>` values, sharing requires either borrowing with complex lifetimes or deep-cloning. The code currently chooses clones in many places.

### Why This Is A Problem

For the current serial executor, this mostly shows up as extra CPU and memory use. For future parallel execution, it becomes more important because snapshots are required for worker safety.

Without shared immutable columns:

- snapshot creation can be expensive
- Rayon workers may duplicate large data
- memory pressure can hide expected parallel speedups
- clone overhead can make a parallel scheduler slower than serial execution

## 4. Proposed Fix

### Recommended Implementation Approach

Implement this after Source Issue 7 creates a stable indexed `ColumnStore`.

Recommended storage model:

```rust
use std::sync::Arc;

pub type SharedNumberColumn = Arc<[f64]>;
pub type SharedStringColumn = Arc<[String]>;

pub struct ColumnStore<T> {
    columns: Vec<(String, Arc<[T]>)>,
    index: HashMap<String, usize>,
}
```

If conversion churn is too high for a first PR, `Arc<Vec<T>>` is acceptable as a stepping stone:

```rust
pub type SharedNumberColumn = Arc<Vec<f64>>;
pub type SharedStringColumn = Arc<Vec<String>>;
```

But `Arc<[T]>` is the better long-term fit because it represents immutable shared column data clearly.

### Code-Level Changes Needed

Add or evolve a column store module:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/column_store.rs
```

Example shape:

```rust
use std::collections::HashMap;
use std::sync::Arc;

pub type SharedColumn<T> = Arc<[T]>;

#[derive(Debug, Clone, Default)]
pub struct ColumnStore<T> {
    columns: Vec<(String, SharedColumn<T>)>,
    index: HashMap<String, usize>,
}

impl<T> ColumnStore<T> {
    pub fn get(&self, name: &str) -> Option<&[T]> {
        self.index
            .get(name)
            .and_then(|idx| self.columns.get(*idx))
            .map(|(_, values)| values.as_ref())
    }

    pub fn get_shared(&self, name: &str) -> Option<SharedColumn<T>> {
        self.index
            .get(name)
            .and_then(|idx| self.columns.get(*idx))
            .map(|(_, values)| Arc::clone(values))
    }

    pub fn insert_owned(&mut self, name: String, values: Vec<T>) {
        self.insert_shared(name, Arc::<[T]>::from(values));
    }

    pub fn insert_shared(&mut self, name: String, values: SharedColumn<T>) {
        if let Some(idx) = self.index.get(&name).copied() {
            self.columns[idx] = (name, values);
        } else {
            let idx = self.columns.len();
            self.index.insert(name.clone(), idx);
            self.columns.push((name, values));
        }
    }

    pub fn iter(&self) -> impl Iterator<Item = (&str, &[T])> {
        self.columns.iter().map(|(name, values)| (name.as_str(), values.as_ref()))
    }
}
```

Update `CalcObjectState`:

```rust
pub struct CalcObjectState {
    pub object_key: String,
    pub object_type: CalcObjectType,
    pub dim_columns: ColumnStore<String>,
    pub number_columns: ColumnStore<f64>,
    pub string_columns: ColumnStore<String>,
    pub connected_dim_columns: ColumnStore<String>,
    pub row_count: usize,
    pub node_ids: HashSet<i64>,
}
```

Introduce snapshot-friendly structures:

```rust
use std::sync::Arc;

pub struct ExecutionSnapshot {
    pub block_key: String,
    pub dim_columns: Arc<ColumnStore<String>>,
    pub number_columns: Arc<ColumnStore<f64>>,
    pub string_columns: Arc<ColumnStore<String>>,
    pub connected_dim_columns: Arc<ColumnStore<String>>,
}

pub struct NodeOutput {
    pub block_key: String,
    pub number_columns: Vec<(String, Vec<f64>)>,
    pub string_columns: Vec<(String, Vec<String>)>,
    pub connected_dim_columns: Vec<(String, Vec<String>)>,
}
```

Keep output vectors owned until merge:

```rust
for (name, values) in node_output.number_columns {
    state.number_columns.insert_owned(name, values);
}
```

Update `build_record_batch` to read from slices:

```rust
for (col_name, values) in state.number_columns.iter() {
    fields.push(Field::new(col_name.to_string(), DataType::Float64, true));
    arrays.push(Arc::new(Float64Array::from(values.to_vec())));
}
```

This still copies at the Arrow boundary, but the copy is now isolated to final materialization or resolver materialization. Full Arrow zero-copy is out of scope for this issue.

### Recommended Implementation Order

1. Implement Source Issue 7 `ColumnStore` with current owned `Vec<T>` values.
2. Add shared aliases and `Arc<[T]>` storage.
3. Update `CalcObjectState` to store shared columns.
4. Add compatibility iterators so existing call sites can migrate gradually.
5. Update formula setup and resolver inputs to accept shared slices where possible.
6. Add snapshot-style structs and tests proving cheap clone behavior.
7. Keep RecordBatch output order and schema unchanged.

### Risks, Tradeoffs, And Alternatives

Risk:

- `Arc<[T]>` is immutable, so replacement must be explicit. This is good for snapshots but requires call-site updates.

Tradeoff:

- `Arc<Vec<T>>` is easier to introduce but semantically weaker because it still looks like a growable vector.

Alternative:

- Use borrowed lifetimes instead of `Arc`. This may reduce allocation further but is much riskier with future Rayon workers and scheduler snapshots.

Recommendation:

- Use `Arc<[T]>` for committed state and owned `Vec<T>` for newly calculated outputs.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

The Rust engine currently copies large columns in several places. This is not always visible on small models, but it can become expensive on larger models and it will make future parallel execution harder.

This issue changes finished column data into shared read-only storage so the engine can create cheap snapshots instead of copying everything.

### Developer-Friendly Explanation

`CalcObjectState` currently owns `Vec<T>` columns directly. Snapshotting, formula setup, resolver updates, and RecordBatch materialization all create clone pressure. We should first introduce the indexed `ColumnStore` from Source Issue 7, then evolve that store to hold `Arc<[T]>`.

Calculated node outputs should remain `Vec<T>` until merge. Once merged, they become immutable shared columns. Future schedulers can then clone `Arc<ColumnStore<T>>` snapshots cheaply.

### Suggested Meeting Talking Points

- This is a prerequisite for worker-safe snapshots.
- It should follow Source Issue 7, not replace it.
- It does not change calculation semantics or API output.
- It intentionally leaves Arrow zero-copy out of scope.
- Success should be measured by clone counters and snapshot clone tests.

## 6. Suggested Diagrams And Documents

### Diagrams

Create a before-vs-after storage diagram:

```text
Before:
CalcObjectState
  number_columns -> Vec<(String, Vec<f64>)>
  snapshot       -> clones Vec<f64>

After:
CalcObjectState
  number_columns -> ColumnStore<Arc<[f64]>>
  snapshot       -> Arc clone only
```

Create an execution lifecycle diagram:

```text
Node calculation
  owned Vec output
    -> merge
      -> Arc<[T]> shared column
        -> read-only snapshot
          -> formula/resolver/future worker reads
```

### Supporting Documents

Prepare:

- list of current clone boundaries
- expected implementation order after Source Issue 7
- benchmark/counter plan for clone reduction
- compatibility note for RecordBatch materialization

## 7. Jira-Ready Issue

### Title

Introduce Snapshot-Friendly Shared Column Storage Using Arc on Top of ColumnStore

### Background / Problem Statement

Future Kahn-style scheduling and Rayon execution require read-only snapshots of calculation state. The current Rust omni-calc state stores columns as owned `Vec<T>` values inside `CalcObjectState`. Creating snapshots from this shape can require deep-cloning large column vectors, which may make snapshot creation slower and more memory-heavy than current serial execution.

### Current Behavior

`CalcObjectState` stores number, string, dimension, and connected dimension columns as `Vec<(String, Vec<T>)>`. Formula setup, resolver update, RecordBatch materialization, and final output creation clone column vectors in multiple places. There is no production `Arc<[T]>` shared column storage or snapshot-friendly execution state.

### Expected Improvement

Finished columns should be stored in immutable shared containers so snapshots can clone references cheaply. Newly calculated outputs should remain owned until they are merged into shared state.

### Proposed Solution

After Source Issue 7 introduces indexed `ColumnStore`, evolve the store to use `Arc<[T]>` column values. Add shared column aliases, snapshot-compatible structures, and merge helpers that convert owned node outputs into shared columns.

### Impact

- Reduces clone pressure in snapshot, formula, resolver, and materialization paths.
- Prepares the executor for Kahn-style scheduling and Rayon worker execution.
- Keeps output schema and ordering stable.
- Isolates remaining Arrow copy costs to materialization boundaries.

### Acceptance Criteria

- Shared column aliases exist for numeric and string columns.
- `ColumnStore` can store and expose shared immutable column data.
- Existing columns can be shared without deep cloning.
- Snapshot-style structures can clone column references cheaply.
- Newly calculated outputs remain owned until merge.
- RecordBatch schema and field order remain unchanged.
- Serial executor output remains identical.
- Unit tests prove shared columns preserve lookup and ordering behavior.
- Tests prove snapshot-style cloning does not deep-clone column vectors.
- Benchmarks or counters show reduced snapshot/column clone overhead or document remaining clone boundaries.

### Notes / Dependencies / Risks

Dependencies:

- Depends on Source Issue 7 `ColumnStore`.
- Enables Source Issue 8 clone reduction.
- Supports Source Issue 19 FormulaEvaluator shared context.
- Supports later scheduler and Rayon work.

Risks:

- `Arc<[T]>` requires explicit replacement semantics.
- Some call sites currently expect owned `Vec<T>`.
- Arrow conversion may still copy, but that should remain out of scope for this issue.
