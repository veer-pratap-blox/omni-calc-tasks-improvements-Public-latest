# Final Merged Cache Performance Issues for Omni-Calc

Date: 2026-05-06

Source cache files reviewed:

- `/Users/veerpratapsingh/Downloads/cache-issues/29-april-omni-calc-CACHE-improvements.md`
- `/Users/veerpratapsingh/Downloads/cache-issues/cache improvements in omni calc.md`

Existing omni-calc task files compared:

- `/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/29-april-omni-calc-parallelisation-FINAL.md`
- `/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/29-april-omni-calc-parallelisation-FINAL-MERGED.md`
- `/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/29-april-omni-calc-parallelisation-FINAL-MERGED1.MD`
- `/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/29-april-omni-calc-parallelisation-FINAL-CODE-VERIFIED.md`
- `/Users/veerpratapsingh/Downloads/omni-calc-tasks-improvements-main/benchmark-jira-issue.md`

Current code paths checked:

- `modelAPI/services/model_metadata_cache_v4.py`
- `modelAPI/calc_engine/rust_bridge.py`
- `modelAPI/calc_engine/dag_manager_v4.py`
- `modelAPI/resources/block_kpi_v4_rust.py`
- `modelAPI/resources/model_data_values_rust.py`
- `modelAPI/omni-calc/src/engine/exec/preload.rs`
- `modelAPI/omni-calc/src/engine/exec/context.rs`
- `modelAPI/omni-calc/src/engine/exec/executor.rs`
- `modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs`
- `modelAPI/omni-calc/src/engine/exec/get_source_data/resolver.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/join_path.rs`
- `modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs`

---

## Validation Summary

### Valid Unique Issues Kept

These issues are real in the current codebase, performance-relevant, and not already fully represented as standalone omni-calc task tickets.

| Final Issue | Source Coverage | Validation |
|---|---|---|
| Issue 1: Define safe invalidation token for cross-request cache reuse | `cache improvements in omni calc.md` issue 1, supports cache file issue 4/7/9 | Valid. Existing metadata cache is rebuilt per request and DB tables do not all expose direct revision fields. Any cross-request cache needs this first. |
| Issue 2: Add optional process-level metadata snapshot cache | `cache improvements in omni calc.md` issue 2 | Valid. `ModelMetadataCacheV4` is instantiated and loads all metadata per request; no process-level reusable snapshot is present. |
| Issue 3: Add O(1) derived lookup indexes in `ModelMetadataCacheV4` | `cache improvements in omni calc.md` issue 3 | Valid. `get_block_indicators`, `find_dimension_property_by_id`, and `find_items_by_property_id_value` still scan dictionaries/lists. |
| Issue 4: Normalize parsed property metadata once in `ModelMetadataCacheV4` | `cache improvements in omni calc.md` issue 4 | Valid. `rust_bridge.py`, `dag_manager_v4.py`, and `block_kpi_v4_rust.py` repeatedly parse `data_format` JSON to read linked dimensions. |
| Issue 5: Spike safe calc-plan/result caching with memory bounds | `29-april-omni-calc-CACHE-improvements.md` issues 4, 7, 9 | Valid as a spike only. No final-result or calc-plan cache exists, but implementation is unsafe until invalidation and memory budgets are defined. |
| Issue 6: Reduce preload fan-out string cloning in Rust property preload | `cache improvements in omni calc.md` issue 9 | Valid and narrow. `preload.rs` clones property values for every target triple even when only one target exists. |
| Issue 7: Validate and narrow all-scenarios property loading | `cache improvements in omni calc.md` issue 10 | Valid as a spike. `ModelMetadataCacheV4` intentionally loads all `DimItemProperties` scenarios for a model, which can inflate setup cost and memory. |

### Removed Because Already Covered

