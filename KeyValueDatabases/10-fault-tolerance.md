# Key-Value Databases — Fault Tolerance

## 1. What Does Fault Tolerance Mean?

**Fault tolerance** is the ability of a system to continue operating correctly (or gracefully degrade) even when one or more components fail. It means the system can:

- **Survive node failures** — if a server dies, other servers take over
- **Recover data** — lost data can be reconstructed from replicas
- **Handle network partitions** — when communication between nodes breaks temporarily
- **Maintain availability** — users can still read/write despite failures
- **Eventually recover** — the system self-heals over time

```
Without Fault Tolerance:
  ┌──────────────┐
  │ Single Node  │  ← Node crashes
  │ (all data)   │    ENTIRE DATABASE LOST ✗
  └──────────────┘

With Fault Tolerance:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Node A       │  │ Node B       │  │ Node C       │
  │ (replica 1)  │  │ (replica 1)  │  │ (replica 1)  │
  └──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
       ├─────── Data replicated across all nodes ────┤
       └─────────────────────────────────────────────┘

  If Node A crashes → Nodes B & C still have the data ✓
```

---

## 2. How Key-Value Stores Deal with Fault Tolerance

### Core Mechanisms

#### A. Replication (Primary + Replicas)

Every piece of data is copied to multiple nodes. When the primary fails, a replica takes over.

```
Write "user:1001":
  ┌──────────────────────────────────┐
  │ Client writes to primary (Node A)│
  └──────────────┬───────────────────┘
                 │
         ┌───────▼────────┐
         │ Primary ACKs   │  → returns success to client
         └───────┬────────┘
                 │
        (async replication)
         ┌───────▼────────┐
         │ Node B (replica) gets copy
         └────────────────┘
         ┌───────┬────────┐
         │ Node C (replica) gets copy
         └────────────────┘

Node A crashes:
  → Nodes B or C detect failure
  → One is elected as new primary
  → Clients automatically redirect
  → New writes go to new primary ✓
```

#### B. Write-Ahead Logging (WAL)

Before modifying in-memory data, KV stores write to a persistent log on disk. If the node crashes, it replays the log on restart.

```
Write Operation:
  1. Write "SET key=value" to WAL (disk) ← durable, can survive crash
  2. Update in-memory structure
  3. ACK to client
  4. Async: send to replicas

Node crashes after step 1 but before step 3:
  → On restart, replay WAL
  → "SET key=value" is reapplied
  → Data is NOT lost ✓
```

#### C. Quorum-Based Consistency

A write is only considered successful if a **quorum** (majority) of replicas acknowledge it. This ensures even if some nodes fail, the data is safe.

```
N = 3 replicas, W = 2 (write quorum)

Write "x = 5":
  ┌─────────────────────────┐
  │ Client → Node A (primary)
  │         Node B (replica)
  │         Node C (replica)
  └─────────────────────────┘

  Node A: ACKs  ✓
  Node B: ACKs  ✓  ← 2/3 = quorum met, write succeeds
  Node C: timeout (network issue)

Result: "x = 5" is durable on at least 2 nodes
        Even if Node A crashes, Node B still has it ✓
```

---

### Popular Implementations

#### DynamoDB (AWS)

**Fault Tolerance Strategy:**
- **Automatic replication** across 3 availability zones (separate data centers)
- **Quorum-based consistency** — W=2, R=1 for eventual consistency
- **DynamoDB Streams** — replicate changes to other regions
- **Point-in-time recovery** — restore to any point in the last 35 days
- **Multi-region replication** — active-active global tables

