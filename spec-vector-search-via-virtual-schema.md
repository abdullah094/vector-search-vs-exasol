Below is a **detailed, staged implementation plan** for a **Lua Exasol Virtual Schema adapter** that overlays a base table and adds **virtual vector columns** stored in an external vector service (Qdrant for E2E), with **generic SQL interface**, **write-optimized DML**, and **ANN pushdown** using a two-phase plan (vector top-k → Exasol refinement). Each stage ends with **comprehensive tests** for everything implemented so far.

The intent is that a coding agent can execute stages sequentially, keeping the system always testable.

---

# Stage 0 — Repo scaffolding, CI, and local dev harness

## Deliverables

* Repo layout with clear modules and test harnesses.
* Docker compose for E2E: Exasol + Qdrant + test runner.
* A minimal “hello adapter” Lua script that can be loaded into Exasol (even if it only exposes base table).

## Suggested repo structure

```
/adapter
  /lua
    adapter.lua
    config.lua
    log.lua
    util.lua
    errors.lua
  /drivers
    driver_interface.lua
    qdrant_driver.lua          # used only in E2E; interface stays generic
  /vector
    codec.lua                  # parse/validate encodings
    canonical.lua              # canonicalization policy
  /pushdown
    ir.lua
    analyzer.lua
    filter_ir.lua
    filter_translate.lua
    planner.lua
  /dml
    dml_ir.lua
    writer.lua                # batching, retries, dirty tracking
    dirty.lua
  /tests
    /unit
    /integration
    /e2e
/docker
  docker-compose.yml
  exasol-init.sql
  qdrant-init.sh
  test-runner.Dockerfile
/scripts
  run_unit.sh
  run_e2e.sh
```

## Testing in this stage

* **Unit**: none required beyond “can run test runner”.
* **E2E smoke**:

  * Bring up containers.
  * Run an init SQL that creates `APP.DOCUMENTS` with a few rows.
  * Load adapter script + create virtual schema.
  * Verify `SELECT COUNT(*) FROM <vtable>` returns expected row count.

Acceptance: CI runs `docker compose up` + smoke query reliably.

---

# Stage 1 — Configuration + generic backend driver interface + diagnostics hook

## Implement

1. `config.lua`

   * Parse Virtual Schema properties:

     * `BASE_SCHEMA`, `BASE_TABLE`, `BASE_KEY_COL`
     * `VECTOR_COLUMNS` JSON
     * `BACKEND_ENDPOINT`, auth fields
     * `DML_MODE`, batching, timeouts
     * `PUSHDOWN_STRATEGY`, `OVERSAMPLE_FACTOR`, `MAX_K`
     * `DIAGNOSTICS_MODE = off|table|log`
2. `drivers/driver_interface.lua`

   * Define methods (generic):

     * `query_topk(req) -> [{id, score}]`
     * `fetch_vectors(req) -> map[id]map[name]vector`
     * `upsert_points(batch)`
     * `delete_points(ids)`
     * `remove_vectors([{id, names[]}])`
     * `capabilities()`
3. `log.lua` + `diagnostics`

   * If `DIAGNOSTICS_MODE=table`, write rows to a configured Exasol table.
   * Record: op_type, vector_name, count, latency_ms, status, error.

## Testing

### Unit

* Config parsing:

  * required props missing → clear error
  * VECTOR_COLUMNS JSON schema validation
* Diagnostics:

  * generates correct record structure

### E2E

* Create diagnostics table `APP.VEC_DIAG`.
* Run a simple read query from the virtual table.
* Assert at least one diagnostic row exists (or none if off).

Acceptance: adapter loads and config errors are actionable.

---

# Stage 2 — Vector encoding/decoding (generic), canonicalization, and validation

## Implement

1. `vector/codec.lua`

   * Supported input encodings:

     * `json:[...]`
     * `f32b64:<...>`
   * Output internal representation:

     * `float[]` (Lua table) + `dim`
   * Validation:

     * dimension check against configured `dim`
     * numeric validation: reject NaN/Inf by default (configurable)
2. `vector/canonical.lua`

   * Canonical output encoding for virtual column values:

     * Choose one (recommended: keep as input; or canonicalize to `f32b64:`).
     * Provide `canonicalize(vec_string, target_encoding)`
3. Update adapter read path (still base-only, but now understands vector strings when present for DML stages later).

## Testing

### Unit (comprehensive)

* Parse valid JSON vectors with whitespace/scientific notation
* Invalid JSON (trailing comma, non-numeric, empty if disallowed)
* Base64 decode:

  * invalid base64
  * wrong length
  * correct length for dim
* Dimension mismatch → error message includes expected/actual
* NaN/Inf policy
* Canonicalization stable

### E2E

* Create a dummy table with a few vector strings (if you keep any local storage for tests) OR just exercise codec via a small UDF test harness.
* Not strictly required yet; mostly unit-driven stage.