| Removed Issue | Source File | Reason |
|---|---|---|
| Avoid rebuilding same raw property maps during Rust execution | `29-april-omni-calc-CACHE-improvements.md` issue 1 | Already implemented in current code through `ExecutionContext.string_property_map_cache` and `ExecutionContext.numeric_property_map_cache`. The remaining aligned-column cache concern is already covered in `FINAL-CODE-VERIFIED.md`. |
| Demote hot-path `warn!` logs and guard expensive log construction | `29-april-omni-calc-CACHE-improvements.md` issue 2 | Already covered by benchmark/perf tracing work and `FINAL-CODE-VERIFIED.md` logging issue. |
| Optimize `preload_connected_dimensions` iteration/allocation | `29-april-omni-calc-CACHE-improvements.md` issue 3 | Already covered by connected-dimension preload and pre/post-processing issues in the omni-calc task files. |
| Parallelize independent nodes inside calc planner execution steps | `29-april-omni-calc-CACHE-improvements.md` issue 5 | Already covered by ready-node/Rayon scheduler issues. Not a cache-specific remaining issue. |
| Reduce Arrow `RecordBatch` construction clones and allocation overhead | `29-april-omni-calc-CACHE-improvements.md` issue 6 | Already covered by ColumnStore, shared column storage, clone reduction, and resolver materialization issues. |
| Add benchmark and correctness harness against development | `29-april-omni-calc-CACHE-improvements.md` issue 8 | Already covered by benchmark runtime tracing work and `benchmark-jira-issue.md`. |
| Add per-execution row-aligned property-column cache | `cache improvements in omni calc.md` issue 5 | Valid but already captured in `FINAL-CODE-VERIFIED.md` as "Property caches store maps, not target-aligned columns." |
| Cache cross-object join artifacts per execution | `cache improvements in omni calc.md` issue 6 | Valid but already covered by cross-object join path/lookup/key representation issues. |
| Replace full resolver `RecordBatch` rebuilds with incremental cached representation | `cache improvements in omni calc.md` issue 7 | Valid but already covered by dependency-aware resolver refresh and shared RecordBatch/materialization issues. |
| Cache connected-dimension alignments per block/execution | `cache improvements in omni calc.md` issue 8 | Valid but already covered by connected-dimension preload and join/alignment optimization issues. |

### Removed or Rewritten Because Too Broad / Unsafe

| Original Idea | Reason |
|---|---|
| Implement final calculation result caching directly | Rewritten as Issue 5 spike. Result caching is performance-relevant but unsafe without dependency fingerprints, invalidation, memory limits, and parity tests. |
| Add unbounded in-memory caches for metadata/results | Rewritten with bounded cache and memory-budget requirements. Unbounded caches are not acceptable for large customer models. |
| Cache data based only on model/scenario/block IDs | Invalid on its own. Cache keys must include a correctness token that changes when dimensions, properties, inputs, formulas, scenarios, dependent blocks, or request shape changes. |

### Missing Items Added During Code Validation

The cache source files were generally directionally correct, but the current codebase shows these missing requirements:

- Cross-request caches must expose hit/miss/eviction/staleness metrics.
- Metadata snapshot cache must not hand mutable cache dictionaries to request code.
- Derived metadata indexes should be built once during `_process_results`, not lazily in hot accessors.
- Property metadata normalization should include parsed linked dimension IDs and parse-error handling.
- Any final result or calc-plan cache must include request-shape inputs such as block selection, requested indicators, time range, scenario, filters, actuals/forecast mode, and dependent block fingerprints.
- Rust preload clone reduction should preserve Rust-owned strings and avoid Python-borrowed lifetime complexity.

---

# Final Unique Cache Issues

## Issue 1: Define Safe Invalidation Token for Cross-Request Cache Reuse

### Context / Problem

`ModelMetadataCacheV4` loads model metadata per request, but not all underlying metadata tables expose direct revision or timestamp columns. The current code confirms direct timestamp availability for high-level tables like `Models`, `ModelScenarios`, and `Blocks`, while metadata-heavy tables such as dimensions, dimension items, dimension properties, item properties, item scenarios, and data inputs do not provide a simple revision token in the cache layer.

Cross-request caching is therefore unsafe unless we can compute a deterministic invalidation token that changes whenever any relevant metadata or input source changes.

### Goal

Define a correctness-safe invalidation-token contract for future metadata, plan, and result caches.

This issue is a prerequisite for any cross-request cache implementation.

### Current Gap in Code

- `ModelMetadataCacheV4` is instantiated per request and immediately runs metadata queries.
- There is no shared invalidation token for cache reuse.
- Existing cache keys would be unsafe if they only used `(model_id, scenario_id)` or `(model_id, scenario_id, block_id)`.
- Query 8 loads `DimItemProperties` for all scenarios of a model, which makes invalidation more complex.

