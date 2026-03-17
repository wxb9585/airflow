# Airflow 3.1.8 TiDB v7.3 Adaptation Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adapt Airflow 3.1.8 metadata database layer to run on self-hosted TiDB v7.3 by modifying 3 core files and auditing migration scripts.

**Architecture:** Reuse Airflow's existing MySQL dialect path. Add a `IS_TIDB` detection flag in `settings.py`, degrade `SKIP LOCKED` to plain `FOR UPDATE` in `sqlalchemy.py`, and update collation from `utf8mb3_bin` to `utf8mb4_bin` in `base.py`. Migration scripts that use `dialect.name == "mysql"` automatically apply to TiDB — only audit for TiDB-incompatible DDL patterns.

**Tech Stack:** Python 3, SQLAlchemy, Alembic, TiDB v7.3 (MySQL protocol)

**Spec:** `docs/superpowers/specs/2026-03-17-airflow-tidb-adaptation-design.md`

---

## File Structure

| Action | File | Responsibility |
|--------|------|----------------|
| Modify | `airflow-core/src/airflow/settings.py` | Add `IS_TIDB` detection flag |
| Modify | `airflow-core/src/airflow/utils/sqlalchemy.py` | Degrade `SKIP LOCKED` for TiDB |
| Modify | `airflow-core/src/airflow/models/base.py` | Change collation to `utf8mb4_bin` |
| Modify | `airflow-core/tests/unit/core/test_settings.py` | Test `IS_TIDB` detection |
| Modify | `airflow-core/tests/unit/utils/test_sqlalchemy.py` | Test SKIP LOCKED degradation |
| Modify | `airflow-core/tests/unit/models/test_base.py` | Test collation change |
| Audit | `airflow-core/src/airflow/migrations/versions/*.py` | Verify TiDB DDL compatibility |

---

### Task 1: Add `IS_TIDB` detection flag in `settings.py`

**Files:**
- Modify: `airflow-core/src/airflow/settings.py:46,372-435`
- Test: `airflow-core/tests/unit/core/test_settings.py`

- [ ] **Step 1: Write the failing test**

Add test to `airflow-core/tests/unit/core/test_settings.py`:

```python
from unittest.mock import MagicMock, patch

class TestTiDBDetection:
    @patch("airflow.settings.engine")
    def test_detect_tidb_returns_true_for_tidb(self, mock_engine):
        """IS_TIDB should be True when VERSION() contains 'TiDB'."""
        from airflow.settings import _detect_tidb

        mock_conn = MagicMock()
        mock_conn.execute.return_value.scalar.return_value = "5.7.25-TiDB-v7.3.0"
        mock_engine.connect.return_value.__enter__ = MagicMock(return_value=mock_conn)
        mock_engine.connect.return_value.__exit__ = MagicMock(return_value=False)

        assert _detect_tidb(mock_engine) is True

    @patch("airflow.settings.engine")
    def test_detect_tidb_returns_false_for_mysql(self, mock_engine):
        """IS_TIDB should be False for regular MySQL."""
        from airflow.settings import _detect_tidb

        mock_conn = MagicMock()
        mock_conn.execute.return_value.scalar.return_value = "8.0.32"
        mock_engine.connect.return_value.__enter__ = MagicMock(return_value=mock_conn)
        mock_engine.connect.return_value.__exit__ = MagicMock(return_value=False)

        assert _detect_tidb(mock_engine) is False

    @patch("airflow.settings.engine")
    def test_detect_tidb_returns_false_on_error(self, mock_engine):
        """IS_TIDB should be False if VERSION() query fails."""
        from airflow.settings import _detect_tidb

        mock_engine.connect.side_effect = Exception("connection failed")

        assert _detect_tidb(mock_engine) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest airflow-core/tests/unit/core/test_settings.py::TestTiDBDetection -v`
Expected: FAIL with `ImportError` — `_detect_tidb` does not exist yet.

- [ ] **Step 3: Implement `_detect_tidb` function and `IS_TIDB` global**

In `airflow-core/src/airflow/settings.py`, add at module level (after line 56, near the `USE_PSYCOPG3` declaration):

```python
IS_TIDB: bool = False
```

Then add the detection function (before `configure_orm`):

```python
def _detect_tidb(eng) -> bool:
    """Detect if the database is TiDB by checking VERSION() output."""
    try:
        with eng.connect() as conn:
            version = conn.execute(text("SELECT VERSION()")).scalar()
            return version is not None and "TiDB" in version
    except Exception:
        return False
```

