# Module 6: Deep Dive - Buffer Pool Internals in Real Systems

## PostgreSQL shared_buffers: How It Works Internally

PostgreSQL's buffer pool is called **shared_buffers** and lives in System V shared memory (or POSIX shared memory on modern systems). It is shared across all backend processes.

### Architecture

```mermaid
graph TD
    subgraph "PostgreSQL Shared Memory"
        subgraph "Buffer Pool (shared_buffers)"
            BD["Buffer Descriptors Array<br/>(BufferDesc[NBuffers])"]
            BA["Buffer Blocks Array<br/>(8KB pages x NBuffers)"]
            BT["Buffer Mapping Table<br/>(hash table: tag -> buf_id)"]
        end

        subgraph "Other Shared Structures"
            WAL["WAL Buffers"]
            LT["Lock Tables"]
            PA["ProcArray"]
        end
    end

    subgraph "Backend Process 1"
        B1["Local buffer for temp tables"]
    end

    subgraph "Backend Process 2"
        B2["Local buffer for temp tables"]
    end

    B1 -->|"shared memory attach"| BD
    B2 -->|"shared memory attach"| BD

    style BD fill:#ff9,stroke:#333
    style BA fill:#9f9,stroke:#333
    style BT fill:#69f,stroke:#333
```

### Buffer Descriptor (BufferDesc)

Each buffer in the pool has a descriptor containing:

```c
typedef struct BufferDesc {
    BufferTag   tag;            // (tablespace, database, relation, forknum, blocknum)
    int         buf_id;         // Index in the buffer array

    /* State bits packed into a single atomic uint32 */
    pg_atomic_uint32 state;     // Contains:
                                //   - reference count (18 bits)
                                //   - usage_count (4 bits, 0-5 for clock sweep)
                                //   - BM_LOCKED
                                //   - BM_DIRTY
                                //   - BM_VALID
                                //   - BM_TAG_VALID
                                //   - BM_IO_IN_PROGRESS
                                //   - BM_IO_ERROR

    int         wait_backend_pgprocno;  // Backend waiting for I/O
    int         freeNext;               // Link in freelist chain
    LWLock      content_lock;           // Lock for reading/writing page contents
} BufferDesc;
```

### The Buffer Mapping Table

PostgreSQL uses a **partitioned hash table** for the page table to reduce lock contention. The hash table is split into 128 partitions (NUM_BUFFER_PARTITIONS), each with its own lightweight lock.

```
Lookup path:

1. Compute BufferTag from (tablespace_oid, db_oid, rel_oid, fork, blocknum)
2. Hash the tag to find the partition: partition = hash(tag) % 128
3. Acquire LWLock on that partition (shared lock for reads)
4. Look up the tag in the hash table
5. Get the buf_id
6. Release the partition lock
7. Pin the buffer (atomically increment refcount)
```

---

## PostgreSQL's Clock Sweep Algorithm

PostgreSQL does NOT use classic LRU. Instead, it uses a **clock sweep** algorithm with a **usage count** (0 to 5). This is a generalization of the basic clock algorithm.

### How It Works

```mermaid
graph TD
    subgraph "PostgreSQL Clock Sweep"
        CH["Clock Hand<br/>(nextVictimBuffer)"]

        subgraph "Buffer Ring"
            B0["Buf 0<br/>usage=3<br/>refcount=0"]
            B1["Buf 1<br/>usage=0<br/>refcount=0"]
            B2["Buf 2<br/>usage=5<br/>refcount=2"]
            B3["Buf 3<br/>usage=1<br/>refcount=0"]
            B4["Buf 4<br/>usage=0<br/>refcount=0"]
            B5["Buf 5<br/>usage=2<br/>refcount=0"]
        end
    end

    CH -->|"current"| B0

    B0 -->|"usage>0, decrement"| B1
    B1 -->|"usage=0, refcount=0<br/>VICTIM!"| V["Evict this buffer"]
    B2 -->|"refcount>0, skip"| B3
    B3 -->|"usage>0, decrement"| B4

    style V fill:#f66,stroke:#333
    style B1 fill:#f66,stroke:#333
    style B2 fill:#9f9,stroke:#333
```

**Algorithm (from freelist.c)**:

```
StrategyGetBuffer():
    loop forever:
        buf = BufferDescriptors[nextVictimBuffer]
        nextVictimBuffer = (nextVictimBuffer + 1) % NBuffers

        // Skip pinned buffers
        if buf.refcount > 0:
            continue

        // Check usage count
        if buf.usage_count > 0:
            buf.usage_count--    // Give it a "second chance"
            continue

        // Found victim: usage_count == 0 and refcount == 0
        return buf
```

