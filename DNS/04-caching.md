# DNS — Caching

Caching is the reason DNS is both fast and scalable.

## Where DNS Is Cached

- Application/browser cache
- OS cache (stub resolver layer)
- Recursive resolver cache (shared by many clients)
- Sometimes intermediary network appliances

## Positive Caching

If answer exists, resolvers cache RRset for up to TTL seconds.

Example:
- `api.example.com A 60` means cached up to 60 seconds.
- Lower TTL = faster change propagation, but more query load.
- Higher TTL = better cache hit rate and lower authoritative load.

## Negative Caching

If name does not exist (NXDOMAIN) or no data for type (NODATA), resolvers cache negative result.

Negative cache duration is derived from SOA fields (commonly the SOA minimum / negative TTL behavior in modern practice).

Operational impact:
- Accidentally querying before creating a record can cause temporary "still not found" behavior even after record is added.

## TTL Trade-Offs (Design View)

- **Low TTL** (e.g. 30-60s):
  - Pros: rapid traffic shifts, quick rollback
  - Cons: higher QPS to authoritative servers, more resolver load
- **High TTL** (e.g. 1h-24h):
  - Pros: low load, stable cache hit ratios
  - Cons: slow failover and rollout propagation

## Common Strategy

- Normal state: moderate/high TTL (5m-1h)
- Planned change window: lower TTL ahead of change
- After validation: raise TTL again

## Cache and Consistency

DNS consistency is not instantaneous globally. Effective behavior is bounded by TTLs and resolver refresh timings.

---

**Navigation:**
- [<- Previous: Record Types and Format](03-record-types-and-format.md)
- [Back to Index](README.md)
- [Next: Live Resolution Walkthrough ->](05-live-resolution-example.md)