### Required Changes

1. Audit real DB fields available for invalidation:
   - `Models`
   - `ModelScenarios`
   - `Blocks`
   - `Indicators`
   - `Dimensions`
   - `DimensionItems`
   - `DimensionProperties`
   - `DimItemProperties`
   - `DimItemScenarios`
   - `DataInputs`
   - Any table used by model filters, actuals, or formula inputs.

2. Define a deterministic cache token using real schema fields only.

3. For tables without revision/timestamp fields, define one of:
   - deterministic aggregate fingerprint,
   - count plus max known timestamp from related mutation tables,
   - explicit unsupported mutation list,
   - schema change proposal to add revision columns.

4. Include request-shape inputs in token design where needed:
   - model ID,
   - scenario ID,
   - block IDs,
   - requested indicator IDs,
   - time range,
   - filters,
   - actuals/forecast mode,
   - connected dimension/property dependencies,
   - cross-block dependency fingerprints.

5. Add a mutation test matrix proving the token changes when relevant metadata changes.

6. Document unsupported or ambiguous invalidation paths.

### Dependencies

None.

This must be completed before cross-request metadata snapshot caching, plan caching, or result caching.

### Risks / Tradeoffs

- Fingerprints can be expensive if they scan large tables.
- Too narrow a token can serve stale data.
- Too broad a token can invalidate too often and reduce cache value.
- Adding DB revision columns may be the cleanest long-term solution but is larger in scope.

### Acceptance Criteria

- A documented invalidation-token spec exists with exact SQL sources.
- Token inputs use real schema fields only.
- Mutation tests cover model, scenario, block, indicator, dimension, property, item, input, actuals, and filter-relevant changes.
- Unsupported mutation paths are explicitly documented.
- Cache consumers can depend on the token without guessing table-level freshness.

---

## Issue 2: Add Optional Process-Level Metadata Snapshot Cache

### Context / Problem

`ModelMetadataCacheV4` currently loads metadata for each request using parallel queries. This avoids N+1 query behavior inside one request, but repeated requests for the same model/scenario still rebuild the same metadata snapshot and repeat JSON processing.

This is visible in the current code because `ModelMetadataCacheV4.__init__` calls `_load_all_metadata()`, which builds and runs the full query set immediately.

### Goal

Add an optional bounded process-level metadata snapshot cache for repeated model/scenario requests, keyed by the invalidation token from Issue 1.

### Current Gap in Code

- No process-level LRU cache exists for `ModelMetadataCacheV4` snapshots.
- Snapshot dictionaries are mutable instance fields.
- Request setup code repeatedly constructs `ModelMetadataCacheV4` in Rust endpoint paths.
- There is no cache hit/miss/eviction visibility.

### Required Changes

1. Add an opt-in metadata snapshot cache behind a feature/config flag.

2. Cache key:

```text
(model_id, scenario_id, invalidation_token)
```

3. Cache value:
   - immutable or copy-on-write metadata snapshot,
   - no request-specific mutable state,
   - no live DB/session objects,
   - no Python objects that can be mutated by downstream code.

4. Add bounded eviction:
   - max entry count,
   - max estimated bytes,
   - optional TTL,
   - explicit invalidation on token change.

5. Add observability:
   - hit count,
   - miss count,
   - eviction count,
   - token mismatch count,
   - estimated snapshot size.

6. Integrate only at the metadata cache creation boundary:
   - `model_metadata_cache_v4.py`,
   - `block_kpi_v4_rust.py`,
   - `model_data_values_rust.py`.

7. Preserve current per-request behavior when the feature flag is disabled.

### Dependencies

- Depends on Issue 1 invalidation token.

### Risks / Tradeoffs

- Stale metadata would create correctness bugs.
- Shared mutable dictionaries can leak request-side mutations between requests.
- Large models can increase worker memory if cache limits are too loose.

### Acceptance Criteria

- Repeated unchanged requests hit the metadata snapshot cache.
- Metadata changes produce a cache miss via token change.
- Cache can be disabled without behavior changes.
- Output parity matches the uncached path.
- Cache metrics are visible in logs or diagnostics.
- Memory bounds and eviction behavior are documented and tested.

---

