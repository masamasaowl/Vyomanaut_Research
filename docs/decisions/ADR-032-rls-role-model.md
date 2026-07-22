# ADR-032 — Append-Only RLS Role Model (FORCE RLS + Least-Privilege Roles)

**Status:** Accepted
**Topic:** #6 Proof of Storage / Audit trail (database enforcement)
**Supersedes:** the implicit single-role assumption in data-model.md §6 and §9
**Superseded by:** —
**Research source:** Papers 07, 11 (tamper-evident audit log); PostgreSQL RLS semantics

---

## Context

data-model.md §3 (Invariants 1 and 2) states that the append-only guarantee on
`audit_receipts`, `escrow_events`, and `chunk_assignments` is *"enforced by row
security policy."* An audit of the Milestone 4 migration found that, as generated,
it was **not** — for two independent reasons, each of which alone defeats the
guarantee:

1. **No `FORCE ROW LEVEL SECURITY`.** The migration used `ENABLE` only. In
   PostgreSQL, `ENABLE` exempts the **table owner** (and any superuser) from every
   policy. So the policies bound nobody with ownership.

2. **The application *was* the owner.** Both `deployments/dev/docker-compose.yml`
   and `.github/workflows/ci.yml` set `POSTGRES_USER: vyomanaut_app`, making the
   request-path role the bootstrap superuser that owns the schema. Combined with (1),
   the app bypassed RLS entirely. Verified live: `DELETE FROM audit_receipts` as the
   app returned `DELETE 1`.

The two failure modes are also mutually exclusive in a subtle way. If an operator
"fixed" (2) by demoting the app to a non-owner, a **third** latent defect surfaced:
there were **no `SELECT` policies**. Under RLS a table with no `SELECT` policy
returns zero rows to a non-owning role, so the two-phase audit write's
`UPDATE ... WHERE audit_result IS NULL` matched nothing and silently returned
`UPDATE 0` — every audit would hang in `PENDING`, GC-abandon would no-op, and all
reads would break. So the schema was only ever "correct" in one of two broken modes.

Finally, the two service roles were created with **no privileges and no attributes**
(`CREATE ROLE vyomanaut_app;`), leaving DM §9's *"with appropriate privileges"*
unmet, and `scripts/ci/migration_check.sh` was **never invoked by CI** (check-07
only applied the migration), so none of this was caught.

## Decision

Adopt a **three-role model** with `FORCE` RLS and explicit least-privilege grants.

### Roles

| Role | Provisioned by | Attributes | Purpose |
|---|---|---|---|
| `vyomanaut_migrator` | **Environment** (bootstrap `POSTGRES_USER` in dev/CI; DBA in prod) | Owner of the schema; `SUPERUSER` **or** `NOSUPERUSER` + `BYPASSRLS` | Runs migrations; refreshes materialised views; prunes/archives. |
| `vyomanaut_app` | **The migration** | `LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS` | Microservice request path. Fully subject to RLS. |
| `vyomanaut_gc` | **The migration** | `LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS` | Garbage collector. Abandons stale `PENDING` receipts. |

`vyomanaut_migrator` is **not** created by the migration — a migration cannot create
the role that runs it (chicken-and-egg). It must hold `BYPASSRLS` (or be superuser)
so that maintenance and `REFRESH MATERIALIZED VIEW` can read the `FORCE`-RLS base
tables. Passwords for `vyomanaut_app` / `vyomanaut_gc` are set by the deployment
(`ALTER ROLE ... PASSWORD` from a secrets store) — **never** committed to VCS.

### Enforcement

1. **`FORCE ROW LEVEL SECURITY`** on `audit_receipts`, `escrow_events`,
   `chunk_assignments` (and, per ADR-033, `audit_receipt_nonces`). Policies now bind
   even a table owner.
2. **`SELECT` policies** (`USING (TRUE)`) for `vyomanaut_app` on all three tables and
   for `vyomanaut_gc` on `audit_receipts`, so the scoped `UPDATE`s can read their
   target rows.
3. **Least-privilege `GRANT`s**, emitted last (after all objects exist). **No
   `DELETE` on any table.** Append-only ledgers (`audit_receipts`, `escrow_events`)
   and soft-delete tables (`files`, `chunk_assignments`) and never-deleted providers
   (Invariant 3) are all protected at the grant layer as well as by RLS.
4. **A read-only assertion** at the top of the migration aborts with a clear message
   if `vyomanaut_app` / `vyomanaut_gc` were provisioned with `rolsuper` or
   `rolbypassrls`. It is read-only (any migrator can run it) because clearing the
   `SUPERUSER` attribute itself requires superuser — an assertion fails loudly
   without needing that privilege.

### MV refresh and maintenance run as the migrator

`REFRESH MATERIALIZED VIEW` reads the `FORCE`-RLS base tables. Rather than granting
the request-path role that power, all maintenance (migrations, MV refresh, pruning,
archival) uses the `vyomanaut_migrator` identity (`BYPASSRLS`). This is a deliberate
separation of the maintenance identity from the request-serving identity — a
downstream-milestone contract (M12): the microservice opens its hot-path pool as
`vyomanaut_app` and a separate maintenance connection as `vyomanaut_migrator`.

