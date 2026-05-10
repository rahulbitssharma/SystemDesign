# Key-Value Databases — Scalability and Performance

## Why SQL Databases Struggle to Scale

SQL databases often struggle with scaling compared to key-value stores due to their relational nature, their reliance on JOINs, and their strict ACID guarantees. The three root causes are described below.

### 1. Relational Data Requires Complex Structures

SQL databases follow a **relational model** built on structured tables, primary/foreign keys, normalisation rules, and referential constraints. These structures enforce data integrity but make horizontal scaling hard because:

- Every query must respect relationships and maintain constraints across tables.
- As data grows across multiple machines, keeping those relationships consistent becomes increasingly expensive.
- Adding a new machine (shard) does not simply double throughput — you have to decide which tables live where, and queries that span tables may now span machines.

```
SQL E-Commerce Schema (simplified)
─────────────────────────────────
 Users            Orders              Products
┌──────────┐    ┌────────────────┐   ┌────────────┐
│ userID   │◄──│ userID (FK)    │   │ productID  │
│ name     │    │ orderID        │──►│ name       │
│ email    │    │ productID (FK) │   │ price      │
└──────────┘    │ quantity       │   └────────────┘
                └────────────────┘
```

Any query like "fetch all orders for a user with product details" must JOIN across all three tables. At scale, those three tables may live on different shards — turning one logical query into multiple network round-trips.

### 2. Joins and Transactions Slow Everything Down

Two features that make SQL databases powerful also make them expensive to scale:

- **JOINs** — SQL queries routinely join two, three, or more tables. In a single-machine setup this is fast because all data is local. In a distributed (sharded) setup, a JOIN might have to pull data from several nodes, adding network latency for every row that must be matched.
- **ACID Transactions** — SQL databases guarantee Atomicity, Consistency, Isolation, and Durability. In a distributed environment, achieving these guarantees requires **distributed locking** or **two-phase commit** (2PC), both of which require nodes to coordinate with each other. A single slow or unavailable node can stall every transaction that touches its data.

```
Distributed Transaction across 2 Shards (2-Phase Commit)
──────────────────────────────────────────────────────────

Coordinator
  │
  ├──── Phase 1 (PREPARE) ────►  Shard A  (holds lock on row A)
  │                         └──► Shard B  (holds lock on row B)
  │
  │◄──── PREPARE OK ──────────── Shard A
  │◄──── PREPARE OK ──────────── Shard B
  │
  ├──── Phase 2 (COMMIT)  ────►  Shard A  (releases lock, writes)
  │                         └──► Shard B  (releases lock, writes)
  │
  └──── Done (but both nodes were locked the whole time)
```

During those lock-hold periods, no other operation on the affected rows can proceed — this is a hard scalability ceiling.

### 3. Scaling Mechanisms: Vertical vs. Horizontal

| Approach               | SQL Databases                          | Key-Value Stores                         |
|------------------------|----------------------------------------|------------------------------------------|
| **Vertical scaling**   | Easy — buy a bigger machine            | Supported, but rarely needed             |
| **Horizontal scaling** | Hard — sharding breaks JOINs & txns    | Native — data is partitioned by key hash |
| **Ceiling**            | Hardware limit (typically one primary) | Near-linear — add nodes as needed        |

SQL databases were originally designed for monolithic, single-server deployments where vertical scaling (more CPU, RAM, faster disks) was the expected growth path. Sharding was bolted on later, and it shows: managing shard keys, cross-shard queries, and distributed transactions requires significant operational complexity.

---

## Detailed Example: Scaling a Relational Database for a Large E-commerce Website

### Initial State — Single Powerful Machine

```
Single PostgreSQL Server
┌──────────────────────────────────────────┐
│  CPU: 64 cores   RAM: 512 GB             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Users   │ │  Orders  │ │ Products │ │
│  │ 50M rows │ │ 500M rows│ │  2M rows │ │
│  └──────────┘ └──────────┘ └──────────┘ │
└──────────────────────────────────────────┘
```

The signature query for this system — "get all orders for a user with their product details" — looks like this:

```sql
SELECT
    u.name,
    o.order_id,
    o.quantity,
    p.product_name,
    p.price
FROM   Users    AS u
JOIN   Orders   AS o ON u.user_id   = o.user_id
JOIN   Products AS p ON o.product_id = p.product_id
WHERE  u.user_id = 456;
```

