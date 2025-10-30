# designing-data-intensive-notes
My chapter-wise notes and coding labs from studying “Designing Data-Intensive Applications"

The following is a detailed, structured summary of the book **Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems** by Martin Kleppmann.

***

# Designing Data-Intensive Applications: Book Summary

## Overview and Philosophy

The goal of this book is to help engineers navigate the fast-changing landscape of technologies for processing and storing data. It digs into the key algorithms, principles, and trade-offs of successful data systems. The book addresses the challenges of building applications that must meet requirements for **Reliability, Scalability, and Maintainability**.

The book is structured into three parts: Foundations, Distributed Data, and Derived Data.

***

## Part I: Foundations of Data Systems

This part covers fundamental ideas applicable to all data systems, whether running on a single machine or distributed across a cluster.

### Chapter 1: Reliable, Scalable, and Maintainable Applications

The chapter defines the three main concerns for data-intensive applications:

1.  **Reliability:** The system must continue working correctly, even when facing faults such as hardware failures, software errors (bugs), and human mistakes. Fault tolerance mechanisms are necessary to prevent faults from causing failures.
2.  **Scalability:** The ability to cope with growth in data volume, traffic volume, or complexity. Scalability is measured by describing **load** (e.g., requests per second) and **performance** (e.g., latency percentiles).
3.  **Maintainability:** Making life easier for the engineering and operations teams over time. This includes:
    *   **Operability:** Making the system easy to keep running smoothly.
    *   **Simplicity:** Managing complexity to reduce bugs and maintenance costs.
    *   **Evolvability (Agility):** Making it easy to adapt the system to new requirements and changes.

### Chapter 2: Data Models and Query Languages

This chapter compares different methods of representing data:

*   **Relational Model vs. Document Model:**
    *   The **Relational Model** (based on SQL) organizes data into relations (tables).
    *   The **Document Model** (e.g., JSON) is often preferred for data with a tree structure (one-to-many relationships), as it reduces the object-relational impedance mismatch.
    *   **Many-to-Many Relationships** usually require identifiers (foreign keys or document references) and joins in both relational and document models.
*   **Query Languages:** Compares **Imperative languages** (specifying steps) with **Declarative languages** (specifying results, like SQL or CSS). MapReduce is also used as a query framework.
*   **Graph-Like Data Models:** Used when data has complex many-to-many relationships.
    *   **Property Graphs** (e.g., Neo4j) use vertices (nodes) and edges (relationships). Cypher is a common query language.
    *   **Triple-Stores** (e.g., RDF) use triples (subject, predicate, object). SPARQL is the associated query language.
    *   **Datalog** serves as a foundation, allowing complex queries to be built up from rules and supporting recursion.

### Chapter 3: Storage and Retrieval

This chapter explores how databases physically store and retrieve data:

*   **Row-Oriented Storage Engines (OLTP):** Optimized for transaction processing (OLTP) access patterns, typically involving indexed lookups of a small number of records.
    *   **Hash Indexes:** Simple key-value store using an in-memory hash map pointing to byte offsets in an append-only log file.
    *   **SSTables and LSM-Trees:** Log-Structured Merge-Trees use segments of sorted key-value pairs (SSTables). Merging and compaction keep the structure efficient, turning random-access writes into sequential writes for higher throughput.
    *   **B-Trees:** The standard indexing structure, organizing data into fixed-size pages. B-trees update data *in place* and are found in almost all major relational databases.
*   **Column-Oriented Storage (OLAP):** Optimized for analytics (OLAP), where queries scan massive numbers of rows.
    *   Instead of storing all row values together, it stores all values from each column together. This minimizes disk I/O because only necessary columns are read.
    *   Supports efficient **Column Compression** (e.g., using bitmap encoding) and **Vectorized Processing**.
*   **Data Warehousing:** Often uses a **Star Schema** (fact tables joined to dimension tables).
*   **Indexes:** Includes secondary indexes, clustered indexes, covering indexes, and specialized indexes like geospatial (R-trees) and full-text search (Lucene).

### Chapter 4: Encoding and Evolution

This chapter discusses how data is serialized and how schemas change over time (evolvability).

*   **Encoding Formats:** Data must be translated from in-memory structures into a byte sequence (**encoding/serialization**) for storage or network transmission.
    *   **Textual Formats (JSON, XML, CSV):** Widespread but often ambiguous regarding datatypes and inefficient for binary strings.
    *   **Binary Schema-Driven Formats (Thrift, Protocol Buffers, Avro):** Provide compact, fast encoding with clear rules for compatibility. Protocol Buffers and Thrift use **field tags** for compatibility, while Avro relies on the **writer's schema** and the **reader's schema** to resolve differences.
*   **Compatibility:** Essential for **Rolling Upgrades**.
    *   **Backward Compatibility:** New code can read data written by old code.
    *   **Forward Compatibility:** Old code can read data written by new code.