## Issue 3: Add O(1) Derived Lookup Indexes in `ModelMetadataCacheV4`

### Context / Problem

`ModelMetadataCacheV4` stores metadata dictionaries, but several hot accessor methods still perform linear scans.

Current code examples:

- `get_block_indicators(block_id)` scans all indicators.
- `find_dimension_property_by_id(prop_id)` scans all dimension property lists.
- `find_items_by_property_id_value(property_id, value)` scans all item property cache entries and parses composite string keys.

These calls happen during DAG construction, payload preparation, filter handling, and Rust bridge setup.

### Goal

Build derived lookup indexes once during `_process_results()` so repeated metadata access is O(1) or close to O(1).

### Current Gap in Code

- `_process_results()` builds base caches but not all lookup indexes.
- Accessors repeatedly scan full dictionaries/lists.
- Composite string keys are split repeatedly for property searches.

### Required Changes

1. Add derived maps built during `_process_results()`:

```python
_indicators_by_block_id: Dict[int, List[Dict]]
_dimension_property_by_id: Dict[int, Dict]
_dimension_id_by_property_id: Dict[int, int]
_items_by_property_value: Dict[Tuple[int, str], List[ItemPropertyProxyData]]
```

2. Normalize keys to integers where internal Python code benefits from it, while preserving external response shape.

3. Replace accessor scans:
   - `get_block_indicators`,
   - `find_dimension_property_by_id`,
   - `find_items_by_property_id_value`.

4. Avoid repeated parsing of composite item-property keys in accessor methods.

5. Add tests proving returned values match the existing scan-based behavior.

6. Include benchmark or trace evidence on a model with many indicators/properties.

### Dependencies

None.

This can be implemented before cross-request metadata caching because it improves each request independently.

### Risks / Tradeoffs

- Extra indexes increase memory per request.
- Indexes must be rebuilt whenever the metadata snapshot is rebuilt.
- Case-insensitive property value matching must preserve current behavior.

### Acceptance Criteria

- Target accessors no longer scan the full cache on each call.
- Returned values match current behavior exactly.
- Index construction happens once per metadata snapshot.
- Tests cover duplicate property values, missing properties, and case-insensitive lookup.
- Performance evidence shows reduced CPU in metadata setup or filter/property lookup paths.

---

## Issue 4: Normalize Parsed Property Metadata Once in `ModelMetadataCacheV4`

### Context / Problem

The current code repeatedly parses dimension property `data_format` JSON to discover linked dimensions.

Code validation found repeated parsing in:

- `modelAPI/calc_engine/rust_bridge.py`
- `modelAPI/calc_engine/dag_manager_v4.py`
- `modelAPI/resources/block_kpi_v4_rust.py`

This creates repeated JSON parsing, repeated branching, and duplicate error-handling logic during DAG/filter extraction and Rust payload assembly.

### Goal

Parse and normalize frequently used property metadata once when `ModelMetadataCacheV4` processes query results.

### Current Gap in Code

- `data_format` is stored as raw JSON/string metadata.
- Callers repeatedly call `json.loads(...)`.
- Linked dimension ID extraction is duplicated in multiple files.
- Parse failures are silently handled differently in different call sites.

### Required Changes

1. During `_process_results()`, normalize each dimension property with derived fields:

```python
{
  "id": ...,
  "dimension_id": ...,
  "name": ...,
  "type": ...,
  "data_format": ...,
  "data_format_parsed": ...,
  "linked_dimension_id": ...,
  "is_dimension_type_property": ...
}
```

2. Add helper accessors:

```python
get_dimension_type_properties(dim_id)
get_linked_dimension_id_for_property(prop_id)
```

3. Update Rust bridge and DAG setup paths to consume normalized fields instead of reparsing `data_format`.

4. Preserve original `data_format` in output payloads where existing consumers expect it.

5. Add consistent parse-error handling:
   - malformed JSON should not crash request setup,
   - parse errors should be counted/logged at debug level,
   - invalid linked dimension values should resolve to `None`.

### Dependencies

- Can be implemented independently.
- Works well with Issue 3 because normalized property metadata can also populate derived indexes.

### Risks / Tradeoffs

- Some callers may expect raw `data_format`; do not remove it.
- Normalization must preserve existing behavior for malformed or missing `data_format`.
- Extra normalized fields slightly increase metadata snapshot size.

