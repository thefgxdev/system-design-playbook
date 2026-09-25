# Graceful degradation

## The principle

A dependency will be down. The decision of what still works when it is down must be made in a meeting, not during the incident.

## Classify every feature

| Class | Example | When the dependency is down |
|---|---|---|
| Critical | Login, checkout, the core write | Must work, or the system is down. Redundancy and fallbacks are budgeted here. |
| Important | Search, recommendations, notifications | Degrade: cached results, simpler results, queued for later. |
| Optional | Analytics, A/B tests, third-party widgets | Turn off silently. |

If every feature is classified as critical, nothing degrades and everything fails together.

## Techniques

- **Timeouts everywhere.** A call without a timeout is a call that can hang your whole request thread pool. Timeouts are shorter than the caller's timeout, always.
- **Circuit breakers.** After N failures, stop calling the dependency for a while and use the fallback. Half-open to test recovery.
- **Bulkheads.** Separate thread pools or connection pools per dependency, so a slow search service cannot starve checkout.
- **Cached fallbacks.** Serve the last known good response with a "may be stale" marker. Better than an error for anything that is not a write.
- **Queues for writes that can wait.** Email, webhooks, analytics events: accept, enqueue, return. The user does not need the email to be sent inside the request.
- **Kill switches.** A flag per optional feature, flippable without a deploy.

## Honest UI

When degraded, say so. "Search is temporarily limited" beats an empty result that looks like a bug. Match the words to the state: "we received your order and will confirm by email" is honest; "your order is confirmed" while it sits in a retry queue is not.

## How it fails

- Timeouts longer than the upstream timeout, so the caller gives up first and the work is wasted.
- Fallbacks that were never exercised and are themselves broken.
- Circuit breakers on critical paths that open during a brief blip and turn a hiccup into an outage.
- Degraded mode that nobody can see in the dashboards.

## Checklist

- [ ] Every feature classified: critical, important, optional.
- [ ] Every outbound call has a timeout shorter than the inbound one.
- [ ] Fallbacks exist for important features and are tested in staging monthly.
- [ ] Kill switches for optional features.
- [ ] Degraded state visible in metrics and in the UI.