*   **Modes of Dataflow:**
    *   **Through Databases:** The writer encodes data, and the reader (potentially a later version of the application) decodes it.
    *   **Through Services (REST/RPC):** Client/server interaction where requests and responses must be compatible.
    *   **Message-Passing:** Asynchronous communication between processes using messaging systems (message brokers or actors).

***

## Part II: Distributed Data

This part examines systems where data is distributed across multiple machines, focusing on **shared-nothing architectures**.

### Chapter 5: Replication

Replication keeps copies of data on multiple nodes for high availability, low latency, and increased read scalability.

*   **Leader-Based Replication:** One replica is the **leader** (accepts writes); **followers** asynchronously or synchronously replicate changes from the leader using a log. Failover is required to handle leader failure.
*   **Problems with Replication Lag:** Asynchronous replication leads to eventual consistency. Issues include **Reading Your Own Writes**, **Monotonic Reads**, and **Consistent Prefix Reads**.
*   **Multi-Leader Replication:** Multiple nodes accept writes. Often used for multi-datacenter deployments. Requires complex **conflict resolution** (e.g., LWW, custom code, Operational Transformation, or CRDTs).
*   **Leaderless Replication:** Writes sent directly to multiple replicas. Uses **quorums** ($W$ writes and $R$ reads, with $W+R > N$ nodes) for fault tolerance. Relies on mechanisms like **read repair** and **hinted handoff**. Concurrent writes are detected using version vectors and the happens-before relationship.

### Chapter 6: Partitioning

Partitioning (or sharding) divides a dataset into subsets (partitions) for increased scalability, aiming to spread data and load evenly to avoid **hot spots**.

*   **Key-Value Partitioning:**
    *   **By Key Range:** Assigns contiguous blocks of keys to a partition (good for range queries, prone to skew).
    *   **By Hash of Key:** Distributes keys uniformly (good for reducing hot spots, poor for range queries).
*   **Secondary Indexes and Partitioning:**
    *   **Document-Partitioned:** Each partition maintains its local index (fast writes, slow reads using scatter/gather).
    *   **Term-Partitioned (Global Index):** Index is separated and partitioned by the term itself (fast reads, slow/complex writes).
*   **Rebalancing Partitions:** Necessary to adapt to growing data or changing cluster size. Strategies often use a fixed number of partitions much larger than the number of nodes to simplify data movement.
*   **Request Routing:** Systems must determine the correct node for a key (e.g., using a routing layer or a partition-aware client).

### Chapter 7: Transactions

Transactions provide guarantees (ACID) against errors and concurrent access.

*   **ACID Properties:** **Atomicity** (all-or-nothing), **Consistency** (data validity), **Isolation** (hiding concurrent changes), and **Durability** (data survives faults).
*   **Weak Isolation Levels:**
    *   **Read Committed:** Prevents reading uncommitted changes (**dirty reads**) and overwriting uncommitted changes (**dirty writes**).
    *   **Snapshot Isolation (Repeatable Read):** Guarantees reads see a consistent state of the database from a single point in time, preventing **read skew**.
    *   Prevents **Lost Updates** (using explicit locks, CAS, or atomic operations).
    *   Still vulnerable to anomalies like **Write Skew**.
*   **Serializability:** The strongest level, guaranteeing that concurrent transactions result in the same state as if they ran strictly sequentially.
    *   **Actual Serial Execution:** Running transactions in a single thread, often viable for in-memory systems.
    *   **Two-Phase Locking (2PL):** Locks prevent subsequent access until a transaction commits.
    *   **Serializable Snapshot Isolation (SSI):** An optimistic technique that allows reads from a snapshot but detects and aborts transactions that might have caused a non-serializable change.

### Chapter 8: The Trouble with Distributed Systems

This chapter details the inherent complexities and uncertainties in distributed environments.

*   **Faults and Partial Failures:** Unlike single systems, distributed systems experience partial failures, where some components are working and others are broken, making it impossible for the software to know the exact state.
*   **Unreliable Networks:** The network between nodes is unreliable and asynchronous; delays, losses, and reordering are possible. Timeouts must be used to detect potential faults, but this is ambiguous (did the remote system fail, or was the message delayed?).
*   **Unreliable Clocks:** Hardware clocks are inaccurate and subject to **skew** and time jumps (e.g., from NTP adjustments or leap seconds). Relying on synchronized clocks (like Google’s TrueTime) is necessary for strong consistency, but requires careful handling.
*   **Knowledge, Truth, and Lies:** Systems often rely on a majority (quorum) to define **truth** (e.g., node membership). Systems must also account for **Byzantine Faults** (malicious or erroneous actions).

### Chapter 9: Consistency and Consensus

This chapter explores mechanisms for fault tolerance and agreement.

