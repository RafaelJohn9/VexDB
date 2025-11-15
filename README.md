# **📐 High-Level Architecture Overview**

Your vector database consists of four major subsystems:

1. **API Layer** – handles client requests (`insert`, `search`, `delete`, `update`)
2. **Index Manager** – keeps vectors searchable (brute-force or ANN)
3. **Storage Engine** – persists data on disk
4. **Metadata Registry** – tracks IDs, offsets, deleted flags

Think of it like:
**API → Coordinator → (Index + Storage)**

---

# **UML: Component Diagram**

```
+-------------------------------------------------------------+
|                        VectorDB (System)                    |
|                                                             |
|  +------------------+       +----------------------------+  |
|  |     API Layer    |<----->|    VectorDB Coordinator    |  |
|  +------------------+       +----------------------------+  |
|                               |                |            |
|                               |                |            |
|                        +------+-----+   +------+-------+    |
|                        |  Index     |   | Storage      |    |
|                        |  Manager   |   | Engine       |    |
|                        +------+-----+   +------+-------+    |
|                               |                |            |
|                        +------+-----+   +------+-------+    |
|                        | Metadata  |   |   File Store   |   |
|                        | Registry  |   |  (Append Log)  |   |
|                        +------------+   +----------------+   |
|                                                             |
+-------------------------------------------------------------+
```

---

# **UML: Class Diagram (Core Components)**

```
+-------------------+
| VectorDB          |
+-------------------+
| +coordinator      |
+-------------------+
| +insert()         |
| +search()         |
| +delete()         |
| +update()         |
+-------------------+

          |
          v

+-------------------------------------+
| VectorDBCoordinator                 |
+-------------------------------------+
| +index_manager: IndexManager        |
| +storage_engine: StorageEngine      |
| +metadata: MetadataRegistry         |
+-------------------------------------+
| +handle_insert()                    |
| +handle_search()                    |
| +handle_delete()                    |
| +handle_update()                    |
+-------------------------------------+

+-----------------------+        +-------------------------+
| IndexManager          |        | StorageEngine           |
+-----------------------+        +-------------------------+
| +vectors_in_mem: map  |        | +append_log             |
| +index_structure (opt)|        | +compaction_worker      |
+-----------------------+        +-------------------------+
| +add_vector()         |        | +write_record()         |
| +remove_vector()      |        | +read_record(offset)    |
| +search()             |        | +compact()              |
+-----------------------+        +-------------------------+

+----------------------------+
| MetadataRegistry           |
+----------------------------+
| +id → offset               |
| +id → deleted_flag         |
+----------------------------+
| +mark_deleted()            |
| +update_offset()           |
| +get_offset()              |
+----------------------------+
```

---

# **Why This Architecture? (Architectural Characteristics)**

The choices in the design optimize for **simplicity, durability, performance, extensibility, and correctness**.

Here’s why each decision was made:

---

# **1. Append-Only Storage → Durability + Simplicity**

### Architectural Characteristics:

* **Durability** - once a write is acknowledged, it must survive crashes and power loss. Achieved by durable flushes (fsync/fdatasync), write‑ahead logs or replication. Note: true durability depends on the **OS/filesystem** and device **write caches**; you must use the right **flush/barrier semantics** to avoid lost or reordered writes.
* **Write Efficiency** -  minimize write amplification and I/O by batching, using **append-oriented structures**, and compacting data intelligently. Efficient writes reduce latency, increase throughput, and prolong SSD lifetime.
* **Crash Safety** - after an unexpected crash the system should recover to a consistent state. Use atomic updates, checksums, and replayable logs to detect and repair partial or torn writes and to ensure invariants hold after recovery.
* **Simplicity** - prefer small, well-defined invariants and minimal code paths. Simpler designs are easier to reason about, test, and maintain; they reduce the chance of subtle bugs in failure scenarios.

An append-only log:

* Gives fast writes (sequential disk writes)
* Is corruption-resistant
* Allows easy recovery after crash
* Makes replication simpler later