**Usage count behavior**:
- When a page is accessed: `usage_count = min(usage_count + 1, 5)`
- Maximum usage count is **BM_MAX_USAGE_COUNT = 5**
- A page with usage_count=5 gets 5 "second chances" before eviction
- This makes frequently accessed pages very sticky

### Ring Buffer Optimization

For large sequential scans, PostgreSQL uses a **ring buffer** -- a small, fixed-size subset of the buffer pool (typically 256 KB = 32 pages). The scan only uses pages within this ring, preventing buffer pool pollution.

```mermaid
graph TD
    subgraph "Ring Buffer for Sequential Scans"
        subgraph "Main Buffer Pool (e.g., 8 GB)"
            MB["Thousands of shared buffers<br/>Protected from scan pollution"]
        end

        subgraph "Ring Buffer (256 KB)"
            R0["Page N"]
            R1["Page N+1"]
            R2["Page N+2"]
            R3["Page N+3"]
            R0 --> R1 --> R2 --> R3 --> R0
        end
    end

    SQ["Sequential Scan"] --> R0
    SQ -.->|"does NOT pollute"| MB

    style MB fill:#9f9,stroke:#333
    style R0 fill:#ff9,stroke:#333
    style R1 fill:#ff9,stroke:#333
    style R2 fill:#ff9,stroke:#333
    style R3 fill:#ff9,stroke:#333
```

Ring buffers are used for:
- Sequential scans (256 KB)
- VACUUM (256 KB)
- Bulk writes / COPY (16 MB)

---

## InnoDB Buffer Pool: LRU with Young/Old Sublists

MySQL's InnoDB uses a modified LRU list split into two sublists: **young** (hot) and **old** (cold).

### Architecture

```mermaid
graph LR
    subgraph "InnoDB Buffer Pool LRU List"
        subgraph "Young Sublist (5/8 of pool)"
            Y1["MRU"] --> Y2["Page A<br/>hot"] --> Y3["Page B<br/>hot"] --> Y4["Page C<br/>hot"] --> Y5["..."]
        end

        subgraph "Midpoint"
            MP["--- midpoint ---<br/>(3/8 from tail)"]
        end

        subgraph "Old Sublist (3/8 of pool)"
            O1["Page X<br/>cold"] --> O2["Page Y<br/>cold"] --> O3["Page Z<br/>cold"] --> O4["LRU tail<br/>(evict here)"]
        end
    end

    Y5 --> MP --> O1

    NP["New page read<br/>from disk"] -->|"inserted at midpoint"| MP

    style MP fill:#ff9,stroke:#333,stroke-width:2px
    style Y1 fill:#9f9,stroke:#333
    style O4 fill:#f66,stroke:#333
```

### How It Works

1. **New pages** are inserted at the **midpoint** (head of old sublist), NOT at the MRU end
2. Pages in the old sublist are only promoted to the young sublist if they are accessed again **after a configurable delay** (`innodb_old_blocks_time`, default 1000ms)
3. This prevents a full table scan from flushing the entire buffer pool
4. Pages in the young sublist are moved to the head only if they are in the bottom 3/4 of the young sublist (to reduce list manipulation overhead)

```
InnoDB LRU Parameters:

innodb_buffer_pool_size     = Total buffer pool size (e.g., 8GB)
innodb_old_blocks_pct       = Percentage for old sublist (default 37, i.e., 3/8)
innodb_old_blocks_time      = Delay in ms before promoting to young (default 1000)
innodb_buffer_pool_instances = Number of buffer pool instances (partitioning)
```

### InnoDB Buffer Pool Instances

InnoDB supports **multiple buffer pool instances** to reduce mutex contention:

```mermaid
graph TD
    subgraph "InnoDB Multi-Instance Buffer Pool"
        H["page_id hash"] --> I0["Instance 0<br/>LRU + Free List<br/>Mutex 0"]
        H --> I1["Instance 1<br/>LRU + Free List<br/>Mutex 1"]
        H --> I2["Instance 2<br/>LRU + Free List<br/>Mutex 2"]
        H --> I3["Instance 3<br/>LRU + Free List<br/>Mutex 3"]
    end

    I0 --> D["Tablespace Files"]
    I1 --> D
    I2 --> D
    I3 --> D

    style H fill:#ff9,stroke:#333
```

Recommended: Set `innodb_buffer_pool_instances` to the number of CPU cores (up to 64) when `innodb_buffer_pool_size` >= 1 GB.

---

## Adaptive Replacement Cache (ARC) - Used in ZFS

