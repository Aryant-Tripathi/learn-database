# Module 11 - Capstone: Architecture Decisions for Building Your Own Database

## Introduction

Building a database from scratch requires making dozens of architectural decisions, each with far-reaching consequences. This module examines how real-world databases make these decisions, why they chose the paths they did, and how you should reason about these trade-offs when building your own.

Every database is a unique combination of choices across storage, data model, process model, memory management, concurrency, and recovery. There is no single "best" architecture -- only architectures that are better suited for specific workloads.

---

## 1. Choosing Your Storage Engine: B+Tree vs LSM-Tree

The storage engine is the heart of any database. The two dominant paradigms are B+Trees and LSM-Trees, and the choice between them shapes nearly everything else.

### B+Tree Storage Engines

B+Trees are the classic choice. They maintain sorted data in a balanced tree structure where all values live in leaf nodes connected by sibling pointers.

**Strengths:**
- Excellent read performance (O(log n) point lookups)
- Predictable latency -- no background compaction storms
- Range scans are efficient via leaf-level linked lists
- In-place updates avoid write amplification from compaction
- Mature, well-understood, battle-tested for 40+ years

**Weaknesses:**
- Random I/O for writes (each insert may touch a different page)
- Write amplification from page splits
- Space amplification from partially-filled pages (typically 50-70% fill factor)
- Fragmentation over time requires periodic reorganization

**Used by:** PostgreSQL, MySQL/InnoDB, SQLite, Oracle, SQL Server

### LSM-Tree Storage Engines

Log-Structured Merge Trees buffer writes in memory (memtable) and flush sorted runs to disk, merging them in the background through compaction.

**Strengths:**
- Sequential write I/O (append-only)
- Excellent write throughput
- Better space utilization (compressed sorted runs)
- Tunable trade-offs via compaction strategies

**Weaknesses:**
- Read amplification (must check multiple levels)
- Write amplification from compaction
- Space amplification from temporary duplicate keys
- Compaction can cause latency spikes
- More complex implementation

**Used by:** RocksDB, LevelDB, Cassandra, HBase, CockroachDB (via RocksDB/Pebble)

```mermaid
graph TD
    subgraph "B+Tree Write Path"
        W1[Write Request] --> W2[Find Leaf Page in Buffer Pool]
        W2 --> W3[Modify Page In-Place]
        W3 --> W4[Write WAL Record]
        W4 --> W5[Mark Page Dirty]
        W5 --> W6[Background Flush to Disk]
    end

    subgraph "LSM-Tree Write Path"
        L1[Write Request] --> L2[Write to WAL]
        L2 --> L3[Insert into MemTable]
        L3 --> L4{MemTable Full?}
        L4 -->|Yes| L5[Flush to L0 SSTable]
        L4 -->|No| L6[Done]
        L5 --> L7[Background Compaction]
        L7 --> L8[Merge SSTables into Lower Levels]
    end

    style W1 fill:#4CAF50,color:white
    style L1 fill:#2196F3,color:white
```

### Decision Matrix

| Factor | B+Tree | LSM-Tree |
|--------|--------|----------|
| Read latency | Low, predictable | Higher, variable |
| Write throughput | Moderate | High |
| Space efficiency | Moderate (fragmentation) | Good (after compaction) |
| Write amplification | Low-moderate | High (compaction) |
| Read amplification | Low (1 tree) | High (multiple levels) |
| Implementation complexity | Moderate | High |
| Latency predictability | High | Lower (compaction spikes) |

---

## 2. Choosing Your Data Model

### Relational Model

Tables with rows and columns, enforced schemas, SQL interface, ACID transactions.

**When to choose:** General-purpose applications, complex queries with joins, strong consistency requirements, well-defined schemas.

**Examples:** PostgreSQL, MySQL, SQLite, CockroachDB

### Key-Value Model

Simple get/put/delete interface on opaque byte strings.

**When to choose:** Caching, session storage, simple lookups, building blocks for higher-level systems.

**Examples:** RocksDB, LevelDB, Redis, etcd

### Document Model

JSON/BSON documents with flexible schemas, nested data.

**When to choose:** Rapid prototyping, semi-structured data, content management, catalogs with varied attributes.

**Examples:** MongoDB, CouchDB, FerretDB

