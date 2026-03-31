# Module 9: LSM Trees & Log-Structured Storage -- Deep Dive

## 1. LevelDB Architecture in Detail

LevelDB is Google's foundational LSM-Tree implementation, written in C++ by Jeff Dean and Sanjay Ghemawat (the same pair behind MapReduce, Bigtable, and many other core Google systems). It is a single-node, embedded key-value store -- not a database server, but a library you link into your application.

### Core components

```mermaid
graph TB
    subgraph "LevelDB Architecture"
        API["DB Interface<br/>Put / Get / Delete / Iterator"]
        WL["Write-Ahead Log<br/>(*.log files)"]
        MT["MemTable<br/>(Skip List)"]
        IMT["Immutable MemTable"]
        VC["Version Control<br/>(VersionSet, VersionEdit)"]
        MAN["MANIFEST File<br/>(SSTable metadata)"]
        CUR["CURRENT File<br/>(Points to active MANIFEST)"]

        subgraph "SSTables on Disk"
            L0["Level 0<br/>(max 4 files, overlapping)"]
            L1["Level 1<br/>(max 10 MB, non-overlapping)"]
            L2["Level 2<br/>(max 100 MB)"]
            L3["Level 3<br/>(max 1 GB)"]
            L4["Level 4<br/>(max 10 GB)"]
            L5["Level 5<br/>(max 100 GB)"]
            L6["Level 6<br/>(max 1 TB)"]
        end

        TC["Table Cache<br/>(LRU cache of open SSTable file handles + index blocks)"]
        BC["Block Cache<br/>(LRU cache of recently read data blocks)"]
    end

    API --> WL
    API --> MT
    MT --> IMT
    IMT --> L0
    L0 --> L1 --> L2 --> L3 --> L4 --> L5 --> L6
    TC --> L0
    BC --> TC

    style API fill:#e1f5fe
    style MT fill:#c8e6c9
    style IMT fill:#fff9c4
```

### Key files on disk

A LevelDB database directory typically contains:

| File | Purpose |
|---|---|
| `000003.log` | Current WAL (write-ahead log) |
| `000004.ldb` | SSTable file (Level 0--6) |
| `MANIFEST-000002` | Metadata: which SSTables exist, at which level, with what key ranges |
| `CURRENT` | Text file containing the name of the active MANIFEST |
| `LOCK` | File lock preventing concurrent access by multiple processes |
| `LOG` | Human-readable operational log (compaction events, etc.) |

### Version control in LevelDB

LevelDB uses a clever **versioning** system to manage concurrent reads and compaction:

- A `Version` is a snapshot of the LSM state: which SSTables exist at each level.
- A `VersionEdit` describes a change: "add SSTable X to Level 2, remove SSTable Y from Level 1."
- The `VersionSet` maintains the current Version and applies VersionEdits.
- Readers hold a reference to a Version, so they can read from a consistent snapshot even while compaction modifies the file set.

This is essentially **MVCC for the LSM metadata itself**.

---

## 2. RocksDB Enhancements over LevelDB

RocksDB started as Facebook's fork of LevelDB in 2012. It has since diverged significantly, adding many features for production workloads.

```mermaid
graph TD
    subgraph "RocksDB Enhancements"
        direction TB
        MT["Multiple MemTable<br/>Types"]
        CF["Column Families"]
        CC["Configurable<br/>Compaction Strategies"]
        PP["Parallel Compaction<br/>& Sub-compaction"]
        MO["Merge Operator"]
        RD["Range Deletions"]
        RateLim["Rate Limiter"]
        Stats["Statistics &<br/>Perf Context"]
        TTL["TTL Support"]
        BF["Pluggable<br/>Bloom Filters"]
        Comp["Pluggable<br/>Compression"]
        DIO["Direct I/O Support"]
        TX["Transactions<br/>(Optimistic & Pessimistic)"]
        BC["Block-Based &<br/>Plain Table Format"]
        WBM["Write Buffer<br/>Manager"]
    end

    style MT fill:#c8e6c9
    style CF fill:#c8e6c9
    style CC fill:#e1f5fe
    style PP fill:#e1f5fe
    style MO fill:#fff9c4
    style TX fill:#fff9c4
```

### Key differences from LevelDB

