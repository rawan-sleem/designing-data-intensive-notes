## Chapter 5: Replication

Replication involves keeping a copy of the same data on multiple machines (replicas) connected via a network.

### Core Reasons for Replication

1.  **Latency Reduction:** Keeping data geographically close to users.
2.  **High Availability:** Allowing the system to continue operating even if parts fail (increasing fault tolerance).
3.  **Scalability:** Distributing read requests across multiple machines (increasing read throughput).

The primary difficulty in replication is managing **changes** (writes) to the replicated data. The chapter discusses three main algorithms for change replication: single-leader, multi-leader, and leaderless replication.

### I. Leaders and Followers (Single-Leader Replication)

This is the most common model (also known as master–slave or active/passive replication).

| Role | Responsibility |
| :--- | :--- |
| **Leader (Primary/Master)** | Accepts **all** client write requests. Writes data to its local storage first. |
| **Followers (Secondaries/Slaves)** | Replicate data changes from the leader and serve read queries. Clients cannot write directly to followers. |

#### A. Synchronous Versus Asynchronous Replication

Replication can be configured to operate synchronously or asynchronously.

| Type | Durability and Performance | Configuration |
| :--- | :--- | :--- |
| **Synchronous Replication** | The leader waits for the follower to confirm receipt of the write before acknowledging success to the client. Ensures durability but increases write latency. | Often used in **semi-synchronous** mode (one follower is synchronous, the others are asynchronous). |
| **Asynchronous Replication** | The leader proceeds immediately without waiting for follower confirmation. If the leader fails, any changes not yet replicated are lost (weakening durability). Allows the leader to continue processing writes even if followers lag significantly. Widely used, especially for geographically distributed followers. |
| **Research on Replication** | **Chain replication** is a variant of synchronous replication implemented in systems like Microsoft Azure Storage. |

#### B. Setting Up New Followers

To ensure the new follower has an accurate and consistent copy of the data, the process typically involves:

1.  Taking a **consistent snapshot** of the leader's database, ideally without locking the entire database (a feature also needed for backups, like `innobackupex` in MySQL).
2.  Copying the snapshot to the new follower.
3.  The follower connects to the leader and requests all data changes that have occurred since the snapshot was taken, using an **exact position in the replication log** (e.g., PostgreSQL's log sequence number or MySQL's binlog coordinates).
4.  Once the backlog is processed, the follower has **caught up** and continues to process current data changes.

#### C. Handling Node Outages

1.  **Follower Failure:** The follower can usually reconnect later and catch up using its log position.
2.  **Leader Failure (Failover):** This critical process involves promoting a follower to be the new leader.
    *   **Automatic Failover Steps:** Failure detection (often via timeout). Choosing a new leader (preferably the most up-to-date replica). Reconfiguring the system (clients redirect writes, other followers replicate from the new leader).
    *   **Failover Problems:** Difficulty in reliable failure detection. Risk of **Split Brain** (two nodes believe they are the leader), which can lead to data corruption. Solutions often require **fencing** or **STONITH** (Shoot The Other Node In The Head) to prevent the old leader from causing damage. Data loss is possible if failover occurs with asynchronous replication.

#### D. Implementation of Replication Logs

Databases use different methods to transport changes from leader to followers:

1.  **Statement-based Replication:** The leader logs and sends every SQL write request (e.g., `INSERT`, `UPDATE`). This can break down if statements are **nondeterministic** (relying on current time or random functions), or if they rely on the exact data order (e.g., statements using `LIMIT`).
2.  **Write-Ahead Log (WAL) Shipping:** The leader sends the physical log used by the storage engine, which contains low-level changes to disk blocks. This tightly couples replication to the storage engine. It often prevents running different database versions on the leader and followers, complicating zero-downtime upgrades.
3.  **Logical (Row-based) Replication:** The leader uses a logical log describing changes at the row or document level. Since it is decoupled from the storage engine, it allows leader and followers to run different versions. This format is also useful for **Change Data Capture (CDC)**, feeding data changes to external systems like data warehouses or caches.
4.  **Trigger-based Replication:** Uses database triggers to execute application code when data is changed. Offers flexibility (replicating subsets, cross-database replication, custom conflict resolution) but has higher overhead and is more bug-prone than built-in methods.

### II. Problems with Replication Lag (Eventual Consistency)

When read requests are served by asynchronous followers (read-scaling architecture), the system provides only **eventual consistency**: replicas will eventually converge, but temporary lag and inconsistency are common. This lag introduces specific consistency anomalies:

1.  **Reading Your Own Writes (Read-after-write consistency):** A user performs a write on the leader but immediately reads from a lagging follower, making the write appear lost.
    *   *Solution Approaches:* Reading data the user may have modified directly from the leader. Monitoring follower lag and preventing reads from stale replicas. Implementing **cross-device consistency** if the user accesses the service from multiple devices.
2.  **Monotonic Reads:** Prevents a user from seeing time go backward (i.e., reading newer data, and then reading older data in a subsequent query).
    *   *Solution:* Ensuring that a specific user always routes all their reads to the same replica (e.g., using a hash of the User ID).