### Acceptance Criteria

- Hot setup paths no longer call `json.loads` for the same property metadata repeatedly.
- Linked dimension detection is centralized.
- Rust payloads remain identical except for optional internal derived fields.
- Tests cover valid dimension-type properties, non-dimension properties, missing `data_format`, malformed JSON, and invalid linked dimension IDs.
- Timing or profiling shows reduced setup overhead on property-heavy models.

---

## Issue 5: Spike Safe Calc-Plan and Result Caching With Memory Bounds

### Context / Problem

The current engine can rebuild calculation plans and recompute outputs even when a model/scenario/request shape has not meaningfully changed. A cache could improve repeated request latency, but this is much riskier than metadata lookup caching.

The cache issue files mention:

- DAG/cache of computed calculation plan,
- final calculated result cache,
- block-level versus indicator-level caching,
- memory-bounded caching,
- possible disk/Parquet-backed storage.

The current Rust and Python code does not provide a correctness-safe final result cache.

### Goal

Create a design spike for safe calc-plan and final-result cache reuse. Do not implement production result caching until invalidation and memory constraints are proven.

### Current Gap in Code

- No final result cache exists.
- No calc-plan cache exists.
- No dependency fingerprint exists for result reuse.
- Existing execution mutates block state and resolver state during calculation, so reusable result snapshots need careful ownership boundaries.

### Required Changes

1. Compare cache levels:
   - metadata snapshot cache,
   - calc-plan / DAG step cache,
   - block-level result cache,
   - indicator-level result cache,
   - hybrid cache.

2. Define cache keys for each level.

3. Include correctness inputs:
   - invalidation token from Issue 1,
   - model/scenario/block IDs,
   - indicator formulas and variables,
   - requested indicator set,
   - time range and granularity,
   - filters,
   - actuals/forecast start behavior,
   - connected dimensions and property dependencies,
   - cross-block dependency fingerprints,
   - request flags affecting output shape.

4. Define cache storage options:
   - process memory LRU,
   - Redis,
   - DB-backed metadata,
   - Parquet or disk-backed result snapshots.

5. Define memory controls:
   - max cache bytes,
   - max entry count,
   - TTL,
   - eviction policy,
   - per-worker/process memory budget.

6. Define output parity tests:
   - cached versus uncached response payload hash,
   - row count,
   - column count,
   - warning count,
   - representative indicator totals,
   - connected dimension cases,
   - actuals/forecast cases.

7. Produce recommendation for first implementation phase.

### Dependencies

- Depends on Issue 1.
- Should come after or alongside runtime benchmarking so cache hit/miss impact can be measured.

### Risks / Tradeoffs

- Incorrect invalidation can serve stale model results.
- Indicator-level caching has high dependency complexity.
- Block-level caching is simpler but may invalidate too much.
- Disk-backed caches reduce memory pressure but add I/O and lifecycle complexity.
- Result cache implementation can hide underlying execution regressions if benchmarks are not separated.

### Acceptance Criteria

- A spike document compares block-level, indicator-level, plan-level, and hybrid caching.
- Cache keys and invalidation rules are specified.
- Memory budget and eviction strategy are specified.
- First implementation recommendation is documented.
- No production cache is enabled until parity and invalidation tests pass.

---

## Issue 6: Reduce Preload Fan-Out String Cloning in Rust Property Preload

### Context / Problem

Rust preload builds `PreloadedMetadata.property_maps` keyed by `(dimension_id, property_id, scenario_id)`. During bulk and fallback property loading, the current code inserts property values with `value.clone()` for every target triple.

This is correct but can allocate more than necessary when a property value maps to a single target triple.

### Goal

Reduce avoidable string cloning during preload while keeping preloaded metadata fully Rust-owned.

### Current Gap in Code

In `modelAPI/omni-calc/src/engine/exec/preload.rs`, both bulk and fallback property population paths clone string values when inserting into target property maps.

This is most visible when a single source property row fans out to one target triple. In that case the value could be moved rather than cloned.

### Required Changes

1. Refactor property map insertion so values are moved when there is exactly one target triple.

2. Clone only when one source value fans out to multiple target triples.

3. Keep `PreloadedMetadata` fully Rust-owned:
   - no borrowed Python strings,
   - no lifetime tied to PyO3 objects,
   - no shared mutable Python cache references.

