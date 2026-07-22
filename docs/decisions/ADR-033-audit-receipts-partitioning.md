# ADR-033 — `audit_receipts` Monthly Partitioning + Global Nonce Guard

**Status:** Accepted
**Topic:** #6 Proof of Storage / Audit trail (scalability & archival)
**Supersedes:** the unpartitioned `audit_receipts` DDL in data-model.md §4.7; the
partitioning line in data-model.md §9
**Superseded by:** —
**Research source:** `Architectural_Probable_Problems/capacity.md`; architecture.md §25 (ADR-015 archival trade-off), §26

---

## Context

data-model.md §9 mandates that `audit_receipts` be
`PARTITION BY RANGE (server_challenge_ts)` monthly *"from day one."* The Milestone 4
migration never implemented it, and the mandate collides with the table's own
constraints: PostgreSQL requires the **partition key to appear in every UNIQUE / PK
constraint**, but the table declares `PRIMARY KEY (receipt_id)` and
`UNIQUE (challenge_nonce)` — neither of which includes `server_challenge_ts`.

Two settled facts frame the decision:

- **architecture.md §25 (ADR-015):** *"INSERT-only audit receipts … We accept
  monotonically growing table size requiring periodic archival."* Archival is an
  explicitly accepted mechanism. But Invariant 1 forbids a DML `DELETE`, so the only
  way to archive an append-only table is to move data at the **DDL** level —
  i.e. `DETACH PARTITION`. Partitioning is therefore not a workaround; it is the
  enabling mechanism for a trade-off the architecture already accepted.
- **architecture.md §26:** *"V2 launches at hundreds of providers — far below this
  limit."* The alarming `capacity.md` figures (~3.6 billion rows/year, 1–2 TB/year,
  a disk-bound global `challenge_nonce` index) are the **V3, 100k-provider**
  projection. So the fix must satisfy the day-one mandate and preserve security
  **without** dragging in V3-scale automation that §26 defers.

`capacity.md` specifically flags the **global unique index on `challenge_nonce`** as
a growth bottleneck. This matters because the naive partitioning fix — making the
nonce unique key `(challenge_nonce, server_challenge_ts)` — would **silently weaken
replay protection** (Invariant 5): the same nonce could recur in a different month.
That is a security regression, not an acceptable cost.

## Decision

Implement **native declarative monthly range partitioning** on `audit_receipts`,
with a dedicated table preserving *global* nonce uniqueness, and a maintenance
function instead of an external extension.

### 1. Partition the table

```sql
CREATE TABLE audit_receipts ( … ) PARTITION BY RANGE (server_challenge_ts);
```

- **Composite primary key** `(receipt_id, server_challenge_ts)` and **local unique**
  `(challenge_nonce, server_challenge_ts)` — the minimum change that satisfies the
  partition-key rule. `receipt_id` remains globally unique in practice
  (`gen_random_uuid`); the timestamp is appended only to satisfy Postgres.

### 2. A `DEFAULT` partition (proportionate to V2 scale)

```sql
CREATE TABLE audit_receipts_default PARTITION OF audit_receipts DEFAULT;
```

At V2 volume all rows land here and the table "just works" — no operator action
required. Emitting a `DEFAULT` partition (rather than a `NOW()`-based monthly
partition) also keeps the generator's output **deterministic**, preserving the
"committed `.sql` == generator output" invariant.

### 3. Global nonce guard table (preserves Invariant 5)

```sql
CREATE TABLE audit_receipt_nonces (
    challenge_nonce      BYTEA        PRIMARY KEY CHECK (octet_length(challenge_nonce) = 33),
    server_challenge_ts  TIMESTAMPTZ  NOT NULL
);
```

A partitioned table cannot enforce global uniqueness on `challenge_nonce` alone, so
this small **non-partitioned** table does. The microservice writes the nonce here in
the **same transaction** as the receipt (IC §6); a duplicate raises a PK violation
and aborts the audit write — the replay is rejected. Because replay is only feasible
inside the challenge-validity / secret-rotation window, this table has **bounded
retention**: the migrator prunes rows past the window, keeping its index small even
at V3 scale. It is `ENABLE`d + `FORCE`d RLS, insert-only for `vyomanaut_app`
(ADR-032), so a compromised app credential cannot delete guard rows to enable a
replay.

### 4. Maintenance function, not `pg_partman`

```sql
SELECT vyomanaut_create_audit_receipts_partition('2026-02-15'); -- creates audit_receipts_2026_02
ALTER TABLE audit_receipts DETACH PARTITION audit_receipts_2026_01;  -- archival (DDL, not DELETE)
```

