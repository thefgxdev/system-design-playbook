# Idempotency keys

## The problem

A user clicks "pay". Nothing visibly happens for 800 ms. They click again. Two requests arrive. Or one request arrives, the charge succeeds, the response is lost, and the client retries. In both cases the system must charge once.

The boundary that matters is the one where an action becomes irreversible or expensive: charging money, sending an email, calling a third-party API with a quota, creating a record another system reacts to. Everything before that boundary can be replayed for free. Everything after it needs a key.

## The pattern

1. The client generates a key per **intent** (not per request): a UUID created when the form is shown, sent in an `Idempotency-Key` header on every retry of that intent.
2. The server, **before** doing the side effect, records the key with state `in_progress` in a store with a uniqueness constraint. If the insert fails because the key exists, it returns the stored response (or `409` if still in progress).
3. After the side effect, the server stores the result under the key and returns it.
4. Keys expire after a window that is longer than any plausible retry (24 h is common).

```
INSERT INTO idempotency (key, scope, status, created_at)
VALUES (:key, :user_id, 'in_progress', now())
ON CONFLICT (key, scope) DO NOTHING;
-- 0 rows inserted → return stored response or 409
```

## Scope matters

The key must be scoped to the principal. Two users must be able to send the same key without colliding, and one user must not be able to replay another user's key. Scope = `(key, user_id)` or `(key, tenant_id)`.

## What it costs

- A store with a uniqueness constraint and an expiry job. Postgres is enough; Redis works if you accept losing keys on failover.
- Clients must be taught to reuse the key on retry. Most HTTP clients retry with a new request; wire the key into the retry logic.

## How it fails

- **Key stored after the side effect.** The process dies in between and the retry charges again. Store first.
- **Key expires before the last retry.** Mobile clients retry hours later. Size the window for your worst client.
- **Different payload, same key.** Decide: reject with `422`, or treat the key as authoritative. Reject is safer.
- **Idempotency at the edge but not downstream.** The API is idempotent; the worker it enqueues is not. Every boundary that costs money needs its own key.

## Checklist

- [ ] Every endpoint that charges, sends, deletes or creates a record another system reacts to accepts an idempotency key.
- [ ] The key is scoped to the principal.
- [ ] The key is persisted before the side effect.
- [ ] Concurrent requests with the same key get a deterministic answer.
- [ ] Expiry is longer than the longest retry your clients perform.
