# Source Issue 19 Analysis - FormulaEvaluator Shared Context

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes. This issue is strongly valid and supported directly by the current code.

`FormulaEvaluator` is a hot path for calculation steps. Its current `EvalContext` owns cloned input data, and `with_raw_properties()` clones the full context when evaluating raw-property child expressions for functions such as `max`, `min`, `mod`, `if`, and `ifelse`.

This issue should remain separate from the broader clone-reduction issue because it has a narrow root cause: formula evaluator input context is mutable/owned instead of shared/immutable.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2053-navbar-dropdown-backend-readiness-persist-model-banner-image-and-provide-lightweight-blocks-listing-for-navigation
```

Primary file:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

`EvalContext` owns full column maps:

```rust
#[derive(Debug, Default, Clone)]
pub struct EvalContext {
    columns: HashMap<String, Vec<f64>>,
    row_count: usize,
    dim_string_columns: HashMap<String, Vec<String>>,
    item_date_ranges: HashMap<(i64, String), ItemDateRange>,
    time_values: Vec<String>,
    time_bounds: (String, String),
    integer_columns: HashSet<String>,
}
```

`FormulaEvaluator` owns `EvalContext` directly:

```rust
pub struct FormulaEvaluator {
    ctx: EvalContext,
    warnings: Vec<EvalWarning>,
    actuals_context: Option<ActualsContext>,
    prior_called: bool,
    property_filter_context: PropertyFilterContext,
    last_result_is_integer: bool,
}
```

`with_raw_properties()` clones the full context:

```rust
fn with_raw_properties(&self) -> Self {
    Self {
        ctx: self.ctx.clone(),
        warnings: Vec::new(),
        actuals_context: self.actuals_context.clone(),
        prior_called: false,
        property_filter_context: PropertyFilterContext::without_filtering(),
        last_result_is_integer: false,
    }
}
```

Time and dimension setup clones string columns:

```rust
self.ctx.dim_string_columns.insert(col_name, values.clone());
time_values = values.clone();
self.ctx.item_date_ranges.insert((dim.id, item.name.clone()), date_range);
```

Formula setup clones existing numeric columns into the evaluator:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
```

```rust
for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}
```

Formula evaluation clones source columns when returning a column reference:

```rust
Ok(values.clone())
```

`get_non_time_dim_columns()` clones dimension column names and values:

```rust
.map(|(name, values)| (name.clone(), values.clone()))
.collect()
```

Raw-property child evaluators are created for multiple formula functions:

```rust
"max" => {
    let mut raw_eval = self.with_raw_properties();
    ...
}

"min" => {
    let mut raw_eval = self.with_raw_properties();
    ...
}

"mod" => {
    let mut raw_eval = self.with_raw_properties();
    ...
}

"if" | "ifelse" => {
    let mut raw_eval = self.with_raw_properties();
    ...
}
```

### Affected Files And Functions

Primary affected file:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/formula_eval.rs
```

Affected parts:

- `EvalContext`
- `FormulaEvaluator`
- `FormulaEvaluator::new`
- `FormulaEvaluator::new_with_filter_context`
- `FormulaEvaluator::with_raw_properties`
- `FormulaEvaluator::init_time_functions`
- `FormulaEvaluator::add_column`
- `FormulaEvaluator::evaluate`
- `FormulaEvaluator::eval_ast`
- `FormulaEvaluator::get_non_time_dim_columns`
- formula functions using raw-property child evaluation

Secondary affected file:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/calculation.rs
```

Affected parts:

- `CalculationStepHandler::process_with_dim_columns`
- `CalculationStepHandler::process_with_filter_context`
- `CalculationStepHandler::process`

### Estimated Impact

Performance impact:

- High for formula-heavy models.
- Higher when formulas use `if`, `ifelse`, `max`, `min`, or `mod`, because those create child evaluators that clone the full context.

Memory impact:

- Medium to high for wide blocks and large row counts because each evaluator context can duplicate input columns and time/dimension strings.

Maintainability impact:

- Medium. Splitting immutable input data from evaluator-local mutable state will make formula behavior easier to reason about.

Correctness impact:

- The current behavior is not necessarily incorrect.
- The fix is correctness-sensitive because property filtering, actuals, warnings, integer handling, and sequential function behavior must remain unchanged.

## 2. Simple Explanation

The formula evaluator is like the calculator that runs formulas for indicators.

Right now, before it calculates, it copies a lot of input data into itself. Then for some formula functions, it creates a child calculator and copies the same data again.

The fix is to keep formula input data in a shared read-only context. Child evaluators can reuse that context instead of copying it.

In simple terms:

```text
Before:
FormulaEvaluator owns copied columns.
Child evaluator copies the same columns again.

After:
FormulaEvaluator points to shared input columns.
Child evaluator shares the same input context.
```

The formula result still stays owned because it is newly calculated data.