## Alternatives considered

- **`ENABLE` + keep app as owner (status quo).** Rejected: owner bypasses RLS;
  append-only is unenforced.
- **`FORCE` but no `SELECT` policies.** Rejected: the two-phase `UPDATE` silently
  returns `UPDATE 0`; the audit pipeline stalls.
- **Defensive `ALTER ROLE ... NOSUPERUSER NOBYPASSRLS` (self-healing).** Rejected:
  clearing `SUPERUSER` requires superuser, so this **fails under a least-privilege
  migrator** (caught during the strength test — see below). Replaced with the
  read-only assertion.

## Strength test (prerequisite — run before this ADR was finalised)

All tests were run against PostgreSQL 16 in the **production posture** — the hardest
case: a `NOSUPERUSER` `vyomanaut_migrator` that owns the schema and holds only
`CREATEROLE` + `BYPASSRLS` (superuser trivially passes, so it is not the bar).

| # | Check | Result |
|---|---|---|
| 1 | Migration applies as non-superuser migrator | `exit 0` |
| 2 | `FORCE` RLS active on all 3 tables (`relforcerowsecurity = t`) | pass |
| 3 | `vyomanaut_app` / `gc` created `NOSUPERUSER NOBYPASSRLS LOGIN` | pass |
| 4 | app `SELECT` receipts | rows returned |
| 5 | app phase-1 `INSERT` PENDING | `INSERT 0 1` |
| 6 | app phase-2 promote PENDING→PASS | **`UPDATE 1`** (was `UPDATE 0`) |
| 7 | app `DELETE FROM audit_receipts` | `permission denied` |
| 8 | app tamper terminal row (`SET audit_result` on a PASS) | `UPDATE 0` (USING blocks) |
| 9 | app escrow `INSERT` / `UPDATE` / `DELETE` | insert ok; update & delete denied |
| 10 | **FORCE teeth:** grant app `DELETE`, retry `DELETE` | **`DELETE 0`** (rows invisible to DELETE) |
| 11 | gc abandon stale PENDING | `UPDATE 1` |
| 12 | migrator `REFRESH MATERIALIZED VIEW CONCURRENTLY` × 4 | all succeed |
| 13 | Assertion fires when app pre-created with `BYPASSRLS` | migration **aborts** with `ADR-032 violation` |
| 14 | Backward-compat: superuser migrator applies | `exit 0` |
| 15 | `migration_check.sh` (25 checks incl. 5 new) | **all pass, `exit 0`** |

Test #10 is the proof that `FORCE` RLS — not merely the absence of a `DELETE` grant
— is what stops deletion: even after `GRANT DELETE`, RLS makes the rows invisible to
`DELETE`, so `0` rows are removed. Test #6 is the previously-broken two-phase write,
now working. The defensive-`ALTER` defect in the first draft was caught by test #1
(it failed with *"permission denied to alter role"* under a non-superuser migrator)
and replaced with the assertion of test #13 before this ADR was accepted.

## Exact changes made

**`migrations/generator.go`** (the schema is generated; `.sql` files are regenerated
byte-for-byte from it):

- `rolesSection`: create `vyomanaut_app` / `vyomanaut_gc` with
  `LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS`; add the read-only
  ADR-032 assertion `DO` block.
- `rspSection`: add `ALTER TABLE ... FORCE ROW LEVEL SECURITY` to `audit_receipts`,
  `escrow_events`, `chunk_assignments`; add `SELECT` policies
  `audit_receipts_app_select`, `audit_receipts_gc_select`,
  `escrow_events_app_select`, `chunk_assignments_app_select`.
- New `grantsSection` (appended last, after views): least-privilege `GRANT`s, **no
  `DELETE`** anywhere.

**`migrations/001_initial_schema.sql`, `..._demo.sql`** — regenerated (deterministic;
profile diff remains header + the 2 profile-variable bounds only).

**`deployments/dev/docker-compose.yml`** — `POSTGRES_USER` and the healthcheck →
`vyomanaut_migrator`.

**`.github/workflows/ci.yml`** — postgres service `POSTGRES_USER` → `vyomanaut_migrator`;
check-07 `PGUSER` → `vyomanaut_migrator`, add `PGDATABASE`, and **invoke
`scripts/ci/migration_check.sh`** after applying the prod schema (it was never run).

**`scripts/ci/migration_check.sh`** — replace the owner-filtered
`information_schema.check_constraints` nonce check with a `pg_constraint` query
(works for any role); add `force_rls_*` ×3, `app_gc_no_bypassrls`,
`app_select_policies_present`.

**`docs/data-model.md`** — §6, §9 corrected (banner + checklist) to reflect this model.

## Consequences

- Append-only Invariants 1 and 2 are now enforced against **every** identity,
  including the schema owner.
- The microservice (M12) must open two pools: `vyomanaut_app` (hot path) and
  `vyomanaut_migrator` (migrations, MV refresh, maintenance).
- Deployments must set `vyomanaut_app` / `vyomanaut_gc` passwords out-of-band.
- CI now actually runs the DM §9 checklist and will fail if any of the above regress.