| Feature | LevelDB | RocksDB |
|---|---|---|
| Compaction strategies | Leveled only | Leveled, Universal, FIFO |
| Column families | No | Yes |
| Concurrent compaction | Single thread | Multiple threads + sub-compaction |
| Merge operator | No | Yes (read-modify-write without read) |
| Transactions | No | Optimistic + Pessimistic |
| Compression | Snappy only | Snappy, LZ4, Zstd, Zlib, BZip2 (per-level) |
| Bloom filter | Basic | Full filter, partitioned filter, ribbon filter |
| Rate limiting | No | Yes (smooths I/O spikes) |
| Statistics | Minimal | Extensive (500+ metrics) |
| Range deletions | No | Yes (efficient bulk deletes) |
| Backup/Checkpoint | No | Built-in |

---

## 3. Leveled Compaction Deep Dive

Leveled compaction is the default in both LevelDB and RocksDB. Understanding its mechanics is critical for tuning LSM-based systems.

### How files move between levels

```mermaid
sequenceDiagram
    participant L0 as Level 0
    participant L1 as Level 1
    participant L2 as Level 2

    Note over L0: 4 SSTables accumulated<br/>Trigger compaction

    L0->>L0: Select ALL L0 files<br/>(they overlap, so must take all)
    L0->>L1: Find L1 files whose key ranges<br/>overlap with L0 files
    Note over L0,L1: Merge-sort L0 files + overlapping L1 files<br/>Write new L1 files<br/>Delete old L0 + old L1 files

    Note over L1: L1 size exceeds 10 MB limit

    L1->>L1: Select ONE L1 file<br/>(round-robin or least overlap)
    L1->>L2: Find L2 files whose key ranges<br/>overlap with selected L1 file
    Note over L1,L2: Merge-sort selected L1 file + overlapping L2 files<br/>Write new L2 files<br/>Delete old files
```

### Level size ratios

The **size ratio** between adjacent levels is a critical parameter (called `max_bytes_for_level_multiplier` in RocksDB, default 10).

With a base level size of 10 MB and a multiplier of 10:

| Level | Max Size | Max Files (at 2 MB each) |
|---|---|---|
| L0 | 4 files | 4 |
| L1 | 10 MB | 5 |
| L2 | 100 MB | 50 |
| L3 | 1 GB | 500 |
| L4 | 10 GB | 5,000 |
| L5 | 100 GB | 50,000 |
| L6 | 1 TB | 500,000 |

### Write amplification analysis

In the worst case with leveled compaction and a size ratio of R:

- Data written to Level 0: 1x (the original write).
- Compaction from Level N to Level N+1: in the worst case, a single file at Level N overlaps with R files at Level N+1, so the merge reads and writes R+1 files.
- With L levels, total write amplification is approximately: 1 + L * R.
- With R=10 and L=6: write amplification up to 61x.

In practice, the write amplification is lower because:
- Not every L_N file overlaps with R files at L_(N+1).
- L0 compaction takes all L0 files at once.
- Tombstones and overwrites reduce the data that reaches deeper levels.

RocksDB reports write amplification is typically **10--30x** for leveled compaction.

### File picking strategy

When Level N exceeds its size limit, which file to compact?

1. **Round-robin** (LevelDB): cycle through files by key range to ensure even coverage.
2. **Minimize overlap** (RocksDB option): pick the L_N file that overlaps with the fewest L_(N+1) files, to minimize the work per compaction.
3. **Coldest file** (RocksDB option): pick the least recently updated file, pushing cold data deeper faster.

---

## 4. Size-Tiered Compaction (STCS)

Size-tiered compaction groups SSTables by size and merges groups of similarly-sized files.

```mermaid
graph TD
    subgraph "Size-Tiered Compaction"
        direction TB
        T1["Tier: 4 small SSTables<br/>(~16 MB each)"]
        T2["Tier: 4 medium SSTables<br/>(~64 MB each)"]
        T3["Tier: 4 large SSTables<br/>(~256 MB each)"]
        T4["Tier: 4 huge SSTables<br/>(~1 GB each)"]
    end

    T1 -->|"Merge 4 small into<br/>1 medium"| T2
    T2 -->|"Merge 4 medium into<br/>1 large"| T3
    T3 -->|"Merge 4 large into<br/>1 huge"| T4

    style T1 fill:#c8e6c9
    style T2 fill:#e1f5fe
    style T3 fill:#fff9c4
    style T4 fill:#ffccbc
```

### When to use STCS

- **Write-heavy workloads**: STCS has much lower write amplification than leveled compaction. Each key is rewritten approximately log_T(N) times (where T is the tier size, typically 4, and N is total data size).
- **Bulk loading**: ingesting large datasets quickly.
- **Time-series data**: where reads mostly access recent data.