## 3. Technical Explanation

### Current Behavior

`EvalContext` is cloneable and owns:

- numeric columns
- dimension string columns
- time values
- item date ranges
- integer-column tracking

`FormulaEvaluator` owns that context directly. When `with_raw_properties()` is called, Rust clones the entire `EvalContext`, including all input column vectors.

This happens inside raw-property evaluation flows needed for Python parity:

- `max`
- `min`
- `mod`
- `if`
- `ifelse`

### Root Cause

Immutable formula input data and mutable evaluator state are mixed together.

These are immutable input data:

- numeric source columns
- dimension string columns
- time values
- item date ranges
- row count

These are evaluator-local mutable state:

- warnings
- property filter mode
- actuals context pointer
- `prior_called`
- `last_result_is_integer`

Because they are in one owned structure, child evaluators clone more than they need.

### Why This Is A Problem

Formula evaluation is one of the most frequently executed parts of omni-calc. Cloning source columns during setup and raw-property child evaluation can add significant overhead without changing results.

This also blocks future shared-snapshot work because the formula evaluator still expects owned vectors.

## 4. Proposed Fix

### Recommended Implementation Approach

After Source Issues 7, 11, and 8 introduce shared column storage, refactor `FormulaEvaluator` so immutable input data is shared through `Arc`.

Target shape:

```rust
use std::sync::Arc;

pub struct EvalContext {
    columns: HashMap<String, Arc<[f64]>>,
    dim_string_columns: HashMap<String, Arc<[String]>>,
    item_date_ranges: Arc<HashMap<(i64, String), ItemDateRange>>,
    time_values: Arc<[String]>,
    time_bounds: (String, String),
    integer_columns: HashSet<String>,
    row_count: usize,
}

pub struct FormulaEvaluator {
    ctx: Arc<EvalContext>,
    warnings: Vec<EvalWarning>,
    actuals_context: Option<Arc<ActualsContext>>,
    property_filter_context: PropertyFilterContext,
    prior_called: bool,
    last_result_is_integer: bool,
}
```

### Code-Level Changes Needed

Add shared column APIs:

```rust
impl EvalContext {
    pub fn add_column_shared(&mut self, name: impl Into<String>, values: Arc<[f64]>) {
        self.columns.insert(name.into(), values);
    }

    pub fn get_column(&self, name: &str) -> Option<&[f64]> {
        self.columns.get(name).map(|values| values.as_ref())
    }
}
```

Change child evaluator creation:

```rust
fn with_raw_properties(&self) -> Self {
    Self {
        ctx: Arc::clone(&self.ctx),
        warnings: Vec::new(),
        actuals_context: self.actuals_context.as_ref().map(Arc::clone),
        prior_called: false,
        property_filter_context: PropertyFilterContext::without_filtering(),
        last_result_is_integer: false,
    }
}
```

Add a builder for evaluator setup:

```rust
pub struct EvalContextBuilder {
    row_count: usize,
    columns: HashMap<String, Arc<[f64]>>,
    dim_string_columns: HashMap<String, Arc<[String]>>,
    item_date_ranges: HashMap<(i64, String), ItemDateRange>,
    time_values: Arc<[String]>,
    time_bounds: (String, String),
    integer_columns: HashSet<String>,
}

impl EvalContextBuilder {
    pub fn add_number_column(mut self, name: String, values: Arc<[f64]>) -> Self {
        self.columns.insert(name, values);
        self
    }

    pub fn build(self) -> Arc<EvalContext> {
        Arc::new(EvalContext {
            columns: self.columns,
            dim_string_columns: self.dim_string_columns,
            item_date_ranges: Arc::new(self.item_date_ranges),
            time_values: self.time_values,
            time_bounds: self.time_bounds,
            integer_columns: self.integer_columns,
            row_count: self.row_count,
        })
    }
}
```

Update calculation step setup:

```rust
let eval_context = EvalContextBuilder::new(row_count)
    .with_time_functions(block_spec, shared_dim_columns)
    .with_number_columns(shared_existing_columns)
    .build();

let mut evaluator = FormulaEvaluator::with_context(
    eval_context,
    filter_context,
);
```

Keep formula outputs owned:

```rust
pub fn evaluate(&mut self, formula: &str) -> Result<Vec<f64>> {
    ...
}
```

Update formula helper functions to accept slices:

```rust
fn eval_binary_op(&self, op: BinaryOp, lhs: &[f64], rhs: &[f64]) -> Vec<f64>
```

Where a function needs to return a source column unchanged, the current signature forces `Vec<f64>`. That can remain for this issue, but the clone boundary should be measured. A later expression engine refactor could return copy-on-write values:

```rust
enum EvalValue {
    Borrowed(Arc<[f64]>),
    Owned(Vec<f64>),
}
```

That larger change should not be forced into this ticket unless the team wants a bigger formula-engine refactor.