```python
# DynamoDB fault tolerance example
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("Users")

# 1. Write with durability guarantee
# By default, DynamoDB uses W=1 (only primary must ACK)
# For stronger durability, apps can retry on failure:

def write_user_with_retry(user_id: str, user_data: dict, max_retries=3):
    """Write with automatic retry on transient failures."""
    for attempt in range(max_retries):
        try:
            table.put_item(Item={"user_id": user_id, **user_data})
            print(f"✓ Write succeeded on attempt {attempt + 1}")
            return True
        except ClientError as e:
            if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
                # Transient failure — DynamoDB is handling it
                if attempt < max_retries - 1:
                    wait_time = 2 ** attempt  # exponential backoff
                    print(f"⏳ Throttled, retrying in {wait_time}s...")
                    time.sleep(wait_time)
                    continue
            raise

write_user_with_retry("user:1001", {"name": "Rahul", "email": "rahul@example.com"})

# 2. Read with consistency options
def read_user_eventually_consistent(user_id: str):
    """Fast read from any replica (may be stale)."""
    response = table.get_item(
        Key={"user_id": user_id},
        ConsistentRead=False  # read from any replica, faster
    )
    return response.get("Item")

def read_user_strongly_consistent(user_id: str):
    """Slower read guaranteed to be latest."""
    response = table.get_item(
        Key={"user_id": user_id},
        ConsistentRead=True   # read from primary only, slower but up-to-date
    )
    return response.get("Item")

# 3. Global Tables (multi-region replication)
# DynamoDB replicates writes across regions automatically
# If us-east-1 region goes down, app can failover to us-west-2

# 4. Point-in-time recovery
# Restore a table to any point in the last 35 days
# Useful if a bug deletes data or corrupts a table
```

#### Cassandra (Open Source)

**Fault Tolerance Strategy:**
- **Distributed replication** — each key stored on multiple nodes (replication factor RF=3)
- **Eventual consistency** — all replicas eventually converge
- **Hinted handoff** — if a replica is down, the primary stores the write and replays it later
- **Read repair** — when reading, if replicas disagree, the latest version is repaired across all
- **Anti-entropy repair** — periodic full sync between all replicas

```python
# Cassandra fault tolerance example
from cassandra.cluster import Cluster
from cassandra.policies import RetryPolicy, DowngradingConsistencyRetryPolicy
import time

cluster = Cluster(['127.0.0.1', '127.0.0.2', '127.0.0.3'])
session = cluster.connect('my_keyspace')

# 1. Write with replication factor 3
# Every write is replicated to 3 nodes
create_table_query = """
    CREATE TABLE IF NOT EXISTS users (
        user_id UUID PRIMARY KEY,
        name TEXT,
        email TEXT
    ) WITH replication = {
        'class': 'SimpleStrategy',
        'replication_factor': 3  ← replicate to 3 nodes
    };
"""

# 2. Write at different consistency levels
from cassandra import ConsistencyLevel

def write_user_quorum(user_id: str, name: str, email: str):
    """Write with quorum consistency: 2 out of 3 nodes must ACK."""
    query = session.prepare("""
        INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)
    """)
    # Set consistency to QUORUM (majority of replicas)
    bound_stmt = query.bind([user_id, name, email])
    bound_stmt.consistency_level = ConsistencyLevel.QUORUM  # 2/3
    session.execute(bound_stmt)
    print(f"✓ Write to {user_id} succeeded (2/3 nodes acknowledged)")

def write_user_all(user_id: str, name: str, email: str):
    """Write with strongest consistency: ALL 3 nodes must ACK."""
    query = session.prepare("""
        INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)
    """)
    bound_stmt = query.bind([user_id, name, email])
    bound_stmt.consistency_level = ConsistencyLevel.ALL  # 3/3 (slower, stronger)
    try:
        session.execute(bound_stmt)
        print(f"✓ Write to {user_id} succeeded (all 3 nodes acknowledged)")
    except Exception as e:
        # If even 1 node is down, ALL consistency fails
        print(f"✗ Write failed: {e}")

# 3. Read with consistency levels
def read_user_one(user_id: str):
    """Fast read from ONE replica (potentially stale)."""
    query = session.prepare("SELECT * FROM users WHERE user_id = ?")
    bound_stmt = query.bind([user_id])
    bound_stmt.consistency_level = ConsistencyLevel.ONE  # fastest, may be stale
    result = session.execute(bound_stmt)
    return result.one()

def read_user_quorum(user_id: str):
    """Read from quorum (2/3 nodes), likely up-to-date."""
    query = session.prepare("SELECT * FROM users WHERE user_id = ?")
    bound_stmt = query.bind([user_id])
    bound_stmt.consistency_level = ConsistencyLevel.QUORUM
    # Cassandra automatically reads from 2 nodes and returns latest version
    result = session.execute(bound_stmt)
    return result.one()

# 4. Hinted Handoff (automatic recovery)
# If Node A is down during a write:
#   → Node B receives the write
#   → Node B stores a "hint" for Node A
#   → When Node A comes back online, Node B replays the hint
#   → Data is automatically restored ✓

# 5. Handling node failures
def write_with_downgrade_retry(user_id: str, name: str, email: str):
    """
    If nodes fail, automatically downgrade consistency level.
    QUORUM write fails → retry at ONE
    """
    cluster.default_retry_policy = DowngradingConsistencyRetryPolicy()
    
    query = session.prepare("""
        INSERT INTO users (user_id, name, email) VALUES (?, ?, ?)
    """)
    bound_stmt = query.bind([user_id, name, email])
    bound_stmt.consistency_level = ConsistencyLevel.QUORUM
    
    # If 1 node is down, automatically downgrades to LOCAL_ONE
    # Ensures availability even with failures
    session.execute(bound_stmt)
    print(f"✓ Write succeeded (may have downgraded consistency due to failures)")

# 6. Periodic repair (anti-entropy)
# Run periodically to detect and fix data inconsistencies
# Example: `nodetool repair -pr` in production
```