Add `text` import at top of file (it's already imported via sqlalchemy in many places, but verify):

```python
from sqlalchemy import create_engine, text
```

Then in `configure_orm()`, after the `engine = create_engine(...)` line (after line 407), add:

```python
    global IS_TIDB
    IS_TIDB = _detect_tidb(engine)
    if IS_TIDB:
        log.info("TiDB detected as database backend")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest airflow-core/tests/unit/core/test_settings.py::TestTiDBDetection -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add airflow-core/src/airflow/settings.py airflow-core/tests/unit/core/test_settings.py
git commit -m "feat(tidb): add IS_TIDB detection flag in settings.py"
```

---

### Task 2: Degrade `SKIP LOCKED` for TiDB in `with_row_locks()`

**Files:**
- Modify: `airflow-core/src/airflow/utils/sqlalchemy.py:313-354`
- Test: `airflow-core/tests/unit/utils/test_sqlalchemy.py`

- [ ] **Step 1: Write the failing test**

Add test to `airflow-core/tests/unit/utils/test_sqlalchemy.py`:

```python
from unittest.mock import MagicMock, patch, PropertyMock

class TestWithRowLocksTiDB:
    @patch("airflow.utils.sqlalchemy.IS_TIDB", True)
    @patch("airflow.utils.sqlalchemy.USE_ROW_LEVEL_LOCKING", True)
    def test_skip_locked_degraded_on_tidb(self):
        """When IS_TIDB=True, skip_locked should be silently dropped."""
        from airflow.utils.sqlalchemy import with_row_locks

        mock_query = MagicMock()
        mock_session = MagicMock()
        mock_dialect = MagicMock()
        mock_dialect.name = "mysql"
        mock_dialect.supports_for_update_of = True
        mock_session.bind.dialect = mock_dialect

        with_row_locks(mock_query, mock_session, skip_locked=True, key_share=False)

        mock_query.with_for_update.assert_called_once()
        call_kwargs = mock_query.with_for_update.call_args[1]
        assert "skip_locked" not in call_kwargs

    @patch("airflow.utils.sqlalchemy.IS_TIDB", True)
    @patch("airflow.utils.sqlalchemy.USE_ROW_LEVEL_LOCKING", True)
    def test_nowait_preserved_on_tidb(self):
        """When IS_TIDB=True, nowait should still be passed through."""
        from airflow.utils.sqlalchemy import with_row_locks

        mock_query = MagicMock()
        mock_session = MagicMock()
        mock_dialect = MagicMock()
        mock_dialect.name = "mysql"
        mock_dialect.supports_for_update_of = True
        mock_session.bind.dialect = mock_dialect

        with_row_locks(mock_query, mock_session, nowait=True, skip_locked=False, key_share=False)

        call_kwargs = mock_query.with_for_update.call_args[1]
        assert call_kwargs.get("nowait") is True

    @patch("airflow.utils.sqlalchemy.IS_TIDB", False)
    @patch("airflow.utils.sqlalchemy.USE_ROW_LEVEL_LOCKING", True)
    def test_skip_locked_preserved_on_mysql(self):
        """When IS_TIDB=False, skip_locked should be passed through normally."""
        from airflow.utils.sqlalchemy import with_row_locks

        mock_query = MagicMock()
        mock_session = MagicMock()
        mock_dialect = MagicMock()
        mock_dialect.name = "mysql"
        mock_dialect.supports_for_update_of = True
        mock_session.bind.dialect = mock_dialect

        with_row_locks(mock_query, mock_session, skip_locked=True, key_share=False)

        call_kwargs = mock_query.with_for_update.call_args[1]
        assert call_kwargs.get("skip_locked") is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest airflow-core/tests/unit/utils/test_sqlalchemy.py::TestWithRowLocksTiDB -v`
Expected: FAIL — `IS_TIDB` not importable from `airflow.utils.sqlalchemy`.

- [ ] **Step 3: Implement SKIP LOCKED degradation**

In `airflow-core/src/airflow/utils/sqlalchemy.py`, add import near the top of the file (after existing airflow imports):

```python
from airflow.settings import IS_TIDB
```

Then modify the `with_row_locks()` function. Replace lines 348-352 with:

```python
    if nowait:
        kwargs["nowait"] = True
    if skip_locked:
        if IS_TIDB:
            # TiDB v7.3 does not support SKIP LOCKED; degrade to plain FOR UPDATE
            pass
        else:
            kwargs["skip_locked"] = True
    if key_share:
        kwargs["key_share"] = True
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest airflow-core/tests/unit/utils/test_sqlalchemy.py::TestWithRowLocksTiDB -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add airflow-core/src/airflow/utils/sqlalchemy.py airflow-core/tests/unit/utils/test_sqlalchemy.py
git commit -m "feat(tidb): degrade SKIP LOCKED to plain FOR UPDATE for TiDB"
```

---

### Task 3: Update collation from `utf8mb3_bin` to `utf8mb4_bin`

**Files:**
- Modify: `airflow-core/src/airflow/models/base.py:63-81`
- Test: `airflow-core/tests/unit/models/test_base.py`

- [ ] **Step 1: Write the failing test**

Add test to `airflow-core/tests/unit/models/test_base.py`:

```python
from unittest.mock import patch

class TestCollationArgs:
    @patch("airflow.models.base.conf")
    def test_mysql_connection_uses_utf8mb4_bin(self, mock_conf):
        """MySQL/TiDB connections should use utf8mb4_bin collation."""
        from airflow.models.base import get_id_collation_args

        mock_conf.get.side_effect = lambda section, key, fallback=None: {
            ("database", "sql_engine_collation_for_ids"): None,
            ("database", "sql_alchemy_conn"): "mysql+pymysql://user:pass@host/db",
        }.get((section, key), fallback)

        result = get_id_collation_args()
        assert result == {"collation": "utf8mb4_bin"}

    @patch("airflow.models.base.conf")
    def test_non_mysql_connection_has_no_collation(self, mock_conf):
        """Non-MySQL connections should return empty collation args."""
        from airflow.models.base import get_id_collation_args

        mock_conf.get.side_effect = lambda section, key, fallback=None: {
            ("database", "sql_engine_collation_for_ids"): None,
            ("database", "sql_alchemy_conn"): "postgresql+psycopg2://user:pass@host/db",
        }.get((section, key), fallback)

        result = get_id_collation_args()
        assert result == {}

    @patch("airflow.models.base.conf")
    def test_custom_collation_overrides_default(self, mock_conf):
        """Explicit sql_engine_collation_for_ids should override auto-detection."""
        from airflow.models.base import get_id_collation_args

        mock_conf.get.side_effect = lambda section, key, fallback=None: {
            ("database", "sql_engine_collation_for_ids"): "utf8mb4_general_ci",
            ("database", "sql_alchemy_conn"): "mysql+pymysql://user:pass@host/db",
        }.get((section, key), fallback)

        result = get_id_collation_args()
        assert result == {"collation": "utf8mb4_general_ci"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest airflow-core/tests/unit/models/test_base.py::TestCollationArgs -v`
Expected: `test_mysql_connection_uses_utf8mb4_bin` FAILS — returns `utf8mb3_bin` instead of `utf8mb4_bin`.

- [ ] **Step 3: Change collation in `get_id_collation_args()`**

In `airflow-core/src/airflow/models/base.py`, line 80, change:

```python
# Before:
        return {"collation": "utf8mb3_bin"}
# After:
        return {"collation": "utf8mb4_bin"}
```

Also update the comment on line 69 to reflect the change:

```python
# Before:
    # Automatically use utf8mb3_bin collation for mysql
# After:
    # Automatically use utf8mb4_bin collation for mysql/tidb
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest airflow-core/tests/unit/models/test_base.py::TestCollationArgs -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add airflow-core/src/airflow/models/base.py airflow-core/tests/unit/models/test_base.py
git commit -m "feat(tidb): change default collation from utf8mb3_bin to utf8mb4_bin"
```

---

### Task 4: Audit migration scripts for TiDB compatibility

**Files:**
- Audit: `airflow-core/src/airflow/migrations/versions/*.py` (88 files)

This task is an audit — no code changes expected unless specific incompatibilities are found.

**Patterns to scan for:**

1. `CREATE TABLE ... AS SELECT` (CTAS) — TiDB does not support this
2. `SKIP LOCKED` in raw SQL — TiDB does not support this
3. Multi-column ALTER TABLE on same column — TiDB may reject
4. Stored procedures / triggers / events — TiDB does not support
5. `ALGORITHM=` hints in ALTER TABLE — TiDB parses but ignores; verify no assertion failures

- [ ] **Step 1: Search for CTAS pattern**

Run:
```bash
grep -rn "CREATE TABLE.*SELECT\|CREATE TABLE.*AS" airflow-core/src/airflow/migrations/versions/
```
Expected: Any hits should already be in MySQL-guarded paths using the `CREATE TABLE ... LIKE` workaround.

- [ ] **Step 2: Search for SKIP LOCKED in migrations**

Run:
```bash
grep -rn "SKIP.LOCKED\|skip_locked" airflow-core/src/airflow/migrations/versions/
```
Expected: No hits (SKIP LOCKED is used in runtime code, not migrations).

- [ ] **Step 3: Search for multi-column ALTER TABLE**

Run:
```bash
grep -rn "batch_alter_table\|alter_column\|add_column\|drop_column" airflow-core/src/airflow/migrations/versions/ | head -40
```
Expected: Alembic's `batch_alter_table` context manager handles this safely for MySQL dialect by issuing individual ALTER statements. Verify no single `op.execute(text("ALTER TABLE ... CHANGE col1 ..., CHANGE col1 ..."))` patterns exist.

- [ ] **Step 4: Search for stored procedures or triggers**

Run:
```bash
grep -rn "CREATE PROCEDURE\|CREATE TRIGGER\|CREATE EVENT\|CREATE FUNCTION" airflow-core/src/airflow/migrations/versions/
```
Expected: No hits.

- [ ] **Step 5: Review key migration files with dialect branching**

Read and verify these migrations handle TiDB (via MySQL path) correctly:
- `0082_3_1_0_make_bundle_name_not_nullable.py` — `INSERT IGNORE` (TiDB supports)
- `0063_3_0_0_use_ti_id_as_fk_to_taskreschedule.py` — FK operations (TiDB v7.3 supports)
- `0042_3_0_0_add_uuid_primary_key_to_task_instance_.py` — UUID column + PK change (verify MySQL path)
- `0060_3_0_0_add_try_id_to_ti_and_tih.py` — Column additions with MySQL path
- `0041_3_0_0_rename_dataset_as_asset.py` — Table rename with MySQL path

- [ ] **Step 6: Document audit results**

If all checks pass with no issues found, add a note to the spec document confirming the audit is complete. If issues are found, create follow-up tasks to fix them.

- [ ] **Step 7: Commit audit documentation (if changes made)**

```bash
git add -A
git commit -m "chore(tidb): complete migration scripts audit — no changes needed"
```

---

### Task 5: Final integration verification

- [ ] **Step 1: Run all modified tests together**

Run:
```bash
python -m pytest airflow-core/tests/unit/core/test_settings.py::TestTiDBDetection airflow-core/tests/unit/utils/test_sqlalchemy.py::TestWithRowLocksTiDB airflow-core/tests/unit/models/test_base.py::TestCollationArgs -v
```
Expected: All tests PASS.

- [ ] **Step 2: Verify no import cycles**

Run:
```bash
python -c "from airflow.settings import IS_TIDB; print(f'IS_TIDB={IS_TIDB}')"
python -c "from airflow.utils.sqlalchemy import with_row_locks; print('import OK')"
python -c "from airflow.models.base import COLLATION_ARGS; print(f'COLLATION_ARGS={COLLATION_ARGS}')"
```
Expected: No `ImportError` or circular import errors.

- [ ] **Step 3: Review all changes**

Run:
```bash
git log --oneline airflow-tidb-adaptation ^3.1.8
git diff 3.1.8..airflow-tidb-adaptation --stat
```
Expected: 4-5 commits, 6 files changed (3 source + 3 test files + spec/plan docs).

- [ ] **Step 4: Final commit summary**

Verify the branch `airflow-tidb-adaptation` contains:
1. `feat(tidb): add IS_TIDB detection flag in settings.py`
2. `feat(tidb): degrade SKIP LOCKED to plain FOR UPDATE for TiDB`
3. `feat(tidb): change default collation from utf8mb3_bin to utf8mb4_bin`
4. `chore(tidb): complete migration scripts audit`

---

## Notes for TiDB Deployment

After code changes, the following TiDB server configuration is recommended:

```sql
-- Use pessimistic transactions (matches MySQL InnoDB behavior)
SET GLOBAL tidb_txn_mode = 'pessimistic';
```

Airflow config (`airflow.cfg`):
```ini
[database]
sql_alchemy_conn = mysql+pymysql://user:password@tidb-host:4000/airflow
```
