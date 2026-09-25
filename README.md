# System Design Playbook

Patterns, trade-offs and decision records from real production systems. Written by [Felipe Guedes](https://fgxdev.com), Software Engineer and Systems Architect, from 600+ systems built, reviewed or audited.

This is not a list of buzzwords. Every entry answers three questions: when to use it, what it costs, and how it fails.

## Contents

| Pattern | Use it when | It fails when |
|---|---|---|
| [Transactional outbox](docs/outbox.md) | A database write and a message publish must both happen or neither | You publish first and the write rolls back, or write first and the process dies before publishing |
| [Idempotency keys](docs/idempotency.md) | Any request that costs money or creates a record can be retried | The key is scoped wrong, stored after the side effect, or expires too early |
| [Multi-tenancy boundaries](docs/multi-tenancy.md) | Many customers share one deployment | One forgotten `WHERE tenant_id` leaks data between customers |
| [Capacity planning](docs/capacity.md) | Before the marketing campaign, not after | You measure averages and get killed by the p99 |
| [Reversible migrations](docs/migrations.md) | Every schema or data change in a system that cannot stop | The rollback was never rehearsed |
| [Graceful degradation](docs/degradation.md) | A dependency will be down and you have to decide in advance what still works | Every feature is treated as critical, so nothing degrades and everything falls |

## Principles behind every pattern

1. **The failure is in a boundary someone trusted.** Network calls, queues, third-party APIs, the line between two teams. Design the boundary, not just the happy path.
2. **Decide what error is acceptable before choosing a technology.** "Exactly once" is a promise nobody keeps. "At least once plus idempotent consumer" is a design.
3. **Everything that costs money or is irreversible gets a key, a log and an owner.**
4. **Reversibility is a feature.** A decision you can undo in a sprint is cheap. A decision you cannot undo needs a written record of why.
5. **Measure the tail, not the average.** Users live at the p99.

## How to use this playbook

- Read the pattern before the incident, not during it.
- Copy the decision-record template in [`adr/TEMPLATE.md`](adr/TEMPLATE.md) into your repository and write one record per expensive decision.
- If a pattern here contradicts your context, your context wins. Write down why.

## Related

- [Architecture review checklist](https://github.com/thefgxdev/architecture-review-checklist): the questions I ask in every review.
- [Postgres operations runbook](https://github.com/thefgxdev/postgres-operations-runbook): the database side of the same problems.
- Articles on System Design at [fgxdev.com/articles](https://fgxdev.com/articles/tag/system-design/).

## Em português

Padrões, trade-offs e registros de decisão tirados de sistemas reais em produção. Cada padrão responde: quando usar, quanto custa e como falha. Os artigos em português estão em [fgxdev.com/pt/articles](https://fgxdev.com/pt/articles/tag/system-design/).

## License

MIT. Use it, adapt it, keep the attribution.
