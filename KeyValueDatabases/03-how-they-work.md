# Key-Value Databases — How They Work

## Write Workflow

```
Client
  │
  │  SET "user:1001" → {"name":"Rahul"}
  ▼
┌─────────────────────────────────┐
│          KV Store Node          │
│                                 │
│  1. Hash the key                │
│     hash("user:1001") = 0x3F2A  │
│                                 │
│  2. Write to WAL                │
│     (durable, sequential write) │
│                                 │
│  3. Update in-memory structure  │
│     (MemTable / Hash Table)     │
│                                 │
│  4. ACK to client               │
└─────────────────────────────────┘
         │  (async / on threshold)
         ▼
  Flush to disk (SSTable / B-Tree page)
         │
         ▼
  Replicate to follower nodes
```

```python
# Simplified write path simulation
import hashlib
import json
import time

class SimpleKVStore:
    def __init__(self):
        self._store = {}
        self._wal = []      # Write-Ahead Log
        self._expiry = {}

    def set(self, key: str, value, ttl_seconds: int = None):
        # Step 1: Write to WAL for durability
        self._wal.append({"op": "SET", "key": key, "value": value, "ts": time.time()})

        # Step 2: Update in-memory hash table
        self._store[key] = value

        # Step 3: Set TTL if provided
        if ttl_seconds is not None:
            self._expiry[key] = time.time() + ttl_seconds

    def get(self, key: str):
        # Check expiry first
        if key in self._expiry and time.time() > self._expiry[key]:
            del self._store[key]
            del self._expiry[key]
            return None
        return self._store.get(key)

    def delete(self, key: str):
        self._wal.append({"op": "DEL", "key": key, "ts": time.time()})
        self._store.pop(key, None)
        self._expiry.pop(key, None)

# Usage
kv = SimpleKVStore()
kv.set("user:1001", {"name": "Rahul", "age": 30}, ttl_seconds=300)
print(kv.get("user:1001"))   # {'name': 'Rahul', 'age': 30}
kv.delete("user:1001")
print(kv.get("user:1001"))   # None
```

## Read Workflow

```
Client
  │
  │  GET "user:1001"
  ▼
┌──────────────────────────────────────────┐
│             KV Store Node                │
│                                          │
│  1. Hash the key                         │
│     hash("user:1001") = 0x3F2A           │
│                                          │
│  2. Check in-memory cache / MemTable     │
│     → FOUND? Return immediately (fast)  │
│     → NOT FOUND? Continue...            │
│                                          │
│  3. Check Bloom Filter for each SSTable  │
│     → Definitely NOT present? Skip file │
│     → Might be present? Read SSTable    │
│                                          │
│  4. Return value (or null if not found)  │
└──────────────────────────────────────────┘
```

## Distributed Architecture

In a distributed KV store, data is spread across multiple nodes using **consistent hashing**. Each key maps to a position on a virtual ring, and the node responsible for that range of positions owns the key.

```
              Consistent Hash Ring
              ┌─────────────────┐
          0°  │                 │  360°
              │   Node A        │
          ┌───┤  (0° – 120°)    ├───┐
          │   │                 │   │
  Node C  │   │                 │   │  Node B
(240°–360°│   │                 │   │ (120°–240°)
          └───┤                 ├───┘
              │                 │
              └─────────────────┘

  "user:1001"  →  hash → 85°  → owned by Node A
  "product:55" →  hash → 175° → owned by Node B
  "session:zz" →  hash → 310° → owned by Node C
```

**Replication:** Each key is replicated to N successor nodes on the ring (typically N=3). This ensures availability if a node fails.

```
  Key "user:1001" hashes to Node A (primary)
  Replicas on: Node B, Node C (the next 2 nodes clockwise)

  Node A  ──replicate──►  Node B  ──replicate──►  Node C
  (primary)               (replica 1)              (replica 2)
```

## Consistency Models

### Strong Consistency

Every read reflects the most recent write. Achieved by requiring all replicas to acknowledge a write before the client receives a success response.

```
Client  ──SET "x"=5──►  Node A (primary)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
                 Node B               Node C
               (replica)            (replica)
                    │                    │
                    └────── ACK ─────────┘
                              │
Node A waits for ALL ACKs, then responds to client.

Client  ──GET "x"──►  Node B  →  returns 5  ✓ (always latest)
```

**Trade-off:** Higher latency (must wait for all replicas). A single slow/unavailable replica can block writes.

### Eventual Consistency

The system returns quickly without waiting for all replicas. Replicas catch up asynchronously. Reads may return stale data briefly but all replicas will *eventually* converge.

```
Client  ──SET "x"=5──►  Node A (primary)
                              │ ACK immediately
                              ▼
                     Client gets success
                              │
                   (async in background)
                    ┌─────────┴──────────┐
                    ▼                    ▼
                 Node B               Node C
               (replica)            (replica)
              x = 5 (synced)      x = 4 (stale — not yet synced)

Client  ──GET "x"──►  Node C  →  returns 4  ✗ (stale for now)
                   (after sync)  →  returns 5  ✓
```

**Trade-off:** Low write latency, high availability. Applications must tolerate brief inconsistency.

### Quorum Reads and Writes (tunable consistency)

Systems like DynamoDB and Cassandra let you tune consistency per operation using quorums:

- **N** = total replicas
- **W** = replicas that must ACK a write
- **R** = replicas that must respond to a read

If `W + R > N`, you are guaranteed to read the latest write (strong consistency).

```
N=3 replicas, W=2, R=2  →  W+R=4 > 3  →  Strong consistency
N=3 replicas, W=1, R=1  →  W+R=2 < 3  →  Eventual consistency (fast)
```

```python
# Quorum simulation
N = 3    # total replicas
W = 2    # write quorum
R = 2    # read quorum

def is_strongly_consistent(W, R, N):
    return (W + R) > N

print(is_strongly_consistent(2, 2, 3))  # True  — strong consistency
print(is_strongly_consistent(1, 1, 3))  # False — eventual consistency
```

---

**Navigation:**
- [← Previous: Design Principles](02-design-principles.md)
- [Back to Index](README.md)
- [Next: Advantages & Limitations →](04-advantages-limitations.md)