ARC, developed at IBM, adaptively tunes itself to favor recency or frequency based on workload patterns. It is used in ZFS (the file system) but not in most databases due to patent restrictions (the patent expired in 2024).

### ARC Data Structures

```mermaid
graph TD
    subgraph "ARC Cache (capacity c)"
        subgraph "Actual Cache (size c)"
            T1["T1: Recently accessed ONCE<br/>(recency list)<br/>Target size: p"]
            T2["T2: Accessed 2+ times<br/>(frequency list)<br/>Target size: c - p"]
        end

        subgraph "Ghost Lists (metadata only)"
            B1["B1: Ghost entries evicted from T1<br/>Tracks recently evicted recency pages"]
            B2["B2: Ghost entries evicted from T2<br/>Tracks recently evicted frequency pages"]
        end
    end

    New["New page<br/>(first access)"] -->|"insert"| T1
    T1 -->|"re-access"| T2
    T1 -->|"evict (when |T1| > p)"| B1
    T2 -->|"evict (when |T2| > c-p)"| B2

    B1 -->|"HIT: increase p<br/>(grow T1, favor recency)"| T2
    B2 -->|"HIT: decrease p<br/>(grow T2, favor frequency)"| T2

    style T1 fill:#69f,stroke:#333
    style T2 fill:#f96,stroke:#333
    style B1 fill:#ddd,stroke:#333
    style B2 fill:#ddd,stroke:#333
```

### The Adaptive Parameter p

The genius of ARC is the parameter `p` that dynamically adjusts:

```
When there's a hit in B1 (a recently evicted recency page is requested again):
  -> We should have kept it! Increase p to make T1 bigger.
  -> p = min(p + max(1, |B2|/|B1|), c)

When there's a hit in B2 (a recently evicted frequency page is requested again):
  -> We should have kept it! Decrease p to make T2 bigger.
  -> p = max(p - max(1, |B1|/|B2|), 0)
```

This makes ARC self-tuning. It automatically adapts to workloads that are recency-heavy (like temporal data) or frequency-heavy (like hot records).

---

## Buffer Pool Sizing: Too Small vs Too Large

### Too Small

```
Symptoms of an undersized buffer pool:
- High buffer cache miss rate
- Excessive disk I/O (high iowait)
- Frequent evictions of hot pages
- In PostgreSQL: low pg_stat_bgwriter.buffers_alloc vs buffers_backend
- In InnoDB: Innodb_buffer_pool_reads >> Innodb_buffer_pool_read_requests
```

### Too Large

```
Symptoms of an oversized buffer pool:
- Wasted memory that could be used by the OS or other processes
- Longer checkpoint times (more dirty pages to write)
- Longer crash recovery (more dirty pages to replay)
- On NUMA systems: cross-node memory access latency
- Reduced OS page cache for WAL files and other I/O
```

### Sizing Guidelines

```
PostgreSQL:
  shared_buffers = 25% of total RAM (starting point)
  Rarely benefits from > 40% of RAM
  Rest goes to OS page cache (which helps with WAL, temp files)

MySQL/InnoDB:
  innodb_buffer_pool_size = 70-80% of total RAM (dedicated server)
  InnoDB relies less on OS page cache than PostgreSQL

General Rule:
  Monitor hit ratio. Target > 99% for OLTP workloads.
  Hit ratio = 1 - (physical_reads / logical_reads)
```

---

## Background Writer and Checkpoint Processes

Dirty pages must eventually be written to disk. Two background processes handle this.

### Background Writer (bgwriter)

Continuously scans the buffer pool and writes dirty pages to disk **before** they are needed for eviction. This ensures that when a backend needs to evict a page, it can find a clean page quickly.

### Checkpointer

Periodically writes ALL dirty pages to disk and records a checkpoint in the WAL. After a checkpoint, the database can recover by replaying WAL from the checkpoint position.

```mermaid
sequenceDiagram
    participant BE as Backend
    participant BP as Buffer Pool
    participant BG as Background Writer
    participant CP as Checkpointer
    participant D as Disk

    Note over BG: Runs continuously

    loop Every bgwriter_delay (200ms default)
        BG->>BP: Scan for dirty, unpinned pages<br/>with low usage_count
        BG->>D: Write selected dirty pages
        BG->>BP: Clear dirty flag
    end

    Note over CP: Runs periodically (checkpoint_timeout)
    CP->>BP: Identify ALL dirty pages
    CP->>D: Write ALL dirty pages (spread over time)
    CP->>D: fsync() all files
    CP->>D: Write checkpoint record to WAL
    Note over CP: Recovery can start from here

    BE->>BP: FetchPage - need to evict
    alt Clean page available
        BE->>BP: Evict clean page (instant)
    else Only dirty pages
        BE->>D: Write dirty page (backend does I/O!)
        Note over BE: This is SLOW - bgwriter should prevent this
    end

    style BG fill:#9f9,stroke:#333
    style CP fill:#69f,stroke:#333
```

