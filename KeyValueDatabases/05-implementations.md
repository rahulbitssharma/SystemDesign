# Key-Value Databases — Popular Implementations

## Redis

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

## Amazon DynamoDB

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

## Other Notable Implementations

| Store         | Type            | Notable For                                              |
|---------------|-----------------|----------------------------------------------------------|
| **Memcached** | In-memory       | Pure cache, extremely simple, no persistence            |
| **Etcd**      | Distributed     | Kubernetes config store, strong consistency via Raft    |
| **RocksDB**   | Embedded LSM    | Ultra-fast embedded KV (used inside Cassandra, TiKV)    |
| **Aerospike** | Hybrid (RAM+SSD)| Very high throughput with SSD-backed persistence        |
| **Riak KV**   | Distributed     | Eventual consistency, inspired by Amazon Dynamo paper   |
| **LevelDB**   | Embedded LSM    | Google's open-source KV library, underpins many systems |

---

**Navigation:**
- [← Previous: Advantages & Limitations](04-advantages-limitations.md)
- [Back to Index](README.md)
- [Next: Caching Example →](06-caching-example.md)