```mermaid
graph TD
    A[Choose Data Model] --> B{Need complex joins<br/>and transactions?}
    B -->|Yes| C{Need distributed<br/>scalability?}
    B -->|No| D{Need flexible schema?}
    C -->|Yes| E[Distributed Relational<br/>CockroachDB, TiDB]
    C -->|No| F[Traditional Relational<br/>PostgreSQL, MySQL]
    D -->|Yes| G{Need nested data?}
    D -->|No| H[Key-Value<br/>RocksDB, etcd]
    G -->|Yes| I[Document<br/>MongoDB]
    G -->|No| H

    style A fill:#9C27B0,color:white
    style E fill:#4CAF50,color:white
    style F fill:#4CAF50,color:white
    style H fill:#2196F3,color:white
    style I fill:#FF9800,color:white
```

---

## 3. Process Model: Embedded vs Client-Server

### Embedded Database

The database runs inside the application process. No separate server, no IPC overhead, no network latency.

**Advantages:**
- Zero deployment complexity
- No network overhead
- Simpler operations (no separate process to manage)
- Lower total resource usage for single-application use

**Disadvantages:**
- Single-process access (or complex locking for multi-process)
- No concurrent access from multiple applications
- Application crash kills the database process
- Limited by single-machine resources

**Examples:** SQLite, DuckDB, LevelDB, RocksDB (library), LMDB

### Client-Server Database

A standalone server process that accepts connections from multiple clients over a network protocol.

**Advantages:**
- Multiple concurrent clients
- Independent lifecycle from applications
- Resource isolation
- Network-accessible

**Disadvantages:**
- Deployment complexity
- Network overhead for every query
- Connection management overhead
- More moving parts to monitor and debug

**Examples:** PostgreSQL, MySQL, MongoDB, CockroachDB

```mermaid
graph LR
    subgraph "Embedded Model"
        APP1[Application Process]
        DB1[Database Library]
        DISK1[(Disk)]
        APP1 --- DB1
        DB1 --> DISK1
    end

    subgraph "Client-Server Model"
        APP2[Client App 1]
        APP3[Client App 2]
        APP4[Client App 3]
        SERVER[Database Server Process]
        DISK2[(Disk)]
        APP2 -->|TCP/Unix Socket| SERVER
        APP3 -->|TCP/Unix Socket| SERVER
        APP4 -->|TCP/Unix Socket| SERVER
        SERVER --> DISK2
    end

    style APP1 fill:#4CAF50,color:white
    style DB1 fill:#4CAF50,color:white
    style SERVER fill:#2196F3,color:white
```

---

## 4. Memory Management Strategy

### Buffer Pool (Page-Oriented)

The database manages its own memory, maintaining a fixed-size pool of pages loaded from disk.

**How it works:**
1. Allocate a fixed-size buffer pool at startup
2. Pages are loaded from disk into buffer pool frames
3. Replacement policy (LRU, Clock, LRU-K) evicts cold pages
4. Dirty pages are flushed to disk on eviction or checkpoint
5. Pin counts prevent eviction of actively-used pages

**Advantages:** Full control over eviction policy, predictable memory usage, can optimize for database access patterns, avoids double-buffering with OS page cache.

**Used by:** PostgreSQL, MySQL/InnoDB, Oracle, SQL Server

### Memory-Mapped I/O (mmap)

Let the operating system handle page management via virtual memory.

**How it works:**
1. Map database file into virtual address space with mmap()
2. OS handles page faults -- loads pages on demand
3. OS handles eviction via its page replacement algorithm
4. Writes go through OS page cache

**Advantages:** Simpler implementation, OS handles complex page management, can exceed physical memory via virtual memory.

**Disadvantages:** No control over eviction policy, no control over flush ordering (critical for crash recovery), TLB shootdowns on multi-core, no async I/O control, SIGBUS on I/O errors.

**Used by:** SQLite (optional), LMDB, early MongoDB (abandoned it), WiredTiger (partial)

```mermaid
graph TD
    subgraph "Buffer Pool Management"
        REQ[Page Request] --> BP{Page in<br/>Buffer Pool?}
        BP -->|Yes| HIT[Return Page Pointer<br/>Increment Pin Count]
        BP -->|No| MISS[Buffer Pool Miss]
        MISS --> EVICT{Free Frame<br/>Available?}
        EVICT -->|Yes| LOAD[Load Page from Disk]
        EVICT -->|No| REPLACE[Run Replacement Policy]
        REPLACE --> DIRTY{Evicted Page<br/>Dirty?}
        DIRTY -->|Yes| FLUSH[Flush to Disk First]
        DIRTY -->|No| LOAD
        FLUSH --> LOAD
        LOAD --> HIT
    end

    subgraph "mmap Management"
        REQ2[Page Access] --> VM[Virtual Memory Access]
        VM --> PF{Page Fault?}
        PF -->|No| DONE[Return Data]
        PF -->|Yes| OS[OS Loads Page<br/>from Disk]
        OS --> DONE
    end

    style REQ fill:#4CAF50,color:white
    style REQ2 fill:#2196F3,color:white
```

