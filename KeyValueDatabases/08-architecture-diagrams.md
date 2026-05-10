# Key-Value Databases — Architecture Diagrams

## Single-Node Architecture

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

## Distributed Cluster Architecture

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

## Write Workflow (Distributed)

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

## Read Workflow (Distributed, Cache Miss Path)

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

**Navigation:**
- [← Previous: Real-World Use Cases](07-use-cases.md)
- [Back to Index](README.md)
- [Next: Scalability & Performance →](09-scalability.md)
