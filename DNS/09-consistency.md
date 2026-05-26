# DNS — Consistency Guarantees and Propagation

DNS is generally **eventually consistent** at Internet scale.

## What Consistency You Actually Get

- Authoritative servers can be strongly consistent internally (depending on provider design).
- Recursive resolvers return cached data until TTL expiry.
- Global clients may observe different answers during TTL windows.

So client-visible behavior is not instant global consistency.

## Propagation Explained

When you update a record:

1. Authoritative zone data changes.
2. Some resolvers still hold old cached RRset.
3. New lookups gradually converge as caches expire and refresh.

Convergence time is bounded mainly by previous TTLs (and negative cache TTL for NXDOMAIN/NODATA cases).

## SOA Serial and Zone Transfer Context

In primary-secondary authoritative setups:
- SOA serial increments signal zone updates.
- Secondaries pull updates via AXFR/IXFR.
- During transfer lag, authoritative nodes themselves may temporarily diverge.

## Operational Patterns

- Lower TTL before planned migration.
- Wait at least old TTL duration before assuming full cutover.
- Keep old and new backends live during transition period.
- Monitor from multiple recursive resolvers and regions.

## Practical Guarantee Statement

DNS usually offers:
- High availability
- Bounded staleness (approximately TTL-bound from client perspective)
- Eventual convergence

It does not guarantee globally linearizable name-to-address reads.

---

**Navigation:**
- [<- Previous: How DNS Scales](08-scaling.md)
- [Back to Index](README.md)
- [Next: Troubleshooting and Runbook ->](11-troubleshooting-and-runbook.md)
