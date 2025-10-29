# Chapter 4: Encoding and Evolution

Applications must be designed for **evolvability**, allowing changes to systems and schemas over time. Since complex applications typically undergo **rolling upgrades** (deploying new versions gradually), different versions of code and data formats will coexist simultaneously.

To ensure stability during evolution, two types of compatibility are necessary:

1.  **Backward compatibility:** Newer code must be able to read data written by older code.
2.  **Forward compatibility:** Older code must be able to read data that was written by newer code. This is often trickier as older code must safely ignore unknown additions made by the newer version.

## I. Formats for Encoding Data

When data is sent over a network or written to a file, it must be converted from its **in-memory representation** (optimized for CPU access, often containing pointers) into a **sequence of bytes**. This translation is called **encoding** (or serialization/marshalling), and the reverse is **decoding** (parsing/deserialization/unmarshalling).

### A. Language-Specific Formats

Many programming languages offer built-in encoding mechanisms (e.g., Java's `Serializable`). However, these are often restricted to a single language and generally **fail to provide the necessary compatibility** for systems designed for independent evolution and cross-language communication.

### B. JSON, XML, and Binary Variants

**JSON, XML, and CSV** are popular textual formats.

*   **Datatype and Binary Issues:** These formats have poor handling of numerical types (e.g., JSON does not distinguish integers from floating-point numbers, potentially losing precision on large numbers greater than $2^{53}$). They also lack native support for binary strings, requiring Base64 encoding, which is considered "hacky" and increases data size by 33%.
*   **Verbosity:** Without a required schema, these formats must include verbose metadata, such as all object field names (e.g., `"userName"`), within the encoded data itself.
*   **Binary Variants:** Formats like **MessagePack** aim for compactness by using binary encoding markers for types and lengths. However, they retain the JSON data model and still must include field names in the data. Encoding a sample record (Example 4-1) in MessagePack requires **59 bytes**.

### C. Thrift and Protocol Buffers

**Protocol Buffers (protobuf)** and **Apache Thrift** achieve much higher compactness and better evolvability by requiring a formal schema definition using an Interface Definition Language (IDL).

*   **Encoding Mechanism:** Fields are identified by **numerical tag numbers** (e.g., 1, 2, 3) defined in the schema, rather than repeating verbose field names.
    *   **Thrift:** Offers protocols like **BinaryProtocol** (59 bytes for the example record) and **CompactProtocol** (34 bytes), which uses variable-length integers to save space.
    *   **Protocol Buffers:** Encodes the sample record in **33 bytes**.
*   **Schema Evolution Rules:**
    *   **Field Tags are Critical:** The **field tag number is immutable** and defines the meaning of the field in the encoded data. Changing a field name is safe, but changing its tag number is not.
    *   **Forward Compatibility:** Old code encountering a new, unrecognized tag number skips the appropriate number of bytes (determined by the datatype annotation), maintaining forward compatibility.
    *   **Backward Compatibility:** New fields must be marked as **optional** or assigned a **default value** to ensure old data (lacking the new field) can still be read by new code.
    *   **Datatype Changes:** Changing datatypes (e.g., from 32-bit to 64-bit integer) is possible but risky, as older code might truncate or lose precision on data written by newer code.
    *   **Lists (Protobuf):** Protocol Buffers uses a `repeated` marker, allowing an optional (single-valued) field to be converted to a `repeated` (multi-valued) field safely.

### D. Avro

Avro handles schema evolution differently, relying on **schema resolution** rather than strict tag numbering.

*   **Encoding Efficiency:** Avro is the most compact binary encoding discussed, requiring only **32 bytes** for the sample record. The encoded data contains no field markers or type information; fields are simply concatenated values.
*   **Schema Resolution:** The Avro library must compare the **writer’s schema** (used to encode) against the **reader’s schema** (expected by the reading application) during decoding. Fields are matched up based on **field name**.
*   **Compatibility Rules:**
    *   Any field added or removed must have a **default value** to ensure compatibility.
    *   If a reader expects a field not present in the written data, the default value from the reader's schema is substituted.
    *   Field name changes are backward compatible if the reader's schema defines an **alias** for the old name, but they are not forward compatible.
*   **Schema Discovery:** For large files (like those used in Hadoop), the writer’s schema is included once at the start of the file in an **Avro object container file**.
*   **Dynamic Generation:** Avro is particularly well-suited for dynamically generated schemas (e.g., derived from a database) because it does not rely on manual tag assignment like Thrift or Protocol Buffers.

### E. The Merits of Schemas

Schema-based binary formats provide significant advantages over text or schemaless variants:

*   **Documentation and Validation:** The schema serves as guaranteed-up-to-date documentation and allows compatibility checks before deployment.
*   **Tooling:** Enables code generation for statically typed languages, aiding compile-time type checking.
*   **Efficiency:** Achieves greater compactness by omitting verbose field names from the encoded data.

## II. Modes of Dataflow

Data encoding and compatibility are critical for enabling independent evolution across different modes of communication where processes do not share memory.

### A. Dataflow Through Databases

In database interaction, the process writing the data encodes it, and the process reading it decodes it.

*   **Compatibility Requirement:** **Forward compatibility** is often required because a newer version of the code might write data that must later be read by an older version still running.
*   **Data Longevity:** The fundamental observation is that **data outlives code**; records written years ago persist in their original encoding.
*   **Schema Evolution in Place:** Most relational databases allow non-destructive schema changes (e.g., adding a nullable column) without rewriting all existing data immediately. When an old row is read, the database fills in default values (like nulls) for missing columns, making the entire database *appear* as if it conforms to the latest schema (schema evolution).
*   **Unknown Field Preservation:** When old code reads a record written by new code, updates the record, and writes it back, it must be careful not to discard unknown fields (fields added by the newer version) during the decode/re-encode cycle.
*   **Archival Storage:** Database snapshots taken for backups or data warehouses (see "Data Warehousing" on page 91) are typically re-encoded into the latest schema version, often using formats like Avro object container files or column-oriented formats like Parquet.

### B. Dataflow Through Services: REST and RPC

Network communication often involves clients and servers interacting via a service API. Architectures like **SOA** (Service-Oriented Architecture) and **Microservices** emphasize independent deployment, requiring compatible data encoding between coexisting versions of clients and servers.

*   **Service Uses:** This mode includes internal service communication, client applications calling services, and data exchange between different organizations (e.g., public APIs).
*   **REST:** REST APIs commonly use JSON/XML over HTTP.
*   **RPC:** Remote Procedure Call frameworks (like Thrift, Protocol Buffers, gRPC) attempt to make network requests behave like local function calls. However, RPC is complicated by network issues and the necessity to translate datatypes between potentially different programming languages. Older RPC systems often struggled with compatibility issues.

### C. Message-Passing Dataflow

Asynchronous message-passing systems (such as message brokers/queues) act as intermediaries between processes.

*   **Mechanism:** A client sends a message (byte sequence) to a **message broker** (or message queue), which stores it temporarily before delivering it to one or more consumers.
*   **Compatibility Needs:** These systems require strong **forward and backward compatibility** to allow publishers (encoders) and consumers (decoders) to evolve and deploy independently.
*   **Actor Frameworks:** Distributed actor systems (e.g., Erlang OTP, Akka) use internal message passing and face similar compatibility challenges during rolling upgrades.

## III. Summary

The choice of encoding format significantly impacts a system's evolvability and efficiency. While language-specific formats lack compatibility, schema-based binary formats like Avro, Protocol Buffers, and Thrift provide robust mechanisms—such as field tags or schema resolution—to handle evolving schemas while maintaining both backward and forward compatibility. This compatibility is vital for enabling rolling upgrades and independent deployment across databases, services, and message queues.
