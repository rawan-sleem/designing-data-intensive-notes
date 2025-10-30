**Chapter 6: Partitioning** (also known as sharding). 

***

# Chapter 6: Partitioning (Sharding)

## I. Core Concepts and Architecture

Partitioning is necessary when dealing with very large datasets or very high query throughput, exceeding the capacity of a single machine.

*   **Terminology:** The term "partitioning" is the most established, but it is also known as sharding. Other terms used include: region (HBase), tablet (Bigtable), shard (MongoDB, Elasticsearch, SolrCloud), vnode (Cassandra, Riak), and vBucket (Couchbase).
*   **Goal:** The primary objective is to spread the **data and the query load evenly** across multiple nodes to achieve horizontal scalability, thereby avoiding **hot spots** (nodes with disproportionately high load).
*   **Partitioning and Replication:** Partitioning is typically combined with replication (Chapter 5) so that copies of each partition are stored on multiple nodes for fault tolerance. Although each record belongs to exactly one partition, copies of that partition may reside on several nodes. If a leader–follower replication model is used, a node may act as the leader for some partitions and a follower for others.

## II. Partitioning Strategies for Key-Value Data

These strategies address how records are assigned to partitions when accessed by a primary key.

### 1. Partitioning by Key Range
This method assigns a **contiguous range of keys** to each partition.

*   **Mechanism:** If the boundaries between ranges are known, the partition containing a specific key can be easily determined. The boundaries of the key ranges must **adapt to the data distribution** to ensure the data is distributed evenly, as the data itself may not be evenly spaced (e.g., some alphabet ranges are denser than others).
*   **Advantage:** Allows for **efficient range queries**.
*   **Disadvantage (Hot Spots):** This approach is vulnerable to **hot spots** if the application often accesses keys that are close together in the sorted order (e.g., time-series data using timestamps as keys).
*   **Rebalancing:** Partitions are typically rebalanced dynamically by **splitting the range into two subranges** when a partition becomes too large.

### 2. Partitioning by Hash of Key
This method utilizes a **hash function** to distribute keys randomly across partitions.

*   **Advantage:** Effective at **relieving hot spots** caused by skewed workloads and ensures uniform key distribution.
*   **Disadvantage:** Range queries are **not efficient**.
*   **Consistent Hashing:** This term is sometimes used, referring to the partition boundaries being randomly chosen.

### 3. Compound Keys (Hybrid Approach)

*   **Cassandra's Method:** Cassandra utilizes a compromise by hashing only the **first part** of a compound primary key to determine the partition, while subsequent columns are used as a concatenated index for **sorting the data within** that partition (stored in SSTables).
*   **Benefit:** This allows for efficient **range scans** over the secondary columns of the key, provided the first column value is fixed in the query.

## III. Partitioning Secondary Indexes

Secondary indexes require distinct partitioning strategies, depending on whether they are kept local to a partition or distributed globally.

### 1. Document-Partitioned Indexes (Local Indexes)

*   **Mechanism:** Each partition maintains its own local secondary index, covering **only** the documents within that partition.
*   **Writes:** They are fast, as a write only needs to update the indexes in the single partition containing the document.
*   **Reads (Querying):** They are slow, requiring the query to be sent to **all partitions** and the results collected and merged—known as the **scatter/gather** approach.

### 2. Term-Partitioned Indexes (Global Indexes)

*   **Mechanism:** A global index is constructed to cover data across all partitions, but the index itself is partitioned based on the value (or **term**) being indexed.
*   **Reads:** Efficient, as the query is routed directly to the specific partition of the index holding the term.
*   **Writes:** They are slower and more complex. Updating a single document may require updating index partitions on different nodes.
*   **Consistency:** Updates to term-partitioned indexes are often **asynchronous** in practice. Synchronous updates would require complex distributed transactions (discussed in Chapter 7 and 9).

## IV. Rebalancing Partitions

Rebalancing is necessary when load or dataset size changes, or when cluster nodes are added or removed.

*   **Objective:** To maintain an even load distribution and move only the necessary data.
*   **Strategy (Fixed Number of Partitions):** The standard strategy is to create a large, **fixed number of partitions** (e.g., 1,000 partitions for 10 nodes). Partitions are treated as atomic units: when a new node is added, it "steals" **entire partitions** from existing nodes. This is done while keeping the number of partitions and the assignment of keys to partitions constant.
*   **Alternative Strategy (Fixed Partitions per Node):** Some systems (e.g., Cassandra) keep the number of partitions proportional to the number of nodes.
*   **Operations:** Rebalancing can be performed **automatically or manually**. The old partition assignment is used for reads and writes while the data transfer is in progress.

## V. Request Routing and Execution

The system must determine which node a client request should be sent to for data retrieval or modification.

*   **Routing Approaches:**
    1.  A **routing layer** (proxy) that forwards requests to the correct node (e.g., `mongos` in MongoDB).
    2.  A **partition-aware client** that is configured to know the partitioning and connects directly to the appropriate node.
    3.  Requests sent to **any node**, which then forwards them internally to the correct partition node (e.g., Cassandra and Riak use a gossip protocol).
*   **Metadata Management:** The mapping of partitions to nodes is stored as cluster metadata, often tracked by coordination services like **ZooKeeper** (used by HBase, SolrCloud, and Kafka) or Helix (used by Espresso).

*   **Parallel Query Execution:** Partitioning is leveraged in **Massively Parallel Processing (MPP) relational database systems** to break down complex queries (joins, filtering, aggregation) and execute the components in parallel across multiple nodes.
