## Phase 84: `metap-reconciler` sync-trigger ledger drift — found live on `metap-demo-waf`, fixed at the root (2026-09-12)

Trigger: debugging a real WAF portal report ("tạo policy được nhưng portal không hiện" — a DDoS
policy create appeared to succeed but the portal seemed not to reflect it). Root-caused during
that debugging session as 3 separate incidents, all `metap-demo-waf`'s own live database, all the
same *class* of bug — a tracking/ledger table saying "done" while the real database object it
stands in for had silently stopped existing. Full incident narrative, evidence, and the exact SQL
used to unblock lives in `../../metap-demo-waf/CLAUDE.md` (search "2 more incidents found live
2026-09-12" — bugs #7/#8/#9 in that file's running incident log); this entry is the `metap` core
side: what was actually fixed in the platform, not the WAF-specific unblock.

**7th (environment, not a `metap` code bug): the shared dev Postgres's `pg_trgm` extension had
silently vanished** despite `_sqlx_migrations` recording migration 16
(`CREATE EXTENSION IF NOT EXISTS pg_trgm;`) as applied — `zones-service` failed to boot entirely
(`operator class "gin_trgm_ops" does not exist for access method "gin"`). Not root-caused why the
extension disappeared. Fixed by re-running the `CREATE EXTENSION` by hand.

**8th (real `metap-reconciler` bug, fixed at the root): `introspect()` trusted
`reconciler_backfill_progress.completed` as proof a `Generated` column's `BEFORE INSERT OR UPDATE`
sync trigger (`executor::build_sync_trigger_sql`) actually exists, without ever re-checking
`pg_trigger`/`pg_proc`.** `waf.ddos_policies.zoneId`'s trigger (and its backing function) had been
dropped from the live database by something outside the reconciler's own lifecycle (not
root-caused how), while the ledger still said `completed = true` from the original 2026-09-07/08
table-per-entity migration. Consequence: `diff()`'s `Some(actual_spec)` branch for an
already-existing `Generated` column only re-asserts the trigger when `introspect()` reports
`backfilled: false` — with the ledger stuck at `true`, `ops_applied` stayed `0` on every boot
forever, and every `createWafDdosPolicies` since the trigger vanished wrote a correct
`data->>'zoneId'` (so reads via the JSONB blob looked fine) while leaving the real `"zoneId"`
column `NULL` — Postgres's `NULL <> NULL` semantics meant `uniq_waf_ddos_policies_zoneId ...
WHERE deleted = false` enforced nothing at all, and 11 duplicate policies for one zone got created
back to back.

**Fix**: `crates/metap-reconciler/src/introspect.rs` gained `sync_trigger_exists(pool, schema,
bare_table, field)`, querying `pg_trigger`/`pg_class`/`pg_namespace` directly for the exact
trigger name `build_sync_trigger_sql` would have created (`trg_sync_{bare_table}_{field}`). A
`Generated` column now reads back as `backfilled: true` only when **both** the ledger says so
**and** the trigger genuinely exists — either being false re-triggers `diff()`'s existing
`push_sync_and_backfill` re-assertion path (already idempotent `CREATE OR REPLACE`, no new `DdlOp`
variant needed). New regression test,
`reconcile_recreates_a_sync_trigger_dropped_outside_its_own_lifecycle`
(`crates/metap-reconciler/tests/reconcile_postgres.rs`) — reconciles once, hand-drops the trigger
+ function exactly as found live, reconciles again, asserts it's actually re-created rather than
trusting the stale ledger row. `cargo test -p metap-reconciler -- --ignored` (25/25) and
`cargo check`/`clippy --workspace -- -D warnings` both clean against real Postgres.

**9th (found while manually unblocking the 8th on the live WAF database, NOT yet fixed — flagged
for a direction decision, not resolved unilaterally): `backfill::run_batched_update`'s batch
UPDATE scopes every row by a single `tenant_id`, which is correct and already regression-tested
for a genuinely per-tenant table, but silently backfills zero rows — while still calling
`mark_completed` — when the caller reconciles with a sentinel tenant id that owns no real rows in
a *shared*, `Schema`-strategy table.** `zones-service` (and `scanning-service`/`alerting-service`,
same pattern) always reconciles at boot with `metap::control::PLATFORM_TENANT_ID` (`Uuid::nil()`),
which is correct for schema-wide DDL (table/column/trigger creation) but can never reach any real
tenant's existing rows for the row-level backfill step — `run_batched_update`'s own doc comment
states the assumption this violates: "every dedicated table belongs to exactly one `DedicatedDb`
tenant", true for `../../metap-demo-jira`'s per-tenant databases, false for every `Schema`-strategy
shared table (all 9 WAF entities, and any future one). Two directions flagged, neither decided:
(a) give `run_batched_update`/`BackfillColumn` a "backfill every tenant's rows in this table"
variant a caller opts into when it already knows the table is shared, or (b) have `executor.rs`
resolve the real distinct tenant-id set present in the table and backfill once per real tenant
instead of once with whatever sentinel the caller's own `reconcile()` invocation used.
`mark_completed` firing after a query that matched zero rows for the *wrong reason* (not "already
done", but "was never going to find anything" for this tenant id) needs to stop looking identical
to genuine completion either way.

**Broader same-day audit of `metap-reconciler`** for the same failure class (ledger trusted
without re-verifying `pg_catalog`) across `diff.rs`, `watchdog.rs`, `migration.rs`/`migrate.rs`,
and `orchestrator.rs`: found one more instance of the same shape as the 9th finding, lower risk —
`migrate.rs::copy_generic_records`'s `mark_completed` also fires unconditionally regardless of
rows actually moved, but every current caller of that specific function always passes a real
tenant id for a real one-shot migration, never the `PLATFORM_TENANT_ID` sentinel that triggered
the 9th finding — a latent defense-in-depth gap, not a live bug today. No other instances found;
`watchdog.rs` re-derives everything from the next `reconcile()`'s own `introspect()` call (no
independent cached state of its own), and `orchestrator.rs`'s `reconciler_entity_deployments` is
written immediately after a real `reconcile()` call succeeds in the same call chain, not read back
later as a substitute for checking reality independently.
