# Module 6: Resources - Buffer Pool & Memory Management

## Lectures and Courses

### CMU 15-445/645: Database Systems (Andy Pavlo)
The gold standard for learning buffer pool management.

- **Lecture 05 - Buffer Pools**
  - https://15445.courses.cs.cmu.edu/fall2023/notes/05-bufferpool.pdf
  - Covers buffer pool architecture, replacement policies, and optimizations
  - Video: https://www.youtube.com/watch?v=GxGkHam4Tko (Fall 2023)

- **Lecture 06 - Memory Management + Buffer Pools (continued)**
  - Covers multiple buffer pools, pre-fetching, scan sharing, O_DIRECT
  - Video: https://www.youtube.com/watch?v=wTMCJkGU5Aw (Fall 2023)

- **Project 1 - Buffer Pool Manager**
  - https://15445.courses.cs.cmu.edu/fall2023/project1/
  - The definitive buffer pool implementation project (LRU-K replacer)
  - Reference implementation framework in C++

### CMU 15-721: Advanced Database Systems
- **Lecture 03 - Memory Management**
  - https://15721.courses.cs.cmu.edu/spring2024/slides/03-memory.pdf
  - Covers mmap debate, huge pages, NUMA, memory allocators
  - Video available on YouTube

### UC Berkeley CS 186: Introduction to Database Systems
- **Lecture on Buffer Management**
  - https://cs186berkeley.net/
  - Good alternative perspective on buffer pool fundamentals

---

## Papers

### Essential Reading

- **The LRU-K Page Replacement Algorithm for Database Disk Buffering**
  - O'Neil, O'Neil, Weikum (1993)
  - https://www.cs.cmu.edu/~christos/courses/721-resources/p297-o_neil.pdf
  - The original paper introducing LRU-K
  - Key insight: tracking last K references distinguishes hot pages from scan pages

- **2Q: A Low Overhead High Performance Buffer Management Replacement Algorithm**
  - Johnson, Shasha (1994)
  - https://www.vldb.org/conf/1994/P439.PDF
  - Simpler alternative to LRU-K with similar scan resistance

- **ARC: A Self-Tuning, Low Overhead Replacement Cache**
  - Megiddo, Modha (2003)
  - https://www.usenix.org/legacy/events/fast03/tech/full_papers/megiddo/megiddo.pdf
  - IBM's adaptive replacement algorithm balancing recency and frequency
  - Used in ZFS; patent expired in 2024

- **Are You Sure You Want to Use MMAP in Your Database Management System?**
  - Crotty, Leis, Pavlo (CIDR 2022)
  - https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf
  - Comprehensive analysis of why mmap is problematic for databases
  - Documents real bugs in MongoDB, LevelDB, SQLite, and others
  - Essential reading for anyone considering mmap for data management

### Advanced Reading

- **LIRS: An Efficient Low Inter-reference Recency Set Replacement Policy to Improve Buffer Cache Performance**
  - Jiang, Zhang (2002)
  - https://web.cse.ohio-state.edu/~zhang.574/lirs-sigmetrics-02.pdf
  - Alternative to LRU-K with low overhead

- **The Five-Minute Rule for Trading Memory for Disc Accesses and the 10 Byte Rule for Trading Memory for CPU Time**
  - Gray, Putzolu (1987)
  - http://research.microsoft.com/pubs/68614/tr-86-09.pdf
  - Classic paper on when to cache pages in memory vs re-reading from disk
  - Updated versions: Five-Minute Rule 20 Years Later (2007), and 30 Years Later (2017)

- **Managing Non-Volatile Memory in Database Systems**
  - van Renen et al. (2018)
  - https://db.in.tum.de/~leis/papers/nvm.pdf
  - How persistent memory (Intel Optane) changes buffer pool design

- **LeanStore: In-Memory Data Management beyond Main Memory**
  - Leis et al. (2018)
  - https://db.in.tum.de/~leis/papers/leanstore.pdf
  - Modern buffer pool design for NVMe SSDs

- **Umbra: A Disk-Based System with In-Memory Performance**
  - Neumann, Freitag (2020)
  - https://db.in.tum.de/~freitag/papers/p29-neumann-cidr20.pdf
  - Uses mmap with careful workarounds; interesting case study

---

## Source Code References

### PostgreSQL Buffer Pool

- **bufmgr.c** - Main buffer manager
  - https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/bufmgr.c
  - Key functions: `ReadBuffer_common()`, `BufferAlloc()`, `FlushBuffer()`

- **freelist.c** - Clock sweep replacement algorithm
  - https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/freelist.c
  - Key function: `StrategyGetBuffer()` implements the clock sweep
  - Ring buffer logic: `GetBufferFromRing()`, `StrategyRejectBuffer()`

- **buf_internals.h** - Buffer descriptor structure
  - https://github.com/postgres/postgres/blob/master/src/include/storage/buf_internals.h
  - `BufferDesc` struct with atomic state field

- **localbuf.c** - Local buffer management for temp tables
  - https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/localbuf.c

- **PostgreSQL Memory Contexts**
  - https://github.com/postgres/postgres/blob/master/src/backend/utils/mmgr/aset.c
  - `AllocSetAlloc()`, `AllocSetFree()`, `AllocSetReset()`

### MySQL/InnoDB Buffer Pool

- **buf0buf.cc** - Main buffer pool implementation
  - https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/buf/buf0buf.cc
  - `buf_page_get_gen()` - main page fetch function

- **buf0lru.cc** - LRU list management
  - https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/buf/buf0lru.cc
  - `buf_LRU_get_free_block()`, `buf_LRU_add_block()`