### STCS trade-offs

| Metric | STCS | Leveled |
|---|---|---|
| Write amplification | ~4--8x | ~10--30x |
| Read amplification | High (many files to check) | Low (one file per level) |
| Space amplification | Up to 2x (during compaction, need space for input + output) | ~1.1x |
| Temporary space spike | Can double disk usage | Controlled |

Cassandra defaults to STCS and it is one of the reasons Cassandra excels at write throughput. However, read-heavy workloads on Cassandra often benefit from switching to Leveled Compaction Strategy (LCS).

---

## 5. Universal Compaction in RocksDB

Universal compaction is RocksDB's size-tiered variant with additional controls to bound space amplification.

### Rules

Universal compaction uses a set of heuristic rules to decide when and what to compact:

1. **Space amplification trigger**: if total_size / last_sorted_run_size > max_size_amplification_percent, compact everything.
2. **Size ratio trigger**: if the ratio of the first (newest) sorted run to the second exceeds size_ratio, merge them. Extend greedily to include more runs.
3. **Sorted runs count trigger**: if the number of sorted runs exceeds level0_file_num_compaction_trigger, compact.

This gives tunable knobs between pure size-tiered (low write amp) and leveled-like (low space amp) behavior.

---

## 6. FIFO Compaction for TTL Data

FIFO compaction is the simplest strategy: when total data size exceeds a threshold, drop the oldest SSTable entirely.

```mermaid
graph LR
    subgraph "FIFO Compaction"
        SST1["SST-1 (oldest)<br/>Created: 2h ago"]
        SST2["SST-2<br/>Created: 1.5h ago"]
        SST3["SST-3<br/>Created: 1h ago"]
        SST4["SST-4<br/>Created: 30m ago"]
        SST5["SST-5 (newest)<br/>Created: 5m ago"]
    end

    SST1 -->|"Total size > limit<br/>DROP oldest"| X["Deleted"]

    style SST1 fill:#ffcdd2
    style X fill:#ffcdd2
    style SST5 fill:#c8e6c9
```

**Use cases**:
- Time-series data with a fixed retention window.
- Cache-like workloads (recent data only).
- Monitoring/metrics data.

**Advantages**: Near-zero write amplification, no merge overhead.
**Disadvantages**: Cannot handle updates or deletes. No deduplication of keys. Read amplification is unbounded.

---

## 7. Bloom Filter Math

Bloom filters are so critical to LSM performance that understanding their math helps with capacity planning.

### Structure

A Bloom filter consists of:
- A bit array of `m` bits, all initially 0.
- `k` independent hash functions, each mapping a key to one of `m` positions.

### Insert operation

To insert key X: compute h_1(X), h_2(X), ..., h_k(X) and set those bit positions to 1.

### Query operation

To check if key X is in the set: compute h_1(X), h_2(X), ..., h_k(X). If ALL corresponding bits are 1, return "probably yes." If ANY bit is 0, return "definitely no."

```mermaid
graph TD
    subgraph "Bloom Filter: 16-bit array, 3 hash functions"
        Insert["Insert keys: 'apple', 'banana'"]
        BA["Bit Array:<br/>[0,1,0,1,0,0,1,0,0,1,0,1,0,0,0,1]"]

        Q1["Query 'apple':<br/>h1=1, h2=3, h3=6<br/>Bits: 1,1,1 --> MAYBE YES"]
        Q2["Query 'cherry':<br/>h1=2, h2=5, h3=9<br/>Bits: 0,0,0 --> DEFINITELY NO"]
        Q3["Query 'date':<br/>h1=1, h2=6, h3=15<br/>Bits: 1,1,1 --> MAYBE YES<br/>(false positive!)"]
    end

    Insert --> BA
    BA --> Q1
    BA --> Q2
    BA --> Q3

    style Q1 fill:#c8e6c9
    style Q2 fill:#c8e6c9
    style Q3 fill:#ffcdd2
```

### False positive rate formula

After inserting `n` keys into an `m`-bit filter with `k` hash functions:

```
FPR = (1 - (1 - 1/m)^(kn))^k approximately (1 - e^(-kn/m))^k
```

### Optimal hash function count

For a given `m/n` (bits per key), the FPR is minimized when:

```
k = (m/n) * ln(2) approximately 0.693 * (m/n)
```

### Practical sizing guide