On a single machine this is fine, but as the site grows to tens of millions of users the machine eventually hits its limits.

### Attempting Horizontal Scaling — Sharding

The engineering team decides to split the `Users` and `Orders` tables across two shards based on `user_id`:

```
Shard 1 (user_id 1 – 25 000 000)       Shard 2 (user_id 25 000 001 – 50 000 000)
┌──────────────────────────────┐        ┌──────────────────────────────┐
│  Users (rows 1–25M)          │        │  Users (rows 25M–50M)        │
│  Orders (rows for those      │        │  Orders (rows for those      │
│         users)               │        │         users)               │
└──────────────────────────────┘        └──────────────────────────────┘
```

Problems immediately appear:

1. **Cross-shard JOINs** — `Products` cannot be split the same way as `Users`, so every order query that needs product details must reach across shards:

   ```
   App Server
     │
     ├── Query Shard 1 for user 456's orders ──────► Shard 1 DB   (~5 ms)
     │
     ├── For each orderID, query Products table ──► Products DB   (~5 ms × N orders)
     │
     └── Assemble result in application code
   ```

   What was one SQL query is now N+1 database calls assembled in application memory.

2. **Distributed transactions** — If a user places an order and payment must be recorded in both `Orders` (Shard 1) and a `Payments` table (potentially on Shard 2), a two-phase commit is required, dramatically increasing write latency:

   ```python
   # Pseudocode illustrating the coordination overhead
   def place_order(user_id: int, product_id: int, amount: float):
       shard = get_shard(user_id)         # route to correct shard

       # 2PC across Orders shard and Payments shard
       with DistributedTransaction([shard, payments_shard]) as txn:
           txn.execute(shard,           "INSERT INTO Orders ...")
           txn.execute(payments_shard,  "INSERT INTO Payments ...")
           txn.commit()                 # round-trips to BOTH shards
       # Any failure on either shard rolls back both — adds 10–50 ms
   ```

3. **Vertical scaling limit** — Even before sharding, the single primary server eventually runs out of CPU or RAM. Upgrading hardware provides diminishing returns and has a hard ceiling.

---

## How Key-Value Databases Excel at Scaling

Key-value stores sidestep the problems above through two core design choices.

### 1. Partitioning by Key (No JOINs Required)

Every item is stored and retrieved using a single key. The system hashes that key to determine which node owns it. Because there are no relationships to maintain across keys, nodes are completely independent — adding a new node is just a matter of redistributing hash ranges.

```
DynamoDB Partition Routing
──────────────────────────
  Key: "user#456"
  hash("user#456") = 0x7F3A
       │
       ▼
  Partition Router
  ┌────────────────────────────────────────────┐
  │  0x0000 – 0x3FFF  →  Partition 1 (Node A) │
  │  0x4000 – 0x7FFF  →  Partition 2 (Node B) │  ◄── "user#456" goes here
  │  0x8000 – 0xBFFF  →  Partition 3 (Node C) │
  │  0xC000 – 0xFFFF  →  Partition 4 (Node D) │
  └────────────────────────────────────────────┘
```

No coordination between nodes is required for a simple read or write — the router sends the request directly to the one node that owns the key.

### 2. Simple Access Patterns Enable Direct Lookups

Because access is always `GET key` or `SET key → value`, the database never needs to plan or optimise a query. There is no query planner, no statistics collection, no index selection — just a hash computation and a direct I/O operation. This keeps latency consistent regardless of dataset size.

```python
# Access pattern complexity comparison

# SQL: query planner must consider indexes, join order, statistics
sql_query = """
    SELECT u.name, o.order_id, p.product_name
    FROM users u
    JOIN orders o   ON u.user_id   = o.user_id
    JOIN products p ON o.product_id = p.product_id
    WHERE u.user_id = 456
"""
# → planner overhead + 2 JOINs + possible cross-shard network calls

# Key-Value: one direct lookup, O(1)
kv_key = "user#456#orders"
value  = kv_store.get(kv_key)
# → hash(key) → node → I/O → return   (no planning, no joins)
```

---

## Detailed Example: Using DynamoDB for the Same E-commerce Application

### Data Modelling — Denormalise into Key-Value Pairs

