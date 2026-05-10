# Key-Value Databases — Building a Caching System

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

## Full Python Caching System Example

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

## Cache Eviction Policies

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

**Navigation:**
- [← Previous: Implementations](05-implementations.md)
- [Back to Index](README.md)
- [Next: Real-World Use Cases →](07-use-cases.md)