---

## 5. Concurrency Model

### Process-Per-Connection (PostgreSQL Model)

Each client connection gets its own OS process via fork().

**Advantages:** Strong isolation, crash of one connection does not affect others, simple programming model.

**Disadvantages:** High memory overhead per connection, expensive context switches, limited to thousands of connections.

### Thread-Per-Connection (MySQL Model)

Each client connection gets its own OS thread within a single process.

**Advantages:** Lower overhead than processes, shared memory space, faster context switches.

**Disadvantages:** Thread safety complexity, shared memory bugs can crash entire server, still limited scalability.

### Event-Driven / Async I/O

A small number of threads handle many connections using non-blocking I/O and event loops.

**Advantages:** Handles tens of thousands of connections, low memory overhead, efficient CPU usage.

**Disadvantages:** Complex programming model (callback hell or coroutines), harder to debug, CPU-bound work can block event loop.

**Examples:** Redis (single-threaded event loop), modern PostgreSQL connection poolers (PgBouncer)

```mermaid
graph TD
    subgraph "Process-per-Connection (PostgreSQL)"
        PM[Postmaster Process]
        PM -->|fork| P1[Backend Process 1]
        PM -->|fork| P2[Backend Process 2]
        PM -->|fork| P3[Backend Process 3]
        P1 --> SM[Shared Memory<br/>Buffer Pool, WAL, etc.]
        P2 --> SM
        P3 --> SM
    end

    subgraph "Thread-per-Connection (MySQL)"
        MS[mysqld Process]
        MS --- T1[Thread 1]
        MS --- T2[Thread 2]
        MS --- T3[Thread 3]
        T1 --> HP[Heap / Shared State]
        T2 --> HP
        T3 --> HP
    end

    subgraph "Event-Driven (Redis)"
        EL[Event Loop<br/>Single Thread]
        EL --> C1[Connection 1]
        EL --> C2[Connection 2]
        EL --> C3[Connection N]
    end

    style PM fill:#4CAF50,color:white
    style MS fill:#FF9800,color:white
    style EL fill:#2196F3,color:white
```

---

## 6. Recovery Strategy

### Write-Ahead Logging (WAL) with ARIES

The dominant recovery strategy. Every modification is first recorded in a sequential log before modifying data pages.

**Key principles:**
1. **Write-Ahead:** Log record must be flushed before data page is written
2. **Redo:** Log contains enough information to redo any committed operation
3. **Undo:** Log contains enough information to undo any uncommitted operation
4. **Checkpointing:** Periodic snapshots reduce recovery time

**Used by:** PostgreSQL, MySQL/InnoDB, SQLite (in WAL mode), SQL Server, Oracle

### Shadow Paging

Instead of logging, maintain two copies of the database. Modifications go to a shadow copy; on commit, atomically swap the shadow and current copies.

**Advantages:** No WAL overhead, simpler recovery (just discard the shadow).

**Disadvantages:** High write amplification (entire pages copied), random I/O, not practical for large databases.

**Used by:** Early SQLite (rollback journal mode), System R (historical)

### Log-Structured Recovery

In LSM-tree systems, the WAL and the memtable together provide recovery. On crash, replay the WAL to reconstruct the memtable.

**Used by:** RocksDB, LevelDB, Cassandra

```mermaid
sequenceDiagram
    participant App as Application
    participant WAL as WAL (Sequential Log)
    participant BP as Buffer Pool
    participant Disk as Data Files

    App->>WAL: 1. Write log record (LSN=101)
    WAL->>WAL: 2. Flush log to disk (fsync)
    App->>BP: 3. Modify page in memory
    Note over BP: Page marked dirty<br/>pageLSN = 101
    BP->>Disk: 4. Background flush<br/>(only after WAL flushed)

    Note over App,Disk: --- CRASH ---

    WAL->>BP: 5. Recovery: Redo from checkpoint
    Note over BP: Replay all logged ops<br/>for committed txns
    WAL->>BP: 6. Undo uncommitted txns
    Note over BP: Rollback incomplete<br/>transactions
```