4. Add clone/allocation counters or benchmark evidence before/after.

5. Add tests covering:
   - single-target insertion,
   - multi-target fan-out,
   - missing/null property values,
   - fallback `_item_properties_cache` path if still supported.

### Dependencies

None.

This is a narrow low-risk preload optimization. It is compatible with broader clone-reduction work but does not require ColumnStore first.

### Risks / Tradeoffs

- Refactoring ownership in the fallback PyO3 path must be done carefully.
- Over-optimizing this path may add complexity for small gains on models with little property fan-out.
- Do not introduce borrowed data into `PreloadedMetadata`.

### Acceptance Criteria

- Preload output remains identical.
- Single-target property insertion avoids unnecessary clone.
- Multi-target fan-out remains correct.
- Clone/allocation measurement improves or stays neutral.
- Existing preload and calculation tests pass.

---

## Issue 7: Validate and Potentially Narrow All-Scenarios Property Loading

### Context / Problem

`ModelMetadataCacheV4` intentionally loads `DimItemProperties` for all scenarios in a model:

```sql
WHERE d.model_id = $1
```

The code comment says this preserves Python v2 behavior because some property filters were not scenario-scoped.

This can significantly increase metadata cache size and preload cost for large models with many scenarios.

### Goal

Validate whether all-scenarios item-property loading is still required for Rust endpoints. If not, narrow loading to the active scenario with explicit fallback rules.

### Current Gap in Code

- Query 8 loads item properties for every scenario of the model.
- Preload later selects requested `(item_id, property_id, scenario_id)` triples.
- The current all-scenario cache can create large Python metadata snapshots even when Rust only needs one scenario for the request.
- The parity requirement behind the old behavior is not documented as a test matrix.

### Required Changes

1. Identify every consumer of `_item_properties_cache`:
   - Rust bridge payload creation,
   - DAG/filter extraction,
   - model data values Rust path,
   - block KPI Rust path,
   - legacy Python compatibility paths.

2. Build parity tests for:
   - active scenario property filters,
   - base/reference scenario behavior,
   - connected dimension filters,
   - start/end properties,
   - actuals/forecast imports if relevant,
   - old Python v2-compatible fallback paths.

3. Measure memory and setup time for:
   - all-scenario loading,
   - active-scenario loading,
   - active-scenario plus explicitly needed reference scenario loading.

4. If safe, change query behavior behind a feature flag:

```text
OMNI_CALC_SCENARIO_SCOPED_PROPERTIES=1
```

5. If not safe, document why all-scenario loading is still required and add metrics so its cost is visible.

### Dependencies

- Can be done after Issue 1 if using invalidation token work.
- Can be done independently as a spike if only measuring and documenting behavior.

### Risks / Tradeoffs

- Narrowing too aggressively can break property filters that depend on non-active scenario values.
- Keeping all-scenario loading can be expensive for large multi-scenario models.
- Behavior may differ between legacy Python and Rust data-values paths.

### Acceptance Criteria

- A parity report states whether all-scenario property loading is required.
- Memory and setup-time impact are measured.
- If narrowed, behavior remains identical for validated request types.
- If not narrowed, the reason is documented and tracked as known cost.
- Feature flag fallback exists for safe rollout if implementation proceeds.

---

## Recommended Implementation Order

1. Issue 3: Add O(1) derived lookup indexes in `ModelMetadataCacheV4`
2. Issue 4: Normalize parsed property metadata once in `ModelMetadataCacheV4`
3. Issue 6: Reduce preload fan-out string cloning in Rust property preload
4. Issue 1: Define safe invalidation token for cross-request cache reuse
5. Issue 2: Add optional process-level metadata snapshot cache
6. Issue 7: Validate and potentially narrow all-scenarios property loading
7. Issue 5: Spike safe calc-plan and result caching with memory bounds

Reasoning:

- Issues 3, 4, and 6 are local and improve current request execution without cross-request invalidation risk.
- Issue 1 must precede any shared cross-request cache.
- Issue 2 depends directly on Issue 1.
- Issue 7 can be measured early, but behavior changes should be gated and validated carefully.
- Issue 5 is highest-risk and should stay a spike until invalidation, memory budgets, and parity tests are proven.

