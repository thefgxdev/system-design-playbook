# Capacity planning

## The question

Not "how many users can we handle" but "what breaks first, at what load, and what happens then".

## The method

1. **Find the unit of work that matters.** Checkouts per minute, messages per second, reports per hour. Not "requests".
2. **Measure the cost of one unit** across every resource it touches: CPU, database connections, queue slots, third-party calls, disk. Use a real trace, not an estimate.
3. **Find the bottleneck.** The resource that saturates first at a given rate. It is almost never the one people guess; it is usually database connections or a single-threaded step.
4. **Set a ceiling and a plan.** At 70 % of the bottleneck, what do you do? Scale (how, how fast), shed load (which requests are dropped first), or degrade (which features turn off).
5. **Rehearse it.** A load test that stops at the current traffic level tells you nothing. Push until something breaks and write down what.

## Averages lie

A service that handles 1,000 requests per second on average sees 5,000 for a few seconds at the top of every hour, and 20,000 during a campaign. Plan for the p99 of the *rate*, and measure latency at the p99 of the *requests*.

## The three limits people forget

- **Connection pools.** Postgres handles a few hundred connections well. Every web instance opens its own pool. Ten instances × 50 connections = 500 connections and a database that spends its time context-switching. Use a pooler (PgBouncer) and size pools from the database side.
- **Third-party quotas.** Your payment provider allows 100 requests per second. Your campaign needs 300. Ask before, not during.
- **Single-threaded steps.** A cron job that processes tenants serially, a queue consumer with concurrency 1, a report generator that holds a lock. They are fine until they are not.

## Cost is a capacity dimension

Autoscaling that works technically can bankrupt the month. Every scaling plan has a cost ceiling and a person who is paged when it is approached.

## Checklist

- [ ] Unit of work defined and measured end to end.
- [ ] Bottleneck identified by load test, not by intuition.
- [ ] Plan for 70 %, 90 % and 120 % of the bottleneck, written down.
- [ ] Connection pools sized from the database side.
- [ ] Third-party quotas known and monitored.
- [ ] Cost ceiling and alert.
