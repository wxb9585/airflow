# Airflow 3.1.8 TiDB v7.3 Adaptation Design

## Overview

Adapt the Airflow 3.1.8 metadata database layer to run on a self-hosted TiDB v7.3 cluster. The adaptation targets TiDB only (no need to maintain PostgreSQL/SQLite compatibility). The approach reuses Airflow's existing MySQL dialect path with targeted modifications for TiDB-specific differences.

## Context

- **Airflow version**: 3.1.8 (tag `3.1.8`, commit `7c1b4e1cc6`)
- **Target database**: TiDB v7.3 (self-hosted cluster)
- **Scope**: Metadata database only (ORM models, migrations, locking, sessions)
- **Strategy**: TiDB-only support, reusing the MySQL dialect path

## TiDB v7.3 Compatibility Summary

| Feature | TiDB v7.3 Status | Airflow Usage |
|---|---|---|
| `GET_LOCK()` / `RELEASE_LOCK()` | Supported | Global lock in `utils/db.py` |
| `SELECT ... FOR UPDATE` | Supported (pessimistic mode) | Row-level locking |
| `NOWAIT` | Supported (since v3.0) | Row-level locking |
| `SKIP LOCKED` | **Not supported** | Row-level locking in scheduler |
| `INSERT IGNORE` | Supported | Migration scripts |
| `ON DUPLICATE KEY UPDATE` | Supported | Variable upsert, asset manager |
| Foreign keys | Supported (since v6.6.0) | Model relationships, migrations |
| `CREATE TABLE ... SELECT` (CTAS) | **Not supported** | DB cleanup (already worked around) |
| `CREATE TABLE ... LIKE` | Supported | DB cleanup MySQL path |
| `MEDIUMTEXT` / `LONGBLOB` | Supported | XCom, Variable models |
| `utf8mb3_bin` collation | Deprecated (use `utf8mb4_bin`) | All ID columns |

## Modifications

### 1. TiDB Detection Flag

**File**: `airflow-core/src/airflow/settings.py`

**What**: Add a global `IS_TIDB` flag detected at engine initialization time via `SELECT VERSION()`.

**Why**: Some code paths need TiDB-specific behavior (e.g., SKIP LOCKED handling). A centralized flag avoids repeated detection.

**How**:
- After engine creation (around line 398-410), execute `SELECT VERSION()` and check for `"TiDB"` in the result string.
- Set `IS_TIDB = True/False` as a module-level variable.
- The flag is only needed for runtime behavior; connection/dialect setup uses standard MySQL protocol.

**Impact**: Minimal. Read-only detection, no behavioral change by itself.

### 2. SKIP LOCKED Degradation

**File**: `airflow-core/src/airflow/utils/sqlalchemy.py`

**What**: In the `with_row_locks()` function (lines 314-356), when TiDB is detected and `skip_locked=True`, drop the `skip_locked` kwarg and fall back to plain `FOR UPDATE`.

**Why**: TiDB v7.3 does not support `SKIP LOCKED`. Without this change, the scheduler will error when trying to acquire task locks.

**How**:
```python
# After the existing MySQL dialect check (line 347)
if IS_TIDB and skip_locked:
    skip_locked = False  # TiDB v7.3 does not support SKIP LOCKED
```

**Impact**: The scheduler will use `FOR UPDATE` instead of `FOR UPDATE SKIP LOCKED`. This means concurrent schedulers may block briefly when contending for the same rows instead of skipping them. For single-scheduler deployments this has zero impact. For HA scheduler setups, throughput may be slightly reduced under contention.

**Alternative considered**: Disable row-level locking entirely (`USE_ROW_LEVEL_LOCKING=False`). Rejected because `FOR UPDATE` + `NOWAIT` still provides useful concurrency control.

### 3. Collation: utf8mb3_bin to utf8mb4_bin

**File**: `airflow-core/src/airflow/models/base.py`

**What**: In `get_id_collation_args()` (lines 64-82), change the auto-detected collation from `utf8mb3_bin` to `utf8mb4_bin` for TiDB connections.