---

## 3. Why KV Stores Have Fault Tolerance Advantages Over SQL Databases

### Simple Replication Model

**KV Stores:**
- Data is partitioned by key hash → each node is independent
- Adding replicas is trivial: "store the same key on nodes 1, 2, 3"
- No coordination needed between nodes

```
KV Store Replication (simple):
  Key "user:1001" → hash to node 3
                  → also replicate to nodes 5, 7
  Nodes are completely independent ✓
```

**SQL Databases:**
- Data is **normalized** across multiple tables with foreign keys
- Replicating a write requires propagating changes across entire schema
- A single row write may trigger cascading updates in related tables

```
SQL Replication (complex):
  Write to Users table row 1001
    → CASCADE UPDATE Orders table (all orders for user 1001)
    → CASCADE UPDATE Payments table (all payments for those orders)
    → Must maintain referential integrity across all replicas
    → Slow, complex, prone to inconsistencies ✗
```

### Built-in Quorum Logic

**KV Stores:**
- Quorum writes are built into the core (DynamoDB, Cassandra, Riak)
- Clients automatically retry on failure
- No application logic needed

**SQL Databases:**
- Quorum concepts don't exist in traditional SQL
- Replication is typically primary-secondary (one direction)
- Failover is manual or requires external tools (Patroni, MHA)

```
KV Store failover (automatic):
  Primary fails → Quorum recognizes it → elect new primary → automatic

SQL Database failover (manual):
  Primary fails → DBA alerted → manual DNS change → application reconnects
```

### Tunable Consistency

**KV Stores:**
- Per-operation consistency tuning (W=1 for speed, W=3 for safety)
- Applications choose the trade-off they need

**SQL Databases:**
- All-or-nothing: either ACID or you're doing replication hacks
- No per-operation tuning

---

## 4. How Relational Databases Handle Fault Tolerance

### Traditional Approach: Primary-Replica Setup

Enterprises typically use **master-slave** or **primary-replica** replication.

```python
# PostgreSQL with primary-replica setup (typical enterprise pattern)

# 1. WRITE to primary (synchronous or asynchronous)
import psycopg2

# Primary database
primary_conn = psycopg2.connect("host=primary.db.company.com user=app dbname=production")

def write_user_primary(user_id: int, name: str, email: str):
    """Write to primary. Replicas get updates asynchronously."""
    cursor = primary_conn.cursor()
    cursor.execute(
        "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
        (user_id, name, email)
    )
    primary_conn.commit()
    # PostgreSQL asynchronously sends this write to replicas
    # Replicas lag behind by ~seconds (eventual consistency)

# 2. READ from replica (for scale)
replica_conn = psycopg2.connect("host=replica.db.company.com user=app dbname=production")

def read_user_replica(user_id: int):
    """Read from replica (may be stale)."""
    cursor = replica_conn.cursor()
    cursor.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
    return cursor.fetchone()

# 3. Failover: manual or semi-automatic
def promote_replica_to_primary():
    """
    When primary fails, manually run:
    1. Stop replica from accepting replication from old primary
    2. Promote replica to primary
    3. Update application connection strings
    """
    replica_cursor = replica_conn.cursor()
    # PostgreSQL command to promote replica
    replica_cursor.execute("SELECT pg_promote();")
    replica_conn.commit()
    print("✓ Replica promoted to primary")
    # Application must update its connection string to point to new primary
```

