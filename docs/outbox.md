# Transactional outbox

## The problem

An order service saves an order to Postgres, then publishes `OrderCreated` to a queue so billing, email and analytics can react. Two writes, two systems. Between them, the process can die, the network can fail, the broker can time out.

- Publish first, then write: the write fails and the world reacts to an order that does not exist.
- Write first, then publish: the process dies after the commit and nobody reacts to an order that exists.

There is no order of two writes that is safe. The only safe design is one write.

## The pattern

1. In the **same database transaction** as the order, insert a row into an `outbox` table: `id`, `aggregate_id`, `type`, `payload`, `created_at`, `published_at NULL`.
2. A separate **relay** process polls (or tails the WAL) for rows with `published_at IS NULL`, publishes them, and marks them published.
3. Consumers are **idempotent**, because the relay can crash between publishing and marking, and the same message will be published again.

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox (id, aggregate_id, type, payload)
VALUES (gen_random_uuid(), :order_id, 'OrderCreated', :json);
COMMIT;
```

## What it costs

- One more table and one more process to operate and monitor.
- Latency between commit and publish: polling interval, typically 100 ms to 1 s. WAL tailing (Debezium, pg logical replication) brings it near zero at the price of more infrastructure.
- Consumers must be idempotent. This is not optional and it is the part teams skip.

## How it fails

- **Relay stops and nobody notices.** Alert on `MIN(created_at) WHERE published_at IS NULL` older than N seconds.
- **Outbox grows forever.** Delete or archive published rows on a schedule.
- **Ordering assumptions.** Rows are published roughly in order, not strictly. If order matters, partition by `aggregate_id` and process each partition serially.
- **Payload schema drifts.** Version the `type` (`OrderCreated.v2`) and keep consumers tolerant of unknown fields.

## When not to use it

- The two writes are in the same database: use one transaction.
- Losing the event is acceptable (analytics samples, non-critical notifications): publish after commit and accept the gap. Write down that you accepted it.

## Decision record

See [`../adr/0001-outbox-instead-of-dual-write.md`](../adr/0001-outbox-instead-of-dual-write.md) for a filled-in example.