**Why**: TiDB defaults to `utf8mb4` and `utf8mb3` is deprecated. Using `utf8mb3_bin` can cause charset mismatch warnings or errors. All Airflow IDs are ASCII, so `utf8mb4_bin` is fully backward-compatible.

**How**:
```python
if conn.startswith(("mysql", "mariadb")):
    return {"collation": "utf8mb4_bin"}  # Changed from utf8mb3_bin
```

**Impact**: Affects all `StringID()` columns across all models. For a fresh TiDB installation this is clean. If migrating from an existing MySQL installation, a collation migration may be needed (out of scope for this design).

### 4. Migration Scripts Audit

**Directory**: `airflow-core/src/airflow/migrations/versions/` (88 files)

**What**: Audit all migration files for TiDB compatibility. The MySQL path is the primary code path; most migrations should work without changes.

**Known patterns that need review**:

1. **Multi-column ALTER TABLE**: TiDB may reject `ALTER TABLE` statements that modify the same column/index multiple times. Split into separate statements if found.

2. **Foreign key operations**: TiDB v7.3 supports FK but behavior may differ subtly (e.g., `ON DELETE CASCADE` timing). Verify migration `0027` (custom MySQL FK helpers) and `0082` (FK drop/recreate around `bundle_name`).

3. **INSERT IGNORE**: Used in migration `0082` for MySQL path. TiDB supports this — no change needed.

4. **Dialect branching**: Migrations that check `dialect.name == "mysql"` will match TiDB since TiDB uses the MySQL dialect. This is the desired behavior.

5. **Batch alter operations**: Alembic's `batch_alter_table` is used in several migrations. This works with MySQL/TiDB but wraps operations in a temporary table copy for SQLite. For MySQL dialect it issues direct ALTER statements.

**Approach**: Run each migration file through a static analysis pass, flagging any DDL that uses patterns known to be problematic on TiDB (per `mysql-compatibility-notes.md`). Fix only what fails.

**Estimated effort**: Most files need no changes. Expect 3-5 files that need minor adjustments.

## Files Not Requiring Changes

These files use MySQL-compatible paths that work on TiDB v7.3 without modification:

| File | Reason |
|---|---|
| `utils/db.py` (global lock) | `GET_LOCK()` / `RELEASE_LOCK()` supported |
| `models/variable.py` (upsert) | `ON DUPLICATE KEY UPDATE` supported |
| `assets/manager.py` (upsert) | MySQL slow path (savepoint-based) works |
| `utils/db_cleanup.py` | MySQL path (`CREATE TABLE ... LIKE` + INSERT) works |
| `settings.py` (isolation level) | `READ COMMITTED` supported |
| `settings.py` (async driver) | `aiomysql` works with TiDB MySQL protocol |

## Testing Strategy

1. **Unit tests**: Run existing Airflow unit tests against TiDB v7.3 instead of MySQL.
2. **Migration test**: Run `airflow db migrate` on a fresh TiDB instance to verify all 88 migrations complete.
3. **Scheduler test**: Run scheduler with `USE_ROW_LEVEL_LOCKING=True` to verify `FOR UPDATE` without `SKIP LOCKED` works correctly.
4. **Concurrency test**: Run two scheduler instances to verify global lock contention is handled properly.

## Out of Scope

- TiDB Provider for DAGs (Hook/Operator for user TiDB access)
- MySQL/PostgreSQL/SQLite backward compatibility
- Migration from existing MySQL Airflow installation to TiDB
- TiDB Cloud-specific adaptations (SSL, region restrictions)
- Performance tuning (AUTO_RANDOM for write hotspots, TiFlash, etc.)

## Risk Assessment

| Risk | Likelihood | Mitigation |
|---|---|---|
| SKIP LOCKED degradation causes scheduler contention | Low (single scheduler) / Medium (HA) | Monitor lock wait times; can disable row locking as fallback |
| Migration DDL fails on TiDB | Low | Audit + test each migration individually |
| Collation mismatch with existing data | N/A (fresh install) | Document if migration path needed later |
| Undiscovered TiDB incompatibility in raw SQL | Low | Comprehensive test suite against TiDB |
