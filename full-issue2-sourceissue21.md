# Source Issue 21 Analysis - Structured Typed IDs And Gradual Migration

> Code branch analyzed: `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`
>
> This document is based on the Omni-Calc implementation in that branch only. BLOX-2143 still uses string-backed wrappers and ad-hoc string parsing for core execution identities.

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes. The issue is valid, but it must be scoped as a gradual migration.

The current Rust omni-calc code encodes semantic IDs into strings such as `ind123`, `prop456`, `b789`, `block789___ind123`, and `dim111___prop222`. Existing wrappers provide type labels, but they still store strings and do not represent semantic numeric IDs.

This issue is strong enough for Jira, but not as a single big-bang rewrite. The first ticket should add typed ID structures, centralized parsing/formatting helpers, tests, and migrate one narrow internal path.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline
```

Current wrappers still store strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/types.rs
```

```rust
pub struct BlockId(pub String);
pub struct IndicatorId(pub String);
pub struct PeriodId(pub String);
```

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs
```

```rust
pub struct ColumnId(pub String);
```

The executor repeatedly parses and formats semantic IDs from strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
```

Examples:

```rust
let dim_key = format!("d{}", block_dim.id);
let col_name = format!("_{}", linked_dim_id_str);
let ind_id: i64 = node_id.strip_prefix("ind").and_then(|s| s.parse().ok()).unwrap_or(0);
let prop_node_id = format!("prop{}", prop_id);
let ref_name = format!("dim{}___prop{}", dim_id, prop_id);
let dim_key_lookup = format!("d{}", prop_spec.dimension_id);
```

Cross-object and dimension-property references are parsed from string formats:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs
```

```rust
pub fn parse_cross_object_ref(reference: &str) -> Option<CrossObjectRef> {
    let parts: Vec<&str> = reference.split("___").collect();
    let block_id = block_part.strip_prefix("block")?;
    let block_key = format!("b{}", block_id);
    ...
}

pub fn parse_dim_prop_ref(reference: &str) -> Option<DimPropRef> {
    let parts: Vec<&str> = reference.split("___").collect();
    let dim_id: i64 = parts[0].strip_prefix("dim")?.parse().ok()?;
    ...
}
```

Resolver node-map keys are string-based:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
```

```rust
struct NodeMapKey {
    pub source_block_key: String,
    pub target_block_key: String,
    pub variable_name: String,
}
```

Plan structures also store string variable names and block keys:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs
```

Examples in comments and fields:

```rust
// variable name from formula, e.g. "block39951___ind259068"
// variable filters keyed by variable name
```

### Affected Files And Functions

Primary affected files:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/types.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/integration/calc_plan.rs
```

Candidate migration areas:

- node ID parsing (`ind*`, `prop*`)
- block key parsing (`b*`, `d*`)
- column identity (`Indicator`, `Dimension`, `Property`, `ConnectedDimension`)
- cross-object reference parsing (`block{id}___ind{id}`)
- dimension property parsing (`dim{id}___prop{id}`)
- resolver `NodeMapKey`
- future `ColumnStore` key type

### Estimated Impact

Performance impact:

- Medium. Typed IDs can reduce repeated parsing and string formatting in hot paths.

Maintainability impact:

- High. Semantic typed IDs make code clearer and reduce ad-hoc string parsing.

Scalability impact:

- Medium to high as model complexity grows, especially with cross-object references and future scheduler graph construction.

Correctness impact:

- Potentially positive long term because centralized parsing reduces inconsistent behavior.
- Migration risk is high if done too broadly at once.

## 2. Simple Explanation

The engine often hides real IDs inside text.

For example:

```text
"ind123" means indicator 123
"b456" means block 456
"dim10___prop20" means property 20 on dimension 10
```

The code then has to keep pulling those numbers back out of strings.

Typed IDs make those meanings explicit:

```text
IndicatorId(123)
BlockId(456)
ColumnId::Property { dimension_id: 10, property_id: 20 }
```

The engine can still convert back to the same strings when building output or reading formulas.

## 3. Technical Explanation

### Current Behavior

Semantic identity is encoded into strings across executor, resolver, formula dependency handling, and plan structures. The code uses `format!`, `strip_prefix`, `split`, `starts_with`, and `contains` in many execution paths.

Existing wrappers like `BlockId(pub String)` and `ColumnId(pub String)` do not solve this. They still carry string-form IDs and still require parsing/formatting elsewhere.

### Root Cause

The Rust engine inherited external string formats as internal execution identity. There is no central typed ID layer that understands:

- block IDs
- dimension IDs
- indicator IDs
- property IDs
- item IDs
- scenario IDs
- structured column identity
- cross-object references

### Why This Is A Problem

Ad-hoc string identity has several costs:

- repeated parsing
- repeated formatting
- repeated string hashing and comparison
- risk of inconsistent parsing rules
- harder future scheduler graph construction
- unclear boundaries between external names and internal identity

Typed IDs give the engine a stable internal representation while keeping external strings unchanged.

## 4. Proposed Fix

### Recommended Implementation Approach

Add semantic numeric typed IDs and centralized parse/format helpers first. Then migrate one narrow hot path.

Recommended new module:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/ids.rs
```

Example types:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct BlockId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct IndicatorId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct DimensionId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct PropertyId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct NodeId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ScenarioId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ItemId(pub i64);
```

Add structured column identity:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ColumnId {
    Indicator(IndicatorId),
    Dimension(DimensionId),
    Property {
        dimension_id: DimensionId,
        property_id: PropertyId,
    },
    CrossObjectIndicator {
        block_id: BlockId,
        indicator_id: IndicatorId,
    },
    ConnectedDimension {
        source_dimension_id: DimensionId,
        linked_dimension_id: DimensionId,
    },
}
```

Add conversion helpers:

```rust
impl ColumnId {
    pub fn to_external_name(self) -> String {
        match self {
            ColumnId::Indicator(id) => format!("ind{}", id.0),
            ColumnId::Dimension(id) => format!("_{}", id.0),
            ColumnId::Property { dimension_id, property_id } => {
                format!("dim{}___prop{}", dimension_id.0, property_id.0)
            }
            ColumnId::CrossObjectIndicator { block_id, indicator_id } => {
                format!("block{}___ind{}", block_id.0, indicator_id.0)
            }
            ColumnId::ConnectedDimension { linked_dimension_id, .. } => {
                format!("_{}", linked_dimension_id.0)
            }
        }
    }
}
```

Centralize parsing:

```rust
pub fn parse_block_key(value: &str) -> Option<BlockId> {
    value.strip_prefix('b')?.parse::<i64>().ok().map(BlockId)
}

pub fn parse_dimension_key(value: &str) -> Option<DimensionId> {
    value.strip_prefix('d')?.parse::<i64>().ok().map(DimensionId)
}

pub fn parse_indicator_column(value: &str) -> Option<IndicatorId> {
    value.strip_prefix("ind")?.parse::<i64>().ok().map(IndicatorId)
}

pub fn parse_dimension_column(value: &str) -> Option<DimensionId> {
    value.strip_prefix('_')?.parse::<i64>().ok().map(DimensionId)
}

pub fn parse_property_column(value: &str) -> Option<(DimensionId, PropertyId)> {
    let (dim_part, prop_part) = value.split_once("___")?;
    let dim_id = dim_part.strip_prefix("dim")?.parse::<i64>().ok()?;
    let prop_id = prop_part.strip_prefix("prop")?.parse::<i64>().ok()?;
    Some((DimensionId(dim_id), PropertyId(prop_id)))
}

pub fn parse_cross_object_ref(value: &str) -> Option<(BlockId, IndicatorId)> {
    let (block_part, ind_part) = value.split_once("___")?;
    let block_id = block_part.strip_prefix("block")?.parse::<i64>().ok()?;
    let indicator_id = ind_part.strip_prefix("ind")?.parse::<i64>().ok()?;
    Some((BlockId(block_id), IndicatorId(indicator_id)))
}
```

Migrate one internal path first. Good first candidates:

1. node classification in `executor.rs`
2. `parse_cross_object_ref` return type
3. resolver `NodeMapKey`
4. future `ColumnStore` internal key

Example typed resolver key:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct TypedNodeMapKey {
    source_block_id: BlockId,
    target_block_id: BlockId,
    variable: ColumnId,
}
```

Keep string compatibility at boundaries:

```rust
// Formula parser input remains string.
// RecordBatch field names remain string.
// API output remains string.
// Warnings and errors remain readable strings.
```

### Recommended Implementation Order

1. Add semantic ID module.
2. Add parse and external-name conversion helpers.
3. Add unit tests for all supported formats.
4. Replace scattered parsing in one narrow path with helper functions.
5. Add typed `ColumnId` behind `ColumnStore` as a documented migration path.
6. Migrate resolver `NodeMapKey` in a follow-up if tests are stable.
7. Do not change external output names.