Instead of three normalised tables, the entire order record is stored as a single self-contained item under a composite key:

```json
{
    "PK":          "USER#456",
    "SK":          "ORDER#2024-01-15#123",
    "order_id":    "123",
    "user_name":   "Rahul",
    "order_date":  "2024-01-15",
    "items": [
        { "product_id": "789", "name": "Laptop", "price": 1200, "qty": 1 },
        { "product_id": "101", "name": "Mouse",  "price": 40,   "qty": 2 }
    ],
    "total":       1280
}
```

All information needed to display an order — user details, product names, prices — lives in one place. There is nothing to JOIN.

### Fetching All Orders for a User

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table    = dynamodb.Table("EcommerceOrders")

def get_orders_for_user(user_id: str) -> list:
    """
    Single DynamoDB Query call — no JOINs, no cross-node coordination.
    Routed directly to the partition that owns USER#<user_id>.
    """
    response = table.query(
        KeyConditionExpression=Key("PK").eq(f"USER#{user_id}") &
                               Key("SK").begins_with("ORDER#")
    )
    return response["Items"]

orders = get_orders_for_user("456")
for order in orders:
    print(f"Order {order['order_id']} — total ${order['total']}")
    for item in order["items"]:
        print(f"  • {item['name']} × {item['qty']} @ ${item['price']}")
```

```
# Output
Order 123 — total $1280
  • Laptop × 1 @ $1200
  • Mouse  × 2 @ $40
```

One API call, one network round-trip to one partition — regardless of whether the table holds 10 000 or 10 billion orders.

### Strong-Consistency Read (when required)

When the application needs a guaranteed up-to-date view (e.g., immediately after a payment is confirmed), DynamoDB supports strongly-consistent reads per call without sacrificing the horizontal scalability of the table:

```python
def get_order_with_strong_consistency(user_id: str, order_id: str, order_date: str) -> dict:
    """
    ConsistentRead=True forces DynamoDB to read from the primary replica,
    guaranteeing the latest data.  The table's partitioning is unchanged.
    """
    response = table.get_item(
        Key={
            "PK": f"USER#{user_id}",
            "SK": f"ORDER#{order_date}#{order_id}",
        },
        ConsistentRead=True,   # strong consistency, not eventual
    )
    return response.get("Item", {})
```

### Seamless Horizontal Scaling

DynamoDB partitions grow automatically. When throughput on a partition exceeds its limit, DynamoDB splits the partition and redistributes data — transparently, with zero downtime:

```
Before scaling (1 partition):
  ┌───────────────────────────────┐
  │ Partition 1                   │
  │  USER#1 … USER#1 000 000      │
  │  10 000 RCU / 10 000 WCU      │
  └───────────────────────────────┘

After automatic split (2 partitions):
  ┌───────────────────────────┐   ┌───────────────────────────┐
  │ Partition 1               │   │ Partition 2               │
  │  USER#1 … USER#500 000    │   │  USER#500 001 … #1 000 000│
  │  5 000 RCU / 5 000 WCU    │   │  5 000 RCU / 5 000 WCU    │
  └───────────────────────────┘   └───────────────────────────┘
  (application code is unchanged — routing is handled internally)
```

---

## Side-by-Side Comparison

| Dimension                | SQL Database (PostgreSQL/MySQL)            | Key-Value Store (DynamoDB/Redis)          |
|--------------------------|--------------------------------------------|-------------------------------------------|
| **Data model**           | Normalised tables with foreign keys        | Self-contained key-value items            |
| **Query for order data** | 3-table JOIN, query planner overhead       | Single `GET` or `Query` by key            |
| **Horizontal scaling**   | Complex — sharding breaks JOINs            | Native — hash-partitioned by key          |
| **Write latency**        | Higher under distributed transactions      | Single-digit ms regardless of cluster size|
| **Cross-node coordination** | Required for distributed transactions  | Not required for single-key operations    |
| **Schema changes**       | `ALTER TABLE` — can lock large tables      | No schema — add attributes per item       |
| **When to prefer it**    | Complex relational queries, ad-hoc reports | High-throughput, predictable access by key|

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

---

**Navigation:**
- [← Previous: Architecture Diagrams](08-architecture-diagrams.md)
- [Back to Index](README.md)
- [Next: Fault Tolerance →](10-fault-tolerance.md)
