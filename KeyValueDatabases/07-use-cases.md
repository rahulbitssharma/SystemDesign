# Key-Value Databases — Real-World Use Cases

## Session Management

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

## Rate Limiting

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

## Leaderboard / Ranked Data

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

## Pub/Sub Messaging

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

## Feature Flags / Configuration Store

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

**Navigation:**
- [← Previous: Caching Example](06-caching-example.md)
- [Back to Index](README.md)
- [Next: Architecture Diagrams →](08-architecture-diagrams.md)