| Bits per key (m/n) | Optimal k | False Positive Rate |
|---|---|---|
| 5 | 3 | 9.18% |
| 8 | 6 | 2.16% |
| 10 | 7 | 0.82% |
| 12 | 8 | 0.31% |
| 15 | 10 | 0.07% |
| 20 | 14 | 0.0009% |

RocksDB defaults to **10 bits per key** with 7 hash functions, giving approximately a **1% false positive rate**. This is an excellent trade-off: 10 bits (1.25 bytes) per key is cheap, and eliminating 99% of unnecessary SSTable reads is huge.

For 1 billion keys at 10 bits/key, the Bloom filter requires only ~1.25 GB of memory.

### Full filter vs. partitioned filter

- **Full filter** (LevelDB, RocksDB default before 7.0): one Bloom filter per SSTable. The entire filter must be loaded into memory.
- **Partitioned filter** (RocksDB): the Bloom filter is split into partitions, each corresponding to a range of data blocks. Only the relevant partition needs to be loaded. Reduces memory pressure for large SSTables.
- **Ribbon filter** (RocksDB 7.0+): a more space-efficient alternative to Bloom filters using XOR-based techniques. ~30% less space for the same FPR.

---

## 8. Block Cache vs. Row Cache

```mermaid
graph TD
    subgraph "RocksDB Cache Hierarchy"
        REQ["Read Request"] --> RC{"Row Cache<br/>(optional)"}
        RC -->|"Hit"| RES["Return Value"]
        RC -->|"Miss"| BC{"Block Cache<br/>(default 8 MB)"}
        BC -->|"Hit"| RES
        BC -->|"Miss"| DISK["Read from SSTable on Disk"]
        DISK --> BC
        DISK --> RES
    end

    style RC fill:#c8e6c9
    style BC fill:#e1f5fe
    style DISK fill:#ffcdd2
```

### Block cache

- Caches uncompressed **data blocks** from SSTables.
- Shared across all column families by default.
- LRU eviction (or LRU with clock-based approximation).
- Also caches index blocks and filter blocks (configurable).
- Default size: 8 MB (far too small for production -- typically set to 30--50% of available RAM).

### Row cache

- Caches individual **key-value pairs** (the logical row).
- Only effective for point lookups (not range scans).
- Higher hit rate for hot-key workloads but higher memory overhead per entry (each entry has cache metadata).
- Not commonly used -- block cache is usually sufficient.

---

## 9. Write Stalls and Rate Limiting

One of the biggest operational challenges with LSM Trees is **write stalls** -- the system temporarily slows or stops accepting writes because compaction cannot keep up.

### Stall triggers in RocksDB

| Condition | Action |
|---|---|
| L0 file count >= `level0_slowdown_writes_trigger` (default 20) | Slow down writes |
| L0 file count >= `level0_stop_writes_trigger` (default 36) | Stop writes entirely |
| Pending compaction bytes >= `soft_pending_compaction_bytes_limit` | Slow down writes |
| Pending compaction bytes >= `hard_pending_compaction_bytes_limit` | Stop writes |
| Too many MemTables (write buffer count exceeded) | Stop writes |

### Rate limiter

RocksDB's rate limiter smooths I/O by capping the total bytes/sec that compaction and flush can write to disk. This prevents compaction from saturating the disk and starving foreground reads/writes.

```
options.rate_limiter.reset(NewGenericRateLimiter(
    100 * 1024 * 1024,  // 100 MB/s rate limit
    100 * 1000,          // refill period (100ms)
    10                   // fairness factor
));
```

---

## 10. Compression: Per-Block with Snappy/LZ4/Zstd

SSTables in RocksDB support per-data-block compression. This is critical because it reduces disk I/O (fewer bytes to read) at the cost of CPU cycles for decompression.

### Compression algorithms comparison

| Algorithm | Compression Ratio | Compress Speed | Decompress Speed | Use Case |
|---|---|---|---|---|
| None | 1.0x | - | - | In-memory / SSD with spare space |
| Snappy | 1.5--1.8x | ~500 MB/s | ~500 MB/s | Default for speed |
| LZ4 | 1.5--2.0x | ~400 MB/s | ~800 MB/s | Good default choice |
| Zstd | 2.0--3.0x | ~150 MB/s | ~400 MB/s | Best ratio with acceptable speed |
| Zlib | 2.0--2.5x | ~50 MB/s | ~300 MB/s | Legacy, avoid |

### Per-level compression

