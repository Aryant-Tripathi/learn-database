# Module 5: Resources -- Transactions & Concurrency Control

## Foundational Papers

### ACID and Transaction Theory

- **"The Transaction Concept: Virtues and Limitations" -- Jim Gray (1981)**
  https://jimgray.azurewebsites.net/papers/theTransactionConcept.pdf
  The foundational paper that formalized the transaction concept and ACID properties. Essential reading for understanding why transactions exist.

- **"A Critique of ANSI SQL Isolation Levels" -- Berenson et al. (1995)**
  https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
  A landmark paper that exposed the inadequacy of the ANSI SQL isolation level definitions. Introduced Snapshot Isolation as a distinct level and identified anomalies like write skew that the standard failed to address. Required reading.

- **"Granularity of Locks and Degrees of Consistency in a Shared Data Base" -- Gray et al. (1976)**
  https://jimgray.azurewebsites.net/papers/granularity%20of%20locks%20and%20degrees%20of%20consistency%20rj%201654.pdf
  Introduced hierarchical locking with intent locks and the degrees of consistency that later became isolation levels.

### Serializable Snapshot Isolation

- **"Making Snapshot Isolation Serializable" -- Fekete et al. (2005)**
  https://dsf.berkeley.edu/cs286/papers/ssi-tods2005.pdf
  The theoretical foundation for SSI. Proved that Snapshot Isolation can be made serializable by detecting dangerous structures (two consecutive rw-dependencies).

- **"Serializable Snapshot Isolation in PostgreSQL" -- Ports & Grittner (2012)**
  https://drkp.net/papers/ssi-vldb12.pdf
  Describes the actual implementation of SSI in PostgreSQL 9.1. Details the SIREAD lock mechanism, rw-conflict detection, and the engineering trade-offs made.

### Concurrency Control Protocols

- **"Concurrency Control and Recovery in Database Systems" -- Bernstein, Hadzilacos, Goodman (1987)**
  https://www.microsoft.com/en-us/research/wp-content/uploads/2016/05/ccontrol.zip
  The definitive textbook on concurrency control theory. Covers 2PL, timestamp ordering, optimistic concurrency control, and multiversion protocols. Free online.

- **"On Optimistic Methods for Concurrency Control" -- Kung & Robinson (1981)**
  https://www.eecs.harvard.edu/~htk/publication/1981-tods-kung-robinson.pdf
  The original paper on optimistic concurrency control (OCC). Describes the read-validate-write protocol.

### MVCC

- **"Multiversion Concurrency Control -- Theory and Algorithms" -- Bernstein & Goodman (1983)**
  https://dl.acm.org/doi/10.1145/319996.320006
  Foundational paper on MVCC theory, covering multiversion timestamp ordering and multiversion 2PL.

---

## Database-Specific Documentation

### PostgreSQL

- **PostgreSQL MVCC Documentation**
  https://www.postgresql.org/docs/current/mvcc.html
  Official documentation covering transaction isolation, explicit locking, and MVCC mechanics.

- **PostgreSQL Source: heapam_visibility.c**
  https://github.com/postgres/postgres/blob/master/src/backend/access/heap/heapam_visibility.c
  The actual visibility check implementation. Reading this code is the best way to understand PostgreSQL's MVCC.

- **PostgreSQL Source: deadlock.c**
  https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/deadlock.c
  PostgreSQL's deadlock detection implementation. Well-commented and instructive.

- **PostgreSQL Source: predicate.c**
  https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/predicate.c
  SSI implementation. Contains extensive header comments explaining the algorithm.

- **PostgreSQL Source: lock.c**
  https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/lock.c
  The main lock manager. Demonstrates production-grade lock table and wait queue management.

- **PostgreSQL Source: xact.c**
  https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xact.c
  Transaction lifecycle management (begin, commit, abort, savepoints).

- **"How Postgres Makes Transactions Atomic" -- Brandon Tilley**
  https://brandur.org/postgres-atomicity
  Excellent blog post explaining how PostgreSQL achieves atomicity using MVCC, pg_xact, and hint bits.

### MySQL / InnoDB

- **InnoDB Multi-Versioning Documentation**
  https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html
  Official documentation of InnoDB's MVCC implementation.

- **InnoDB Locking Documentation**
  https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html
  Covers shared/exclusive locks, intention locks, record locks, gap locks, next-key locks, and auto-increment locks.

- **"InnoDB MVCC Implementation" -- Jeremy Cole**
  https://blog.jcole.us/innodb/
  Deep-dive blog series into InnoDB's physical storage format, page structure, and MVCC implementation.

---

## Books