3.  **Consistent Prefix Reads:** Guarantees that if a sequence of writes is causally related, any reader sees them appear in the correct, causal order. This is particularly relevant in **partitioned (sharded) databases** where different partitions operate independently.
4.  **Solutions:** While solving these issues in application code is complex, databases can offer stronger guarantees through **transactions** (Chapter 7) or **consensus** algorithms (Chapter 9).

### III. Multi-Leader Replication

In this setup (active/active), multiple nodes are configured to accept writes.

#### A. Use Cases for Multi-Leader Replication

1.  **Multi-Datacenter Operation:** Provides lower write latency for users by allowing them to write to the leader in their local datacenter, with asynchronous replication occurring between datacenters.
2.  **Clients with Offline Operation:** Each device/client holds a replica that acts as a leader, accepting local writes while offline and syncing changes when connectivity resumes.
3.  **Collaborative Editing:** Used in real-time editing applications (like Google Docs) where changes are applied locally and replicated asynchronously, necessitating conflict resolution.

#### B. Handling Write Conflicts

Since writes can occur concurrently on different leaders, **write conflicts** are inevitable. All replicas must resolve the conflict in a **convergent** way to reach the same final state.

*   **Conflict Resolution Methods:**
    *   **Last Write Wins (LWW):** Chooses the write with the largest timestamp or highest ID. This can result in data loss if clocks are inaccurate or concurrent changes occur rapidly.
    *   **Merging Values:** Combining conflicting data, such as concatenating items alphabetically.
    *   **Custom Application Logic:** Implementing specific code to decide the final value, executed either during the write (replication) or during the read operation.
*   **Automatic Resolution Research:** Includes **Conflict-Free Replicated Datatypes (CRDTs)** and **Operational Transformation (OT)**, designed to handle collaborative editing scenarios automatically.

#### C. Multi-Leader Replication Topologies

The configuration defines the paths writes take to propagate to all nodes.

*   **Topologies:** Examples include **circular** (ring), **star**, or **all-to-all**.
*   **Loop Prevention:** Nodes must use unique identifiers and tag writes with the IDs of all nodes they have passed through, ignoring writes tagged with their own ID.
*   **Causality Issues:** Network delays can cause writes to "overtake" each other in all-to-all topologies, resulting in replicas receiving causally dependent events (e.g., an update) before the antecedent event (e.g., the insert). This requires advanced ordering techniques like **version vectors**.

### IV. Leaderless Replication

In this model (used by Dynamo-style systems like Cassandra and Riak), clients send writes to multiple replicas simultaneously, and no single node is designated as the primary leader.

#### A. Writing to the Database When a Node Is Down

1.  **Quorum Writes ($w$):** A write is considered successful only if acknowledged by at least $w$ replicas.
2.  **Quorum Reads ($r$):** Reads must query at least $r$ replicas.
3.  **Quorum Condition:** The condition $\mathbf{w + r > n}$ (where $n$ is the total replicas) ensures that the read and write sets overlap, guaranteeing that a read retrieves the most recent successful write.
    *   *Customization:* $n$ is typically odd, and $w$ and $r$ are often $(n+1)/2$ (e.g., 2 out of 3, or 3 out of 5).

#### B. Ensuring Eventual Consistency

Two mechanisms are used to ensure data propagates to all replicas:

1.  **Read Repair:** When a client reads multiple versions during a quorum read, it writes the newer version back to any replica that returned stale data. Effective for frequently read data.
2.  **Anti-Entropy:** A background process runs to detect differences between replicas and copy missing data.

#### C. Sloppy Quorums and Hinted Handoff

*   **Sloppy Quorums:** When a node fails, writes may be accepted by other reachable nodes, even if they are not the designated **home nodes** for that key. This improves write availability during network partitions.
*   **Hinted Handoff:** The temporary node stores a "hint" and forwards the write to the intended home node once it recovers.
*   *Limitation:* Sloppy quorums violate the $w+r > n$ overlap guarantee, meaning a quorum read may not return the latest write. Leaderless systems, even with strict quorums, **do not provide linearizability**.

#### D. Detecting Concurrent Writes

Leaderless systems must explicitly handle concurrent writes since there is no centralized ordering.

1.  **Last Write Wins (LWW):** Simple approach using timestamps to choose a "winner". Prone to losing data if clocks are unreliable or writes are truly concurrent.
2.  **Causality ("Happens-Before"):** The system tracks the causal dependencies to determine if operations are truly concurrent or if one preceded the other.
3.  **Version Vectors:** Used to track causal dependencies across all operations and replicas. They distinguish between sequential overwrites and **concurrent siblings**.
4.  **Merging Siblings:** Version vectors ensure no data is silently dropped, but the client or application logic must handle the **merging** of concurrent values (siblings).
5.  **Tombstones:** Special deletion markers required to prevent deleted items from reappearing when conflicting versions (siblings) are later merged.
