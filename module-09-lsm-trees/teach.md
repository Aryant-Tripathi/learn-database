# Module 9: LSM Trees & Log-Structured Storage -- Core Teaching Content

## 1. The Write Amplification Problem with B-Trees

B-Trees are the workhorse of traditional databases -- PostgreSQL, MySQL/InnoDB, and Oracle all rely on them. They provide excellent **read** performance with O(log N) lookups. But they have a fundamental problem for write-heavy workloads: **write amplification**.

### What is write amplification?

Write amplification is the ratio of the actual bytes written to disk versus the logical bytes the application intended to write. When you update a single 100-byte row in a B-Tree:

1. The database reads the entire 4 KB--16 KB page containing that row.
2. It modifies the row in memory.
3. It writes the **entire page** back to disk.
4. It may also write a WAL record.
5. If the page splits, two new pages are written plus updates to the parent.

A 100-byte logical write can easily become 32 KB or more of physical I/O. That is a write amplification factor of 320x.

```mermaid
graph TD
    A["Application: Update 100 bytes"] --> B["Read 16 KB page from disk"]
    B --> C["Modify row in memory"]
    C --> D["Write 16 KB page back to disk"]
    C --> E["Write WAL record ~200 bytes"]
    D --> F["If page splits: write 2 new pages + parent update = 48 KB more"]

    style A fill:#e1f5fe
    style D fill:#ffcdd2
    style F fill:#ffcdd2
```

### Why this matters

On SSDs, write amplification has three costs:

- **Throughput**: every unnecessary write byte consumes I/O bandwidth.
- **Latency**: random writes to B-Tree pages cannot be fully parallelized.
- **Device lifetime**: SSDs have a finite number of program/erase cycles per cell. Excessive writes wear out the drive faster.

On HDDs, the situation is even worse because random writes require physical head seeks, making each B-Tree page write extremely expensive.

---

## 2. The Big Idea: Turn Random Writes into Sequential Writes

The key insight behind Log-Structured Merge Trees (LSM Trees) is deceptively simple:

> **Never modify data in place. Instead, always append new data sequentially, and reorganize it in the background.**

Sequential writes are dramatically faster than random writes on all storage media:

| Storage Type | Random Write (4 KB) | Sequential Write (4 KB) | Ratio |
|---|---|---|---|
| HDD (7200 RPM) | ~100 IOPS | ~50 MB/s (~12,800 IOPS equiv.) | 128x |
| SATA SSD | ~30,000 IOPS | ~500 MB/s | ~16x |
| NVMe SSD | ~200,000 IOPS | ~3 GB/s | ~15x |

LSM Trees exploit this gap. Instead of updating pages in-place like B-Trees, they:

1. Buffer writes in memory (fast).
2. Flush the buffer to disk as a sorted, immutable file (sequential write).
3. Merge and reorganize files in the background (sequential reads + writes).

Patrick O'Neil et al. introduced this idea in the 1996 paper *"The Log-Structured Merge-Tree (LSM-Tree)."*

---

## 3. LSM-Tree Architecture Overview

An LSM-Tree organizes data across multiple layers, from fast memory to slower disk. Each layer is larger than the previous one, typically by a factor of 10.

```mermaid
graph TB
    subgraph Memory
        WAL["Write-Ahead Log (WAL)<br/>On disk, sequential append"]
        MT["Active MemTable<br/>(Skip List / Red-Black Tree)<br/>~64 MB"]
        IMT["Immutable MemTable<br/>(Being flushed to disk)"]
    end

    subgraph "Disk -- Level 0"
        L0_1["SSTable 0-1"]
        L0_2["SSTable 0-2"]
        L0_3["SSTable 0-3"]
        L0_4["SSTable 0-4"]
    end

    subgraph "Disk -- Level 1 (10x size)"
        L1_1["SSTable 1-1"]
        L1_2["SSTable 1-2"]
        L1_3["SSTable 1-3"]
        L1_4["SSTable 1-4"]
        L1_5["SSTable 1-5"]
    end

    subgraph "Disk -- Level 2 (100x size)"
        L2_1["SSTable 2-1"]
        L2_2["SSTable 2-2"]
        L2_3["SSTable 2-3"]
        L2_4["SSTable 2-4"]
        L2_5["SSTable 2-5"]
        L2_6["SSTable 2-6"]
        L2_7["SSTable 2-7"]
    end

    MT --> IMT
    IMT --> L0_1
    L0_1 --> L1_1
    L1_1 --> L2_1

    style MT fill:#c8e6c9
    style IMT fill:#fff9c4
    style WAL fill:#ffccbc
```

