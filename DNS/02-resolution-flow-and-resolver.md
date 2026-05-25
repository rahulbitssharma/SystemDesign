# DNS — Resolution Flow and Resolver Role

## What Is a DNS Resolver?

A DNS resolver is the component that answers "what is the IP (or other record) for this name?"

There are two common resolver roles:

- **Stub resolver** (on host/OS): minimal client that forwards queries.
- **Recursive resolver** (in network): performs full lookup by querying other nameservers and caching results.

When people say "DNS resolver" in production architecture, they usually mean the **recursive resolver**.

## End-to-End Resolution (Typical)

1. App asks OS for `www.example.com`.
2. OS checks local sources (browser cache, OS cache, `/etc/hosts`).
3. On miss, OS stub sends query to configured recursive resolver.
4. Recursive resolver checks its cache.
5. On miss, recursive resolver asks a root server for `.com` delegation.
6. Resolver asks `.com` TLD server for `example.com` delegation.
7. Resolver asks authoritative server for `www.example.com` record.
8. Authoritative server replies (e.g. CNAME -> A/AAAA chain).
9. Recursive resolver caches each RRset by TTL and returns final answer to client.

## Query Types: Recursive vs Iterative

- **Recursive query**: Client asks resolver "give me final answer".
- **Iterative query**: Resolver asks nameservers and gets referrals until it reaches authority.

### Why It Is Called a Recursive Resolver

The resolver is called "recursive" because of the service contract it offers to clients, not because every upstream hop is recursive.

- Stub/host sends query with recursion desired.
- Recursive resolver takes full responsibility for returning a final answer (or final error).
- Internally, that resolver usually performs iterative queries to root, TLD, and authoritative servers.

So both statements are true at once: client->resolver interaction is recursive; resolver->upstream interaction is usually iterative.

## How Browser Requests Reach a DNS Resolver

When a browser needs `www.example.com`, name resolution usually flows like this:

1. Browser asks OS networking APIs to resolve the hostname (for example, via `getaddrinfo`-style calls).
2. OS stub resolver checks local sources/caches and then queries configured recursive resolver(s).
3. Recursive resolver returns answer to OS, and OS returns it to the browser.

Who configures which recursive resolver is used?

- Commonly the OS/network stack via DHCP-provided DNS servers from router/ISP, or manual/enterprise policy settings.
- Not directly "the internet" deciding per request.
- Browser code initiates lookup, but resolver choice is typically owned by OS/network configuration.

Important exception: if browser DNS-over-HTTPS is enabled, browser may send DNS directly to its configured DoH provider instead of OS-configured UDP/TCP resolver path.

## Why Recursive Resolvers Are Critical

- Reduce latency through shared cache hits.
- Protect authoritative servers from repeated traffic.
- Apply policy/security (DNSSEC validation, filtering, split-horizon rules).
- Improve resilience using retries and alternate upstream paths.

## Failure Scenarios to Understand

- Resolver unavailable: apps fail to resolve names quickly.
- Stale cache after change: traffic may continue to old endpoint until TTL expiry.
- Bad delegation (NS/glue mismatch): domain partially or fully unresolvable.

---

**Navigation:**
- [<- Previous: Introduction](01-introduction.md)
- [Back to Index](README.md)
- [Next: Record Types and Format ->](03-record-types-and-format.md)
