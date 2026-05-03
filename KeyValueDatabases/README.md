# Key-Value Databases

## Table of Contents
1. [Introduction](#1-introduction)
2. [Design Principles](#2-design-principles)
3. [How They Work](#3-how-they-work)
4. [Advantages and Limitations](#4-advantages-and-limitations)
5. [Popular Implementations](#5-popular-implementations)
6. [Detailed Example Usage — Building a Caching System](#6-detailed-example-usage--building-a-caching-system)
7. [Real-World Use Cases](#7-real-world-use-cases)
8. [Architecture Diagrams](#8-architecture-diagrams)

---

## 1. Introduction

A **Key-Value Database** (also called a Key-Value Store) is the simplest form of NoSQL database. It stores data as a collection of **key-value pairs**, where:

- The **key** is a unique identifier (a string, integer, UUID, etc.).
- The **value** is any arbitrary data associated with that key — a string, number, JSON document, binary blob, or even a complex data structure.

Think of it like a dictionary (or hash map) at massive scale, optimised for extremely fast reads and writes.

```
┌─────────────────────────────────────────────────┐
│              Key-Value Store                    │
│                                                 │
│   "user:1001"    →   {"name": "Rahul",          │
│                        "age": 30}               │
│                                                 │
│   "session:abc"  →   "eyJhbGciOiJIUzI1NiJ9..." │
│                                                 │
│   "counter:hits" →   42819                      │
│                                                 │
│   "product:555"  →   {"title": "Laptop",        │
│                        "price": 999.99}         │
└─────────────────────────────────────────────────┘
```

### Simple Code Example

Below is a conceptual Python example showing how data is stored and retrieved using key-value pairs. This mimics what a real KV store does internally.

```python
# Conceptual representation of a Key-Value Store as a Python dict
store = {}

# --- WRITE ---
store["user:1001"] = {"name": "Rahul", "age": 30, "city": "Bangalore"}
store["session:abc123"] = "eyJhbGciOiJIUzI1NiJ9.payload.signature"
store["counter:page_hits"] = 42819

# --- READ ---
user = store.get("user:1001")
print(user)  # {'name': 'Rahul', 'age': 30, 'city': 'Bangalore'}

session = store.get("session:abc123")
print(session)  # eyJhbGciOiJIUzI1NiJ9.payload.signature

# --- DELETE ---
del store["session:abc123"]

# --- CHECK EXISTENCE ---
if "user:1001" in store:
    print("User found!")
```

Key observations:
- The lookup is **O(1)** — no scanning, no joins.
- The key is the *only* way to access a value (no queries by field).
- Values are **opaque** to the store — it does not parse or index their contents.

---

## 2. Design Principles

### 2.1 The Data Model

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

### 2.2 Storage Mechanisms

Under the hood, different KV stores use different storage engines:

#### a) Hash Table (In-Memory)

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

#### b) Log-Structured Merge Tree (LSM Tree)

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

#### c) B-Tree / B+ Tree (On-Disk)

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

### 2.3 Indexing for Fast Lookups

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

## 3. How They Work

### 3.1 Write Workflow

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

### 3.2 Read Workflow

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

### 3.3 Distributed Architecture

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

### 3.4 Consistency Models

#### Strong Consistency

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

#### Eventual Consistency

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

#### Quorum Reads and Writes (tunable consistency)

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

## 4. Advantages and Limitations

### 4.1 Advantages

| Advantage           | Why                                                                 |
|---------------------|---------------------------------------------------------------------|
| **Blazing Speed**   | O(1) hash-based lookups; in-memory stores like Redis hit sub-millisecond latency |
| **Horizontal Scale**| Consistent hashing makes adding/removing nodes seamless            |
| **Simple API**      | `GET`, `SET`, `DELETE` — easy to integrate into any application    |
| **Flexible Values** | Values can be strings, JSON, binary, counters, lists, sets, etc.   |
| **TTL Support**     | Native expiry makes caching and session management trivial          |
| **High Throughput** | Millions of reads/writes per second on modern hardware             |

### 4.2 Limitations and Trade-offs

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

## 5. Popular Implementations

### 5.1 Redis

Redis (Remote Dictionary Server) is an **in-memory** KV store supporting rich data structures (strings, hashes, lists, sets, sorted sets, streams, and more). It is the most widely-used KV store in the world.

**Key features:**
- Sub-millisecond latency (everything in RAM)
- Persistence via RDB snapshots and AOF (Append-Only File)
- Pub/Sub messaging
- Lua scripting
- Cluster mode for horizontal scaling
- Built-in TTL per key

```python
# Redis usage with the redis-py client
# Install: pip install redis

import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# --- Basic String ---
r.set("user:1001:name", "Rahul")
r.set("user:1001:age", 30)
print(r.get("user:1001:name"))   # "Rahul"
print(r.get("user:1001:age"))    # "30"

# --- Key Expiry (TTL) ---
r.set("session:abc123", "token_xyz", ex=3600)  # expires in 1 hour
print(r.ttl("session:abc123"))   # seconds remaining

# --- Hash (store a whole object under one key) ---
r.hset("user:1001", mapping={"name": "Rahul", "age": "30", "city": "Bangalore"})
print(r.hgetall("user:1001"))    # {'name': 'Rahul', 'age': '30', 'city': 'Bangalore'}
print(r.hget("user:1001", "city"))  # "Bangalore"

# --- Counter (atomic increment) ---
r.set("counter:page_views", 0)
r.incr("counter:page_views")
r.incr("counter:page_views")
r.incrby("counter:page_views", 10)
print(r.get("counter:page_views"))  # "12"

# --- List (queue / recent items) ---
r.rpush("queue:emails", "email_job_1", "email_job_2", "email_job_3")
job = r.lpop("queue:emails")        # dequeue from left
print(job)                          # "email_job_1"

# --- Sorted Set (leaderboard) ---
r.zadd("leaderboard", {"Alice": 9500, "Bob": 8800, "Charlie": 9200})
top3 = r.zrevrange("leaderboard", 0, 2, withscores=True)
print(top3)  # [('Alice', 9500.0), ('Charlie', 9200.0), ('Bob', 8800.0)]

# --- Delete ---
r.delete("user:1001:name", "session:abc123")
```

### 5.2 Amazon DynamoDB

DynamoDB is a **fully managed**, serverless KV + document store on AWS. It uses a primary key (partition key or partition + sort key) to locate items. Under the hood it runs on a distributed hash ring with SSDs.

**Key features:**
- Fully managed — no servers to run
- Single-digit millisecond latency at any scale
- Auto-scaling
- Global Tables for multi-region replication
- DAX (DynamoDB Accelerator) for microsecond caching
- On-demand and provisioned capacity modes

```python
# DynamoDB usage with boto3
# Install: pip install boto3

import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("Users")

# --- PUT (write an item) ---
table.put_item(
    Item={
        "user_id": "1001",           # Partition key
        "name": "Rahul",
        "age": 30,
        "city": "Bangalore",
    }
)

# --- GET (read by primary key — O(1)) ---
response = table.get_item(Key={"user_id": "1001"})
user = response.get("Item")
print(user)  # {'user_id': '1001', 'name': 'Rahul', 'age': Decimal('30'), 'city': 'Bangalore'}

# --- UPDATE (modify specific attributes) ---
table.update_item(
    Key={"user_id": "1001"},
    UpdateExpression="SET age = :new_age",
    ExpressionAttributeValues={":new_age": 31},
)

# --- DELETE ---
table.delete_item(Key={"user_id": "1001"})

# --- QUERY (partition key + sort key range) ---
# Assuming table has partition key: user_id, sort key: timestamp
orders_table = dynamodb.Table("Orders")
response = orders_table.query(
    KeyConditionExpression=Key("user_id").eq("1001") & Key("timestamp").begins_with("2024-")
)
orders = response["Items"]
```

### 5.3 Other Notable Implementations

| Store         | Type            | Notable For                                              |
|---------------|-----------------|----------------------------------------------------------|
| **Memcached** | In-memory       | Pure cache, extremely simple, no persistence            |
| **Etcd**      | Distributed     | Kubernetes config store, strong consistency via Raft    |
| **RocksDB**   | Embedded LSM    | Ultra-fast embedded KV (used inside Cassandra, TiKV)    |
| **Aerospike** | Hybrid (RAM+SSD)| Very high throughput with SSD-backed persistence        |
| **Riak KV**   | Distributed     | Eventual consistency, inspired by Amazon Dynamo paper   |
| **LevelDB**   | Embedded LSM    | Google's open-source KV library, underpins many systems |

---

## 6. Detailed Example Usage — Building a Caching System

A cache sits in front of a slower data source (e.g., a relational database) and serves frequently-accessed data at high speed. KV stores are the canonical cache tier.

```
  Client Request
        │
        ▼
  ┌─────────────┐       CACHE HIT        ┌───────────────┐
  │  App Server │ ──── GET "user:1001" ──►│  Redis Cache  │
  │             │◄──── return data  ──── │  (fast, ~1ms) │
  └─────────────┘                        └───────────────┘
        │                                       │
        │  CACHE MISS                           │ (not found)
        ▼                                       │
  ┌───────────────┐                            │
  │  PostgreSQL   │◄───────────────────────────┘
  │  (slow, ~5ms) │  fetch from DB, then cache result
  └───────────────┘
```

### Full Python Caching System Example

```python
import redis
import json
import time
import random
from typing import Optional

# ─── Simulated database ───────────────────────────────────────────────────────

DATABASE = {
    "1001": {"id": "1001", "name": "Rahul",   "email": "rahul@example.com",   "plan": "premium"},
    "1002": {"id": "1002", "name": "Priya",   "email": "priya@example.com",   "plan": "basic"},
    "1003": {"id": "1003", "name": "Arjun",   "email": "arjun@example.com",   "plan": "premium"},
}

def slow_db_query(user_id: str) -> Optional[dict]:
    """Simulates a slow database query (50–150 ms)."""
    time.sleep(random.uniform(0.05, 0.15))
    return DATABASE.get(user_id)

# ─── Cache layer ──────────────────────────────────────────────────────────────

class UserCache:
    def __init__(self, host="localhost", port=6379, ttl_seconds=300):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)
        self.ttl = ttl_seconds
        self.hits = 0
        self.misses = 0

    def _cache_key(self, user_id: str) -> str:
        return f"user:{user_id}"

    def get_user(self, user_id: str) -> Optional[dict]:
        key = self._cache_key(user_id)

        # 1. Try cache first
        cached = self.r.get(key)
        if cached is not None:
            self.hits += 1
            print(f"[CACHE HIT]  user:{user_id}")
            return json.loads(cached)

        # 2. Cache miss — query the database
        self.misses += 1
        print(f"[CACHE MISS] user:{user_id} — querying DB...")
        user = slow_db_query(user_id)

        # 3. Store result in cache for future requests
        if user is not None:
            self.r.set(key, json.dumps(user), ex=self.ttl)

        return user

    def invalidate(self, user_id: str):
        """Remove a user from cache (e.g., after update)."""
        key = self._cache_key(user_id)
        self.r.delete(key)
        print(f"[INVALIDATE] user:{user_id} removed from cache")

    def update_user(self, user_id: str, updates: dict):
        """Update database and invalidate cache."""
        if user_id in DATABASE:
            DATABASE[user_id].update(updates)
        self.invalidate(user_id)

    def stats(self):
        total = self.hits + self.misses
        hit_rate = (self.hits / total * 100) if total else 0
        print(f"\nCache Stats — Hits: {self.hits}, Misses: {self.misses}, Hit Rate: {hit_rate:.1f}%")

# ─── Demo ─────────────────────────────────────────────────────────────────────

def run_demo():
    cache = UserCache(ttl_seconds=60)

    print("=== First access — all cache misses ===")
    for uid in ["1001", "1002", "1003"]:
        user = cache.get_user(uid)
        print(f"  Got: {user}\n")

    print("=== Second access — all cache hits ===")
    for uid in ["1001", "1002", "1003"]:
        user = cache.get_user(uid)
        print(f"  Got: {user}\n")

    print("=== Update user 1001 ===")
    cache.update_user("1001", {"plan": "enterprise"})

    print("\n=== Access after update — miss then hit ===")
    user = cache.get_user("1001")  # miss (invalidated)
    print(f"  Got: {user}\n")
    user = cache.get_user("1001")  # hit
    print(f"  Got: {user}\n")

    cache.stats()

# run_demo()  # Uncomment when running with a live Redis instance
```

**Expected output:**
```
=== First access — all cache misses ===
[CACHE MISS] user:1001 — querying DB...
  Got: {'id': '1001', 'name': 'Rahul', 'email': 'rahul@example.com', 'plan': 'premium'}

[CACHE MISS] user:1002 — querying DB...
  Got: {'id': '1002', 'name': 'Priya', 'email': 'priya@example.com', 'plan': 'basic'}
...

=== Second access — all cache hits ===
[CACHE HIT]  user:1001
[CACHE HIT]  user:1002
...

Cache Stats — Hits: 4, Misses: 3, Hit Rate: 57.1%
```

### Cache Eviction Policies

When the cache is full, the store must decide which keys to evict. Redis supports several policies, configurable via `maxmemory-policy`:

| Policy         | Description                                           | Best For                    |
|----------------|-------------------------------------------------------|-----------------------------|
| `noeviction`   | Reject new writes when full                           | Persistent stores           |
| `allkeys-lru`  | Evict least recently used keys (any key)              | General caching             |
| `volatile-lru` | Evict LRU keys that have a TTL set                    | Mixed cache + persistent KV |
| `allkeys-lfu`  | Evict least frequently used keys                      | Hot/cold data patterns      |
| `allkeys-random` | Evict a random key                                  | Uniform access patterns     |
| `volatile-ttl` | Evict key with shortest remaining TTL                 | Predictable expiry workloads|

```python
# Simplified LRU cache in Python (illustrates the concept)
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()  # maintains insertion/access order

    def get(self, key: str):
        if key not in self.cache:
            return None
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key: str, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            # Remove the least recently used (first item)
            evicted_key, _ = self.cache.popitem(last=False)
            print(f"[EVICT] Evicted key: {evicted_key}")

lru = LRUCache(capacity=3)
lru.set("a", 1)
lru.set("b", 2)
lru.set("c", 3)
lru.get("a")         # access "a" → moves it to end
lru.set("d", 4)      # cache full → evicts "b" (LRU)
# [EVICT] Evicted key: b
```

---

## 7. Real-World Use Cases

### 7.1 Session Management

Web applications store user session data in a KV store for fast retrieval on every request. Session tokens are the key; session data (user id, permissions, cart) is the value.

```
HTTP Request  →  Extract session token from cookie
                          │
                          ▼
              GET "session:{token}"  →  Redis
                          │
                  ┌───────┴────────┐
               HIT │            MISS │
                  ▼                 ▼
         Return user data    Redirect to login
```

```python
import redis
import json
import uuid
import time
from typing import Optional

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
SESSION_TTL = 1800  # 30 minutes

def create_session(user_id: str, permissions: list) -> str:
    """Create a new session and return the session token."""
    token = str(uuid.uuid4())
    session_data = {
        "user_id": user_id,
        "permissions": permissions,
        "created_at": time.time(),
    }
    r.set(f"session:{token}", json.dumps(session_data), ex=SESSION_TTL)
    return token

def get_session(token: str) -> Optional[dict]:
    """Retrieve session data for a given token."""
    raw = r.get(f"session:{token}")
    if raw is None:
        return None  # session expired or invalid
    # Refresh TTL on activity (sliding expiry)
    r.expire(f"session:{token}", SESSION_TTL)
    return json.loads(raw)

def destroy_session(token: str):
    """Logout: invalidate the session."""
    r.delete(f"session:{token}")

# --- Usage ---
token = create_session("user:1001", ["read", "write"])
print(f"Session token: {token}")

session = get_session(token)
print(f"Session data: {session}")

destroy_session(token)
print(f"After logout: {get_session(token)}")  # None
```

### 7.2 Rate Limiting

KV stores implement rate limiting using atomic counters and TTL. The "fixed window" pattern counts requests per user per time window.

```
Request from user:1001
        │
        ▼
INCR "rate:{user_id}:{window}"
        │
   count ≤ limit?        count > limit?
        │                      │
        ▼                      ▼
   Allow request         Return 429 Too Many Requests
```

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def is_rate_limited(user_id: str, limit: int = 100, window_seconds: int = 60) -> bool:
    """
    Fixed-window rate limiter.
    Allows at most `limit` requests per `window_seconds` seconds per user.
    """
    import math
    window = math.floor(time.time() / window_seconds)
    key = f"rate:{user_id}:{window}"

    # Atomic increment and get
    count = r.incr(key)

    # Set expiry only on first request in this window
    if count == 1:
        r.expire(key, window_seconds)

    return count > limit

# --- Usage ---
import time

user = "user:1001"
for i in range(5):
    limited = is_rate_limited(user, limit=3, window_seconds=10)
    print(f"Request {i+1}: {'BLOCKED' if limited else 'ALLOWED'}")
# Request 1: ALLOWED
# Request 2: ALLOWED
# Request 3: ALLOWED
# Request 4: BLOCKED
# Request 5: BLOCKED
```

### 7.3 Leaderboard / Ranked Data

Redis Sorted Sets allow storing scores with automatic ranking — perfect for game leaderboards, trending content, or priority queues.

```python
import redis
from typing import Optional

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
LEADERBOARD_KEY = "game:leaderboard"

def submit_score(player: str, score: float):
    """Add or update a player's score."""
    r.zadd(LEADERBOARD_KEY, {player: score})

def get_top_n(n: int) -> list:
    """Get the top N players with their scores."""
    return r.zrevrange(LEADERBOARD_KEY, 0, n - 1, withscores=True)

def get_player_rank(player: str) -> Optional[int]:
    """Get a player's rank (1-indexed)."""
    rank = r.zrevrank(LEADERBOARD_KEY, player)
    return rank + 1 if rank is not None else None

# --- Usage ---
submit_score("Alice", 9500)
submit_score("Bob", 8800)
submit_score("Charlie", 9200)
submit_score("Diana", 9800)
submit_score("Eve", 9100)

print("Top 3 Players:")
for player, score in get_top_n(3):
    print(f"  {get_player_rank(player)}. {player}: {int(score)}")
# Top 3 Players:
#   1. Diana:   9800
#   2. Alice:   9500
#   3. Charlie: 9200

submit_score("Bob", 9600)  # Bob improves their score
print(f"\nBob's new rank: {get_player_rank('Bob')}")  # 3
```

### 7.4 Pub/Sub Messaging

Redis supports **publish-subscribe** messaging, enabling decoupled real-time communication between services.

```python
# Publisher (e.g., order service)
import redis
import json

r_pub = redis.Redis(host="localhost", port=6379, decode_responses=True)

def publish_order_event(order_id: str, status: str):
    event = {"order_id": order_id, "status": status}
    r_pub.publish("order:events", json.dumps(event))
    print(f"Published: {event}")

publish_order_event("ORD-001", "shipped")
publish_order_event("ORD-002", "delivered")
```

```python
# Subscriber (e.g., notification service) — runs in a separate thread/process
import redis
import json

r_sub = redis.Redis(host="localhost", port=6379, decode_responses=True)
pubsub = r_sub.pubsub()
pubsub.subscribe("order:events")

print("Listening for order events...")
for message in pubsub.listen():
    if message["type"] == "message":
        event = json.loads(message["data"])
        print(f"Received event: {event}")
        # → Send notification to user
```

### 7.5 Feature Flags / Configuration Store

KV stores are ideal for storing application configuration and feature flags — the entire config is a few key lookups.

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def set_feature_flag(flag: str, enabled: bool, rollout_percent: int = 100):
    data = {"enabled": enabled, "rollout_percent": rollout_percent}
    r.set(f"feature:{flag}", json.dumps(data))

def is_feature_enabled(flag: str, user_id: str) -> bool:
    raw = r.get(f"feature:{flag}")
    if not raw:
        return False  # flag not found → disabled by default
    config = json.loads(raw)
    if not config["enabled"]:
        return False
    # Simple hash-based rollout
    user_bucket = hash(user_id) % 100
    return user_bucket < config["rollout_percent"]

# --- Usage ---
set_feature_flag("dark_mode", enabled=True, rollout_percent=50)
set_feature_flag("new_checkout", enabled=False)

print(is_feature_enabled("dark_mode", "user:1001"))   # True or False (50% rollout)
print(is_feature_enabled("new_checkout", "user:1001")) # False
```

---

## 8. Architecture Diagrams

### 8.1 Single-Node Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Key-Value Store Node                        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Client Interface                     │   │
│  │          GET / SET / DELETE / EXPIRE / INCR             │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐   │
│  │                  Command Processor                      │   │
│  │   (parse command, validate, route to data structure)    │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│        ┌────────────────────┼────────────────────┐             │
│        ▼                    ▼                    ▼             │
│  ┌──────────┐        ┌──────────┐        ┌──────────────┐     │
│  │  String  │        │  Hash    │        │  Sorted Set  │     │
│  │  Store   │        │  Store   │        │  (ZSet)      │     │
│  └──────────┘        └──────────┘        └──────────────┘     │
│        │                    │                    │             │
│  ┌─────▼────────────────────▼────────────────────▼──────┐     │
│  │                  In-Memory Hash Table                 │     │
│  └───────────────────────────────────────────────────────┘     │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐   │
│  │                Persistence Layer                        │   │
│  │   RDB Snapshot (point-in-time)  │  AOF (append log)     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Distributed Cluster Architecture

```
                         ┌─────────────┐
                         │   Client    │
                         └──────┬──────┘
                                │  SET "user:1001" → data
                                ▼
                    ┌───────────────────────┐
                    │    Cluster Router /   │
                    │   Hash Slot Mapper    │
                    │  (consistent hashing) │
                    └───────────┬───────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │   Shard 1    │     │   Shard 2    │     │   Shard 3    │
  │  (Primary)   │     │  (Primary)   │     │  (Primary)   │
  │ slots 0–5461 │     │slots 5462–   │     │slots 10923–  │
  │              │     │   10922      │     │  16383       │
  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
         │                    │                    │
  ┌──────▼───────┐     ┌──────▼───────┐     ┌──────▼───────┐
  │ Replica 1A   │     │ Replica 2A   │     │ Replica 3A   │
  │  (standby)   │     │  (standby)   │     │  (standby)   │
  └──────────────┘     └──────────────┘     └──────────────┘
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Replica 1B   │     │ Replica 2B   │     │ Replica 3B   │
  │  (standby)   │     │  (standby)   │     │  (standby)   │
  └──────────────┘     └──────────────┘     └──────────────┘

  "user:1001" → hash_slot = CRC16("user:1001") % 16384 = 4392
              → routed to Shard 1 (slots 0–5461)
```

### 8.3 Write Workflow (Distributed)

```
Client
  │
  │  SET "user:1001" = {data}
  ▼
Cluster Router
  │  hash_slot("user:1001") = 4392  →  Shard 1
  ▼
Shard 1 Primary
  │
  ├── 1. Append to WAL (Write-Ahead Log)  ─────────────────► WAL on disk
  │
  ├── 2. Update MemTable (in-memory)
  │
  ├── 3. Respond ACK to client  ─────────────────────────►  Client gets OK
  │
  └── 4. Async replicate to replicas ──►  Replica 1A
                                     └►  Replica 1B

                    (MemTable full?)
                          │ YES
                          ▼
               Flush to SSTable (L0)
                          │
               (compaction threshold?)
                          │ YES
                          ▼
               Merge SSTables L0 → L1 → ... (background)
```

### 8.4 Read Workflow (Distributed, Cache Miss Path)

```
Client
  │
  │  GET "user:1001"
  ▼
Cluster Router
  │  hash_slot("user:1001") = 4392  →  Shard 1
  ▼
Shard 1 (Primary or Replica depending on read preference)
  │
  ├── 1. Check MemTable  ──────── FOUND? ──────────────────► Return to Client
  │         (not found)
  │
  ├── 2. Check Bloom Filter (L0)  ─── "definitely not here"? → skip SSTable
  │         (might be here)
  │
  ├── 3. Read SSTable L0  ─────── FOUND? ──────────────────► Return to Client
  │         (not found)
  │
  ├── 4. Check Bloom Filter (L1)
  │
  ├── 5. Read SSTable L1  ─────── FOUND? ──────────────────► Return to Client
  │         (not found)
  │
  └── 6. Key does not exist  ─────────────────────────────► Return NULL
```

---

## Summary

Key-Value Databases are the **simplest and fastest** data stores available. Their power lies in:

1. **O(1) lookups** via hash-based indexing.
2. **Horizontal scalability** via consistent hashing and sharding.
3. **Flexible values** that can model any data structure.
4. **TTL and eviction** that make them the natural fit for caches.
5. **Tunable consistency** (quorums) to balance speed vs. accuracy.

They are not a replacement for relational databases when you need complex queries or joins — but for their intended use cases (caching, sessions, counters, leaderboards, pub/sub), nothing comes close in raw performance.

| Use Case                  | Recommended Store   | Key Pattern             |
|---------------------------|---------------------|-------------------------|
| Application cache         | Redis / Memcached   | `cache:{resource}:{id}` |
| Session storage           | Redis               | `session:{token}`       |
| Rate limiting             | Redis               | `rate:{user}:{window}`  |
| Leaderboards              | Redis Sorted Sets   | `leaderboard:{game}`    |
| Feature flags             | Redis / etcd        | `feature:{flag_name}`   |
| Distributed config        | etcd / DynamoDB     | `config:{service}:{key}`|
| High-throughput writes    | DynamoDB / RocksDB  | `{type}:{id}`           |