- **"Database Internals" -- Alex Petrov (O'Reilly, 2019)**
  Chapters 5 (Transaction Processing and Recovery) and 6 (Concurrency Control) provide an excellent practical overview of transaction internals across multiple database architectures.

- **"Designing Data-Intensive Applications" -- Martin Kleppmann (O'Reilly, 2017)**
  Chapter 7 (Transactions) is one of the best practical introductions to transactions, isolation levels, and their trade-offs. Highly recommended as a first read.

- **"Transaction Processing: Concepts and Techniques" -- Jim Gray & Andreas Reuter (1992)**
  The comprehensive reference on transaction processing. Covers everything from the theory of serializability to practical implementation of lock managers, logging, and recovery. 1000+ pages.

- **"Architecture of a Database System" -- Hellerstein, Stonebraker, Hamilton (2007)**
  https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf
  Section 6 covers transaction management, concurrency control, and recovery. Provides a system-level view of how these components fit together. Free online.

---

## Blog Posts and Articles

- **"How Does a Relational Database Work" -- Christophe Kalenzaga**
  https://coding-geek.com/how-databases-work/
  Long-form article covering lock manager implementation, deadlock detection, and MVCC. Good diagrams.

- **"PostgreSQL's MVCC -- A Deep Dive" -- Hussein Nasser**
  https://www.youtube.com/watch?v=AveRgUrC7FM
  Video walkthrough of PostgreSQL MVCC with practical demonstrations.

- **"Understanding PostgreSQL VACUUM" -- Cybertec**
  https://www.cybertec-postgresql.com/en/what-is-vacuum-in-postgresql/
  Practical guide to VACUUM, autovacuum, and common problems (bloat, wraparound).

- **"Transaction ID Wraparound in PostgreSQL"**
  https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND
  Official documentation on the wraparound problem and how freezing addresses it.

- **"A Primer on Database Isolation Levels" -- Vlad Mihalcea**
  https://vladmihalcea.com/a-beginners-guide-to-database-locking-and-the-lost-update-phenomena/
  Practical examples of isolation level behavior with code demonstrations.

- **"Locks in InnoDB" -- Percona**
  https://www.percona.com/blog/innodb-locking-explained-with-stick-figures/
  Visual explanation of InnoDB's various lock types using stick figures. Makes gap locks and next-key locks intuitive.

---

## Videos and Lectures

- **CMU 15-445/645 Database Systems -- Lecture 15-18: Concurrency Control**
  https://15445.courses.cs.cmu.edu/
  Andy Pavlo's database systems course. Lectures 15-18 cover locking, 2PL, timestamp ordering, MVCC, and OCC. Recordings available on YouTube.

- **CMU 15-721 Advanced Database Systems -- MVCC Lectures**
  https://15721.courses.cs.cmu.edu/
  Graduate-level lectures on MVCC implementation variants, garbage collection strategies, and modern concurrency control protocols.

- **"Transactions: Myths, Surprises, and Opportunities" -- Martin Kleppmann (Strange Loop 2015)**
  https://www.youtube.com/watch?v=5ZjhNTM8XU8
  Excellent talk on common misconceptions about transactions and isolation levels.

---

## Interactive Tools and Playgrounds

- **"Jepsen" -- Kyle Kingsbury**
  https://jepsen.io/
  Distributed systems testing framework that tests databases for isolation level correctness. The analyses of various databases (PostgreSQL, MySQL, CockroachDB, etc.) are invaluable for understanding real-world isolation behavior.

- **"Hermitage" -- Martin Kleppmann**
  https://github.com/ept/hermitage
  Test suite that empirically tests the isolation levels of various databases. The README provides a matrix of which anomalies each database actually prevents at each isolation level.

- **"DB Fiddle"**
  https://www.db-fiddle.com/
  Online SQL playground where you can experiment with transactions and isolation levels on PostgreSQL and MySQL.

---

## Implementation References

- **"Neon: Serverless Postgres" -- Source Code**
  https://github.com/neondatabase/neon
  Modern PostgreSQL-based system with well-structured Rust code for page versioning and WAL.

- **"ToyDB" -- Erik Grinaker**
  https://github.com/erikgrinaker/toydb
  A distributed SQL database written in Rust for educational purposes. Clean implementation of MVCC and transactions.

- **"mini-lsm" -- Alex Chi**
  https://github.com/skyzh/mini-lsm
  An LSM-tree storage engine tutorial in Rust. Week 3 covers MVCC implementation with snapshot isolation.

- **"bustub" -- CMU 15-445**
  https://github.com/cmu-db/bustub
  The C++ educational database used in CMU's database course. Has lock manager and transaction manager implementations as course projects.

---

## RFCs and Standards

- **SQL Standard -- ISO/IEC 9075**
  The formal definition of SQL isolation levels. The 1992 standard defined the four levels based on phenomena (dirty read, non-repeatable read, phantom). The Berenson et al. critique paper (above) is more useful for understanding the actual semantics.

- **PostgreSQL Wiki: SSI**
  https://wiki.postgresql.org/wiki/SSI
  Internal wiki page documenting the design decisions behind PostgreSQL's SSI implementation.