---

## 7. How Real Databases Make These Decisions

### SQLite: The Embedded Powerhouse

SQLite makes a unique set of choices optimized for simplicity, reliability, and embedded use.

| Decision | SQLite's Choice | Why |
|----------|----------------|-----|
| Storage Engine | B+Tree | Predictable reads, simpler implementation |
| Data Model | Relational (with dynamic typing) | SQL compatibility with flexibility |
| Process Model | Embedded library | Zero configuration, zero deployment |
| Memory Management | mmap (optional) + own page cache | Simplicity, works without tuning |
| Concurrency | File-level locking (WAL mode: readers+1 writer) | Simplicity, no server process |
| Recovery | Rollback journal OR WAL | Both options for different use cases |

**Key insight:** SQLite optimizes for simplicity and reliability over raw performance. It is the most widely deployed database in the world (billions of instances) because it just works.

```mermaid
graph TD
    subgraph "SQLite Architecture"
        API[C API Layer]
        TOK[Tokenizer]
        PARSE[Parser]
        CG[Code Generator]
        VDBE[Virtual Machine - VDBE]
        BT[B-Tree Layer]
        PGR[Pager Layer]
        OS[OS Interface - VFS]

        API --> TOK --> PARSE --> CG --> VDBE
        VDBE --> BT --> PGR --> OS
    end

    style API fill:#4CAF50,color:white
    style VDBE fill:#FF9800,color:white
    style BT fill:#2196F3,color:white
    style PGR fill:#9C27B0,color:white
```

### PostgreSQL: The Enterprise Workhorse

PostgreSQL makes choices that prioritize correctness, extensibility, and standards compliance.

| Decision | PostgreSQL's Choice | Why |
|----------|-------------------|-----|
| Storage Engine | Heap files + B+Tree indexes | Flexibility, multiple index types |
| Data Model | Relational (strongly typed) | SQL standard compliance |
| Process Model | Process-per-connection | Isolation, crash safety |
| Memory Management | Shared buffer pool | Control over eviction, no double-buffering |
| Concurrency | MVCC with snapshot isolation | High concurrency without read locks |
| Recovery | WAL with ARIES-style recovery | Industry standard, proven reliable |

**Key insight:** PostgreSQL prioritizes correctness and extensibility. Its catalog-driven design means nearly everything is a table, and new types, operators, and index methods can be added without modifying the core.

```mermaid
graph TD
    subgraph "PostgreSQL Architecture"
        CLIENT[Client Connection]
        PM[Postmaster]
        BE[Backend Process]

        CLIENT -->|connect| PM
        PM -->|fork| BE

        BE --> PARSER[Parser]
        PARSER --> REWRITE[Rewrite System]
        REWRITE --> PLANNER[Query Planner/Optimizer]
        PLANNER --> EXECUTOR[Executor]

        EXECUTOR --> AM[Access Methods<br/>HeapAM, IndexAM]
        AM --> BUFMGR[Buffer Manager]
        BUFMGR --> STORAGE[Storage Manager]

        BE --> WAL[WAL Writer]
        BE --> STATS[Stats Collector]

        BG1[BGWriter] --> BUFMGR
        BG2[Checkpointer] --> BUFMGR
        BG3[Autovacuum] --> AM
    end

    style CLIENT fill:#4CAF50,color:white
    style PM fill:#FF9800,color:white
    style EXECUTOR fill:#2196F3,color:white
    style BUFMGR fill:#9C27B0,color:white
```

### CockroachDB: Distributed SQL

CockroachDB layers a SQL database on top of a distributed key-value store.

| Decision | CockroachDB's Choice | Why |
|----------|---------------------|-----|
| Storage Engine | LSM-Tree (Pebble, a Go RocksDB clone) | High write throughput for replication |
| Data Model | Relational (SQL) over KV | Familiar SQL interface with distribution |
| Process Model | Single process, goroutines | Go's concurrency model |
| Memory Management | Go runtime + RocksDB block cache | Leverages Go GC + manual for hot data |
| Concurrency | Serializable SI (MVCC + clock skew) | Strongest isolation in distributed setting |
| Recovery | Raft consensus + WAL | Replicated state machine for consistency |

**Key insight:** CockroachDB shows that you can build a distributed SQL database by layering abstractions. SQL -> DistSQL -> KV -> Raft -> Pebble (LSM).