### The layers explained

| Component | Location | Sorted? | Mutable? | Typical Size |
|---|---|---|---|---|
| WAL | Disk (sequential) | No (append-only) | Append-only | Same as MemTable |
| Active MemTable | Memory | Yes | Yes | 64--256 MB |
| Immutable MemTable | Memory | Yes | No | 64--256 MB |
| Level 0 SSTables | Disk | Yes (within file), overlapping between files | No | ~256 MB total |
| Level 1 SSTables | Disk | Yes, non-overlapping | No | ~2.56 GB total |
| Level 2 SSTables | Disk | Yes, non-overlapping | No | ~25.6 GB total |
| Level N SSTables | Disk | Yes, non-overlapping | No | 10^N * Level0 |

**Critical property**: At Level 1 and beyond, SSTables within the same level have **non-overlapping key ranges**. Level 0 is the exception -- its files can have overlapping ranges because they are direct flushes from MemTables.

---

## 4. MemTable: The In-Memory Buffer

The MemTable is an in-memory sorted data structure that buffers recent writes. It must support:

- **O(log N) insert**: fast writes
- **O(log N) point lookup**: find a key
- **O(N) ordered iteration**: for flushing to an SSTable in sorted order

### Common MemTable implementations

**Skip List** (used by LevelDB, RocksDB default):
- Probabilistic data structure that provides O(log N) average for insert, delete, and lookup.
- Lock-free variants exist for concurrent access (RocksDB uses a lock-free skip list).
- Cache-friendly due to sequential node traversal.

**Red-Black Tree**:
- Self-balancing BST with O(log N) worst-case for all operations.
- Used in some implementations but less common due to pointer-heavy structure.

**Hash Skip List** (RocksDB option):
- Optimized for prefix-based lookups.

```mermaid
graph TD
    subgraph "Skip List MemTable"
        H["Head"] --> |"Level 3"| N5["Key: dog"]
        H --> |"Level 2"| N2["Key: bat"]
        H --> |"Level 1"| N1["Key: ant"]
        N1 --> |"Level 1"| N2
        N2 --> |"Level 1"| N3["Key: cat"]
        N2 --> |"Level 2"| N5
        N3 --> |"Level 1"| N4["Key: cow"]
        N4 --> |"Level 1"| N5
        N5 --> |"Level 1"| N6["Key: fox"]
        N5 --> |"Level 3"| T["Tail"]
        N5 --> |"Level 2"| T
        N6 --> |"Level 1"| T
    end

    style H fill:#e1f5fe
    style T fill:#e1f5fe
```

---

## 5. The Write Path

Writing to an LSM-Tree is a two-step process that is intentionally simple and fast.

```mermaid
sequenceDiagram
    participant Client
    participant WAL as WAL (Disk)
    participant MT as Active MemTable
    participant IMT as Immutable MemTable
    participant L0 as Level 0 SSTables

    Client->>WAL: 1. Append (key, value) to WAL
    WAL-->>Client: fsync / group commit
    Client->>MT: 2. Insert (key, value) into MemTable
    MT-->>Client: Write acknowledged

    Note over MT: MemTable reaches size threshold (e.g., 64 MB)

    MT->>IMT: 3. Convert active MemTable to immutable
    Note over MT: Create new empty active MemTable + new WAL

    IMT->>L0: 4. Background: flush immutable MemTable to SSTable
    Note over L0: Delete old WAL after flush completes
```

### Step by step

1. **Write to WAL**: The key-value pair is appended to the Write-Ahead Log on disk. This is a sequential append, so it is very fast. The WAL ensures durability -- if the process crashes, the MemTable can be rebuilt by replaying the WAL.

2. **Insert into MemTable**: The key-value pair is inserted into the in-memory sorted structure. This is an O(log N) operation.

3. **MemTable rotation**: When the active MemTable reaches a size threshold (typically 64 MB), it is marked as **immutable** (no more writes accepted), and a new empty MemTable + WAL is created. The application continues writing to the new MemTable without blocking.