- **buf0flu.cc** - Flushing (dirty page write-back)
  - https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/buf/buf0flu.cc

### SQLite Buffer Pool (Pager)

- **pager.c** - SQLite's page cache implementation
  - https://github.com/sqlite/sqlite/blob/master/src/pager.c
  - Simpler than PostgreSQL/InnoDB, good for learning fundamentals

- **pcache1.c** - Default page cache (pluggable cache interface)
  - https://github.com/sqlite/sqlite/blob/master/src/pcache1.c

### Other Notable Implementations

- **BusTub (CMU 15-445 educational DB)**
  - https://github.com/cmu-db/bustub
  - `src/buffer/buffer_pool_manager.cpp`
  - `src/buffer/lru_k_replacer.cpp`
  - Educational implementation designed for clarity

- **DuckDB Buffer Manager**
  - https://github.com/duckdb/duckdb
  - `src/storage/buffer_manager.cpp`
  - Modern implementation with memory-mapped I/O and eviction

---

## Blog Posts and Articles

### Buffer Pool Internals

- **PostgreSQL Buffer Cache Explained**
  - https://www.interdb.jp/pg/pgsql08.html
  - Chapter 8 of "The Internals of PostgreSQL" - excellent visual explanations

- **Understanding PostgreSQL Buffer Pool (shared_buffers)**
  - https://postgresqlco.nf/doc/en/param/shared_buffers/
  - Practical tuning guide

- **InnoDB Buffer Pool Internals**
  - https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html
  - Official MySQL documentation on buffer pool architecture

- **How the InnoDB Buffer Pool Works**
  - https://www.percona.com/blog/2020/01/22/innodb-buffer-pool-how-it-works/
  - Percona's practical deep dive

### Memory Management

- **PostgreSQL Memory Contexts**
  - https://www.pgcon.org/2019/schedule/attachments/547_pgcon2019-memory-contexts.pdf
  - PGCon talk slides on MemoryContext internals

- **Huge Pages and PostgreSQL**
  - https://www.percona.com/blog/2018/12/20/using-huge-pages-with-postgresql/
  - Practical guide to configuring huge pages

- **NUMA and Database Performance**
  - https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/
  - Classic post on NUMA problems with MySQL

### mmap Debate

- **mmap Is Not the Answer (Andy Pavlo)**
  - Summary of the CIDR 2022 paper
  - Covers real bugs caused by mmap in database systems

- **LMDB: Lightning Memory-Mapped Database**
  - https://www.symas.com/lmdb
  - A system that successfully uses mmap (single-writer, copy-on-write B-tree)

---

## Books

- **Database Internals** by Alex Petrov (O'Reilly, 2019)
  - Chapter 5: Transaction Processing and Recovery (touches on buffer management)
  - Chapter 12: Memory Management (detailed coverage)

- **Database System Concepts** by Silberschatz, Korth, Sudarshan
  - Chapter 13: Data Storage Structures
  - Chapter 14: Indexing (buffer pool interaction with B-trees)

- **The Internals of PostgreSQL** (online book)
  - https://www.interdb.jp/pg/
  - Chapter 8: Buffer Manager - the best free resource on PostgreSQL buffer internals

- **Architecture of a Database System** by Hellerstein, Stonebraker, Hamilton
  - https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf
  - Section 4: Memory Management - architectural overview

---

## Tools and Monitoring

### PostgreSQL Buffer Pool Monitoring

```sql
-- Buffer cache hit ratio
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;

-- pg_buffercache extension: see what's in the buffer pool
CREATE EXTENSION pg_buffercache;
SELECT
    c.relname,
    count(*) AS buffers,
    round(100.0 * count(*) / (SELECT setting FROM pg_settings WHERE name='shared_buffers')::integer, 1) AS pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 20;

-- Background writer statistics
SELECT * FROM pg_stat_bgwriter;
```

### MySQL/InnoDB Buffer Pool Monitoring

```sql
-- Buffer pool status
SHOW ENGINE INNODB STATUS\G

-- Key metrics
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Hit ratio
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS hit_ratio
FROM (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_reads
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) a, (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) b;
```

### Linux Performance Tools

```bash
# Monitor page cache usage
vmstat 1

# Detailed memory info
cat /proc/meminfo

# Per-process memory map
pmap -x $(pidof postgres)

# perf for TLB misses
perf stat -e dTLB-load-misses,dTLB-store-misses -p $(pidof postgres) sleep 10

# NUMA statistics
numastat -p $(pidof postgres)
```

---

## Video Talks

- **Andy Pavlo - Buffer Pool Management (CMU 15-445)**
  - Multiple semesters available on YouTube
  - Search: "CMU 15-445 buffer pool" for the latest version

- **Tomas Vondra - PostgreSQL Buffer Cache Internals (PGConf)**
  - Deep dive into PostgreSQL's clock sweep and shared_buffers

- **Mark Callaghan - InnoDB Buffer Pool Internals**
  - Facebook's experience tuning InnoDB buffer pools at scale

- **Adam Prout - Are You Sure You Want to Use MMAP? (CIDR 2022 Talk)**
  - Video presentation of the mmap paper

---

## Related Modules

| Module | Connection to Buffer Pool |
|--------|--------------------------|
| Module 2: Storage Engines | Buffer pool sits above the disk/storage manager |
| Module 3: B-Trees & Indexing | B-tree traversal relies on buffer pool for page access |
| Module 5: Transactions | Pin/unpin semantics interact with transaction isolation |
| Module 7: WAL & Recovery | Dirty page flushing must respect WAL protocol |
| Module 8: Join Algorithms | Hash joins and sort-merge joins consume buffer pool pages |