```mermaid
graph TD
    subgraph "CockroachDB Architecture"
        SQL[SQL Layer<br/>Parser, Planner, Executor]
        DISTSQL[DistSQL Layer<br/>Distributed Query Execution]
        TXN[Transaction Layer<br/>MVCC, Timestamp Oracle]
        DIST[Distribution Layer<br/>Range Routing, Leaseholder]
        REP[Replication Layer<br/>Raft Consensus]
        STORE[Storage Layer<br/>Pebble LSM-Tree]

        SQL --> DISTSQL --> TXN --> DIST --> REP --> STORE
    end

    subgraph "Single Range Raft Group"
        LEADER[Leader]
        F1[Follower 1]
        F2[Follower 2]
        LEADER -->|AppendEntries| F1
        LEADER -->|AppendEntries| F2
    end

    style SQL fill:#4CAF50,color:white
    style DISTSQL fill:#FF9800,color:white
    style REP fill:#2196F3,color:white
    style STORE fill:#9C27B0,color:white
```

---

## 8. Architecture of Notable Databases

### MySQL/InnoDB

MySQL separates the SQL layer from the storage engine, allowing pluggable engines. InnoDB is the default.

**Key architectural decisions:**
- Thread-per-connection model
- Pluggable storage engine API (InnoDB, MyISAM, etc.)
- InnoDB uses clustered B+Tree (primary key IS the table)
- Adaptive hash index for frequently accessed pages
- Double-write buffer for torn page protection
- Change buffer for deferred secondary index updates

### DuckDB: The Analytical Embedded Database

DuckDB is "SQLite for analytics" -- an embedded columnar database optimized for OLAP.

**Key architectural decisions:**
- Columnar storage (great for aggregations, scans)
- Vectorized push-based execution (processes vectors of ~2048 values)
- No buffer pool -- relies on OS virtual memory via mmap
- Parallel intra-query execution
- Morsel-driven parallelism

### RocksDB: The Storage Engine Building Block

RocksDB is not a full database -- it is a storage engine library used to build databases.

**Key architectural decisions:**
- LSM-Tree with tiered/leveled compaction
- Pluggable memtable formats (skiplist, hash-skiplist, vector)
- Pluggable compaction strategies
- Column families for logical separation
- Comprehensive statistics and tuning knobs
- Used as foundation by: CockroachDB, TiKV, YugabyteDB

```mermaid
graph LR
    subgraph "Database Architecture Spectrum"
        direction TB
        SIMPLE[Simple<br/>Key-Value]
        EMBEDDED[Embedded<br/>Relational]
        SERVER[Client-Server<br/>Relational]
        DISTRIBUTED[Distributed<br/>SQL]

        SIMPLE --> EMBEDDED --> SERVER --> DISTRIBUTED

        RDB[RocksDB] -.-> SIMPLE
        SQLT[SQLite] -.-> EMBEDDED
        DUCK[DuckDB] -.-> EMBEDDED
        PG[PostgreSQL] -.-> SERVER
        MY[MySQL] -.-> SERVER
        CRDB[CockroachDB] -.-> DISTRIBUTED
        TIDB[TiDB] -.-> DISTRIBUTED
    end

    style SIMPLE fill:#4CAF50,color:white
    style EMBEDDED fill:#2196F3,color:white
    style SERVER fill:#FF9800,color:white
    style DISTRIBUTED fill:#9C27B0,color:white
```

---

## 9. Trade-Off Analysis: A Systematic Framework

When building your own database, use this framework to evaluate each decision:

### The Three Amplification Factors

Every storage engine trade-off can be analyzed in terms of three amplification factors:

1. **Read Amplification:** How many I/Os per read? B+Trees: O(log n). LSM: O(L * log n) where L is number of levels.
2. **Write Amplification:** How many I/Os per write? B+Trees: ~2 (WAL + page). LSM: ~10-30x (compaction).
3. **Space Amplification:** How much extra space? B+Trees: ~1.5x (fragmentation). LSM: ~1.1-1.3x (temporary duplicates).

```mermaid
graph TD
    subgraph "RUM Conjecture: Pick Two"
        R[Read Optimal<br/>R=1]
        U[Update Optimal<br/>U=1]
        M[Memory Optimal<br/>M=1]

        R ---|B+Tree<br/>Good R, Good M<br/>Poor U| U
        U ---|LSM-Tree<br/>Good U, Good M<br/>Poor R| M
        M ---|Read-Optimized<br/>Good R, Good U<br/>Poor M| R
    end

    style R fill:#f44336,color:white
    style U fill:#4CAF50,color:white
    style M fill:#2196F3,color:white
```