Acceptance: vector parsing is rock-solid before you attach it to DML/pushdown.

---

# Stage 3 — Metadata exposure: virtual table overlay + vector columns (read-only)

## Implement

1. `adapter.lua` metadata handlers:

   * Expose one virtual table matching base table name.
   * Columns = all base columns + configured vector columns:

     * vector columns are `VARCHAR` (sized generously) and nullable.
2. Read execution:

   * For scans/projections: read from base table directly.
   * Vector columns return NULL for now (until vector fetch stage).

## Testing

### Unit

* Metadata builder produces correct columns
* Vector columns appended with correct names

### E2E

* `DESCRIBE VEC.DOCUMENTS` includes configured vector columns
* `SELECT base cols` matches base table
* `SELECT vector col` yields NULLs (expected for this stage)

Acceptance: schema and base read path stable.

---

# Stage 4 — Backend driver implementation for E2E (Qdrant), behind generic interface

*(SQL interface remains generic; this is only the driver used in tests and default dev.)*

## Implement

1. `drivers/qdrant_driver.lua` implementing `driver_interface`.

   * `query_topk`: call Qdrant query endpoint for named vector
   * `fetch_vectors`: retrieve vectors for ids (use Qdrant get/retrieve)
   * `upsert_points`: upsert ids + named vectors
   * `remove_vectors`: remove a named vector (implement via Qdrant-supported method or fetch+rewrite minus vector)
   * `delete_points`
2. `capabilities()`:

   * `supports_remove_single_vector`
   * `supports_id_allowlist` (if implemented)
   * `supports_score_threshold`
3. Ensure driver never stores/relies on business payload except optional technical fields.

## Testing

### Unit (mock HTTP)

* Request formation for each driver method
* Error parsing, retries, timeouts
* Capability flags correct

### E2E

* Start Qdrant, create collection with named vectors matching VECTOR_COLUMNS.
* Driver smoke:

  * upsert one point, query topk returns it, fetch_vectors returns vector

Acceptance: driver works and stays isolated from adapter logic.

---

# Stage 5 — Lazy vector fetching on SELECT projections

## Implement

1. In read execution:

   * Detect whether vector columns are projected.
   * If not projected: do not call backend fetch.
   * If projected:

     * collect ids for the current batch/page
     * call `driver.fetch_vectors(ids, names)` in batches
     * fill projected vector columns with canonical encoded strings
2. Add caching (optional but recommended):

   * LRU keyed by `(id, vector_name)` with TTL.

## Testing

### Unit

* Projection detection logic
* Batching behavior and cache hits
* Missing vectors → NULL

### E2E

* Insert base rows (APP.DOCUMENTS)
* Upsert vectors directly into Qdrant for some ids
* Queries:

  * `SELECT doc_id FROM VEC.DOCUMENTS` → diagnostics show no fetch_vectors
  * `SELECT doc_id, EMB_BODY FROM VEC.DOCUMENTS WHERE doc_id IN (...)`

    * returns non-NULL for ids with vectors
    * diagnostics show fetch_vectors call count consistent with batching

Acceptance: vectors are external-only and fetched only when required.

---

# Stage 6 — ANN pushdown: analyze query shape → two-phase plan (ANN-first + oversample) + rank join

## Implement

1. `pushdown/analyzer.lua`

   * Detect pattern:

     * `ORDER BY VEC_*_DISTANCE(emb_col, const_vec) LIMIT k`
   * Require const vec string literal with supported encoding prefixes (or accept bare JSON and treat as `json:`).
2. `pushdown/ir.lua` + `planner.lua`

   * Produce a plan:

     * ANN request: vector_name, metric, query_vec, k
     * strategy: `ann_first`
     * oversample factor if SQL filters exist
3. Execution:

   * Call `driver.query_topk` with `k0`
   * Join ids to base table:

     * use an inline ranks table (VALUES) or temp table approach (choose one and standardize)
   * Apply original SQL WHERE in Exasol step (post-filter)
   * If fewer than k returned:

     * iterative fetch more candidates until satisfied or max_k hit
4. Preserve ordering:

   * `ORDER BY rank` unless user explicitly overrides.

## Testing

### Unit (heavy)

* Analyzer recognizes eligible queries
* Rejects non-constant query vec
* LIMIT required
* Oversample selection logic
* Iterative “fetch more if filtered out” loop logic

### E2E

* Seed small, deterministic vectors (3–4 dims) for easy expected ordering.
* Test:

  * ANN query returns expected ids in expected order
  * With a restrictive WHERE filter:

    * oversample kicks in
    * still returns full LIMIT when possible
* Diagnostics:

  * verify one query_topk call (or multiple if iterative) and no vector fetch unless projected.

Acceptance: vector DB used only for top-k candidate ids + score.

---

# Stage 7 — SQL-first allowlist strategy (optional, capability-driven)

## Implement

