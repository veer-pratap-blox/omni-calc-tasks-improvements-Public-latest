# Source Issue 13 Analysis - String Interning For Selected Hot-Path Keys

## 1. Issue Validation

### Is The Issue Valid Based On The Code?

Yes. The issue is valid, but it should be scoped narrowly.

The current Rust omni-calc code uses many repeated string identifiers in hot execution paths: block keys, node IDs, indicator names, property names, cross-object references, join paths, variable names, and column names. There is no production string interner or interned ID type in the active execution code.

This is a performance and maintainability optimization, not a correctness fix. It should be added only for measured high-volume paths, not as a broad rewrite of every string in the engine.

### Evidence From The Codebase

Current branch inspected:

```text
BLOX-2053-navbar-dropdown-backend-readiness-persist-model-banner-image-and-provide-lightweight-blocks-listing-for-navigation
```

Search evidence:

```text
No StringInterner found.
No InternedId found.
No production interned-key lookup path found.
```

Current lightweight wrappers still store strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs
```

```rust
pub struct ColumnId(pub String);
```

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/types.rs
```

```rust
pub struct BlockId(pub String);
pub struct IndicatorId(pub String);
pub struct PeriodId(pub String);
```

The active executor repeatedly formats, clones, parses, and compares strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
```

Examples found:

```rust
step.nodes.iter().filter(|n| n.starts_with("ind"))
let dim_key = format!("d{}", block_dim.id);
let col_name = format!("_{}", linked_dim_id_str);
let ind_id: i64 = node_id.strip_prefix("ind").and_then(|s| s.parse().ok()).unwrap_or(0);
let prop_node_id = format!("prop{}", prop_id);
let ref_name = format!("dim{}___prop{}", dim_id, prop_id);
let dim_key_lookup = format!("d{}", prop_spec.dimension_id);
```

Cross-object reference parsing creates and stores strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs
```

```rust
let parts: Vec<&str> = reference.split("___").collect();
let block_id = block_part.strip_prefix("block")?;
let block_key = format!("b{}", block_id);

Some(CrossObjectRef {
    block_key,
    indicator_col: ind_part.to_string(),
    variable_name: reference.to_string(),
})
```

Resolver node-map keys are string-heavy:

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

Lookup maps use join path strings:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
```

```rust
pub type LookupMap = HashMap<String, f64>;
lookup.insert(path.clone(), value);
*lookup.entry(path.clone()).or_insert(0.0) += value;
```

Join path creation builds strings for row matching:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
```

```rust
values.join(JOIN_PATH_SEPARATOR)
```

### Affected Files And Functions

Primary affected files:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/executor.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/state.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/parser.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/steps/sequential.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/calc_buffers/columns.rs
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/types.rs
```

Candidate hot paths:

- `ColumnStore` key lookup after Source Issue 7.
- `NodeMapKey` resolver lookup.
- cross-object reference lookup.
- calculated block lookup.
- formula existing-column lookup.
- node classification for `ind*` and `prop*`.
- lookup map construction and alignment.

### Estimated Impact

Performance impact:

- Medium for string-heavy, cross-object-heavy, lookup-heavy, and wide-column models.
- The improvement depends on applying interning only to measured hot paths.

Memory impact:

- Medium. Interning can reduce duplicate string allocations for repeated identifiers.

Maintainability impact:

- Medium. A scoped interner can centralize key identity, but overuse can make code harder to read.

Correctness impact:

- No direct correctness change expected.
- Risk is accidentally exposing interned IDs in output/debug text or comparing IDs from different interner scopes.

## 2. Simple Explanation

The engine uses a lot of repeated text keys, like `"ind123"`, `"b456"`, and `"block456___ind123"`.

Today those strings are copied, hashed, compared, and sometimes parsed again and again.

String interning means we store each repeated string once and use a small numeric handle internally.

In simple terms:

```text
Before:
Compare and hash full strings repeatedly.

After:
Convert repeated strings once, then compare small IDs.
```

The API output and logs can still show the original strings.

## 3. Technical Explanation

### Current Behavior

The executor uses string identifiers throughout hot paths. Existing wrappers like `BlockId(pub String)` and `ColumnId(pub String)` give some type labeling but do not reduce string allocation, hashing, comparison, or parsing cost.

Cross-object references are parsed from strings. Resolver keys store strings. Lookup maps store join path strings. Column lookup uses string names. Step classification repeatedly checks string prefixes.

### Root Cause

The engine has no interned identity layer. Every component carries its own owned strings and repeated operations happen in loops.

### Why This Is A Problem

String operations are not the only bottleneck, but in wide or cross-object-heavy models they can add overhead:

- repeated allocation
- repeated hashing
- repeated string cloning
- repeated prefix checks and parsing
- larger HashMap keys

Interning can reduce that overhead for selected paths.

## 4. Proposed Fix

### Recommended Implementation Approach

Add a small scoped interner and apply it to one or two measured hot paths first.

Recommended new module:

```text
/Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI/omni-calc/src/engine/exec/interner.rs
```

Example implementation:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct InternedId(u32);

#[derive(Debug, Default)]
pub struct StringInterner {
    strings: Vec<String>,
    index: HashMap<String, InternedId>,
}

impl StringInterner {
    pub fn intern(&mut self, value: &str) -> InternedId {
        if let Some(id) = self.index.get(value).copied() {
            return id;
        }

        let id = InternedId(self.strings.len() as u32);
        self.strings.push(value.to_string());
        self.index.insert(value.to_string(), id);
        id
    }

    pub fn resolve(&self, id: InternedId) -> Option<&str> {
        self.strings.get(id.0 as usize).map(|s| s.as_str())
    }

    pub fn contains(&self, value: &str) -> bool {
        self.index.contains_key(value)
    }

    pub fn len(&self) -> usize {
        self.strings.len()
    }
}
```