*   **Linearizability:** A strong consistency model where operations appear to execute atomically on a single copy of the data, ensuring a total order of operations. It is often expensive, particularly across geographic distances. Linearizability is required for implementing things like uniqueness constraints and mutual exclusion locks.
*   **Ordering Guarantees:** Ordering helps preserve **causality** (the "happens-before" relationship). **Total Order Broadcast** (where messages are delivered reliably to all nodes in the same fixed order) is a powerful primitive that can be used to implement linearizable storage and state machine replication.
*   **Distributed Transactions and Consensus:** **Consensus** is the fundamental problem of getting multiple nodes to agree on a decision.
    *   **Two-Phase Commit (2PC):** A protocol used for **atomic commit** in distributed transactions, but it is blocking and sensitive to coordinator failure.
    *   **Fault-Tolerant Consensus Algorithms:** Protocols like Paxos, Raft, and ZooKeeper’s Zab achieve robust consensus, even when nodes fail.

***

## Part III: Derived Data

This final part focuses on building applications by integrating multiple heterogeneous data systems (databases, caches, indexes) using **derived data** and dataflow principles.

*   **Systems of Record and Derived Data:** The **System of Record** (source of truth) holds the authoritative, normalized data. **Derived Data** (indexes, caches, materialized views) is created from the system of record via a repeatable process. This clarifies complex system architectures.

### Chapter 10: Batch Processing

Batch processing works on bounded (fixed-size) input files to produce new output files.

*   **Unix Tools Philosophy:** MapReduce and modern dataflow engines inherited the Unix principles of immutability, simple interfaces (like stdin/stdout), and composing small tools that "do one thing well".
*   **MapReduce:** A distributed processing framework using a distributed filesystem (HDFS). It consists of a `map` phase (extract key-value pairs) and a `reduce` phase (aggregate values for the same key).
*   **Joins:** Implemented efficiently using partitioning and sorting: **Sort-Merge Joins** (reduce-side) and **Broadcast Hash Joins** (map-side).
*   **Beyond MapReduce:** Modern dataflow engines (Spark, Flink, Tez) focus on minimizing the **materialization of intermediate state** to disk (which is costly in MapReduce) and use pipelining, often keeping intermediate data in memory.
*   **Outputs and Fault Tolerance:** Batch outputs are immutable. This immutability allows for easy rollback and recovery from bugs (referred to as human fault tolerance), simply by rerunning the job.

### Chapter 11: Stream Processing

Stream processing deals with **unbounded** (never-ending) sequences of events.

*   **Messaging Systems:** **Log-based message brokers** (like Kafka) use an append-only log, providing partitioning and ordering guarantees, and retaining messages so consumers can reread old messages (non-destructive reading).
*   **Databases and Streams:** The database state can be viewed as the integration of the event stream over time, and the stream (changelog) is the derivative of the state.
    *   **Change Data Capture (CDC):** Extracts changes from a database into an event log.
    *   **Event Sourcing:** Treats the event log as the **system of record**, and the current state is derived entirely from replaying the log. Immutability simplifies reasoning and enables various read-optimized derived views.
*   **Processing Streams:** Stream processors continually derive new streams or update derived state.
    *   **Time and Windows:** Distinction between **event time** (when the event occurred) and **processing time** (when the processor sees it) is critical, especially when dealing with late-arriving data. Operations are often performed over tumbling, hopping, or sliding **windows**.
    *   **Stream Joins:** Operations such as stream-table joins (stream enrichment) are performed efficiently to keep derived state up to date.
    *   **Fault Tolerance:** Achieved using checkpointing of operator state and guaranteeing **exactly-once semantics** through mechanisms like atomic commit and idempotence.

### Chapter 12: The Future of Data Systems

The final chapter discusses integration and design patterns for future reliable, scalable, and maintainable applications.

*   **Data Integration and Evolution:** Due to the specialization of tools, complex applications require **combining specialized tools by deriving data**. Reprocessing historical data (using batch/stream processing) is key to handling evolving schemas and changing requirements.
*   **Unbundling Databases:** Suggests treating traditional database functions (indexing, caching, replication) as separate, specialized services connected by logs/dataflows. This **"database inside-out"** approach follows the Unix philosophy of loose coupling and composition.
*   **Dataflow Architecture:** Application code acts as derivation functions that consume streams of state changes and produce derived streams. The derived dataset sits at the boundary where the **write path** (eager computation) meets the **read path** (lazy computation on demand). The write path can be extended all the way to stateful, offline-capable client devices.
*   **Aiming for Correctness:** Focuses on guaranteeing **Correctness, Timeliness, and Integrity**. Emphasizes the **End-to-End Argument** (integrity checks should happen at the application level). Enforcing constraints (like uniqueness) often requires coordination and consensus.
*   **Ethics and Responsibility:** Concludes by discussing the ethical responsibility of engineers, particularly concerning **privacy and tracking** (surveillance). Data is a hazardous material, and security, privacy, and accountability must be central concerns.