4. **Flush to SSTable**: A background thread writes the immutable MemTable to disk as a new Level 0 SSTable. Since the MemTable is already sorted, this is a simple sequential write. Once the flush is complete, the corresponding WAL file is deleted.

**Write performance**: Steps 1 and 2 are all that the client waits for. A WAL append + memory insert is dramatically faster than a B-Tree page read-modify-write cycle. This is why LSM Trees excel at write-heavy workloads.

---

## 6. The Read Path

Reading from an LSM-Tree is more complex than reading from a B-Tree because data may be spread across multiple levels.

```mermaid
flowchart TD
    A["Get(key)"] --> B{"Key in Active<br/>MemTable?"}
    B -->|Yes| C["Return value"]
    B -->|No| D{"Key in Immutable<br/>MemTable?"}
    D -->|Yes| C
    D -->|No| E{"Check Level 0<br/>SSTables<br/>(newest first)"}
    E -->|Found| C
    E -->|Not in L0| F{"Check Level 1<br/>SSTables<br/>(binary search on key range)"}
    F -->|Found| C
    F -->|Not in L1| G{"Check Level 2<br/>SSTables"}
    G -->|Found| C
    G -->|Not in L2| H["...Check deeper levels..."]
    H -->|Found| C
    H -->|Not found anywhere| I["Return NOT FOUND"]

    style A fill:#e1f5fe
    style C fill:#c8e6c9
    style I fill:#ffcdd2
```

### Read path details

1. **Check Active MemTable**: O(log N) lookup in the skip list. If found, return immediately.

2. **Check Immutable MemTable(s)**: Same as above. There may be 0--2 immutable MemTables awaiting flush.

3. **Check Level 0**: Each L0 SSTable may contain the key (their ranges overlap). Check each one from newest to oldest. For each SSTable:
   - Check the **Bloom filter** first (if key is definitely not present, skip this file).
   - If Bloom filter says "maybe," look up the **index block** to find the data block.
   - Read and search the data block.

4. **Check Level 1+**: Since SSTables at these levels have non-overlapping key ranges, binary search determines which single SSTable could contain the key. Then check its Bloom filter and index.

5. **Return the first match found** (from the newest source), or NOT FOUND.

### Why reads can be slow

In the worst case (key does not exist), the read path must check every level. With 7 levels, this could mean checking 7+ files. This is the fundamental trade-off of LSM Trees: **writes are fast, reads can be slower**.

Bloom filters are critical for mitigating this. A Bloom filter can tell you with certainty that a key is **not** in an SSTable, avoiding unnecessary disk reads.

---

## 7. SSTable (Sorted String Table) Format

An SSTable is an immutable file containing a sorted sequence of key-value pairs. The format is designed for efficient sequential writes and efficient point/range reads.

```mermaid
graph TD
    subgraph "SSTable File Layout"
        DB1["Data Block 1<br/>Keys: aaa -- app"]
        DB2["Data Block 2<br/>Keys: art -- bat"]
        DB3["Data Block 3<br/>Keys: bee -- cat"]
        DB4["Data Block N<br/>Keys: ... -- zzz"]
        MB["Meta Block<br/>(Bloom Filter)"]
        MIB["Meta Index Block<br/>(Offset to Meta Block)"]
        IB["Index Block<br/>(Key + Offset for each Data Block)"]
        FT["Footer<br/>(48 bytes: offsets to Index & Meta Index blocks)"]
    end

    DB1 --- DB2 --- DB3 --- DB4 --- MB --- MIB --- IB --- FT

    style DB1 fill:#e1f5fe
    style DB2 fill:#e1f5fe
    style DB3 fill:#e1f5fe
    style DB4 fill:#e1f5fe
    style MB fill:#fff9c4
    style IB fill:#c8e6c9
    style FT fill:#ffccbc
```

### Data Block

Each data block (typically 4 KB) contains a sorted sequence of key-value entries using **prefix compression** (also called delta encoding):

```
| Shared key length | Unshared key length | Value length | Unshared key bytes | Value bytes |
```

For example, if consecutive keys are "application" and "approximate":
- Shared prefix: "app" (3 bytes)
- Entry for "approximate": shared=3, unshared=8, key="roximate"

Every N entries (the **restart interval**, typically 16), a full key is stored without prefix compression to allow binary search within the block.

### Index Block

The index block contains one entry per data block. Each entry holds:
- A key >= the last key in the corresponding data block and < the first key in the next block.
- The offset and size of the data block.