### Checkpoint Flow

```mermaid
graph TD
    subgraph "Checkpoint Process"
        A["Start Checkpoint"] --> B["Identify all dirty buffers"]
        B --> C["Sort by file and block number<br/>(sequential I/O)"]
        C --> D["Write dirty pages to disk<br/>(spread over checkpoint_completion_target)"]
        D --> E["fsync() all modified files"]
        E --> F["Write checkpoint record to WAL"]
        F --> G["Delete old WAL segments<br/>(before checkpoint)"]
        G --> H["Checkpoint complete"]
    end

    style A fill:#ff9,stroke:#333
    style H fill:#9f9,stroke:#333
```

PostgreSQL configuration:
```
checkpoint_timeout = 5min          # Maximum time between checkpoints
checkpoint_completion_target = 0.9  # Spread writes over 90% of interval
max_wal_size = 1GB                 # Trigger checkpoint when WAL reaches this
```

---

## The Double Buffering Problem

When a database manages its own buffer pool AND the OS maintains a page cache, the same data can exist in memory twice.

```mermaid
graph TD
    subgraph "Double Buffering Problem"
        subgraph "User Space"
            BP["Database Buffer Pool<br/>(e.g., 8GB)<br/>Page 42 lives here"]
        end

        subgraph "Kernel Space"
            PC["OS Page Cache<br/>(e.g., 20GB)<br/>Page 42 ALSO lives here"]
        end

        subgraph "Disk"
            D["Data file<br/>Page 42 on disk"]
        end
    end

    BP -->|"read()/write()"| PC
    PC -->|"actual I/O"| D

    W["Wasted: Page 42 in<br/>memory TWICE"] -.-> BP
    W -.-> PC

    style W fill:#f66,stroke:#333
    style BP fill:#69f,stroke:#333
    style PC fill:#ff9,stroke:#333
```

### Solutions

**1. O_DIRECT**: Bypass the OS page cache entirely.
```c
fd = open("datafile", O_RDWR | O_DIRECT);
// Reads/writes go directly between user buffer and disk
// No OS page cache involvement
```

Used by: InnoDB (by default), Oracle, and others.

**2. Accept the double buffering**: PostgreSQL takes this approach.
- PostgreSQL does NOT use O_DIRECT by default
- It relies on the OS page cache as a "second level" cache
- The OS cache helps with:
  - WAL file reads during recovery
  - Temp file I/O
  - Providing a safety net for data not in shared_buffers
- Downside: some memory is wasted on duplicate copies

**3. Use mmap**: Some systems (LMDB, early MongoDB) use mmap to avoid double buffering. But this has its own problems (see teach.md).

### Why Some Databases Use O_DIRECT

```
Benefits of O_DIRECT:
+ Eliminates double buffering (saves RAM)
+ Database has full control over write ordering (critical for WAL)
+ Avoids OS readahead that might conflict with DB prefetching
+ More predictable I/O latency

Drawbacks of O_DIRECT:
- Requires aligned buffers (typically 512-byte or 4KB alignment)
- Cannot benefit from OS page cache for the data file
- More complex code
- Some filesystems have bugs with O_DIRECT
```

---

## Huge Pages and TLB Misses

### The TLB Problem

The Translation Lookaside Buffer (TLB) caches virtual-to-physical address mappings. With the default 4 KB page size, a 128 GB buffer pool requires 33 million page table entries. The TLB can only cache a few thousand entries, leading to frequent TLB misses.

```
TLB miss cost: ~10-100 nanoseconds per miss
With 4KB pages and 128GB buffer pool:
  33,554,432 page table entries
  TLB typically holds 1,024-4,096 entries
  TLB miss rate can be very high for random access patterns

With 2MB huge pages:
  65,536 page table entries (512x fewer!)
  TLB can cover much more of the buffer pool
  Significant reduction in TLB misses
```

### Configuring Huge Pages

```
Linux:
  # Reserve huge pages
  echo 4096 > /proc/sys/vm/nr_hugepages  # 4096 x 2MB = 8GB

  # PostgreSQL
  huge_pages = try  # or 'on' to require them

MySQL/InnoDB:
  large-pages = ON  # in my.cnf

  # Grant privileges
  echo "vm.nr_hugepages=4096" >> /etc/sysctl.conf
```