### Modern Approach: Distributed Consensus (Patroni, etcd)

Large enterprises use tools like **Patroni** or **etcd** to automate failover.

```python
# PostgreSQL with Patroni (automated failover)
# Patroni uses Raft consensus algorithm for leader election

import requests

# Patroni REST API
patroni_leader = "http://patroni-cluster-leader:8008"

def write_user_patroni(user_id: int, name: str, email: str):
    """
    Patroni automatically routes writes to current primary.
    If primary fails, Patroni elects a new one (< 1 second).
    """
    # Connect to any Patroni node (e.g., load balancer)
    conn = psycopg2.connect("host=patroni-cluster.company.com user=app")
    cursor = conn.cursor()
    
    try:
        cursor.execute(
            "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
            (user_id, name, email)
        )
        conn.commit()
        print("✓ Write succeeded")
    except Exception as e:
        if "read-only" in str(e):
            # Node became replica (failover happened)
            print("⚠ Primary failed over, retrying...")
            conn.close()
            # Patroni has already elected new primary
            # Reconnect and retry (load balancer redirects to new primary)
            write_user_patroni(user_id, name, email)
        else:
            raise

# Check cluster status
def check_patroni_status():
    response = requests.get(f"{patroni_leader}/api/cluster")
    cluster_info = response.json()
    print(f"Primary: {cluster_info['members'][0]['name']}")
    print(f"Replicas: {[m['name'] for m in cluster_info['members'][1:]]}")
    # Example output:
    # Primary: pg-node-1
    # Replicas: ['pg-node-2', 'pg-node-3']
```

### Enterprise Approach: Dual Active-Active Setup

Some large enterprises run **two independent databases** in different data centers (active-active or active-passive).

```python
# Oracle Data Guard or MySQL Group Replication (active-active)
# Both databases can accept writes, changes are replicated bidirectionally

def write_user_dual_dc(user_id: int, name: str, email: str, prefer_datacenter: str = "us-east"):
    """
    Write to primary DC, replicates to secondary DC.
    If primary DC fails, app can switch to secondary.
    """
    if prefer_datacenter == "us-east":
        primary = psycopg2.connect("host=db-us-east-1.company.com")
        secondary = psycopg2.connect("host=db-us-west-2.company.com")
    else:
        primary = psycopg2.connect("host=db-us-west-2.company.com")
        secondary = psycopg2.connect("host=db-us-east-1.company.com")
    
    try:
        cursor = primary.cursor()
        cursor.execute(
            "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
            (user_id, name, email)
        )
        primary.commit()
        print(f"✓ Write to primary ({prefer_datacenter}) succeeded")
        # Secondary gets copy via replication (async, ~100ms lag)
    except Exception as e:
        print(f"✗ Primary failed: {e}")
        # Failover to secondary
        cursor = secondary.cursor()
        cursor.execute(
            "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
            (user_id, name, email)
        )
        secondary.commit()
        print(f"✓ Write to secondary failover succeeded")

# Read from whichever DC is closer
def read_user_nearest_dc(user_id: int, user_location: str = "us-east"):
    if user_location == "us-east":
        conn = psycopg2.connect("host=db-us-east-1.company.com")
    else:
        conn = psycopg2.connect("host=db-us-west-2.company.com")
    
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
    return cursor.fetchone()
```

### Backup Strategy

```python
# Enterprise backup strategy (very important for fault tolerance!)

# 1. Point-in-time recovery
# PostgreSQL WAL archiving allows recovery to any second in history

# 2. Full backups
# pg_dump every night → stored in S3
# Can restore to completely separate database if needed

import subprocess
import time

def backup_database_daily():
    """Run daily backup (cron job)."""
    backup_file = f"production_backup_{time.strftime('%Y%m%d_%H%M%S')}.sql.gz"
    
    subprocess.run([
        "pg_dump",
        "-h", "primary.db.company.com",
        "-U", "backup_user",
        "production",
    ], stdout=subprocess.PIPE) | subprocess.run([
        "gzip"
    ], stdout=open(f"/backups/{backup_file}", "wb"))
    
    # Upload to S3 for durability
    subprocess.run([
        "aws", "s3", "cp",
        f"/backups/{backup_file}",
        "s3://company-backups/postgres/"
    ])

# 3. Test restores
def test_restore_backup():
    """Verify backups can be restored (CRITICAL!)."""
    # Actually restore to test database to catch corruption
    # Many companies have lost data because backup restoration failed
    subprocess.run([
        "pg_restore",
        "-h", "test.db.company.com",
        "-d", "test_restore",
        "/backups/production_backup_20240510.sql.gz"
    ])
    print("✓ Backup verified — can be restored if needed")
```

