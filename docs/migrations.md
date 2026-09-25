# Reversible migrations

## The rule

Every change to a system that cannot stop must be reversible in the time it takes to notice it went wrong. If a change cannot be reversed, it must be small enough that the forward fix is faster than the rollback would have been.

## Expand, migrate, contract

Any schema change that would break the running code is split into three deploys:

1. **Expand.** Add the new column, table or index. Old code ignores it. New code can start writing to both old and new.
2. **Migrate.** Backfill in batches, with a rate limit and a progress table. Verify counts. Switch reads to the new structure behind a flag.
3. **Contract.** Only after the old structure has been unused for a full business cycle, remove it.

Each step is deployable alone and reversible alone. The most common failure is skipping step 3 forever; write the ticket during step 1.

## Data migrations

- Batches of a few thousand rows, with a sleep between them. A single `UPDATE` on a large table locks it and fills the WAL.
- Idempotent: running the migration twice must be safe.
- Progress persisted in a table, so a restart continues instead of starting over.
- Verified: row counts, checksums or sampled comparisons before switching reads.

## Indexes

- `CREATE INDEX CONCURRENTLY` in Postgres. Without it, the table is locked for writes for the duration.
- Concurrent index creation can fail and leave an invalid index. Check `pg_index.indisvalid` and drop it if it did.

## Rehearse the rollback

A rollback that was never run is a hypothesis. Before the change ships:

- Restore last night's backup to a scratch database and time it. That number is your worst-case recovery.
- Run the down migration on the scratch database.
- Write the exact commands in the deploy ticket.

## Feature flags are part of the migration

The code path that reads the new structure ships dark, behind a flag, and is turned on for one tenant, then ten, then all. The flag is the fastest rollback you have.

## Checklist

- [ ] Change split into expand, migrate, contract.
- [ ] Backfill is batched, idempotent and resumable.
- [ ] Indexes created concurrently and verified valid.
- [ ] Rollback rehearsed on a restored backup and written in the ticket.
- [ ] Read path behind a flag with gradual rollout.
- [ ] Contract step has a ticket with a date.
