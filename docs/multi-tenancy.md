# Multi-tenancy boundaries

## The problem

Many customers share one deployment. The cheapest failure to cause and the most expensive to explain is one customer seeing another customer's data. It happens through one forgotten filter, one admin endpoint, one background job that iterates "all rows".

## Three models

| Model | Isolation | Cost | Use when |
|---|---|---|---|
| Shared schema, `tenant_id` column | Lowest; enforced by discipline or row-level security | Lowest | Many small tenants, one product |
| Schema per tenant | Medium; a connection can only see its schema | Medium; migrations run N times | Dozens to hundreds of tenants with compliance needs |
| Database per tenant | Highest | Highest; operations multiply | Few large tenants, regulated data, per-tenant backups |

Most products start with the first model and should. The rest of this document is about making it safe.

## Making shared schema safe

1. **Row-level security in the database, not only in the application.** `ALTER TABLE orders ENABLE ROW LEVEL SECURITY` with a policy on `current_setting('app.tenant_id')`. The application sets the setting per connection. A forgotten `WHERE` then returns nothing instead of everything.
2. **Tenant comes from the session, never from the request body.** The tenant is derived from the authenticated principal. Any `tenant_id` in a payload is validated against it or ignored.
3. **Every table with tenant data has a `tenant_id` column and an index that starts with it.**
4. **Background jobs are per tenant.** A job that "processes everything" must iterate tenants explicitly and set the tenant context for each.
5. **Admin tooling runs with a named tenant.** "God mode" queries are logged with who ran them and why.

## Noisy neighbours

Isolation is also about capacity. One tenant's batch import must not slow everyone's checkout.

- Rate limits per tenant, not only per user.
- Queues partitioned by tenant, or at least a separate queue for bulk work.
- Per-tenant metrics: p99 latency, error rate, queue depth. You cannot protect what you cannot see.

## How it fails

- A report endpoint written "for the dashboard" without a tenant filter.
- A cache key without the tenant in it.
- A search index shared across tenants without a tenant filter in every query.
- A file storage path that is guessable across tenants.

## Checklist for a review

- [ ] Row-level security or equivalent enforced at the data layer.
- [ ] Tenant derived from the session; request payloads cannot override it.
- [ ] Cache keys, search queries, storage paths and queue messages all carry the tenant.
- [ ] Background jobs set tenant context explicitly.
- [ ] Per-tenant rate limits and metrics exist.
- [ ] There is a test that logs in as tenant A and tries to read tenant B's data through every list endpoint.