A deterministic plpgsql helper creates the month partition covering its argument. A
scheduled job calls it for *next* month before that month's rows arrive; archival is
a `DETACH`. We deliberately avoid `pg_partman`: it is a non-trusted extension, and
architecture.md §25.1 forbids re-introducing a rejected/heavy dependency without an
ADR. This keeps the footprint at the two trusted extensions already in use
(`btree_gist`, `pgcrypto`) and the output deterministic.

## Alternatives considered

- **A — Native partitioning, composite unique on `(challenge_nonce, ts)` only.**
  Rejected: weakens replay protection (Invariant 5) — a **security regression**.
- **C — No partitioning; archive by background `DELETE` of old rows.** Rejected:
  `DELETE` violates Invariant 1. Partitioning + `DETACH` is the only append-only-safe
  archival path.
- **D — Defer partitioning to migration `002`.** Rejected: contradicts the
  day-one mandate and forces an expensive create-copy-swap conversion of a large
  table later; converting from day one is far cheaper.
- **`pg_partman` for automation.** Rejected for V2: non-trusted dependency, §25.1
  discipline, and unnecessary at V2 volume (§26). The maintenance function covers the
  need without it.

## Strength test (PostgreSQL 16, production posture)

| # | Check | Result |
|---|---|---|
| 1 | Migration applies (partitioned parent + default + guard + fn) as non-superuser migrator | `exit 0` |
| 2 | Maintenance fn creates `_2026_01` / `_2026_02`; inserts route correctly | Jan row → `_2026_01`, Feb → `_2026_02` |
| 3 | **Global nonce uniqueness:** replay same nonce, *different* ts | **rejected** by `audit_receipt_nonces_pkey` |
| 4 | RLS through the partitioned parent: app phase-2 `UPDATE` | `UPDATE 1` |
| 5 | app `DELETE FROM audit_receipts` | `permission denied` |
| 6 | **DETACH archival:** `DETACH PARTITION`, rows preserved | `ALTER TABLE`; detached partition still holds its row (no DELETE) |
| 7 | Direct partition access by app (`SELECT FROM audit_receipts_2026_02`) | `permission denied` (defense in depth) |
| 8 | `migration_check.sh` — 30 checks (incl. 5 partitioning/guard) | **all pass, `exit 0`** |
| 9 | Regeneration deterministic; profile diff = header + 2 bounds only | pass |

Test #3 is the decision's crux: partitioning is in place **and** global replay
protection still holds. Test #6 is the payoff: an append-only table whose old data
can be aged out without ever issuing a `DELETE`.

## Exact changes made

**`migrations/generator.go`** (`auditReceiptsSection`):
- `receipt_id` inline `PRIMARY KEY` → `NOT NULL`; add
  `CONSTRAINT audit_receipts_pkey PRIMARY KEY (receipt_id, server_challenge_ts)`.
- `audit_receipts_nonce_unique` → `UNIQUE (challenge_nonce, server_challenge_ts)`.
- `);` → `) PARTITION BY RANGE (server_challenge_ts);`.
- Append `CREATE TABLE audit_receipts_default … DEFAULT`, `audit_receipt_nonces`
  (+ comment), and `vyomanaut_create_audit_receipts_partition(DATE)`.
- `rspSection`: `ENABLE` + `FORCE` RLS on `audit_receipt_nonces` with insert-only +
  select policies for `vyomanaut_app`.
- `grantsSection`: `GRANT SELECT, INSERT ON audit_receipt_nonces TO vyomanaut_app`.

**`migrations/001_initial_schema.sql`, `..._demo.sql`** — regenerated (deterministic;
partitioning/guard are profile-invariant, so the profile diff is unchanged).

**`scripts/ci/migration_check.sh`** — add `audit_receipts_partitioned`,
`audit_receipts_range_partitioned`, `audit_receipts_default_partition`,
`nonce_guard_table_pk`, `partition_maintenance_fn`.

**`docs/data-model.md`** — §4.7 banner, §9 checklist, and the §9 partitioning line
updated.

## Consequences — downstream interface changes (M8+ audit write path)

The application audit write path must change when it is built:

1. **Dual-write in one transaction:** `INSERT INTO audit_receipt_nonces` **then**
   `INSERT INTO audit_receipts`, committed together. A nonce PK violation aborts the
   whole audit (replay rejected).
2. **Carry `server_challenge_ts` in `WHERE` clauses** on `audit_receipts` reads and
   the two-phase `UPDATE`, so the planner can prune to a single partition. Queries
   still work without it (all partitions scanned), but pruning is the point of
   partitioning.
3. **Scheduled maintenance:** a monthly job calls
   `vyomanaut_create_audit_receipts_partition(next month)` as `vyomanaut_migrator`;
   an archival job `DETACH`es closed months after export to cold storage; a pruning
   job trims `audit_receipt_nonces` past the challenge-validity window.

At V2 scale none of the maintenance jobs are load-bearing (everything lives happily
in the `DEFAULT` partition); they become relevant as the network approaches the §26
ceiling.
