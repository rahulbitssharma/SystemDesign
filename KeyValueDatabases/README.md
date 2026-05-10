# Key-Value Databases

A comprehensive guide to understanding, implementing, and scaling key-value databases. This resource covers everything from basic concepts to distributed systems design patterns.

## Table of Contents

1. **[Introduction](01-introduction.md)** — What is a KV store? O(1) lookups and opaque values.
2. **[Design Principles](02-design-principles.md)** — Data models, storage engines (hash tables, LSM trees, B-trees), indexing, and TTL.
3. **[How They Work](03-how-they-work.md)** — Read/write workflows, distributed architecture, consistency models (strong vs eventual), and quorum-based tuning.
4. **[Advantages and Limitations](04-advantages-limitations.md)** — Pros (speed, scale, simplicity) and cons (no queries by value, no JOINs, key design burden).
5. **[Popular Implementations](05-implementations.md)** — Redis, DynamoDB, Memcached, etcd, RocksDB, Aerospike, and others.
6. **[Building a Caching System](06-caching-example.md)** — Full Python example with cache hits/misses, TTL, and eviction policies (LRU, LFU, etc).
7. **[Real-World Use Cases](07-use-cases.md)** — Session management, rate limiting, leaderboards, pub/sub, feature flags, and configuration stores.
8. **[Architecture Diagrams](08-architecture-diagrams.md)** — Single-node, distributed clusters, write/read workflows, and replication patterns.
9. **[Scalability and Performance](09-scalability.md)** — Why SQL databases struggle to scale, horizontal partitioning, denormalisation, and detailed E-commerce examples.
10. **[Fault Tolerance](10-fault-tolerance.md)** — How KV stores and SQL databases handle failures. Replication, quorum writes, failover strategies, case studies (Twitter, Stripe, Netflix), and scenario-based trade-offs.

---

## Quick Start

**New to KV stores?** Start with [Introduction](01-introduction.md) for a gentle introduction.

**Want to understand the internals?** Jump to [Design Principles](02-design-principles.md) and [How They Work](03-how-they-work.md).

**Looking for implementation details?** See [Popular Implementations](05-implementations.md) and [Building a Caching System](06-caching-example.md).

**Scaling a distributed system?** Read [Scalability and Performance](09-scalability.md) for deep dives into sharding, consistency, and real-world trade-offs.

**Concerned about reliability?** Read [Fault Tolerance](10-fault-tolerance.md) to understand how KV stores and relational databases handle failures, with real case studies (Twitter, Stripe, Netflix).

---

## Key Concepts at a Glance

| Concept           | Description                                             |
|-------------------|---------------------------------------------------------|
| **O(1) Lookup**   | Hash-based indexing for instant data retrieval         |
| **Horizontal Scale** | Partition data across nodes using consistent hashing  |
| **TTL / Expiry**  | Per-key expiration for automatic cleanup               |
| **Consistency**   | Tunable via quorums: strong vs eventual                |
| **No JOINs**      | Denormalise data; each item is self-contained          |
| **LSM Trees**     | Write-optimised storage engine (RocksDB, Cassandra)   |
| **Replication**   | Primary + replicas for availability                   |

---

## Archived Content (1477-line original)

All content has been split into 9 focused markdown files listed above. The original monolithic document is available upon request.

