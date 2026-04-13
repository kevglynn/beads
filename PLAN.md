# PR #3232 Convergence Fixes — Hyphenated Database Auto-Sanitization

## Background

PR #3232 fixes auto-sanitization of hyphenated `dolt_database` names when
projects upgrade to embedded-mode-default builds (GH#3231). Five independent
model reviews + convergence pass identified P0 and P1 issues to address before
merge.

Issue: https://github.com/gastownhall/beads/issues/3231
PR: https://github.com/gastownhall/beads/pull/3232

## P0 — Must Fix Before Merge

### 1. Read-only store path triggers writes (F+I)

**Problem:** `newReadOnlyStoreFromConfig` delegates to `newDoltStoreFromConfig`,
which now contains migration logic (dir rename + metadata.json save). Cross-repo
hydration via `routing_read.go` and `store_reopen.go` can mutate foreign projects.

**Fix:** Inline `sanitizeDBName` in `newReadOnlyStoreFromConfig` without calling
`migrateHyphenatedDB`. Sanitize the name in memory only for the SQL connection.

**Files:** `cmd/bd/store_factory.go`

### 2. Init guard ordering — serverMode not set before validation (D)

**Problem:** The `--database` hyphen validation at `init.go:108` calls
`isEmbeddedMode()`, but `serverMode` isn't assigned until line 155. Running
`bd init --server --database my-project` incorrectly rejects the name.

**Fix:** Move `serverMode = initServerMode` before the `--database` validation
block, or check `initServerMode` directly.

**Files:** `cmd/bd/init.go`

### 3. Directory collision — error instead of silent skip (A)

**Problem:** When both `embeddeddolt/my-project/` AND `embeddeddolt/my_project/`
exist, the rename is silently skipped but metadata.json is updated, orphaning
the old directory.

**Fix:** Return an error when both directories exist, telling user to resolve
manually.

**Files:** `cmd/bd/store_factory.go`

## P1 — Should Fix

### 4. Init line 677 dot sanitization gap (K+)

**Problem:** Line 677 of `init.go` only sanitizes hyphens in `cfg.DoltDatabase`,
but line 437-438 sanitizes both hyphens and dots in `dbName`. If a prefix
contains dots, the database is created as `my_project` but metadata.json records
`my.project`, causing reopens to fail.

**Fix:** Use `sanitizeDBName(prefix)` at line 677, or replicate the full logic.

**Files:** `cmd/bd/init.go`

### 5. Widen migration trigger to include dots (B+C)

**Problem:** The trigger at `store_factory.go:78` only checks for hyphens, but
`sanitizeDBName` also replaces dots. A database named `my.project` bypasses
auto-migration. The doctor check has the same gap and its suggestion is
inconsistent with `sanitizeDBName`.

**Fix:** Change trigger to `sanitizeDBName(database) != database`. Align doctor
check and suggestion.

**Files:** `cmd/bd/store_factory.go`, `cmd/bd/doctor/config_values.go`

### 6. Stat error handling (E)

**Problem:** In `migrateHyphenatedDB`, if `os.Stat(newDir)` returns a
non-IsNotExist error (e.g., permission denied), the code silently skips the
rename while still updating metadata.json.

**Fix:** Check for non-nil, non-IsNotExist errors and return them.

**Files:** `cmd/bd/store_factory.go`

## Explicitly Rejected

- **Reverse save/rename order (J):** 5/5 models reject. Current rename-first
  order is self-healing on crash. Saving metadata first creates a data-loss
  window.
- **Env var bypass (G):** Repeated no-op is harmless. Not a real bug.

## Status

- [x] P0-1: Read-only store path
- [x] P0-2: Init guard ordering
- [x] P0-3: Directory collision
- [x] P1-4: Init line 677 dots
- [x] P1-5: Widen trigger to dots
- [x] P1-6: Stat error handling
