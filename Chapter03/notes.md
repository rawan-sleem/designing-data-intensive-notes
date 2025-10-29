# Chapter 3: Storage and Retrieval

Chapter 3 explores how databases manage and retrieve data on physical storage (disk). The discussion focuses on storage engines used in traditional relational databases and many NoSQL databases. The chapter examines two primary families of storage engines: log-structured and page-oriented (B-trees).

## I. Data Structures That Power Your Database

Fundamentally, a database must efficiently find data after it has been stored. While simply appending records to a file (a **log**) is efficient for writes, efficient retrieval requires an **index**. An index is additional metadata derived from the primary data that acts as a signpost to locate the data. Maintaining an index generally slows down writes because the index must also be updated whenever data is written.

### A. Hash Indexes

The simplest index relies on an in-memory hash map.

*   **Mechanism:** The database stores data by appending key-value pairs to an **append-only data file (log)**. The hash map stores every key and maps it to a **byte offset** in the data file where the value can be found.
*   **Write Performance:** Appending to a file is typically efficient, making writes fast.
*   **Read Performance:** Retrieval involves finding the offset in the map, performing a disk seek to that location, and reading the value.
*   **Constraints:** This approach, used by Bitcask (Riak's default storage engine), requires that **all keys fit entirely in available RAM**; however, values can exceed memory capacity as they are loaded from disk.
*   **Log Management:**
    *   To prevent infinite growth, the log is broken into **segments**.
    *   **Compaction** is performed, discarding duplicate keys and retaining only the most recent update for each key.
    *   Segments can be **merged** simultaneously with compaction. Merged segments are written to a new file, as segments are never modified after being written.
    *   **Deletion** requires appending a special marker called a **tombstone** to the log.
*   **Reliability:** In case of a crash, the in-memory hash maps are lost. Crash recovery can be sped up by storing a **snapshot** of the hash map on disk.
*   **Limitation:** This structure is inefficient for **range queries** (scanning keys sequentially).

### B. SSTables and LSM-Trees

The **Sorted String Table (SSTable)** format is an improvement over log segments, requiring that key-value pairs within each segment be **sorted by key**.

*   **Advantages of Sorted Segments:**
    1.  **Efficient Merging:** Merging SSTable segments is simple and efficient, even if files are larger than available memory, using an approach similar to the mergesort algorithm. When merging, the value from the most recent segment is kept.
    2.  **Sparse Indexing:** An in-memory index is still required, but it can be **sparse**, only storing offsets for blocks of keys. Retrieval involves finding the closest key in the index and scanning sequentially from that offset due to the sorted nature of the data.
    3.  **Compression:** Key-value pairs can be grouped into compressed blocks before writing to disk, reducing disk space and I/O bandwidth.
*   **LSM-Tree Principle:** To handle writes, this scheme uses a hybrid approach:
    1.  Writes are first added to an in-memory, sorted data structure called a **memtable** (e.g., a red-black tree).
    2.  When the memtable exceeds a threshold, it is written to disk as a new, sorted SSTable file.
    3.  Reads check the memtable first, then the most recent on-disk segment, and so on.
    4.  A separate append-only log is used for every write to allow the memtable to be restored after a crash.
    5.  A background process continuously handles merging and compaction.
*   **LSM Storage Engines:** Storage engines utilizing this merge and compaction principle (like LevelDB, RocksDB, Cassandra, and HBase) are called **LSM storage engines**.
*   **Full-Text Search:** Lucene, used by Elasticsearch and Solr, utilizes a similar method for storing its term dictionary, based on SSTable-like sorted files that are merged in the background.

### C. B-Trees

B-trees are the traditional indexing structure, generally employing an **update-in-place** philosophy.

*   **Structure:** B-trees organize data into fixed-size **pages** (blocks), typically 4 KB. Queries traverse from the root page down to the leaf page, which contains the key/value or a reference to it.
*   **Branching Factor:** The number of child page references in one page (the branching factor) is typically several hundred. This results in a shallow tree structure, often only three or four levels deep, maintaining a depth of $O(\log n)$.
*   **Updates:** Updates involve overwriting the relevant page on disk. If a page overflows during insertion, it is **split** into two half-full pages, and the parent page is updated.
*   **Reliability:** B-trees rely on a **write-ahead log (WAL)** (or redo log) to prevent index corruption if the database crashes during multi-page operations (such as a page split).
*   **Optimizations:** Keys can be abbreviated in interior pages to maximize the **branching factor**. Leaf pages often include pointers to sibling pages to facilitate fast sequential scanning.

### D. Comparing B-Trees and LSM-Trees

| Feature | B-Trees (Update-in-Place) | LSM-Trees (Log-Structured) |
| :--- | :--- | :--- |
| **Write Mechanism** | Overwrite pages in place. | Append to logs; merge immutable segments. |
| **Write Throughput** | Can be lower due to random I/O required to overwrite pages. | Typically higher, favoring sequential writes. |
| **Read Performance** | Generally faster and more predictable access times. | Slower reads because they may have to check the memtable and multiple SSTable segments. |
| **Storage Efficiency** | Can suffer from disk space fragmentation (unused space in pages). | Lower storage overhead due to compaction removing fragmentation and potentially achieving better compression. |

### E. Other Indexing Structures

*   **Secondary Indexes:** Indexes based on values other than the primary key.
    *   **Clustered Index:** Stores the raw row data directly within the index structure.
    *   **Covering Index:** Includes all required columns needed for a query, allowing the query to be satisfied directly from the index without fetching the full row.
*   **Multi-Column Indexes:** Indexes covering several fields simultaneously, such as concatenated indexes or R-trees (for geospatial data).
*   **Full-Text Search and Fuzzy Indexes:** Used for approximate matching (e.g., misspelled words). Indexing relies on specialized techniques like Levenshtein automata.
*   **In-Memory Databases:** Store data primarily in RAM, relying on logging, periodic snapshots, or replication for durability. New non-volatile memory (NVM) technologies may lead to further changes in storage design.

## II. Transaction Processing or Analytics?

Databases are optimized for different workloads: transaction processing (OLTP) and analytic systems (OLAP).

| Workload Type | OLTP (Online Transaction Processing) | OLAP (Online Analytic Processing) |
| :--- | :--- | :--- |
| **Primary Pattern** | Small number of records fetched by key. | Aggregation (count, sum, average) over large number of records. |
| **Primary Bottleneck** | Disk seek time. | Minimizing data read from disk. |
| **Data Representation** | Latest state of data. | History of events that happened over time. |

### A. Data Warehousing

Many organizations use separate data warehouse systems specifically optimized for analytic access patterns, distinct from their transactional systems. Data is typically loaded into the warehouse via an **ETL** (Extract–Transform–Load) process.

### B. Stars and Snowflakes: Schemas for Analytics

Analytic systems commonly use relational models optimized for querying, such as the **star schema**.

*   **Fact Table:** The central table, containing events (e.g., sales) and numerical attributes.
*   **Dimension Tables:** Peripheral tables containing context (e.g., product, date, region) referenced by foreign keys from the fact table.
*   **Snowflake Schema:** A variant where dimension tables are further normalized.

## III. Column-Oriented Storage

Column-oriented storage is highly effective for OLAP workloads because analytic queries often require reading many rows but only a few columns.

*   **Principle:** Data is stored by column, not by row. All values from a single column are stored together. This minimizes disk I/O, as queries only read the necessary columns.
*   **A. Column Compression:** Columnar data is often homogeneous, allowing for efficient compression techniques (e.g., bitmap encoding), which further reduces the required disk I/O.
*   **B. Sort Order in Column Storage:** Sorting the data can significantly improve compression and query performance. Databases may store multiple versions of the data, each sorted differently, to serve various query patterns efficiently.
*   **C. Writing to Column-Oriented Storage:** Column stores typically utilize LSM-tree approaches for handling writes. Writes go to an in-memory store and are written to columnar segments in **bulk** in the background.
*   **D. Aggregation: Data Cubes and Materialized Views:**
    *   **Data Cubes** and **Materialized Views** are aggregates that are **eagerly precomputed** along multiple dimensions.
    *   They speed up queries dramatically but introduce overhead during write operations, as the views must be kept up-to-date. Materialized views are a form of **derived data**.
