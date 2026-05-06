# Key-Value Databases

---

## 10. Availability and Fault Tolerance

**Key-Value Databases** are well-regarded for their superior **availability** and **fault tolerance** in distributed systems, especially when compared to traditional relational databases (RDBMS). This section explores these advantages, followed by comparative examples, code snippets, and diagrams.

---

### 10.1 Key-Value Databases: Availability and Fault Tolerance

Key-value databases such as Redis, DynamoDB, and Cassandra are optimally designed for availability and fault-tolerant systems, balancing aspects of the **CAP theorem**:

1. **Replication Mechanisms:**
   - Key-value stores replicate data across multiple nodes and regions for **high availability** and **read-write durability**.
   - Example: In Cassandra, data is replicated across nodes using consistent hashing and replication-factor settings.

2. **Eventual Consistency:**
   - Favors availability by allowing asynchronous replication. Reads may be stale initially but eventually converge.
   - Example Code in Cassandra:
      ```cql
      CREATE TABLE users (
         id UUID PRIMARY KEY,
         name TEXT,
         email TEXT
      ) WITH replicas = 3 AND consistency = 'eventual';
      ```

3. **Partition Tolerance:**
   - Networks failures or partition splits do not affect functionality due to replicas always being available.

**Diagram: Distributed Replication in Cassandra**

```
          ┌──────────┐
          │ Node A   │                        ┌──────────┐      
          │ Primary  │ --Replication-->       │ Node B   │ 
          └──────────┘   Table writes         └──────────🔒 Same.
---