RocksDB allows different compression algorithms per level:

```cpp
options.compression_per_level = {
    kNoCompression,       // L0: no compression (fast flush)
    kNoCompression,       // L1: no compression
    kLZ4Compression,      // L2: LZ4
    kLZ4Compression,      // L3: LZ4
    kZSTD,                // L4: Zstd (deeper levels = more compression)
    kZSTD,                // L5: Zstd
    kZSTD                 // L6: Zstd
};
```

The rationale: L0 and L1 are frequently read and rewritten during compaction, so speed matters more. L5 and L6 hold the vast majority of data and are read less often, so higher compression saves significant disk space.

---

## 11. Column Families in RocksDB

Column families allow logically separating different types of data within a single RocksDB instance while sharing the WAL and block cache.

```mermaid
graph TD
    subgraph "RocksDB Instance"
        WAL["Shared WAL"]
        BC["Shared Block Cache"]

        subgraph "Column Family: default"
            MT1["MemTable"]
            SST1["SSTables L0-L6"]
        end

        subgraph "Column Family: metadata"
            MT2["MemTable"]
            SST2["SSTables L0-L6"]
        end

        subgraph "Column Family: timeseries"
            MT3["MemTable"]
            SST3["SSTables L0-L6"]
        end
    end

    WAL --> MT1
    WAL --> MT2
    WAL --> MT3
    BC --> SST1
    BC --> SST2
    BC --> SST3

    style WAL fill:#ffccbc
    style BC fill:#e1f5fe
```

### Benefits

- **Different tuning per data type**: metadata might use leveled compaction with 10 bits/key Bloom filters, while timeseries uses FIFO compaction with no Bloom filters.
- **Atomic writes across column families**: a WriteBatch can atomically write to multiple CFs.
- **Independent compaction**: each CF compacts independently.
- **Shared resources**: WAL, block cache, and rate limiter are shared, reducing overhead vs. separate DB instances.

### Use case: CockroachDB

CockroachDB uses multiple column families to separate SQL row data from MVCC metadata, allowing different compaction tuning for each.

---

## 12. Tombstones and Delete Handling

Deletes in an LSM Tree cannot simply remove data from an immutable SSTable. Instead, a special marker called a **tombstone** is written.

### How tombstones work

1. `Delete(key)` writes a tombstone record: `(key, TOMBSTONE)` to the MemTable and WAL.
2. Reads encountering a tombstone return "key not found" even if older SSTables contain the key.
3. During compaction, when a tombstone meets the actual key-value pair, both are dropped (if no older data exists at deeper levels).

### The tombstone problem

Tombstones can accumulate and cause performance issues:

- **Space**: tombstones occupy space until compacted away.
- **Read amplification**: a range scan must skip over tombstones, potentially reading many dead entries.
- **Compaction delay**: a tombstone at Level 0 cannot be dropped until it is pushed to the deepest level where the corresponding key might exist.

```mermaid
graph TD
    subgraph "Tombstone Propagation"
        T1["L0: Delete(key='user:42')"]
        T2["L1: (key='user:42', tombstone)"]
        T3["L2: (key='user:42', tombstone)"]
        T4["L3: (key='user:42', value='Alice')<br/>Tombstone meets value --> BOTH DROPPED"]
    end

    T1 -->|"Compact L0->L1"| T2
    T2 -->|"Compact L1->L2"| T3
    T3 -->|"Compact L2->L3"| T4

    style T1 fill:#ffcdd2
    style T4 fill:#c8e6c9
```

### Mitigation strategies

- **Compaction filters**: RocksDB allows custom logic to drop entries during compaction (e.g., drop keys older than a TTL).
- **Bottom-level tombstone dropping**: tombstones at the bottom level can always be dropped because there are no deeper levels.
- **Delete-triggered compaction**: RocksDB can be configured to trigger compaction when too many tombstones exist in a key range.

---

## 13. Range Deletions

Deleting millions of keys one-by-one with point tombstones is extremely expensive. RocksDB's **range deletion** feature allows efficiently deleting an entire key range.

### How range deletions work

Instead of writing one tombstone per key, a single range tombstone is written:

```
DeleteRange(start_key, end_key)
```

This writes a single record: `(start_key, end_key, RANGE_TOMBSTONE)`.

During reads, the system checks if the queried key falls within any range tombstone. During compaction, any key within the range is dropped.

### Implementation

Range tombstones are stored in a separate **range deletion block** within each SSTable (not in the data blocks). This prevents them from interfering with point lookup performance.