---

## 5. Real-World Case Studies

### Case Study 1: Twitter/X Migration (KV Stores → Fault Tolerance)

**Problem:** Twitter's early infrastructure used MySQL databases. As scale grew, they hit the wall:
- Replication lag caused inconsistencies (you'd post a tweet but not see it)
- Failovers took minutes (manual process)
- Database hotspots (one shard got more load than others)

**Solution:** Migrated to Cassandra + Redis for certain data:
- Tweets stored in Cassandra with RF=3
- Timeline data in Redis with cross-DC replication
- Eventual consistency model (acceptable for tweets)

**Result:**
- Sub-second failovers (automatic)
- 99.99% uptime (4+ nines)
- Horizontal scaling: add nodes as traffic grows

**Code pattern they adopted:**
```python
def tweet_write_cassandra(user_id: int, text: str):
    """Write optimized for fault tolerance."""
    # Cassandra quorum write (2/3 nodes)
    # Automatically handles node failures
    cassandra_session.execute(
        "INSERT INTO tweets (user_id, timestamp, text) VALUES (?, ?, ?)",
        [user_id, time.time(), text],
        consistency_level=ConsistencyLevel.QUORUM
    )
```

### Case Study 2: Stripe (SQL + Distributed Consensus)

**Problem:** Stripe processes billions in payments. Lost transactions = lost money. SQL primary-replica wasn't enough.

**Solution:**
- Primary PostgreSQL database (ACID transactions for financial data)
- Patroni for automatic failover with < 1 second RTO (recovery time objective)
- Standby replicas in multiple AZs
- Nightly test restores to verify disaster recovery

**Result:**
- Zero data loss (durability)
- Sub-second failover (availability)
- Full audit trail (compliance)

**Code pattern:**
```python
def process_payment_stripe(amount: float, card: str):
    """High-stakes write: must be durable."""
    conn = get_db_connection()  # Patroni ensures we get primary
    
    cursor = conn.cursor()
    try:
        cursor.execute(
            "INSERT INTO transactions (amount, card, status) VALUES (%s, %s, %s)",
            (amount, card, 'pending')
        )
        conn.commit()  # Synchronous commit (wait for WAL flush)
        # Only return success after durable on disk + replicated
        return "payment_processed"
    except Exception as e:
        conn.rollback()
        # Stripe has ACID transaction here, can safely rollback
        raise
```

### Case Study 3: Netflix (DynamoDB for Fault Tolerance)

**Problem:** Netflix's CDN and streaming systems need to handle massive scale with automatic recovery.

**Solution:**
- DynamoDB for configuration, user session data
- Multi-region active-active (Global Tables)
- Automatic failover: if one region dies, clients redirected to others
- Eventual consistency acceptable for these use cases

**Result:**
- No single point of failure
- Automatic regional failovers
- Can lose an entire AWS region and keep streaming

**Code pattern:**
```python
def get_user_preferences_multiregion(user_id: str):
    """
    DynamoDB Global Tables automatically:
    - Replicate writes to all regions
    - Route reads to nearest region
    - Failover if region is unavailable
    """
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.Table('user-preferences')  # Global Table
    
    response = table.get_item(Key={'user_id': user_id})
    return response.get('Item')
```

---

## 6. How Fault Tolerance Varies by Situation

### Scenario 1: Financial Transactions (Banks, Payment Processors)

**Requirements:**
- **Zero data loss** (durability is critical)
- **Consistency > availability** (ACID)
- **Audit trail** (compliance)

**Solution:** SQL database + Patroni
```python
# Synchronous commit: wait for durability
INSERT INTO transactions ... COMMIT;
# Only return success after:
# 1. Written to primary's WAL on disk
# 2. Written to replica's WAL on disk
# This is slow (~10ms) but guarantees no data loss
```

**Trade-off:** Slower writes (10-50ms) but data is never lost.

---

### Scenario 2: Social Media Feeds (Twitter, Instagram)

**Requirements:**
- **High availability** (users can't tolerate downtime)
- **Eventual consistency OK** (seeing a tweet 1 second late is fine)
- **Massive scale** (billions of reads/second)

**Solution:** Cassandra or DynamoDB
```python
# Quorum write with RF=3
INSERT INTO feed_items ... CONSISTENCY QUORUM;
# Succeeds when 2/3 nodes ACK (fast ~5ms)
# If 1 node fails, data is still safe
# Other nodes catch up asynchronously
```

**Trade-off:** Faster writes (5ms) but briefly stale reads.

---

### Scenario 3: Caching Layer (Redis)

**Requirements:**
- **Super fast** (sub-millisecond)
- **Best effort** (data loss OK, can recompute)
- **Availability** (if cache dies, rebuild it)

**Solution:** Redis Cluster with persistence off
```python
# Redis Sentinel for automatic failover
redis_cluster.set("cache:user:1001", user_data)
# If node fails, Sentinel:
#   1. Detects failure (< 100ms)
#   2. Elects new primary
#   3. Client redirects (transparent)
# Data might be lost but cache can be rebuilt from DB
```

**Trade-off:** Fastest (< 1ms) but can lose data on node failure.

---

### Scenario 4: Analytics Data Warehouse

**Requirements:**
- **Durability** (don't lose analysis results)
- **Eventual consistency** (stale data OK)
- **Cost efficient** (don't overspend on redundancy)

**Solution:** Spark + HDFS with replication factor 2
```python
# HDFS replicates each block to 2 nodes (not 3 like Cassandra)
# Saves 33% disk space
# Still survives 1 node failure
df.write.parquet("hdfs:///data/year=2024")
# Internally:
#   Block 1 → Node A, Node B (2 copies)
#   Block 2 → Node C, Node D (2 copies)
# If Node A fails, Block 1 still exists on Node B
```

**Trade-off:** Balanced — good durability without 3x storage cost.

---

### Scenario 5: IoT Sensor Data

**Requirements:**
- **Cheap per message** (millions of sensors)
- **Can tolerate some loss** (1% data loss acceptable)
- **Eventually consistent** (sensor reading 1 minute old is fine)

**Solution:** Kinesis or Kafka with RF=2
```python
# Kafka with replication_factor=2
# Cheaper than RF=3, still survives 1 broker failure
kafka_producer.send(
    'sensor-data',
    {
        'sensor_id': 'sensor-12345',
        'temperature': 23.5,
        'timestamp': time.time()
    }
)
# Message goes to:
#   - Broker 1 (primary)
#   - Broker 2 (replica)
# If Broker 1 fails, Broker 2 serves data
```

**Trade-off:** Cheap (RF=2) with acceptable risk.

---

## Comparison Table

| Situation | Best Store | Consistency | Availability | Replication | Failover Time |
|-----------|-----------|-------|------|---|---|
| Financial Transactions | PostgreSQL + Patroni | Strong | Lower priority | Sync to 1 replica | ~1-5 seconds |
| Social Media | Cassandra | Eventual | High | RF=3, async | <100ms (automatic) |
| Caching Layer | Redis Cluster | Not required | Very High | RF=2, async | ~100ms |
| Analytics | HDFS/Spark | Eventual | Medium | RF=2 | Manual re-run |
| IoT Sensors | Kafka | Eventually Consistent | High | RF=2 | Brokers auto-recover |

---

## Summary: Fault Tolerance Strategies

| Database Type | Fault Tolerance Method | Pros | Cons |
|---|---|---|---|
| **SQL (traditional)** | Primary-replica replication, manual failover | ACID, strong consistency | Manual failover, replication lag |
| **SQL (modern)** | Patroni + consensus (Raft) | ACID + automatic failover | More complex ops |
| **KV Stores (Cassandra)** | Multi-master, quorum reads/writes | Automatic failover, horizontal scale | Eventual consistency, complex tuning |
| **KV Stores (DynamoDB)** | AWS-managed, multi-AZ automatic | Fully managed, automatic failover | Vendor lock-in, pricing |
| **Redis** | Sentinel or Cluster mode | Fast failover, simple | Data loss possible |

---

**Navigation:**
- [← Previous: Scalability & Performance](09-scalability.md)
- [Back to Index](README.md)
