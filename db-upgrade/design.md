# Safe In-Place Database Schema Upgrades (`ramble data upgrade-db`)
* **Status:** Draft
* **Author(s):** rfbgo
* **Last Updated:** 2026-10-27

---

## Overview
This proposal introduces a safe, non-destructive `ramble data upgrade-db` CLI command and validation mechanism for migrating existing result databases forward when Ramble's table schemas evolve.

### Current Issue / Current State / Motivation
Ramble tracks versioned schemas for each database table (`experiments`, `foms`, `metadata`, `experiments_metadata`, `software`) and records their versions in the `metadata` table at creation time. However, there is currently no mechanism to upgrade an existing database when a schema changes:
* Running `ramble data create-db` against an existing database logs a version mismatch warning if `metadata` lags behind Ramble's current schema version, but skips all existing tables.
* Adding new columns to an existing table schema (or introducing a new table) causes subsequent `ramble results upload` and `ramble workspace analyze --upload` runs to fail against pre-existing databases unless users manually inspect and alter their tables.

### User Journey / Use Case
1. A developer updates Ramble to a version that introduces new database columns or tables.
2. To bring an existing SQLite or BigQuery database up to date, the user runs:
   ```bash
   ramble data upgrade-db
   ```
3. Ramble validates that the schema transition is strictly additive and non-destructive, creates any missing tables, adds missing nullable columns in-place while preserving all existing rows, and updates the schema version records in the `metadata` table.
4. Subsequent uploads succeed without manual SQL intervention.

### Requirements
* Schema upgrades must be provably non-destructive: they must never drop columns, alter existing column types, or tighten nullability on existing columns.
* Newly added columns on existing tables must be nullable (`NULLABLE` in BigQuery) so both SQLite and BigQuery can apply in-place `ALTER TABLE` / `update_table` operations without invalidating historical rows.
* Downgrading a database created by a newer version of Ramble must be rejected.

## Goals
* Define programmatic safety checks to verify whether a transition between one or more schema versions is non-destructive (supports iterative upgrade).
* Add `ramble data upgrade-db` to perform safe in-place table upgrades and create missing tables across SQLite and BigQuery backends.

## Non-Goals
* Destructive or complex schema migrations (e.g., dropping or renaming columns, changing column types, or full-table rewrites).
* Backfilling values for newly added columns on historical rows.
* Implicitly altering database schemas during `ramble results upload` without explicit user invocation.

## Implementation Plan / Detailed Design

### 1. Schema Upgrade Safety Validation
Implement check_schema_upgrade and safety checks between $S_{\text{old}}$ and $S_{\text{new}}$:
1. **Forward version**: `upstream_version <= target_version` (refuse downgrades).
2. **No dropped/renamed columns**: $\text{columns}(S_{\text{old}}) \subseteq \text{columns}(S_{\text{new}})$.
3. **No type/format changes**: Every column in $S_{\text{old}}$ retains the exact same `type` and `format` in $S_{\text{new}}$.
4. **No nullability tightening**: No existing optional column becomes `required`.
5. **Nullable additions only**: Every newly added column $c \in \text{columns}(S_{\text{new}}) \setminus \text{columns}(S_{\text{old}})$ is omitted from `required`.

### 2. CLI & Uploader Upgrade Execution
* Add the `upgrade-db` subcommand to `ramble data`
  1. **Validate version chain in memory**: Reject downgrades and verify requested upgraded (chain)`for existing upstream version.
  2. **Inspect live table**: Compare live columns against the target schema to ensure no live columns were dropped, changed type, or tightened in nullability, and that all missing columns are nullable.
  3. **Apply changes**:
     * Create any missing tables that were introduced in newer Ramble versions.
     * Append missing nullable columns in-place.
     * Update schema version entries (`db_schema_version` and per-table `*_schema_version` keys) in the `metadata` table.

### 3. Testing & Verification
* Unit tests for `check_schema_upgrade` covering valid additive changes and each unsafe failure mode (removed column, type change, nullability tightening, required new column).
* Unit tests for `SQLiteUploader.upgrade_tables`, `BigQueryUploader.upgrade_tables`, and `ramble data upgrade-db` CLI execution (verifying column addition, preservation of existing rows, metadata version updates, and rejection of downgrades or unsafe schema changes).

## Alternatives Considered
* **Auto-migrating inside `ramble data create-db`**: Kept separate in `ramble data upgrade-db` so `create-db` remains strictly for initial setup and schema alterations require explicit user intent.

## Open Questions / Future Work
* **Major version migrations**: If a future breaking schema change is ever required, support an opt-in migration strategy (such as creating v2 tables or copying/transforming data).