### Risks, Tradeoffs, And Alternatives

Risks:

- A broad migration can introduce subtle correctness bugs.
- Formula syntax still uses strings, so conversion boundaries must be explicit.
- Existing tests may assert exact strings.

Tradeoff:

- Typed IDs are cleaner than string interning but require more code changes.

Alternative:

- Use string interning first for a smaller performance win, then typed IDs for semantic cleanup.

Recommendation:

- Add typed ID primitives and central helpers first, then migrate path-by-path.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

The engine currently stores important identities as formatted text and repeatedly converts between text and numbers. This issue introduces clearer internal IDs so the engine does less parsing and becomes easier to maintain.

### Developer-Friendly Explanation

Current execution uses string encodings like `ind123`, `b456`, and `dim1___prop2` as internal identity. Existing newtypes still wrap strings. Add numeric typed IDs and structured `ColumnId`, centralize parsing/formatting, and migrate internal paths gradually while preserving external string names.

### Suggested Meeting Talking Points

- Do not make this a big-bang rewrite.
- First PR should add types/helpers/tests and migrate one path.
- External formula syntax and RecordBatch schema must not change.
- This complements, but does not duplicate, Source Issue 13 string interning.
- Typed IDs support future scheduler graph construction.

## 6. Suggested Diagrams And Documents

### Diagrams

Create an identity conversion boundary diagram:

```text
External strings
  formula: block123___ind456
  output: ind456
       |
       v
Parsing boundary
       |
       v
Internal typed IDs
  BlockId(123)
  IndicatorId(456)
  ColumnId::CrossObjectIndicator { ... }
       |
       v
Output boundary converts back to strings
```

Create a migration diagram:

```text
Step 1: typed ID module
Step 2: parse/to_external_name helpers
Step 3: one hot path migrated
Step 4: ColumnStore typed key support
Step 5: resolver/node-map typed keys
Step 6: broader scheduler graph use
```

### Supporting Documents

Prepare:

- supported current string formats
- parse/to-external-name roundtrip table
- list of external compatibility boundaries
- migration plan by module
- benchmark/correctness checklist

## 7. Jira-Ready Issue

### Title

Introduce Structured Node, Block, Dimension, Property, and Column IDs for Gradual Execution Migration

### Background / Problem Statement

The Rust omni-calc engine encodes semantic identity into strings such as `ind123`, `prop456`, `b789`, `block789___ind123`, and `dim111___prop222`. This causes repeated parsing, formatting, hashing, and comparison across executor, resolver, formula handling, joins, dependency planning, and future graph construction.

### Current Behavior

Existing ID wrappers still store strings. Many execution paths use ad-hoc `format!`, `strip_prefix`, `split`, and `starts_with` logic. Resolver node-map keys and column references are string-based.

### Expected Improvement

The engine should have semantic numeric typed IDs and structured column identity for internal use, while preserving existing external string names for formulas, RecordBatch fields, API output, warnings, and errors.

### Proposed Solution

Add typed ID structs for block, indicator, dimension, property, node, scenario, and item IDs. Add a structured `ColumnId` enum. Centralize parsing and `to_external_name()` helpers for all current formats. Migrate one narrow internal path first and document the broader migration plan.

### Impact

- Reduces repeated parsing and formatting.
- Improves type safety and maintainability.
- Supports future scheduler graph and resolver refactors.
- Keeps external behavior unchanged.

### Acceptance Criteria

- Semantic typed IDs exist for block, indicator, dimension, property, node, scenario, and item IDs.
- Structured `ColumnId` exists for indicator, dimension, property, cross-object indicator, and connected dimension identity.
- Parsing helpers centralize current string parsing behavior.
- External name conversion helpers preserve current output names exactly.
- Unit tests cover parse/to-external-name roundtrips for all supported formats.
- At least one internal path uses typed IDs instead of ad-hoc string parsing.
- ColumnStore typed-key migration path is documented.
- Resolver/node-map typed-ID migration path is documented or partially implemented.
- Debug/error/warning messages remain readable.
- Existing serial outputs remain unchanged.
- No Python/PyO3 callbacks are introduced.

### Notes / Dependencies / Risks

Dependencies:

- Should follow Source Issue 7.
- Works better after Source Issue 13 if interning is used as a bridge.
- Supports future scheduler graph work.

Risks:

- Broad migration is risky if done all at once.
- Formula parser and external APIs still use strings, so boundaries must be explicit.
- Typed IDs and interned IDs need clear responsibilities.