### Decision Tree for Your Database

```mermaid
graph TD
    START[Start: What is your<br/>primary workload?]
    START --> OLTP{OLTP or OLAP?}

    OLTP -->|OLTP| WRITE{Write-heavy or<br/>Read-heavy?}
    OLTP -->|OLAP| COLSTORE[Columnar Storage<br/>Vectorized Execution]

    WRITE -->|Write-heavy| LSM[LSM-Tree<br/>RocksDB-style]
    WRITE -->|Read-heavy| BTREE[B+Tree<br/>InnoDB-style]
    WRITE -->|Balanced| BTREE

    LSM --> DIST{Distributed?}
    BTREE --> DIST2{Distributed?}

    DIST -->|Yes| RAFT[Add Raft/Paxos<br/>CockroachDB-style]
    DIST -->|No| SINGLE[Single-node<br/>Embedded or Server]
    DIST2 -->|Yes| RAFT
    DIST2 -->|No| SINGLE

    SINGLE --> EMB{Embedded or<br/>Client-Server?}
    EMB -->|Embedded| SQLITE_STYLE[SQLite/DuckDB Style]
    EMB -->|Server| PG_STYLE[PostgreSQL Style]

    style START fill:#9C27B0,color:white
    style COLSTORE fill:#FF9800,color:white
    style LSM fill:#4CAF50,color:white
    style BTREE fill:#2196F3,color:white
    style RAFT fill:#f44336,color:white
```

---

## 10. Component Interaction: The Full Picture

Every database, regardless of its specific choices, has the same fundamental layers. Here is how they interact:

```mermaid
graph TD
    subgraph "Client Layer"
        CL[Client / Application]
    end

    subgraph "Connection Layer"
        PROTO[Protocol Handler<br/>Wire Protocol]
        CONN[Connection Manager<br/>Auth, Session State]
    end

    subgraph "SQL Layer"
        PARSER[Parser / Tokenizer]
        BINDER[Semantic Analysis / Binder]
        OPT[Query Optimizer]
        EXEC[Query Executor]
    end

    subgraph "Transaction Layer"
        TXN[Transaction Manager]
        LOCK[Lock Manager / MVCC]
        LOG[WAL Manager]
    end

    subgraph "Access Layer"
        HEAP[Heap / Table Access]
        IDX[Index Access<br/>B+Tree, Hash, etc.]
    end

    subgraph "Storage Layer"
        BUF[Buffer Pool Manager]
        DISK[Disk Manager<br/>File I/O]
    end

    CL --> PROTO --> CONN
    CONN --> PARSER --> BINDER --> OPT --> EXEC
    EXEC --> TXN
    TXN --> LOCK
    TXN --> LOG
    EXEC --> HEAP
    EXEC --> IDX
    HEAP --> BUF
    IDX --> BUF
    LOG --> DISK
    BUF --> DISK

    style CL fill:#4CAF50,color:white
    style EXEC fill:#FF9800,color:white
    style TXN fill:#2196F3,color:white
    style BUF fill:#9C27B0,color:white
    style DISK fill:#f44336,color:white
```

---

## Summary

| Database | Storage | Data Model | Process | Memory | Concurrency | Recovery |
|----------|---------|------------|---------|--------|-------------|----------|
| SQLite | B+Tree | Relational | Embedded | mmap/cache | File locks | Journal/WAL |
| PostgreSQL | Heap+B+Tree | Relational | Process-per-conn | Buffer pool | MVCC (SI) | WAL (ARIES) |
| MySQL/InnoDB | Clustered B+Tree | Relational | Thread-per-conn | Buffer pool | MVCC (RC/RR) | WAL + doublewrite |
| CockroachDB | LSM (Pebble) | SQL over KV | Goroutines | Go + block cache | MVCC (SSI) | Raft + WAL |
| DuckDB | Columnar | Relational | Embedded | mmap | MVCC | WAL |
| RocksDB | LSM | Key-Value | Library | Block cache | Lock-free (single writer) | WAL |

**The most important lesson:** There is no universally "best" architecture. Every choice is a trade-off. The key is to understand your workload, your constraints, and which trade-offs you can accept. The best database architects are not those who know one architecture deeply, but those who can reason about trade-offs and choose the right combination of decisions for each specific problem.
