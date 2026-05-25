# DNS (Domain Name System)

A practical, system-design focused guide to how DNS works in production: query flow, records, caching, resolver internals, nameserver hierarchy, scaling, and consistency.

## Table of Contents

1. **[Introduction](01-introduction.md)** — Why DNS exists, key concepts, and where it sits in request paths.
2. **[DNS Resolution Flow and Resolver Role](02-resolution-flow-and-resolver.md)** — Stub resolver, recursive resolver, root/TLD/authoritative flow.
3. **[DNS Record Types and Wire/Data Format](03-record-types-and-format.md)** — A/AAAA/CNAME/MX/TXT/NS/SOA/SRV/PTR and zone file syntax.
4. **[Caching in DNS](04-caching.md)** — Positive/negative caching, TTLs, resolver/browser/OS caches, cache invalidation trade-offs.
5. **[Live Resolution Walkthrough](05-live-resolution-example.md)** — Real command output and step-by-step explanation.
6. **[/etc/hosts and Local Name Overrides](06-etc-hosts-and-local-overrides.md)** — Where hosts file fits before DNS.
7. **[How Name Servers Are Organized](07-nameserver-organization.md)** — Root server letters, TLDs, authoritative providers, glue records.
8. **[How DNS Scales](08-scaling.md)** — Anycast, distributed authoritative fleets, QPS handling, resolver cache economics.
9. **[Consistency Guarantees and Propagation](09-consistency.md)** — Eventual consistency, TTL windows, SOA serials, operational patterns.
10. **[Code Examples and Diagrams](10-code-examples-and-diagrams.md)** — Python examples and sequence/architecture diagrams.
11. **[Troubleshooting and Runbook](11-troubleshooting-and-runbook.md)** — NXDOMAIN/SERVFAIL/timeout diagnostics, DNSSEC checks, and incident response steps.

---

## Quick Start

**New to DNS?** Start with [Introduction](01-introduction.md) and [Resolution Flow and Resolver Role](02-resolution-flow-and-resolver.md).

**Need practical debugging?** Go to [Live Resolution Walkthrough](05-live-resolution-example.md), [Code Examples and Diagrams](10-code-examples-and-diagrams.md), and [Troubleshooting and Runbook](11-troubleshooting-and-runbook.md).

**Designing for reliability?** Read [How DNS Scales](08-scaling.md) and [Consistency Guarantees](09-consistency.md).
