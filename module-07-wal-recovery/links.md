# Module 7: Resources -- WAL & Crash Recovery

## Essential Papers

### The ARIES Paper (Must Read)

- **"ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial
  Rollbacks Using Write-Ahead Logging"**
  - Authors: C. Mohan, Don Haderle, Bruce Lindsay, Hamid Pirahesh, Peter Schwarz
  - Published: ACM Transactions on Database Systems, 1992
  - Link: [https://cs.stanford.edu/people/chr101/cs345/aries.pdf](https://cs.stanford.edu/people/christo/cs345/aries.pdf)
  - This is THE paper on database recovery. Dense but essential. Read sections 1-10 first,
    then the rest. Every concept in this module traces back to this paper.

- **"ARIES/KVL: A Key-Value Locking Method for Concurrency Control of Multiaction
  Transactions Operating on B-Tree Indexes"**
  - Authors: C. Mohan
  - Published: VLDB 1990
  - Covers how ARIES interacts with B-tree concurrency control

### Related Foundational Papers

- **"Principles of Transaction-Oriented Database Recovery"**
  - Authors: Theo Haerder, Andreas Reuter (1983)
  - Link: [https://dl.acm.org/doi/10.1145/289.291](https://dl.acm.org/doi/10.1145/289.291)
  - Introduced the ACID properties and the STEAL/NO-STEAL, FORCE/NO-FORCE taxonomy

- **"A Survey of B-Tree Logging and Recovery Techniques"**
  - Authors: Goetz Graefe (2012)
  - Covers how WAL interacts with B-tree indexes in practice

- **"The Transaction Concept: Virtues and Limitations"**
  - Author: Jim Gray (1981)
  - The original paper on transactions and recovery

---

## Textbook Chapters

- **"Database Internals" by Alex Petrov**
  - Chapter 4: Implementing B-Trees (includes WAL discussion)
  - Chapter 13: Log-Structured Storage (contrasts with WAL approach)

- **"Database System Concepts" by Silberschatz, Korth, Sudarshan (7th Edition)**
  - Chapter 16: Recovery System
  - Excellent academic treatment of ARIES with worked examples

- **"Transaction Processing: Concepts and Techniques" by Jim Gray & Andreas Reuter**
  - The definitive reference on transaction processing and recovery
  - Chapters 9-10 cover WAL and recovery in great depth

- **"Architecture of a Database System" by Hellerstein, Stonebraker, Hamilton**
  - Section 5: Storage Management and Recovery
  - Link: [http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf](http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf)
  - Free and highly recommended overview of database architecture

---

## Lecture Videos and Courses

### CMU Database Group (Andy Pavlo)

- **CMU 15-445/645: Intro to Database Systems**
  - Lecture 20: Crash Recovery -- Algorithms for Recovery and Isolation
  - Lecture 21: Crash Recovery -- ARIES
  - YouTube Playlist: [https://www.youtube.com/playlist?list=PLSE8ODhjZXjbj8BMuIrRcacnQh20hmY9g](https://www.youtube.com/playlist?list=PLSE8ODhjZXjbj8BMuIrRcacnQh20hmY9g)
  - Andy Pavlo's lectures are the gold standard for learning database internals

- **CMU 15-721: Advanced Database Systems**
  - Lecture on Recovery Protocols
  - YouTube Playlist: [https://www.youtube.com/playlist?list=PLSE8ODhjZXjYzlIK4K3p1hS_FHEW9oJr](https://www.youtube.com/playlist?list=PLSE8ODhjZXjYzlIK4K3p1hS_FHEW9oJr)

### UC Berkeley CS186

- **CS186: Introduction to Database Systems**
  - Recovery lectures cover ARIES with excellent visual examples
  - Available on YouTube

---

## Blog Posts and Articles

### WAL Deep Dives

- **"How does a relational database work?" -- Christophe Kalenzaga**
  - [http://coding-geek.com/how-databases-work/](http://coding-geek.com/how-databases-work/)
  - Excellent overview with a full section on recovery and WAL

- **"Understanding WAL in PostgreSQL" -- PostgreSQL Wiki**
  - [https://wiki.postgresql.org/wiki/WAL_Internals_Of_PostgreSQL](https://wiki.postgresql.org/wiki/WAL_Internals_Of_PostgreSQL)

- **"PostgreSQL WAL: Everything You Might Not Know"**
  - Covers segment management, full-page writes, WAL tuning

- **"Write-Ahead Log (WAL) in DBMS" -- GeeksforGeeks**
  - [https://www.geeksforgeeks.org/write-ahead-log-wal/](https://www.geeksforgeeks.org/write-ahead-log-wal/)
  - Good introductory overview

### ARIES Explained

- **"A Quick Summary of ARIES" -- Stanford CS345**
  - Concise summary of the ARIES algorithm

- **"ARIES Recovery Algorithm" -- CMU 15-445 Notes**
  - Companion notes to the lecture videos

- **"Understanding ARIES: The Gold Standard for Database Recovery"**
  - Blog walkthrough with worked examples

### Implementation Perspectives

- **"How SQLite Works" -- SQLite Documentation**
  - [https://www.sqlite.org/atomiccommit.html](https://www.sqlite.org/atomiccommit.html)
  - Detailed explanation of SQLite's atomic commit mechanism
  - Covers both rollback journal and WAL mode

- **"InnoDB Recovery: The WAL and Doublewrite Buffer"**
  - MySQL/InnoDB's approach to crash recovery

- **"Write-Ahead Logging in CockroachDB"**
  - How a distributed database uses WAL at the storage engine level (RocksDB)

---

## Source Code to Study

### PostgreSQL (C)

| File | What to Study |
|------|---------------|
| `src/backend/access/transam/xlog.c` | Core WAL logic (~15,000 lines). Start with `XLogInsertRecord()` and `CreateCheckPoint()` |
| `src/backend/access/transam/xloginsert.c` | WAL record construction. Study `XLogInsert()` |
| `src/backend/access/transam/xlogrecovery.c` | Recovery and redo logic. Study `PerformWalRecovery()` |
| `src/include/access/xlogrecord.h` | WAL record format definitions |
| `src/backend/storage/buffer/bufmgr.c` | Buffer manager -- look for WAL protocol enforcement in `FlushBuffer()` |
| `src/backend/access/transam/xlogarchive.c` | WAL archiving for PITR |

Repository: [https://github.com/postgres/postgres](https://github.com/postgres/postgres)

### SQLite (C)

| File | What to Study |
|------|---------------|
| `src/wal.c` | WAL mode implementation |
| `src/pager.c` | Pager (buffer manager) -- handles both rollback journal and WAL |
| `src/journal.c` | Rollback journal implementation |

Repository: [https://github.com/sqlite/sqlite](https://github.com/sqlite/sqlite)

### BoltDB (Go)

- A simpler codebase to study (uses copy-on-write B+ tree, no traditional WAL)
- Good contrast to understand alternative approaches
- Repository: [https://github.com/etcd-io/bbolt](https://github.com/etcd-io/bbolt)

### BusTub (C++) -- CMU Teaching Database

- Implements a simplified ARIES recovery manager
- Excellent for understanding the algorithm without production complexity
- Repository: [https://github.com/cmu-db/bustub](https://github.com/cmu-db/bustub)
- Look at `src/recovery/` directory

### SimpleDB (Java) -- MIT Teaching Database

- Another teaching database with WAL and recovery
- Repository and materials from the MIT 6.830 course

---

## Conference Talks

- **"The Internals of PostgreSQL WAL" -- PGConf talks**
  - Multiple talks at PostgreSQL conferences cover WAL internals

- **"How Databases Recover from Failure" -- Strange Loop**
  - Accessible talk on recovery concepts

- **"ARIES and Its Influence on Modern Database Recovery" -- SIGMOD retrospective**
  - Historical perspective on the impact of ARIES

---

## Tools and Utilities

### PostgreSQL WAL Tools

- **`pg_waldump`**: Decode and display WAL records in human-readable form
  ```
  pg_waldump 000000010000000000000001
  ```
  Essential for understanding what WAL records look like in practice

- **`pg_basebackup`**: Create a base backup for PITR
  ```
  pg_basebackup -D /backup -Ft -z -P
  ```

- **`pg_receivewal`**: Stream WAL from a running server (for continuous archiving)

### SQLite WAL Tools

- **`.journal_mode` pragma**: Switch between rollback and WAL mode
  ```sql
  PRAGMA journal_mode=WAL;
  ```

---

## Related Topics for Further Study

- **Logical Replication and Change Data Capture (CDC)**: Uses WAL to extract logical
  changes for replication to heterogeneous systems

- **Write-Ahead Logging in LSM Trees**: RocksDB's WAL protects the memtable before it
  is flushed to SSTable

- **Distributed Consensus and WAL**: Raft and Paxos use replicated logs that are
  conceptually similar to WAL

- **NVRAM and the Future of WAL**: Non-volatile memory may change the WAL trade-offs
  by making writes to "memory" durable without fsync

- **Asynchronous Commit**: Trading durability for performance by not waiting for fsync
  (PostgreSQL's `synchronous_commit = off`)

---

## Recommended Reading Order

For a student new to this topic, the recommended order is:

1. **Watch** CMU 15-445 Lectures 20-21 (Andy Pavlo on crash recovery)
2. **Read** the teach.md and explanation.md files in this module
3. **Read** "Architecture of a Database System" Section 5 (free PDF)
4. **Study** SQLite's atomic commit documentation (simpler system, easier to grasp)
5. **Read** the ARIES paper (sections 1-10 first, then the full paper)
6. **Study** BusTub recovery code (teaching database, clean implementation)
7. **Study** PostgreSQL's xlog.c (production system, very complex but well-commented)
8. **Build** the project in project.md
