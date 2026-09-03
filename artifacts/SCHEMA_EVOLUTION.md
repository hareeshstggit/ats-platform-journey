# Schema Evolution

Place the v2.0 baseline artifact here (see VERSION.md placement table).

## Binding checklist (CI enforcement)

- Any schema change (column/index/constraint/trigger/enum/RLS policy) MUST regenerate
  `docs/ci_schema_snapshot.sql` (`pg_dump` from a fresh Alembic-replayed DB) in the SAME
  commit, or `backend-ci.yml`'s G15d schema-definition-drift check
  (`app/scripts/check_schema_definition_drift.py`) fails the build.
