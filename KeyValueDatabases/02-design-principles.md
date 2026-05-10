# Key-Value Databases — Design Principles

## The Data Model

A Key-Value store has the most minimal schema of any database:

```
Key   : unique, immutable, usually a string or binary sequence
Value : opaque blob — the database does not know or care what it contains
```

**Key design conventions** commonly used in production systems:

| Pattern               | Example                        | Purpose                          |
|-----------------------|--------------------------------|----------------------------------|
| `type:id`             | `user:1001`                    | Namespace to avoid collisions    |
| `type:id:field`       | `user:1001:email`              | Fine-grained per-field storage   |
| `prefix:resource:ver` | `cache:product:555:v3`         | Versioned cache keys             |
| `queue:name`          | `queue:email_jobs`             | Message/task queues              |

Because the entire interface is just `get(key)` and `set(key, value)`, the key *is* your query. Designing keys well is the most important modelling decision in a KV store.

## Storage Mechanisms

Under the hood, different KV stores use different storage engines:

### a) Hash Table (In-Memory)

The fastest possible lookup. Keys are hashed to bucket indices, giving O(1) average-case reads/writes. Redis uses this for its primary data structure.

```
Key  →  hash(key)  →  bucket index  →  value

"user:1001"  →  0x3F2A  →  bucket[43]  →  {name: "Rahul"}
"user:1002"  →  0x7B1C  →  bucket[17]  →  {name: "Priya"}
```

```
┌────────────────────────────────────────────────────┐
│                  Hash Table                        │
│                                                    │
│  bucket[0]  → [ "cfg:timeout"   → 30  ]           │
│  bucket[1]  → [ empty                 ]           │
│  bucket[2]  → [ "user:1002"     → {...} ]         │
│  bucket[3]  → [ empty                 ]           │
│     ...                                            │
│  bucket[43] → [ "user:1001"     → {...} ]         │
│     ...                                            │
└────────────────────────────────────────────────────┘
```

### b) Log-Structured Merge Tree (LSM Tree)

Used by RocksDB, Cassandra, and LevelDB. Writes are **always appended** to an in-memory buffer (MemTable). When the MemTable is full, it is flushed to disk as an immutable sorted file (SSTable). Periodic **compaction** merges and garbage-collects SSTables.

```
Write Path:
  New write  →  WAL (Write-Ahead Log)  →  MemTable (in memory)
                                             │
                                   (when full, flush)
                                             ↓
                                        SSTable L0
                                             │
                                   (compaction)
                                             ↓
                                   SSTable L1 … Ln (on disk)
```

```
Read Path:
  Lookup key  →  MemTable  (not found?)
               →  L0 SSTable (not found?)
               →  L1 SSTable (not found?)
               →  ...
               →  found → return value
```

Bloom Filters are used at each level to skip SSTables that definitely do not contain the key — keeping read latency manageable.

### c) B-Tree / B+ Tree (On-Disk)

Traditional databases (including some KV stores) use B-Trees for on-disk storage. Data is stored in balanced tree nodes of fixed page size. Every write updates the tree in-place (unlike LSM which appends).

```
                     [ 30 | 60 ]
                    /     |     \
          [10|20]  [40|50]  [70|80]
         /  |  \   ...
     [5] [15] [25] ...
```

- Good for **read-heavy** workloads (fewer levels to traverse).
- Writes can be slow due to random I/O and page splits.

## Indexing for Fast Lookups

The primary index in a KV store is always the **key itself** (via a hash or sorted tree). There is deliberately no secondary indexing by value — that would add complexity and overhead.

**How lookups stay fast:**

1. **Hash-based index** — O(1) lookup. Key is hashed; the hash points directly to where the value lives in memory or on disk.
2. **Sorted index (B-Tree / SSTable)** — O(log n) lookup. Useful when you need range scans (e.g., all keys from `user:1000` to `user:2000`).
3. **Bloom Filters** — A probabilistic data structure that answers "this key definitely does NOT exist" in O(1). Used alongside LSM trees to skip unnecessary disk reads.

```python
# Conceptual Bloom Filter usage
class BloomFilter:
    def __init__(self, size=1000):
        self.bits = [0] * size
        self.size = size

    def _hashes(self, key):
        import hashlib
        h1 = int(hashlib.md5(key.encode()).hexdigest(), 16) % self.size
        h2 = int(hashlib.sha1(key.encode()).hexdigest(), 16) % self.size
        return h1, h2

    def add(self, key):
        for h in self._hashes(key):
            self.bits[h] = 1

    def might_contain(self, key):
        return all(self.bits[h] == 1 for h in self._hashes(key))

bf = BloomFilter()
bf.add("user:1001")
bf.add("user:1002")

print(bf.might_contain("user:1001"))  # True  (definitely or false positive)
print(bf.might_contain("user:9999"))  # False (definitely NOT in store)
```

**TTL (Time-To-Live):** Most KV stores support per-key expiration. Internally, a background thread periodically evicts expired keys, or expiry is checked lazily on read.

```python
# Redis-style TTL concept
store["session:abc"] = "token_value"
ttl["session:abc"] = time.time() + 3600  # expires in 1 hour

def get(key):
    if key in ttl and time.time() > ttl[key]:
        del store[key]
        del ttl[key]
        return None
    return store.get(key)
```

---

**Navigation:**
- [← Previous: Introduction](01-introduction.md)
- [Back to Index](README.md)
- [Next: How They Work →](03-how-they-work.md)