This enables binary search across data blocks.

### Filter Block (Bloom Filter)

Contains a Bloom filter for the keys in this SSTable. This allows quickly rejecting point lookups for keys that are not present without reading any data blocks.

### Footer

A fixed-size (48 bytes in LevelDB) structure at the end of the file containing:
- The offset and size of the meta index block.
- The offset and size of the index block.
- A magic number for format verification.

---

## 8. Compaction: Why and What

Over time, the LSM Tree accumulates many SSTables. Without maintenance:
- **Read amplification** increases (more files to check).
- **Space amplification** increases (old versions of updated/deleted keys waste space).
- Level 0 can grow without bound.

**Compaction** is the background process that merges SSTables, removes obsolete entries, and pushes data to deeper levels.

```mermaid
graph LR
    subgraph "Before Compaction"
        A1["L0: {a:1, c:3, e:5}"]
        A2["L0: {b:2, c:4, d:6}"]
        A3["L1: {a:0, b:1, d:3, f:7}"]
    end

    subgraph "After Compaction"
        B1["L1: {a:1, b:2, c:4, d:6, e:5, f:7}"]
    end

    A1 --> B1
    A2 --> B1
    A3 --> B1

    style A1 fill:#ffcdd2
    style A2 fill:#ffcdd2
    style A3 fill:#fff9c4
    style B1 fill:#c8e6c9
```

### What compaction does

1. **Selects input files**: chooses SSTables from one or two adjacent levels.
2. **Merge-sorts**: reads all selected files in key order, producing a single sorted stream.
3. **Drops obsolete entries**: if multiple versions of a key exist, keeps only the newest. If a tombstone (delete marker) exists and all older versions are in the merge, drops the key entirely.
4. **Writes new SSTables**: outputs sorted, non-overlapping SSTables at the target level.
5. **Atomically swaps**: updates the LSM metadata (MANIFEST file) to point to the new files and deletes the old input files.

---

## 9. Compaction Strategies

Different compaction strategies optimize for different workload characteristics. The choice of strategy is one of the most impactful configuration decisions for an LSM-based system.

### 9.1 Leveled Compaction (LCS)

Used by LevelDB and RocksDB (default). Each level has a size limit. When a level exceeds its limit, one SSTable is picked and merged with overlapping SSTables in the next level.

```mermaid
graph TD
    subgraph "Leveled Compaction"
        L0["Level 0 (max 4 files)<br/>Overlapping ranges"]
        L1["Level 1 (max 10 MB)<br/>Non-overlapping"]
        L2["Level 2 (max 100 MB)<br/>Non-overlapping"]
        L3["Level 3 (max 1 GB)<br/>Non-overlapping"]
    end

    L0 -->|"Pick file,<br/>merge with<br/>overlapping L1 files"| L1
    L1 -->|"Pick file,<br/>merge with<br/>overlapping L2 files"| L2
    L2 -->|"Pick file,<br/>merge with<br/>overlapping L3 files"| L3

    style L0 fill:#ffcdd2
    style L1 fill:#fff9c4
    style L2 fill:#e1f5fe
    style L3 fill:#c8e6c9
```

**Properties**:
- Low space amplification (~1.1x).
- Low read amplification (at most one file per level to check).
- High write amplification (a key may be rewritten 10x per level transition).
- Best for: read-heavy workloads, limited disk space.

### 9.2 Size-Tiered Compaction (STCS)

Used by Cassandra (default), HBase, and ScyllaDB. SSTables of similar size are grouped and merged together.

**Properties**:
- Low write amplification.
- Higher space amplification (up to 2x during compaction).
- Higher read amplification (more files to check).
- Best for: write-heavy workloads.

### 9.3 FIFO Compaction

Simply drops the oldest SSTable when total size exceeds a threshold. No merge is performed.

**Properties**:
- Minimal write amplification (almost zero -- data is written once and deleted).
- Only suitable for time-series/TTL data where old data is worthless.

### 9.4 Universal Compaction (RocksDB)

A hybrid between size-tiered and leveled. It tries to minimize write amplification while bounding space amplification. It considers the ratio of sizes between adjacent sorted runs and merges when the ratio is too large.

---

## 10. Bloom Filters: Probabilistic Membership Testing