Use it during plan preparation or execution context creation:

```rust
pub struct ExecutionInterners {
    pub columns: StringInterner,
    pub block_keys: StringInterner,
    pub node_ids: StringInterner,
    pub variables: StringInterner,
}
```

Apply first to `ColumnStore` keys:

```rust
pub struct InternedColumnStore<T> {
    columns: Vec<(InternedId, String, Arc<[T]>)>,
    index: HashMap<InternedId, usize>,
}
```

Or keep string compatibility while adding interned lookup:

```rust
pub struct ColumnStore<T> {
    columns: Vec<(String, Arc<[T]>)>,
    string_index: HashMap<String, usize>,
    interned_index: HashMap<InternedId, usize>,
}
```

Apply to resolver node-map keys:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct InternedNodeMapKey {
    source_block_key: InternedId,
    target_block_key: InternedId,
    variable_name: InternedId,
}
```

Preserve string boundaries:

```rust
// External output should still use original strings.
let external_name = interner.resolve(column_id).unwrap_or("<unknown>");
```

### Recommended Implementation Order

1. Add `StringInterner` and `InternedId`.
2. Add unit tests for intern/resolve/duplicate/missing/stable IDs.
3. Add execution-scoped interner storage.
4. Intern plan-level hot keys once during `ExecutionContext::new`.
5. Apply interned IDs to one measured hot path, preferably `ColumnStore` lookup after Source Issue 7.
6. Apply to resolver `NodeMapKey` as a second step.
7. Add counters for string intern hit count and string lookup time.
8. Confirm output names and warning text remain unchanged.

### Risks, Tradeoffs, And Alternatives

Risks:

- IDs from different interner instances must not be compared.
- Over-interning can add complexity without measurable improvement.
- Debugging can become harder if interned IDs leak into logs.

Tradeoff:

- String interning is a bridge optimization. It does not replace semantic typed IDs from Source Issue 21.

Alternative:

- Skip interning and move directly to typed IDs. That may be cleaner long term but riskier because typed ID migration is broader.

Recommendation:

- Add interning only for selected high-volume string keys and keep original strings at external boundaries.

## 5. How To Explain To The Team

### Manager-Friendly Explanation

The engine repeatedly handles the same text labels during calculation. This issue reduces that overhead by storing repeated labels once and using small internal IDs. It should help performance without changing user-facing output.

### Developer-Friendly Explanation

Current hot paths use `String` keys for column lookup, resolver node maps, cross-object references, and lookup maps. Add a scoped `StringInterner` and apply it to selected measured paths. Keep external names available for RecordBatch fields, API output, warnings, and debug logs.

### Suggested Meeting Talking Points

- This is not a full typed ID migration.
- Scope should be limited to one or two measured hot paths first.
- Do not intern every string in the system.
- Original strings must remain available at all external boundaries.
- Compare benchmark metrics before/after to decide whether to expand.

## 6. Suggested Diagrams And Documents

### Diagrams

Create an identity-boundary diagram:

```text
External formula/output strings
  -> intern once
    -> hot path uses InternedId
      -> resolve back to string for output/logs
```

Create a before-vs-after HashMap key diagram:

```text
Before:
HashMap<String, usize>

After:
HashMap<InternedId, usize>
StringInterner keeps InternedId -> original name
```

### Supporting Documents

Prepare:

- list of candidate strings to intern
- benchmark results for chosen hot path
- rules for interner lifetime and scope
- list of external boundaries where strings must be preserved

## 7. Jira-Ready Issue

### Title

Replace Selected Hot-Path String Lookups With Interned IDs

### Background / Problem Statement

The Rust omni-calc executor repeatedly clones, hashes, compares, formats, and parses string identifiers such as block keys, node IDs, column names, variable names, and cross-object references. These repeated string operations add overhead in wide, lookup-heavy, and cross-object-heavy models.

### Current Behavior

There is no production string interner. Existing ID wrappers still store strings. Hot paths use `HashMap<String, ...>`, `String` join paths, string node-map keys, and repeated prefix parsing.

### Expected Improvement

Selected high-volume keys should be interned once per execution so hot paths can use small copyable IDs while preserving original strings for output, warnings, errors, and debug logs.

### Proposed Solution

Add a scoped `StringInterner` and `InternedId`. Apply interning to one measured hot path first, such as `ColumnStore` lookup or resolver `NodeMapKey`, then expand only if benchmarks justify it.

### Impact

- Reduces repeated string allocation and cloning.
- Reduces HashMap key size and comparison cost in selected hot paths.
- Creates a bridge toward later semantic typed IDs.
- Keeps external output unchanged.

### Acceptance Criteria

- `StringInterner` and `InternedId` exist.
- Unit tests cover intern, resolve, duplicate intern, missing resolve, and stable IDs.
- Interning is applied to at least one measured hot path.
- Original string names are preserved for RecordBatch fields, API output, warnings, errors, and debug logs.
- Tests confirm output column names remain unchanged.
- Benchmarks or runtime metrics show improvement or document a neutral result.
- Migration notes identify remaining string-heavy paths and how this relates to Source Issue 21.

### Notes / Dependencies / Risks

Dependencies:

- Should follow Source Issue 7.
- Works better after Source Issues 11 and 8.
- Related to Source Issue 21 typed IDs but not a replacement for it.

Risks:

- Interner IDs must be scoped carefully.
- Over-interning may add complexity without meaningful gain.
- Interned IDs must not leak into external contracts.
