# Module 9: LSM Trees & Log-Structured Storage -- Quiz Questions

## Instructions

These questions test your understanding of LSM-Tree internals, compaction strategies, Bloom filters, and amplification trade-offs. Try to answer each question before revealing the answer.

---

### Question 1: Core Concept

**What is the fundamental insight behind LSM Trees that makes them faster for writes than B-Trees?**

<details>
<summary>Answer</summary>

LSM Trees convert **random writes into sequential writes**. Instead of modifying data in-place on disk (which requires reading a page, modifying it, and writing it back -- a random I/O pattern), LSM Trees buffer writes in an in-memory sorted structure (MemTable) and periodically flush the entire buffer to disk as a sorted, immutable file (SSTable). Sequential writes are 10-100x faster than random writes on both HDDs and SSDs.

</details>

---

### Question 2: Write Path

**Describe the complete write path when a client calls `Put(key, value)` on an LSM-Tree based store. What happens if the process crashes after step 1 but before step 2?**

<details>
<summary>Answer</summary>

1. The key-value pair is appended to the **Write-Ahead Log (WAL)** on disk (sequential append + fsync).
2. The key-value pair is inserted into the in-memory **MemTable** (a sorted structure like a skip list).
3. The write is acknowledged to the client.

If the process crashes after writing to the WAL but before inserting into the MemTable, the data is safe. On recovery, the system replays the WAL to reconstruct the MemTable. This is why the WAL write must happen **before** the MemTable insert and must be durable (fsync'd or group-committed).

</details>

---

### Question 3: MemTable Rotation

**What happens when the active MemTable reaches its size threshold? Why is an "immutable MemTable" needed?**

<details>
<summary>Answer</summary>

When the active MemTable reaches its size threshold (e.g., 64 MB):
1. The active MemTable is marked as **immutable** (read-only).
2. A new empty MemTable and a new WAL file are created.
3. New writes go to the new MemTable.
4. A background thread flushes the immutable MemTable to disk as a Level 0 SSTable.
5. Once the flush completes, the old WAL is deleted.

The immutable MemTable is needed to **decouple flushing from writing**. Without it, the system would have to either block writes during the flush (unacceptable latency) or risk losing data if it writes to a MemTable that is simultaneously being flushed to disk.

</details>

---

### Question 4: Level 0 vs. Level 1+

**Why can Level 0 SSTables have overlapping key ranges, but Level 1+ SSTables cannot?**

<details>
<summary>Answer</summary>

Level 0 SSTables are **direct flushes from MemTables**. Each MemTable accumulates whatever keys were written during its lifetime, so different MemTables (and thus different L0 SSTables) can contain the same keys. Enforcing non-overlapping ranges at L0 would require merging with existing L0 files during flush, which would defeat the purpose of fast sequential writes.

Level 1+ SSTables are the output of **compaction**, which merge-sorts input files and produces output files with non-overlapping ranges. This invariant is maintained so that a point lookup at Level N only needs to check a single SSTable (found via binary search on key ranges), keeping read amplification low.

</details>

---

### Question 5: SSTable Format

**An SSTable contains four main sections. Name them and explain the purpose of each.**

<details>
<summary>Answer</summary>

1. **Data Blocks**: Contains the actual sorted key-value pairs, organized in blocks of ~4 KB. Uses prefix compression to reduce redundancy between consecutive sorted keys. Restart points every N entries enable binary search within a block.

2. **Filter Block (Meta Block)**: Contains a Bloom filter for the keys in this SSTable. Allows quickly determining that a key is NOT in the SSTable without reading any data blocks.

3. **Index Block**: Contains one entry per data block, mapping a separator key to the offset and size of the corresponding data block. Enables binary search to find the right data block for a given key.

4. **Footer**: A fixed-size structure at the end of the file containing the offsets and sizes of the index block and filter block, plus a magic number for format verification.

</details>

---

### Question 6: Read Amplification

**A database has an LSM-Tree with 7 levels (L0 through L6). L0 has 4 SSTables with overlapping ranges. In the worst case (key does not exist, no Bloom filters), how many SSTables might need to be checked for a point lookup?**

<details>
<summary>Answer</summary>

**10 SSTables** in the worst case:
- L0: all 4 SSTables must be checked (overlapping ranges).
- L1 through L6: 1 SSTable per level (non-overlapping ranges, so binary search identifies at most one candidate per level) = 6 SSTables.
- Total: 4 + 6 = 10.

This is why Bloom filters are so critical -- with a 1% false positive rate, only ~0.1 of these checks would actually result in unnecessary disk reads on average.

</details>

---

### Question 7: Bloom Filter Basics

**A Bloom filter uses 10 bits per key and 7 hash functions. If it contains 1 million keys, approximately what is the false positive rate? How much memory does the filter consume?**

<details>
<summary>Answer</summary>

- **False positive rate**: Approximately **0.82%** (~1%). The formula is: `FPR = (1 - e^(-kn/m))^k = (1 - e^(-7*1000000/10000000))^7 = (1 - e^(-0.7))^7 approximately (0.503)^7 approximately 0.0082`.

- **Memory**: 10 bits * 1,000,000 keys = 10,000,000 bits = 1,250,000 bytes = **~1.19 MB**.

This is extremely space-efficient: 1.19 MB of memory eliminates ~99% of unnecessary disk reads for a million keys.

</details>

---

### Question 8: Bloom Filter False Negatives

**Can a Bloom filter ever return a false negative (saying a key is NOT present when it actually IS present)?**

<details>
<summary>Answer</summary>

**No, never.** A Bloom filter has zero false negatives. When a key is inserted, all k hash positions are set to 1. These bits are never reset to 0 (standard Bloom filters do not support deletion). Therefore, querying an inserted key will always find all k bits set to 1 and return "maybe present."

False negatives would only occur if bits were cleared, which does not happen in a standard Bloom filter. (Counting Bloom filters support deletion but introduce the possibility of overflow, which is a different issue.)

</details>

---

### Question 9: Compaction -- Leveled

**In leveled compaction, when Level 1 exceeds its size limit, describe exactly what happens during compaction.**

<details>
<summary>Answer</summary>

1. **Pick one SSTable** from Level 1 (using round-robin or a heuristic like minimum overlap with Level 2).
2. **Identify overlapping SSTables** at Level 2: find all L2 files whose key range intersects with the selected L1 file's key range.
3. **Merge-sort** the selected L1 file with all overlapping L2 files, producing a single sorted stream.
4. During the merge, **discard obsolete entries**: keep only the newest version of each key. If a tombstone exists and no older data remains at deeper levels, drop the tombstone too.
5. **Write new SSTables** at Level 2, splitting at the target file size (e.g., 2 MB). Each output file has a non-overlapping key range.
6. **Atomically update the MANIFEST** to remove the old L1 file and old L2 files, and add the new L2 files.
7. **Delete the old SSTable files** from disk.

</details>

---

### Question 10: Write Amplification Calculation

**With leveled compaction, a size ratio of 10 between levels, and 5 levels (L0 through L4), what is the theoretical worst-case write amplification? Why is the actual write amplification typically lower?**

<details>
<summary>Answer</summary>

**Theoretical worst case**: At each level transition, a file may overlap with up to 10 files at the next level, so the merge reads/writes up to 11 files (1 input + 10 overlap). Across 4 level transitions: `1 (initial flush) + 4 * 10 = 41x` write amplification (some analyses use `1 + L * (R+1)` where R=10 and L=4, giving 45x).

**Why it is lower in practice**:
- Not every file overlaps with the full 10 files at the next level.
- Overwrites reduce the amount of data reaching deeper levels.
- Tombstones eliminate data during compaction.
- L0 compaction takes all L0 files at once and merges them with L1, which is more efficient.
- RocksDB reports typical write amplification of **10-30x** for leveled compaction.

</details>

---

### Question 11: Size-Tiered vs. Leveled

**Compare size-tiered (STCS) and leveled (LCS) compaction across write amplification, read amplification, and space amplification. When would you choose each?**

<details>
<summary>Answer</summary>

| Metric | STCS | LCS |
|---|---|---|
| Write amplification | Low (4-8x) | High (10-30x) |
| Read amplification | High (many unsorted runs) | Low (1 file per level) |
| Space amplification | High (up to 2x during compaction) | Low (~1.1x) |

**Choose STCS** when:
- The workload is **write-heavy** (e.g., logging, time-series ingestion).
- You have ample disk space and can tolerate higher read latency.
- Bulk-loading data.

**Choose LCS** when:
- The workload is **read-heavy** or mixed read/write.
- Disk space is limited and space amplification must be minimized.
- Consistent read performance is important.

</details>

---

### Question 12: Tombstones

**Why can't an LSM Tree delete a key immediately? What is a tombstone, and when is it actually removed?**

<details>
<summary>Answer</summary>

An LSM Tree cannot delete a key immediately because SSTables are **immutable** -- once written, they are never modified in place. The key-value pair to be deleted may exist in any SSTable at any level, and modifying those files would destroy the immutability guarantee that makes LSM Trees efficient.

A **tombstone** is a special marker `(key, DELETE)` written to the MemTable (and subsequently flushed to an SSTable). When a read encounters a tombstone, it returns "key not found" even if older SSTables contain the key.

The tombstone is removed during **compaction** when it meets the actual key-value pair it is covering. Specifically, it can be dropped when:
1. The compaction includes the level where the original key-value pair resides, OR
2. The tombstone has reached the **bottom level** (deepest level with data), where there are no deeper levels that could contain older versions.

</details>

---

### Question 13: RUM Conjecture

**State the RUM Conjecture. Give one example of a data structure that optimizes each pair of the three factors.**

<details>
<summary>Answer</summary>

The **RUM Conjecture** states: An access method that provides optimal performance for any two of **R**ead overhead, **U**pdate (write) overhead, and **M**emory (space) overhead must be sub-optimal for the third.

Examples:
- **Optimize Read + Memory** (sacrifice Update): **B-Tree**. Excellent read performance and moderate space usage, but writes require costly in-place page updates.
- **Optimize Update + Memory** (sacrifice Read): **Leveled LSM-Tree**. Efficient writes (sequential) and low space amplification, but reads must check multiple levels.
- **Optimize Read + Update** (sacrifice Memory): **Size-tiered LSM-Tree** or a fully **in-memory hash table**. Fast reads and writes, but high space usage due to multiple copies of data or memory-only storage.

</details>

---

### Question 14: WAL and Recovery

**If a system crashes with 3 active WAL files (one for the current MemTable, two for immutable MemTables being flushed), describe the recovery process.**

<details>
<summary>Answer</summary>

During recovery:
1. Read the MANIFEST to determine the current LSM state (which SSTables exist at which levels).
2. Identify all WAL files that correspond to MemTables that were **not** successfully flushed to SSTables.
3. Replay each WAL file in order, rebuilding the corresponding MemTable:
   - WAL 1 (oldest): rebuild MemTable, flush to L0 SSTable.
   - WAL 2: rebuild MemTable, flush to L0 SSTable.
   - WAL 3 (current): rebuild MemTable, keep in memory as the active MemTable.
4. Update the MANIFEST with the newly created L0 SSTables.
5. Delete the replayed WAL files (for the flushed ones).
6. Resume normal operation.

The key insight is that WAL files are only deleted **after** the corresponding SSTable is successfully written and the MANIFEST is updated. So any WAL file that still exists on disk corresponds to data that needs to be recovered.

</details>

---

### Question 15: Merge Operator

**What problem does RocksDB's Merge Operator solve? Give an example of a use case where it provides a significant advantage over regular Put/Get.**

<details>
<summary>Answer</summary>

The Merge Operator solves the **read-before-write** problem. Without it, operations like incrementing a counter require: `val = Get(key); val += 1; Put(key, val)`. This forces a read from the LSM Tree before writing, negating the write performance advantage.

With `Merge(key, operand)`, the application writes the operand (e.g., "+1") without reading the current value. Multiple merge operands accumulate in the LSM Tree. They are combined:
- **On read**: the Get path applies operands in order to produce the final value.
- **During compaction**: operands are collapsed into a single Put, freeing space.

**Example**: A social media app tracking view counts per post. Instead of reading the current count, incrementing, and writing back (3 operations, one involving a disk read), it calls `Merge("post:123:views", "+1")` -- a single write with no read. Millions of increments are batched and collapsed during compaction.

</details>

---

### Question 16: Compression Strategy

**Why does RocksDB recommend using no compression for L0-L1 and Zstd compression for deeper levels?**

<details>
<summary>Answer</summary>

- **L0 and L1** are frequently read and rewritten during compaction. Using no compression (or lightweight LZ4) minimizes the CPU overhead of compress/decompress cycles that happen often for these levels. L0 and L1 contain a small fraction of total data, so the space savings from compressing them is negligible.

- **Deeper levels (L4-L6)** contain the vast majority of data (Level 6 alone can hold 10x more data than all other levels combined in a balanced tree). These levels are read less frequently and compacted less often. Using Zstd, which has a higher compression ratio (2-3x vs. 1.5x for LZ4), saves significant disk space and reduces disk I/O when these levels are read. The CPU cost of decompression is amortized over the large amount of data.

</details>

---

### Question 17: Column Families

**What are column families in RocksDB? Why would you use them instead of separate RocksDB instances?**

<details>
<summary>Answer</summary>

Column families are logically separate key-value namespaces within a single RocksDB instance. Each column family has its own MemTable and set of SSTables at each level, but they share:
- The **WAL** (enabling atomic writes across column families).
- The **block cache** (reducing memory overhead).
- The **rate limiter** (coordinated I/O control).

Advantages over separate instances:
1. **Atomic cross-family writes**: A WriteBatch can atomically write to multiple CFs. Separate instances cannot provide this.
2. **Resource sharing**: One block cache, one rate limiter, one set of background threads instead of duplicates.
3. **Different tuning per CF**: metadata CF can use leveled compaction with 15 bits/key Bloom filters while the timeseries CF can use FIFO compaction.
4. **Lower overhead**: fewer file descriptors, less memory for MemTables and caches.

</details>

---

### Question 18: Write Stalls

**What causes write stalls in an LSM-Tree? Name three trigger conditions in RocksDB.**

<details>
<summary>Answer</summary>

Write stalls occur when the system **cannot accept writes at the incoming rate** because background operations (compaction, flushing) cannot keep up. If writes continue unchecked, L0 grows unboundedly, read performance degrades, and memory usage spikes.

Three trigger conditions in RocksDB:
1. **L0 file count >= `level0_slowdown_writes_trigger`** (default 20): writes are slowed (artificially delayed).
2. **L0 file count >= `level0_stop_writes_trigger`** (default 36): writes are completely blocked until compaction reduces the count.
3. **Pending compaction bytes >= `soft_pending_compaction_bytes_limit`** (default 64 GB): writes are slowed proportionally to the pending bytes.

Other conditions include: too many immutable MemTables awaiting flush, and pending compaction bytes exceeding the hard limit.

</details>

---

### Question 19: Range Scans

**How does a range scan (iterate from key A to key B) work in an LSM-Tree? Why is it more complex than in a B-Tree?**

<details>
<summary>Answer</summary>

A range scan in an LSM-Tree uses a **merge iterator** that combines sorted iterators from:
1. The active MemTable.
2. Any immutable MemTables.
3. All L0 SSTables (since they have overlapping ranges).
4. The relevant SSTable from each of L1 through LN.

The merge iterator maintains a min-heap of these iterators, always yielding the smallest key across all sources. When multiple sources have the same key, the newest version wins and older versions are skipped.

This is more complex than a B-Tree range scan because:
- A B-Tree simply follows leaf-level linked list pointers -- one sequential scan through sorted data.
- An LSM Tree must merge K sorted streams in real time, where K can be 10+ (4 L0 files + 6 levels).
- Each step of the merge requires a heap operation: O(log K) per key yielded.
- Data from different sources may be on different parts of the disk, causing random I/O.

</details>

---

### Question 20: Practical Sizing

**You need to store 10 TB of data in an LSM-Tree with leveled compaction and a size ratio of 10. How many levels do you need? What should the L1 size be?**

<details>
<summary>Answer</summary>

Working backward from 10 TB at the deepest level:
- L6: 10 TB (can hold all data)
- L5: 1 TB
- L4: 100 GB
- L3: 10 GB
- L2: 1 GB
- L1: 100 MB

So **6 levels** (L1 through L6) are needed, with L0 as the flush level.

L1 size should be ~100 MB (this is `max_bytes_for_level_base` in RocksDB). In practice, you might round to 256 MB to reduce compaction frequency.

Total "overhead" space used by non-bottom levels: 1 TB + 100 GB + 10 GB + 1 GB + 100 MB ~ 1.11 TB. So space amplification is approximately 11.1/10 = 1.11x.

</details>

---

### Question 21: B-Tree vs. LSM for Mixed Workloads

**A workload is 50% reads and 50% writes, with keys uniformly distributed. The dataset fits in 500 GB. Would you choose a B-Tree or an LSM-Tree? Justify your answer.**

<details>
<summary>Answer</summary>

This is a judgment call with reasonable arguments for both sides.

**LSM-Tree arguments**:
- 50% writes is substantial. LSM Trees will handle the write half much more efficiently.
- With Bloom filters and leveled compaction, read amplification can be kept to ~1-2 SSTable reads per query.
- Modern NVMe SSDs handle the sequential I/O pattern of compaction very well.

**B-Tree arguments**:
- 50% reads means half the workload benefits from B-Tree's predictable read latency.
- B-Trees have no compaction jitter -- latency is more predictable.
- If the workload is OLTP with strict P99 latency requirements, B-Trees may be safer.

**Likely answer**: For most modern systems, an **LSM-Tree with leveled compaction** is the better choice for a 50/50 workload because:
1. The write throughput advantage outweighs the read latency penalty when Bloom filters are used.
2. Modern hardware (NVMe SSDs, high core counts) favors LSM's parallelizable compaction.
3. Systems like RocksDB, TiKV, and CockroachDB demonstrate that LSM Trees can handle mixed workloads effectively.

However, if P99 read latency is the primary concern, a B-Tree (e.g., InnoDB) is safer.

</details>

---

### Question 22: FIFO Compaction

**When is FIFO compaction appropriate? What are its limitations?**

<details>
<summary>Answer</summary>

FIFO compaction is appropriate when:
- Data has a natural **TTL** (time-to-live) and old data can be discarded entirely.
- The workload is **append-only** (no updates or deletes to existing keys).
- Examples: monitoring metrics, log storage, session data with expiration.

Limitations:
- **No deduplication**: if a key is written multiple times, all versions are stored until the SSTable containing them is dropped.
- **No delete support**: tombstones serve no purpose since files are dropped whole.
- **Unbounded read amplification**: without merging, the number of SSTables can grow large.
- **All-or-nothing deletion**: entire SSTables are dropped based on age; you cannot selectively retain certain keys.
- **No update support**: updating a key writes a new version; the old version persists until its SSTable is dropped.

</details>

---

### Question 23: Bloom Filter Sizing

**You have an SSTable with 500,000 keys and want a false positive rate under 0.1%. How many bits per key and how many hash functions should you use?**

<details>
<summary>Answer</summary>

Using the formula `FPR = (1 - e^(-kn/m))^k` and solving for `m/n` (bits per key):

For 0.1% FPR (0.001):
- Approximately **15 bits per key** with `k = 15 * ln(2) = 10.4 ~ 10 hash functions`.
- Verification: `(1 - e^(-10 * 500000 / 7500000))^10 = (1 - e^(-0.667))^10 = (0.487)^10 = 0.00079` which is approximately 0.08%, under the 0.1% target.

Total memory: 500,000 * 15 bits = 937,500 bytes = ~916 KB per SSTable.

</details>

---

### Question 24: Concurrent Access

**How do LSM Trees handle concurrent reads and writes without complex locking?**

<details>
<summary>Answer</summary>

LSM Trees have a natural concurrency advantage due to their design:

1. **SSTables are immutable**: once written, they are never modified. Any number of readers can access them concurrently without locks. This is fundamentally different from B-Trees, where readers and writers both access mutable pages.

2. **MemTable concurrency**: the MemTable is the only mutable structure. RocksDB uses a **lock-free skip list** (based on atomic compare-and-swap operations) to allow concurrent inserts without locking. Alternatively, a mutex protects the MemTable for simpler implementations.

3. **Version snapshots**: readers acquire a reference to the current Version (the set of active SSTables). Even if compaction creates new SSTables and deletes old ones, readers continue reading from their snapshot. Reference counting ensures old SSTables are not deleted until all readers are done.

4. **Single writer (serialization)**: LevelDB serializes all writes through a single thread with a mutex. RocksDB supports **concurrent writers** via a write group protocol where one writer becomes the group leader and batches multiple writes into a single WAL append.

</details>