This is why Kafka, Redis AOF, and RocksDB use append-only patterns.

---

# **2. In-Memory Index → Performance**

### Architectural Characteristics:

* **Low Latency Queries**
* **High Throughput Reads**
* **Predictable Performance**

Keeping vectors in RAM ensures:

* Search is fast
* Distance comparison is instantaneous
* ANN structures (if added later) run at full speed

Main memory holds:

* Active vectors
* Index structures
* Metadata

---

# **3. Coordinator Pattern → Separation of Concerns**

### Architectural Characteristics:

* **Modularity**
* **Testability**
* **Maintainability**
* **Extensibility**

The Coordinator acts like a mini “query planner”:

* API layer talks ONLY to Coordinator
* Coordinator orchestrates storage + indexing
* Index and storage never talk directly

Clean boundaries = fewer bugs.

---

# **4. Metadata Registry → Correctness**

### Architectural Characteristics:

* **Consistency**
* **Correctness**
* **Fast Reload**

Metadata tracks:

* Which vectors exist
* Where they are on disk
* Which have been deleted
* What offsets point to

This prevents:

* Ghost reads
* Bad offsets
* Searching deleted items

Reload is fast because metadata reconstructs RAM state quickly.

---

# **5. Logical Delete + Compaction → Performance + Simplicity**

### Architectural Characteristics:

* **Write Performance**
* **Concurrency Friendly**
* **Crash Resilience**

Instead of rewriting old vectors:

* Mark as deleted (fast)
* Compact later in the background

This avoids locking the whole DB, and is exactly how:

* ElasticSearch
* RocksDB
* Qdrant

work internally.

---

# **6. Optional ANN Layer → Extensibility + Scale**

### Architectural Characteristics:

* **High Scalability**
* **Configurable performance trade-offs**
* **Future-proof**

Start with brute-force search.
Plug in HNSW/IVF later when needed.

The architecture isolates ANN from Storage and API, so you can swap it in without rewriting the DB.

---

# **7. Clear API Layer → Usability**

### Architectural Characteristics:

* **Developer Experience**
* **Clean Interface**
* **Transport Agnostic**

API exposes:

* insert
* search
* delete
* update

Cleanly separated.
You can switch between REST, gRPC, or IPC without touching internals.

---

# **UML: Sequence Diagram (Insert Operation)**

```
Client → API Layer → Coordinator → Index Manager → Storage Engine → File Store

Client:
    POST /insert
API Layer:
    Receives JSON
Coordinator:
    Generates ID
    Calls storage_engine.write_record()
Storage Engine:
    Appends record
    Returns offset
Coordinator:
    metadata.update_offset(id, offset)
    index_manager.add_vector()
API Layer:
    Returns success
```

---

# **Sequence Diagram (Search Operation)**

```
Client → API Layer → Coordinator → Index Manager

Client:
    POST /search?top_k=5
API Layer:
    Parses vector
Coordinator:
    Calls index_manager.search(query_vector)
Index Manager:
    Linear scan or ANN
    Returns top_k matches
API Layer:
    Returns JSON with IDs + distances
```

# File Structure

```
VexDB/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── main.rs            (optional CLI)
    │
    ├── storage/           (append-only segment files)
    │   ├── mod.rs
    │   ├── segment.rs
    │   ├── wal.rs         (optional write-ahead-log)
    │   └── compaction.rs
    │
    ├── index/             (search structures)
    │   ├── mod.rs
    │   ├── flat.rs        (brute-force index)
    │   └── hnsw.rs        (extend later)
    │
    ├── models/            (core data types)
    │   ├── mod.rs
    │   ├── vector.rs
    │   └── record.rs
    │
    ├── engine/            (coordination of storage + index)
    │   ├── mod.rs
    │   └── db.rs
    │
    ├── api/               (user-facing layer)
    │   ├── mod.rs
    │   ├── http.rs        (optional future)
    │   └── cli.rs
    │
    └── utils/
        ├── mod.rs
        └── math.rs        (dot product, cosine, L2)
```