A Bloom filter is a space-efficient probabilistic data structure that answers the question: "Is element X in this set?"

- If the Bloom filter says **NO**: the element is definitely not in the set. (No false negatives.)
- If the Bloom filter says **YES**: the element is *probably* in the set. (Possible false positives.)

```mermaid
graph TD
    subgraph "Bloom Filter: Insert 'cat'"
        K1["Hash1('cat') = 2"]
        K2["Hash2('cat') = 5"]
        K3["Hash3('cat') = 9"]

        BF["Bit Array: [0,0,1,0,0,1,0,0,0,1,0,0]"]
    end

    subgraph "Bloom Filter: Query 'dog'"
        Q1["Hash1('dog') = 1"]
        Q2["Hash2('dog') = 5"]
        Q3["Hash3('dog') = 7"]

        R["Bit 1=0 --> DEFINITELY NOT IN SET"]
    end

    K1 --> BF
    K2 --> BF
    K3 --> BF
    Q1 --> R

    style R fill:#c8e6c9
    style BF fill:#fff9c4
```

### Why Bloom filters are essential for LSM Trees

Without Bloom filters, a point lookup for a non-existent key would require reading the index block (and potentially data blocks) of every SSTable at every level. With Bloom filters:

- Each SSTable has its own Bloom filter (typically loaded into memory).
- Before reading any data from an SSTable, check its Bloom filter.
- With a 1% false positive rate (10 bits per key), 99% of unnecessary SSTable reads are eliminated.

### The math

For a Bloom filter with:
- `m` bits in the array
- `n` keys inserted
- `k` hash functions

The false positive probability is approximately:

```
FPR = (1 - e^(-kn/m))^k
```

The optimal number of hash functions is:

```
k_opt = (m/n) * ln(2) ~ 0.693 * (m/n)
```

With **10 bits per key** and 7 hash functions, the FPR is approximately **0.82%** (~1%).

---

## 11. Amplification Factors: The LSM Trade-off Triangle

Every storage engine must contend with three types of amplification:

| Amplification Type | Definition | B-Tree | LSM (Leveled) | LSM (Size-Tiered) |
|---|---|---|---|---|
| **Write** | bytes written to disk / bytes written by app | 10--30x | 10--30x | 3--5x |
| **Read** | bytes read from disk / bytes requested by app | 1x | 1--2x | 5--20x |
| **Space** | bytes on disk / bytes of actual data | 1--1.5x | 1.1x | 2--3x |

```mermaid
graph TD
    subgraph "Amplification Triangle"
        W["Write Amplification<br/>(how many times data is rewritten)"]
        R["Read Amplification<br/>(how many places to check for a read)"]
        S["Space Amplification<br/>(how much extra disk space is used)"]
    end

    W --- R
    R --- S
    S --- W

    LCS["Leveled: Low Space,<br/>Low Read, High Write"]
    STCS["Size-Tiered: Low Write,<br/>High Read, High Space"]
    BT["B-Tree: Low Read,<br/>Medium Write, Low Space"]

    style W fill:#ffcdd2
    style R fill:#e1f5fe
    style S fill:#fff9c4
```

---

## 12. The RUM Conjecture

The **RUM Conjecture** (Athanassoulis et al., 2016) formalizes the fundamental trade-off:

> An access method that provides optimal performance for any two of **R**ead overhead, **U**pdate (write) overhead, and **M**emory (space) overhead must be sub-optimal for the third.

This is analogous to the CAP theorem for distributed systems, but for storage engine design:

- **Optimize Read + Memory** (minimize reads and space) --> B-Trees. Writes are expensive (in-place updates, page splits).
- **Optimize Update + Memory** (minimize writes and space) --> Leveled LSM. Reads must check multiple levels.
- **Optimize Read + Update** (minimize reads and writes) --> Size-tiered LSM or in-memory stores. Space usage is high (multiple copies of data).

No single data structure can be optimal on all three axes.

---

## 13. LSM Trees vs. B-Trees: A Comprehensive Comparison