### Transparent Huge Pages (THP) Warning

```
WARNING: Most databases recommend DISABLING Transparent Huge Pages (THP)

Why THP is problematic for databases:
- THP causes allocation stalls when the kernel tries to defragment memory
- Latency spikes of 10-100ms are common
- khugepaged background thread competes for CPU
- Memory bloat: a single byte allocation can consume 2MB

Disable THP:
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

---

## NUMA-Aware Buffer Management

On multi-socket servers, memory access times are non-uniform. Accessing memory on the local NUMA node is faster than accessing remote memory.

```mermaid
graph TD
    subgraph "NUMA Architecture"
        subgraph "Node 0"
            CPU0["CPU 0-7"]
            MEM0["Local Memory<br/>64 GB<br/>~80ns access"]
        end

        subgraph "Node 1"
            CPU1["CPU 8-15"]
            MEM1["Local Memory<br/>64 GB<br/>~80ns access"]
        end
    end

    CPU0 --> MEM0
    CPU1 --> MEM1
    CPU0 -->|"~130ns (remote!)"| MEM1
    CPU1 -->|"~130ns (remote!)"| MEM0

    style MEM0 fill:#9f9,stroke:#333
    style MEM1 fill:#69f,stroke:#333
```

### NUMA Strategies for Databases

```
1. Interleave memory allocation (most common)
   numactl --interleave=all postgres
   - Spreads buffer pool evenly across NUMA nodes
   - Prevents one node from being a bottleneck
   - Average access time is predictable

2. NUMA-aware buffer pool partitioning
   - Assign buffer pool partitions to NUMA nodes
   - Route queries to the node holding their data
   - More complex but better locality

3. Memory binding
   numactl --membind=0 postgres
   - Force all allocation to one node
   - Only useful for small buffer pools on one node
```

### Monitoring NUMA Effects

```bash
# Check NUMA topology
numactl --hardware

# Monitor NUMA statistics
numastat -p $(pidof postgres)

# Watch for remote memory accesses
perf stat -e node-loads,node-load-misses -p $(pidof postgres)
```

---

## Putting It All Together: A Production Buffer Pool Configuration

### PostgreSQL Example (128 GB RAM Server)

```
# Buffer pool
shared_buffers = 32GB              # 25% of RAM
huge_pages = try                   # Use huge pages if available

# Background writer
bgwriter_delay = 200ms             # Wake up every 200ms
bgwriter_lru_maxpages = 100        # Write up to 100 pages per round
bgwriter_lru_multiplier = 2.0      # Write 2x the recent rate

# Checkpoints
checkpoint_timeout = 15min         # Checkpoint every 15 minutes
checkpoint_completion_target = 0.9  # Spread writes over 90% of interval
max_wal_size = 4GB                 # Allow 4GB of WAL before forced checkpoint

# Per-query memory
work_mem = 256MB                   # Per-sort/hash operation
maintenance_work_mem = 2GB         # VACUUM, CREATE INDEX
temp_buffers = 256MB               # Per-session temp table buffer

# OS settings
effective_cache_size = 96GB        # Tell planner about OS cache
```

### MySQL/InnoDB Example (128 GB RAM Server)

```
# Buffer pool
innodb_buffer_pool_size = 96G      # 75% of RAM
innodb_buffer_pool_instances = 16  # Reduce mutex contention
innodb_old_blocks_pct = 37         # Old sublist = 37% of pool
innodb_old_blocks_time = 1000      # 1 second before promoting to young

# Flushing
innodb_flush_method = O_DIRECT     # Bypass OS page cache
innodb_flush_neighbors = 0         # Disable for SSD (useful for HDD)

# Page cleaner
innodb_page_cleaners = 8           # Background page cleaner threads
innodb_lru_scan_depth = 1024       # Pages to scan per cleaner per round

# Huge pages
large-pages = ON
```

---

## Summary Table

| System | Replacement Policy | Key Feature | Double Buffer Strategy |
|--------|-------------------|-------------|----------------------|
| PostgreSQL | Clock sweep (usage_count 0-5) | Ring buffers for scans | Accept it (no O_DIRECT) |
| InnoDB | Modified LRU (young/old) | Midpoint insertion, time delay | O_DIRECT by default |
| Oracle | Touch count LRU | Multiple buffer pools by block size | O_DIRECT (async I/O) |
| SQL Server | Clock with LRU-K(2) | NUMA-aware, lazy writer | No O_DIRECT (uses OS cache strategically) |
| ZFS/ARC | Adaptive (recency+frequency) | Ghost lists, self-tuning p | N/A (filesystem, not DB) |