---

## 14. Merge Operator in RocksDB

The merge operator is one of RocksDB's most powerful features. It enables **read-modify-write without the read**.

### The problem

Consider incrementing a counter:

```
val = db.Get("counter")         // Read from disk
val = val + 1                   // Modify
db.Put("counter", val)          // Write back
```

This requires a read before the write, negating the LSM write advantage.

### The solution: Merge

```
db.Merge("counter", "+1")
```

This writes a merge operand `+1` to the MemTable without reading the current value. During compaction (or when reading), the merge operator combines operands:

```mermaid
graph LR
    subgraph "Merge Operator Example: Counter"
        P["L3: Put(counter, 100)"]
        M1["L2: Merge(counter, +5)"]
        M2["L1: Merge(counter, +3)"]
        M3["L0: Merge(counter, +1)"]
    end

    M3 -->|"Read path: combine"| R["Get(counter) = 100 + 5 + 3 + 1 = 109"]

    subgraph "After Compaction"
        CP["Put(counter, 109)"]
    end

    P --> CP
    M1 --> CP
    M2 --> CP
    M3 --> CP

    style R fill:#c8e6c9
    style CP fill:#c8e6c9
```

### Use cases

- **Counters**: increment/decrement without reading.
- **Append to list**: append elements to a list value without reading the full list.
- **Aggregation**: partial aggregates that are combined during compaction.
- **JSON patching**: apply JSON merge patches without reading the full document.

The merge operator must be **associative** (grouping does not matter): `merge(merge(a, b), c) == merge(a, merge(b, c))`.

---

## 15. Compaction Strategies Comparison

```mermaid
graph TD
    subgraph "Choosing a Compaction Strategy"
        Start["What is your workload?"]
        Start -->|"Write-heavy"| WH{"Space constraints?"}
        Start -->|"Read-heavy"| RH["Leveled Compaction"]
        Start -->|"Time-series / TTL"| TS{"Updates/deletes?"}

        WH -->|"Plenty of disk"| ST["Size-Tiered / Universal"]
        WH -->|"Limited disk"| UC["Universal with<br/>space amp limit"]

        TS -->|"No"| FIFO["FIFO Compaction"]
        TS -->|"Yes"| LCS["Leveled + TTL filter"]
    end

    style RH fill:#c8e6c9
    style ST fill:#e1f5fe
    style FIFO fill:#fff9c4
    style UC fill:#ffccbc
    style LCS fill:#c8e6c9
```

### Summary table

| Strategy | Write Amp | Read Amp | Space Amp | Best For |
|---|---|---|---|---|
| Leveled | High (10-30x) | Low (1-2 files/level) | Low (~1.1x) | Read-heavy, OLTP |
| Size-Tiered | Low (4-8x) | High (many files) | High (up to 2x) | Write-heavy, bulk load |
| Universal | Tunable | Tunable | Tunable | Flexible workloads |
| FIFO | ~0 | Very High | 1x (no redundancy) | TTL/time-series, caches |

---

## 16. Advanced: Write Buffer Manager

When multiple column families exist, each has its own MemTable. Without coordination, memory usage can spike unpredictably. The **Write Buffer Manager** sets a global memory budget across all MemTables.

When total MemTable memory exceeds the budget:
1. The largest MemTable (across all CFs) is switched to immutable and scheduled for flush.
2. If memory still exceeds the budget, additional MemTables are flushed.

This prevents OOM situations in applications with many column families.

---

## 17. Key Operational Metrics to Monitor

For any production LSM-based system, these metrics are essential:

| Metric | What It Tells You |
|---|---|
| `rocksdb.num-files-at-level{N}` | File count per level -- imbalance indicates compaction lag |
| `rocksdb.compaction-pending` | Whether compaction is falling behind |
| `rocksdb.actual-delayed-write-rate` | Current write throttling rate |
| `rocksdb.is-write-stopped` | Whether writes are completely stalled |
| `rocksdb.estimate-pending-compaction-bytes` | How much compaction work is queued |
| `rocksdb.mem-table-flush-pending` | Whether a MemTable flush is pending |
| `rocksdb.block-cache-usage` | How much of the block cache is used |
| `rocksdb.bloom-filter-useful` | How many reads were avoided by Bloom filters |

Monitoring these metrics with Prometheus/Grafana is standard practice for systems like TiKV, CockroachDB, and any RocksDB-based deployment.