| Dimension | B-Tree | LSM-Tree |
|---|---|---|
| **Write throughput** | Lower (random I/O, page splits) | Higher (sequential I/O, memory buffering) |
| **Read latency** | Lower, predictable (single path from root to leaf) | Higher, variable (check multiple levels) |
| **Write amplification** | 10--30x (page rewrites) | 10--30x (leveled) or 3--5x (tiered) |
| **Space efficiency** | ~60--70% page utilization | Near 100% after compaction |
| **Concurrency** | Complex latch protocols | Simpler (immutable SSTables, only MemTable needs sync) |
| **Range scans** | Excellent (leaves are linked) | Good after compaction, poor if many levels |
| **Deletes** | Immediate (mark + reclaim) | Deferred (tombstones, reclaimed during compaction) |
| **Predictability** | Steady-state I/O | Compaction spikes can cause latency jitter |
| **Use cases** | OLTP, read-heavy, point lookups | Write-heavy, time-series, logging, analytics ingest |

### Who uses what?

**B-Tree based**: PostgreSQL, MySQL/InnoDB, SQL Server, Oracle, SQLite.

**LSM-Tree based**: RocksDB, LevelDB, Cassandra, HBase, CockroachDB (on top of RocksDB/Pebble), TiKV (TiDB's storage engine), InfluxDB, ScyllaDB, BadgerDB (Go), Pebble (Go, CockroachDB).

```mermaid
graph LR
    subgraph "B-Tree Systems"
        PG["PostgreSQL"]
        MY["MySQL/InnoDB"]
        SQ["SQLite"]
    end

    subgraph "LSM-Tree Systems"
        RDB["RocksDB"]
        LDB["LevelDB"]
        CAS["Cassandra"]
        HB["HBase"]
        CRDB["CockroachDB<br/>(Pebble)"]
        TIKV["TiKV"]
    end

    subgraph "Hybrid"
        WP["WiredTiger<br/>(MongoDB)"]
    end

    style PG fill:#e1f5fe
    style MY fill:#e1f5fe
    style SQ fill:#e1f5fe
    style RDB fill:#c8e6c9
    style LDB fill:#c8e6c9
    style CAS fill:#c8e6c9
    style HB fill:#c8e6c9
    style CRDB fill:#c8e6c9
    style TIKV fill:#c8e6c9
    style WP fill:#fff9c4
```

---

## 14. Key Takeaways

1. **LSM Trees trade read performance for write performance** by converting random writes into sequential writes.
2. The **MemTable** buffers writes in memory; the **WAL** ensures durability before flushing.
3. **SSTables** are immutable, sorted files on disk. They are organized into levels.
4. **Compaction** merges SSTables to reclaim space, remove obsolete data, and reduce read amplification.
5. **Leveled compaction** minimizes space and read amplification at the cost of higher write amplification.
6. **Size-tiered compaction** minimizes write amplification at the cost of higher space and read amplification.
7. **Bloom filters** are essential for reducing read amplification by avoiding unnecessary SSTable reads.
8. The **RUM Conjecture** tells us no data structure can be optimal for reads, writes, and memory simultaneously.
9. **Choose B-Trees** when reads dominate and latency predictability matters.
10. **Choose LSM Trees** when writes dominate and you can tolerate occasional compaction-induced latency spikes.

---

## 15. Summary Diagram: Complete LSM-Tree Data Flow

```mermaid
flowchart TB
    subgraph "Write Path"
        W["Put(key, value)"] --> WAL["Append to WAL"]
        WAL --> MEM["Insert into MemTable<br/>(Skip List)"]
        MEM --> |"Size limit reached"| IMEM["Freeze as Immutable MemTable"]
        IMEM --> |"Background flush"| L0["Write SSTable to Level 0"]
    end

    subgraph "Compaction"
        L0 --> |"Level 0 full"| C1["Compact L0 + L1 overlap"]
        C1 --> L1["Level 1 SSTables"]
        L1 --> |"Level 1 full"| C2["Compact L1 + L2 overlap"]
        C2 --> L2["Level 2 SSTables"]
        L2 --> |"Continue..."| LN["Level N"]
    end

    subgraph "Read Path"
        R["Get(key)"] --> RM["Check MemTable"]
        RM --> |"Miss"| RI["Check Immutable MemTable"]
        RI --> |"Miss"| R0["Check L0 (Bloom + Index)"]
        R0 --> |"Miss"| R1["Check L1 (Bloom + Index)"]
        R1 --> |"Miss"| R2["Check L2...LN"]
        R2 --> |"Miss"| NF["NOT FOUND"]
    end

    style W fill:#c8e6c9
    style R fill:#e1f5fe
    style NF fill:#ffcdd2
```
