# Key-Value Databases — Advantages and Limitations

## Advantages

| Advantage           | Why                                                                 |
|---------------------|---------------------------------------------------------------------|
| **Blazing Speed**   | O(1) hash-based lookups; in-memory stores like Redis hit sub-millisecond latency |
| **Horizontal Scale**| Consistent hashing makes adding/removing nodes seamless            |
| **Simple API**      | `GET`, `SET`, `DELETE` — easy to integrate into any application    |
| **Flexible Values** | Values can be strings, JSON, binary, counters, lists, sets, etc.   |
| **TTL Support**     | Native expiry makes caching and session management trivial          |
| **High Throughput** | Millions of reads/writes per second on modern hardware             |

## Limitations and Trade-offs

| Limitation                      | Detail                                                                  |
|---------------------------------|-------------------------------------------------------------------------|
| **No query by value**           | You cannot do `SELECT * WHERE age > 25`. Keys only.                    |
| **No relationships/joins**      | No foreign keys or relational semantics                                 |
| **No complex transactions**     | Multi-key atomic transactions are limited or unavailable in many stores |
| **Value opaqueness**            | The database doesn't understand the value — no type enforcement         |
| **Memory cost**                 | In-memory stores (Redis) are expensive at scale                        |
| **Key design burden**           | Poorly designed keys lead to hot spots and uneven distribution          |
| **Limited consistency options** | Eventual consistency can cause stale reads without careful tuning       |

---

**Navigation:**
- [← Previous: How They Work](03-how-they-work.md)
- [Back to Index](README.md)
- [Next: Implementations →](05-implementations.md)