1. Add `supports_id_allowlist` capability.
2. Planner decides `sql_first` when filters are selective:

   * Exasol first computes candidate ids (bounded, e.g., `SELECT id FROM base WHERE ... LIMIT allowlist_cap`)
   * Backend query restricted to those ids
3. Add configuration overrides:

   * `PUSHDOWN_STRATEGY = auto|ann_first|sql_first`
   * `ALLOWLIST_MAX_IDS`

## Testing

### Unit

* Strategy selection based on heuristics and caps
* Allowlist formation and truncation rules

### E2E

* Use selective filter (tenant_id) so candidate ids are small.
* Verify diagnostics show:

  * first a base id query
  * then backend query restricted
* Compare result set with ANN-first; must be identical (assuming enough candidates).

Acceptance: improved performance for highly selective SQL filters without moving business data to backend.

---

# Stage 8 — Write-capable DML: INSERT/UPDATE/DELETE with batching and strict/best-effort modes

## Implement

1. DML analyzer / IR:

   * Determine per-row operations:

     * base cols to write to base table
     * vector cols to upsert/remove in backend
2. Implement statement execution with ordering + compensation policy:

   * Recommended default (best-effort):

     * write base table in Exasol (transactional)
     * attempt backend updates
     * on backend failure: record dirty ops
   * Strict mode:

     * if backend failure: fail statement; attempt compensating actions if needed
3. Write batching:

   * Collect backend upserts/removals/deletes in memory and flush in chunks.
4. Null semantics:

   * setting vector col to NULL → remove that vector for the id
5. Dirty tracking:

   * `APP.VEC_DIRTY_OPS` (or configured) with: id, op, vector_name, vec_value(optional), ts, last_error, retry_count

## Testing

### Unit

* DML IR generation for INSERT/UPDATE/DELETE
* Batching flush boundaries
* Null semantics
* Best-effort vs strict behavior
* Dirty records on failure

### E2E

* INSERT with vector:

  * base row created
  * backend vector created
* UPDATE vector:

  * backend updated
* UPDATE vector=NULL:

  * vector removed
* DELETE row:

  * backend record deleted
* Bulk insert from staging (thousands):

  * verify backend call batching via diagnostics
* Failure test:

  * stop backend, run INSERT with vector

    * best-effort: base insert succeeds and dirty recorded
    * strict: statement fails and base insert rolled back (if possible)
* Repair:

  * bring backend back
  * `CALL VEC_REPAIR_DIRTY(limit => ...)` drains dirty ops

Acceptance: write path is robust and doesn’t compromise Exasol as system of record.

---

# Stage 9 — Advanced query expressiveness: explicit TOPK table function (optional but recommended)

This avoids “magic” for complex queries (joins, subqueries, per-group searches).

## Implement

* Expose a virtual helper table/function-like object:

  * `VEC_TOPK(vector_col_name, query_vec, k [, options])`
  * returns `(id, score, rank)`
* Users can join it with arbitrary SQL in Exasol.

Implementation options:

* If VS framework supports table functions: implement as such.
* Otherwise: expose a special virtual table `VEC.__TOPK` parameterized via session variables or a function wrapper (less ideal).

## Testing

### Unit

* Parameter parsing and validation
* Output schema stable

### E2E

* Use `VEC_TOPK` joined to base table with WHERE/GROUP BY
* Validate result correctness and that only backend topk is used.

Acceptance: maximal expressiveness without backend data duplication.

---

# Stage 10 — Hardening: observability, limits, security, and performance regression tests

## Implement

* Strict input limits:

  * max vector dim, max string length
  * max k, max oversample
* Timeouts and retries per operation
* Better error messages (include op type, vector col, id count)
* Diagnostics:

  * metrics counters, request latencies
* Performance regression test suite (E2E):

  * baseline query counts and backend call counts

## Testing

### Unit

* limit enforcement
* error message formatting

### E2E

* Stress tests (moderate scale):

  * 50k base rows, 50k vectors
  * ANN queries with/without filters
  * bulk insert 10k vectors
* Assertions are based on:

  * correctness
  * number of backend calls (not wall-clock time)

Acceptance: stable and operable in production-like conditions.

---

## Global testing conventions for all stages

### Unit tests

* Run in <1–2 seconds, no Docker.
* No real HTTP; driver tests use mocked transport.
* Aim for highest coverage in:

  * vector codec
  * pushdown analyzer/planner
  * DML batching/dirty logic

### E2E tests

* Use deterministic low-dim vectors for correctness assertions.
* Always assert:

  * returned row sets and ordering
  * diagnostics call counts (proves pushdown/lazy fetch/batching)
* Include failure-mode tests early and keep them.

---

## What the coding agent should build first (priority order)

1. Diagnostics hook + vector codec (Stages 1–2)
2. Read-only overlay + backend driver smoke (Stages 3–5)
3. ANN pushdown (Stage 6)
4. DML write path + dirty repair (Stage 8)
5. SQL-first allowlist + explicit TOPK helper (Stages 7, 9)

