# DNS — How DNS Scales

DNS serves extremely high QPS globally through protocol simplicity + aggressive caching + distributed infrastructure.

## Primary Scaling Mechanisms

- **Caching at recursive resolvers**: Most queries do not reach authoritative servers.
- **Anycast routing**: Same IP announced from many PoPs; traffic goes to nearest healthy site.
- **Stateless UDP for most queries**: Low per-request overhead (with TCP fallback when needed).
- **Horizontal authoritative fleets**: Multiple nameserver instances per zone/provider.
- **Short response payloads + compression**: Efficient network usage.

## Anycast in Practice

With anycast:
- Many geographically distributed servers share same IP.
- BGP routes clients to topologically nearest instance.
- Traffic shifts automatically during failures/routing changes.

## Capacity Planning Levers

- Increase resolver cache hit ratio (reasonable TTLs)
- Add more anycast PoPs
- Split large zones operationally (sub-zones/delegation)
- Provision DNS providers with enough edge and DDoS mitigation

## Attack and Failure Resilience

- Overprovision and absorb volumetric spikes
- Response rate limiting for abusive patterns
- Multi-provider authoritative DNS for critical domains
- Health-checked failover records (with caution about TTL delays)

## Performance Model (Simple)

Let $Q_c$ be client query rate and $h$ resolver cache hit rate.

Then authoritative query load is approximately:

$$
Q_a \approx Q_c (1 - h)
$$

Even a modest rise in $h$ significantly reduces authoritative load.

---

**Navigation:**
- [<- Previous: Name Server Organization](07-nameserver-organization.md)
- [Back to Index](README.md)
- [Next: Consistency Guarantees ->](09-consistency.md)