### Recommended Implementation Order

1. Add `Arc<EvalContext>` and `Arc<ActualsContext>` support.
2. Change `with_raw_properties()` to clone only Arcs.
3. Add shared column insertion APIs.
4. Update calculation setup to pass shared columns.
5. Convert time and dimension string data to shared columns.
6. Keep formula output as owned `Vec<f64>`.
7. Add tests around raw-property functions and property filtering.
8. Compare performance counters for formula context clone cost.

### Risks, Tradeoffs, And Alternatives

Risks:

- Property filtering behavior must remain identical.
- `prior()` and integer division behavior must remain evaluator-local.
- Actuals context must stay correct for sequential functions.

Tradeoff:

- Returning `Vec<f64>` from `eval_ast` still means column reference evaluation clones. Fixing that fully requires a larger `EvalValue` or copy-on-write refactor.

Recommendation:

- First remove the full `EvalContext` clone and setup clones. Leave full expression return-shape refactor for a later focused issue if needed.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

The formula engine currently copies input data too often. This issue lets formulas share read-only input data, reducing memory and runtime overhead without changing calculation results.

### Developer-Friendly Explanation

`FormulaEvaluator` mixes immutable input context with evaluator-local mutable state. `with_raw_properties()` clones `EvalContext`, so functions like `ifelse`, `max`, and `min` can duplicate all source columns. Move immutable input data into `Arc<EvalContext>` and keep warnings, filter mode, prior flags, and integer-result state local to each evaluator.

### Suggested Meeting Talking Points

- This issue is narrower than general clone reduction.
- It specifically targets formula context cloning.
- Raw-property behavior must remain Python-compatible.
- Outputs remain owned; inputs become shared.
- Validate with formula-heavy benchmarks and correctness tests.

## 6. Suggested Diagrams And Documents

### Diagrams

Create a before-vs-after evaluator diagram:

```text
Before:
FormulaEvaluator
  EvalContext { cloned columns, cloned time values }
  with_raw_properties -> clones full EvalContext

After:
FormulaEvaluator
  Arc<EvalContext> { shared immutable input data }
  with_raw_properties -> Arc clone only
```

Create a state separation diagram:

```text
Shared:
  numeric columns
  dimension strings
  time values
  item date ranges

Evaluator-local:
  warnings
  property filter context
  prior_called
  last_result_is_integer
```

### Supporting Documents

Prepare:

- list of formula functions that use `with_raw_properties`
- formula correctness test matrix
- benchmark before/after for formula-heavy models
- notes on remaining `Vec<f64>` return clone boundary

## 7. Jira-Ready Issue

### Title

Optimize FormulaEvaluator Context Cloning and Shared Column Access

### Background / Problem Statement

`FormulaEvaluator` is a core hot path in the Rust omni-calc engine. The current evaluator owns cloned input columns and clones the full `EvalContext` when creating raw-property child evaluators for functions such as `max`, `min`, `mod`, `if`, and `ifelse`.

### Current Behavior

Formula setup clones existing numeric columns into the evaluator. Time and dimension string columns are cloned into `EvalContext`. `with_raw_properties()` clones the full `EvalContext` and actuals context for each child evaluator.

### Expected Improvement

Immutable formula input data should be shared by reference. Child evaluators should clone only cheap shared references while keeping evaluator-local mutable state separate.

### Proposed Solution

Refactor `FormulaEvaluator` to store `Arc<EvalContext>`. Move immutable input columns, time values, dimension strings, and item date ranges into shared data. Keep warnings, actuals pointer, property filter context, prior state, and integer-result flags local to each evaluator.

### Impact

- Reduces formula setup clone cost.
- Reduces raw-property child evaluator clone cost.
- Prepares formula evaluation for shared execution snapshots.
- Preserves current formula outputs and external behavior.

### Acceptance Criteria

- FormulaEvaluator input context can be shared by `Arc`.
- `with_raw_properties()` does not clone the full `EvalContext`.
- Existing numeric input columns are not cloned into evaluator setup where shared storage is available.
- Dimension/time string columns are not cloned into evaluator setup where shared storage is available.
- Formula outputs remain owned where necessary.
- Property filtering behavior remains unchanged.
- Raw property behavior inside `IF`, `MAX`, `MIN`, and `MOD` remains unchanged.
- Time functions and sequential-related formula behavior remain unchanged.
- Existing formula, calculation, and sequential tests pass.
- Runtime counters or benchmarks show reduced formula context clone cost.

### Notes / Dependencies / Risks

Dependencies:

- Depends on Source Issue 7 indexed column access.
- Works best after Source Issue 11 shared Arc column storage.
- Related to Source Issue 8 clone reduction.

Risks:

- Mutable evaluator state must not be placed in shared context.
- Formula output still returns owned vectors, which may leave one clone boundary until a later expression-value refactor.
