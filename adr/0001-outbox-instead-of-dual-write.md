# ADR-0001: Publish domain events through a transactional outbox

- **Status:** accepted
- **Date:** 2026-03-10
- **Deciders:** platform team, order team
- **Reversibility:** moderate (a sprint)

## Context

The order service writes to Postgres and publishes `OrderCreated` to the message broker. Twice in six months an order was created without the event (process restarted during deploy between commit and publish). Billing did not run; the customer was not charged; finance found it three weeks later.

## Options considered

1. **Publish inside the transaction, before commit.** Broker acknowledges, transaction rolls back, phantom event. Rejected.
2. **Two-phase commit between Postgres and the broker.** Supported by neither in our setup; operationally heavy. Rejected.
3. **Transactional outbox with a polling relay.** One more table, one more process, consumers must be idempotent (they already are for other reasons). Latency 200 ms.
4. **Do nothing and reconcile nightly.** Cheap, but the gap already cost a customer escalation. Rejected.

## Decision

Option 3. The outbox table lives in the order database; a relay process polls every 200 ms and publishes in `aggregate_id` order; published rows are deleted after seven days.

## Consequences

- Easier: no more lost events; publishing is retried automatically.
- Harder: one more process to monitor. Alert on oldest unpublished row > 5 s.
- Must do: idempotency keys on billing consumer verified (done), outbox cleanup job (ticket PLAT-231), runbook for relay restart.
- Revisit if: event volume exceeds 5,000/s (consider WAL tailing) or if the broker gains transactional support with Postgres.
