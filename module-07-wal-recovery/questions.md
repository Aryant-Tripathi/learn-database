# Module 7: Quiz Questions -- WAL & Crash Recovery

## Instructions

Each question is followed by a collapsible answer section. Try to answer each question
before revealing the answer. For trace-through questions, work through the scenario on
paper first.

---

## Section 1: WAL Fundamentals

### Question 1: The WAL Rule

What is the fundamental rule of Write-Ahead Logging? Why is it called "write-ahead"?

<details>
<summary>Answer</summary>

The fundamental rule is: **before any change is written to the database on disk, the log
record describing that change must first be written to stable storage.**

It is called "write-ahead" because the log is always written **ahead** of (before) the
data. The log leads; the data follows. This ensures that after a crash, we can always
determine what changes were made by reading the log, even if the data pages are in an
inconsistent state.
</details>

---

### Question 2: Why Sequential?

Why is writing to the WAL much faster than writing modified data pages directly to disk?

<details>
<summary>Answer</summary>

Two reasons:

1. **Sequential vs Random I/O**: The WAL is an append-only file, so all writes are
   sequential. Data pages are scattered across the disk, requiring random I/O. Sequential
   I/O is 10-100x faster than random I/O on spinning disks and still significantly faster
   on SSDs.

2. **Write size**: A log record is typically tens to hundreds of bytes. A data page is
   4KB-16KB. Writing a small log record is faster than writing an entire page, especially
   when only a few bytes on the page changed.
</details>

---

### Question 3: What If No WAL?

Consider a database without WAL. Transaction T1 transfers $200 from Account A to Account B.
The system crashes after writing Account A's page to disk but before writing Account B's
page. What is the state of the database after restart?

<details>
<summary>Answer</summary>

Without WAL, the database is **corrupted and unrecoverable**:

- Account A has been debited: balance reduced by $200 (change is on disk)
- Account B has NOT been credited: balance unchanged (change was only in memory)
- $200 has effectively vanished from the system
- There is no log to consult, so the database cannot detect or fix this inconsistency
- The database violates both **atomicity** (partial transaction on disk) and
  **consistency** ($200 disappeared)
</details>

---

### Question 4: LSN Purpose

List three distinct roles that the Log Sequence Number (LSN) serves in a WAL-based system.

<details>
<summary>Answer</summary>

1. **Unique identifier**: Each log record has a unique LSN, allowing any record to be
   referenced unambiguously.

2. **Ordering/timestamp**: LSNs are monotonically increasing, establishing a total order
   over all database modifications. Higher LSN = later in time.

3. **Physical address**: The LSN often equals the byte offset in the log file, enabling
   O(1) lookup of any log record.

4. (Bonus) **Recovery coordination**: The pageLSN on each data page tells recovery whether
   a redo is needed (compare pageLSN with the log record's LSN). The flushedLSN tells the
   buffer manager whether it is safe to write a dirty page.
</details>

---

### Question 5: pageLSN Enforcement

Before the buffer manager writes a dirty page to disk, it checks that
`pageLSN <= flushedLSN`. What would go wrong if this check were omitted?

<details>
<summary>Answer</summary>

If the check is omitted, a dirty page could be written to disk before its corresponding
log record is flushed. If the system then crashes:

- The data page on disk reflects a change that has no corresponding log record on disk
- During recovery, redo cannot re-apply the change (no log record exists)
- If the page was from an uncommitted transaction (STEAL policy), undo cannot reverse the
  change either (no log record to identify what needs undoing)
- The database is left in a corrupted, unrecoverable state

This is a violation of the WAL protocol and is the single most critical correctness
invariant in the entire system.
</details>

---

## Section 2: Steal/Force Policies

### Question 6: Policy Matrix

Fill in the following table:

| Policy | Needs UNDO? | Needs REDO? |
|--------|-------------|-------------|
| No-Steal + Force | ? | ? |
| No-Steal + No-Force | ? | ? |
| Steal + Force | ? | ? |
| Steal + No-Force | ? | ? |

<details>
<summary>Answer</summary>

| Policy | Needs UNDO? | Needs REDO? | Reasoning |
|--------|-------------|-------------|-----------|
| No-Steal + Force | No | No | No-Steal means uncommitted data never reaches disk (no undo needed). Force means all committed data is on disk at commit time (no redo needed). |
| No-Steal + No-Force | No | Yes | No uncommitted data on disk. But committed data might not be on disk yet (need redo). |
| Steal + Force | Yes | No | Uncommitted data might be on disk (need undo). But all committed data is forced to disk (no redo needed). |
| Steal + No-Force | Yes | Yes | Uncommitted data on disk (need undo). Committed data might not be on disk (need redo). |
</details>

---

### Question 7: Why STEAL/NO-FORCE?

STEAL/NO-FORCE requires both undo AND redo logging, making recovery more complex. Why do
all production databases still choose this policy?

<details>
<summary>Answer</summary>

Performance:

- **STEAL** allows the buffer manager to evict any page at any time. Without STEAL
  (NO-STEAL), a long-running transaction could pin many pages in the buffer pool, causing
  memory pressure and potentially deadlock with other transactions needing buffer frames.

- **NO-FORCE** means commit only requires flushing the log (a small sequential write).
  With FORCE, every commit requires flushing ALL dirty pages modified by that transaction
  to disk -- potentially many random writes. This makes commits extremely slow.

The combination gives maximum buffer management flexibility and minimum commit latency.
The cost of implementing both undo and redo is worth the performance benefit.
</details>

---

### Question 8: STEAL Scenario

Transaction T1 modifies page P. Before T1 commits, the buffer manager needs to evict
page P to make room for another page (STEAL policy). Describe exactly what must happen
before page P can be written to disk.

<details>
<summary>Answer</summary>

1. The WAL protocol must be enforced: the log record for T1's modification of page P
   must be flushed to stable storage first. Specifically, `pageLSN <= flushedLSN` must
   hold.

2. If the log buffer contains T1's update record but has not been flushed yet, the
   buffer manager (or WAL writer) must flush the log up to at least page P's `pageLSN`.

3. Only after the log flush completes (including `fsync()`) can the dirty page P be
   written to the data file on disk.

4. The before-image in the log record allows the recovery system to undo this change if
   T1 ultimately does not commit (crash before commit, or explicit abort).
</details>

---

## Section 3: Log Record Types

### Question 9: CLR Purpose

What is a Compensation Log Record (CLR)? Why is the `undoNextLSN` field critical?

<details>
<summary>Answer</summary>

A CLR is a log record written during the undo phase of recovery (or during a normal
transaction abort). It records the undo of a previous UPDATE record.

The `undoNextLSN` field is critical because it handles **crashes during recovery**. If the
system crashes while undoing a transaction, the next recovery must not re-undo operations
that were already undone. The `undoNextLSN` points to the next record that still needs to
be undone, allowing recovery to skip over already-undone operations.

Without `undoNextLSN`, a crash during undo could cause the same operation to be undone
twice -- which is NOT idempotent for many operations and would corrupt data.
</details>

---

### Question 10: COMMIT Record Timing

At what exact moment does a transaction become "committed" (durable)? Is it when the
COMMIT record is written to the log buffer? When it is written to the OS file? When
fsync returns?

<details>
<summary>Answer</summary>

A transaction is committed at the moment `fsync()` (or equivalent) returns successfully
for the WAL file containing the COMMIT record.

- Writing to the log buffer: NOT committed (data only in volatile RAM)
- Writing to the OS file (write() system call): NOT committed (data may be in the OS
  page cache, which is still volatile RAM)
- After fsync() returns: COMMITTED (data is on stable storage)

This is why `fsync()` is on the critical path of every transaction commit and why group
commit (batching multiple commits into one fsync) is so important for performance.
</details>

---

## Section 4: Checkpointing

### Question 11: Sharp vs Fuzzy

What is the key difference between a sharp checkpoint and a fuzzy checkpoint? Why do
production databases use fuzzy checkpoints?

<details>
<summary>Answer</summary>

**Sharp checkpoint**: Stops all transaction processing, flushes ALL dirty pages to disk,
writes a checkpoint record, then resumes. After a sharp checkpoint, the database on disk
is fully consistent.

**Fuzzy checkpoint**: Captures the ATT and DPT without stopping transactions or flushing
pages. Transactions continue running during the checkpoint. The database on disk is NOT
necessarily consistent at the checkpoint point.

Production databases use fuzzy checkpoints because sharp checkpoints cause an unacceptable
pause in service. A database with millions of dirty pages would freeze for seconds or
minutes during a sharp checkpoint. Fuzzy checkpoints add zero downtime but require the
ARIES analysis phase to account for the imprecise checkpoint state.
</details>

---

### Question 12: Checkpoint Contents

What information does a fuzzy checkpoint record contain, and how is each piece used
during recovery?

<details>
<summary>Answer</summary>

A fuzzy checkpoint contains:

1. **Active Transaction Table (ATT)**: Lists all transactions that were running at
   checkpoint time, with their status and lastLSN. During analysis, this is the starting
   point for determining which transactions need to be undone.

2. **Dirty Page Table (DPT)**: Lists all dirty pages and their recLSN (the LSN of the
   first log record that dirtied the page since last flush). During analysis, this is the
   starting point for determining which pages might need redo. The minimum recLSN in the
   DPT determines where the redo phase begins scanning.

The analysis phase refines both tables by scanning the log forward from the checkpoint,
accounting for any changes that happened after the checkpoint was taken.
</details>

---

## Section 5: ARIES Recovery Traces

### Question 13: Analysis Phase Trace

Given the following log and a checkpoint at LSN 40:

```
LSN 10: BEGIN T1
LSN 20: T1 UPDATE page=3 (A: 10->20)
LSN 30: BEGIN T2
LSN 40: CHECKPOINT (ATT: {T1: lastLSN=20, T2: lastLSN=30}, DPT: {page3: recLSN=20})
LSN 50: T2 UPDATE page=5 (B: 30->40)
LSN 60: T1 UPDATE page=3 (C: 5->15)
LSN 70: T1 COMMIT
LSN 80: T2 UPDATE page=7 (D: 50->60)
        *** CRASH ***
```

What are the ATT and DPT after the analysis phase?

<details>
<summary>Answer</summary>

Starting from checkpoint at LSN 40:
- ATT = {T1: lastLSN=20, T2: lastLSN=30}
- DPT = {page3: recLSN=20}

Scanning forward:

| LSN | Action |
|-----|--------|
| 50 | T2 UPDATE page=5: ATT: T2.lastLSN=50. DPT: add page5 with recLSN=50 |
| 60 | T1 UPDATE page=3: ATT: T1.lastLSN=60. page3 already in DPT (keep recLSN=20) |
| 70 | T1 COMMIT: Remove T1 from ATT |
| 80 | T2 UPDATE page=7: ATT: T2.lastLSN=80. DPT: add page7 with recLSN=80 |

**Final ATT**: {T2: lastLSN=80, status=Running}
**Final DPT**: {page3: recLSN=20, page5: recLSN=50, page7: recLSN=80}

T2 needs to be undone. Redo starts at min(recLSN) = LSN 20.
</details>

---

### Question 14: Redo Phase Trace

Continuing from Question 13, suppose the pages on disk have these pageLSNs:
- page3: pageLSN = 60
- page5: pageLSN = 40 (some older value)
- page7: pageLSN = 0 (never written)

Which log records are redone during the redo phase?

<details>
<summary>Answer</summary>

Redo scans from min(recLSN) = LSN 20:

| LSN | Page | In DPT? | LSN >= recLSN? | pageLSN < LSN? | Action |
|-----|------|---------|----------------|-----------------|--------|
| 20 | page3 | Yes (recLSN=20) | 20 >= 20: Yes | pageLSN=60 >= 20: No | **SKIP** |
| 50 | page5 | Yes (recLSN=50) | 50 >= 50: Yes | pageLSN=40 < 50: Yes | **REDO** |
| 60 | page3 | Yes (recLSN=20) | 60 >= 20: Yes | pageLSN=60 >= 60: No | **SKIP** |
| 80 | page7 | Yes (recLSN=80) | 80 >= 80: Yes | pageLSN=0 < 80: Yes | **REDO** |

**Redone**: LSN 50 (T2's update to page5) and LSN 80 (T2's update to page7)
**Skipped**: LSN 20 and LSN 60 (already on page3 per pageLSN)

Note that T2's updates are redone even though T2 will be undone -- this is the "repeating
history" principle of ARIES.
</details>

---

### Question 15: Undo Phase Trace

Continuing from Questions 13-14, trace through the undo phase. What CLRs are written?
What is the final state of each page?

<details>
<summary>Answer</summary>

ATT has T2 with lastLSN=80. ToUndo = {80}.

| Step | Process | Action |
|------|---------|--------|
| 1 | LSN=80: T2 UPDATE page=7 (D: 50->60) | Undo: write D=50 to page7. Write CLR (LSN=90, undoNextLSN=50). ToUndo={50} |
| 2 | LSN=50: T2 UPDATE page=5 (B: 30->40) | Undo: write B=30 to page5. Write CLR (LSN=100, undoNextLSN=30). ToUndo={30} |
| 3 | LSN=30: T2 BEGIN | This is a BEGIN, not an UPDATE. Write END record for T2. Done. |

**CLRs written**:
- LSN=90: CLR undo of LSN=80, undoNextLSN=50
- LSN=100: CLR undo of LSN=50, undoNextLSN=30

**Final page states**:
- page3: C=15, A=20 (T1's committed changes preserved)
- page5: B=30 (T2's change undone)
- page7: D=50 (T2's change undone)
</details>

---

### Question 16: Crash During Recovery

In the scenario from Question 15, suppose the system crashes AGAIN after writing the CLR
at LSN=90 but before writing the CLR at LSN=100. What happens during the second recovery?

<details>
<summary>Answer</summary>

Second recovery:

**Analysis phase**: Scans from checkpoint at LSN=40. Processes all records including the
new CLR at LSN=90. T2 is still in ATT with lastLSN=90 (the CLR).

**Redo phase**: Redoes all actions including the CLR at LSN=90 (restoring D=50 on page7).

**Undo phase**: ToUndo = {90} (T2's lastLSN is now the CLR).

| Step | Process | Action |
|------|---------|--------|
| 1 | LSN=90: CLR, undoNextLSN=50 | It is a CLR, follow undoNextLSN. ToUndo={50} |
| 2 | LSN=50: T2 UPDATE page=5 | Undo: write B=30. Write CLR. ToUndo={30} |
| 3 | LSN=30: BEGIN | Write END for T2. |

The CLR at LSN=90 ensures that the undo of LSN=80 is NOT repeated. The `undoNextLSN`
field skips directly to LSN=50. This is exactly why CLRs exist -- they make recovery
correct even across multiple crashes.
</details>

---

## Section 6: Advanced Topics

### Question 17: Group Commit

Explain the group commit optimization. If a disk has 5ms fsync latency, what is the
theoretical maximum commits per second (a) without group commit, and (b) with group
commit batching 20 transactions per fsync?

<details>
<summary>Answer</summary>

Group commit batches multiple transactions' COMMIT records into a single fsync call.
When one transaction requests a commit flush, the system waits briefly for other
transactions to also commit, then flushes all of them together.

(a) Without group commit: 1 fsync per commit. 1000ms / 5ms = **200 commits/second**

(b) With group commit (20 txns/batch): 1 fsync per 20 commits.
    200 fsyncs/second * 20 commits/fsync = **4,000 commits/second**

This is a 20x improvement with no change in durability guarantees.
</details>

---

### Question 18: Full-Page Writes

PostgreSQL writes a full page image to WAL after each checkpoint for the first modification
to each page. Why? What problem does this solve?

<details>
<summary>Answer</summary>

Full-page writes (also called "backup blocks" or "full-page images") protect against
**torn pages**.

A database page is typically 8KB (PostgreSQL) or 16KB, but the OS and disk may write in
512-byte or 4KB sectors. If the system crashes mid-write, half the page may have the new
data and half may have old data -- a "torn page."

If the page was torn, the before-image in the WAL record does not match what is on disk,
so neither redo nor undo can be applied correctly.

By writing the entire page image to WAL after each checkpoint, recovery can restore the
full page from the WAL before applying incremental redo records, guaranteeing page-level
consistency.
</details>

---

### Question 19: Logical vs Physical Logging

Why does ARIES use physiological logging (physical redo, logical undo) rather than purely
physical or purely logical logging?

<details>
<summary>Answer</summary>

**Physical redo** is used because:
- It is deterministic: "write these bytes at this offset on this page" always produces the
  same result
- It is simple and fast: just a memcpy
- It does not require any schema or index information to replay

**Logical undo** is used because:
- Between the original update and the undo, the page structure may have changed (e.g., a
  B-tree page split moved the tuple to a different page)
- Physical undo would write to the wrong location if the page structure changed
- Logical undo says "delete the tuple that was inserted" which works regardless of where
  the tuple physically ended up

Purely physical logging would be too verbose (logging entire pages). Purely logical logging
would be non-deterministic for redo (where on the page should an inserted tuple go?).
Physiological logging combines the strengths of both approaches.
</details>

---

### Question 20: Nested Top Actions

A B-tree page split inserts a new key into a leaf, splits the leaf, and updates the parent.
If the inserting transaction later aborts, should the page split be undone? Why or why not?

<details>
<summary>Answer</summary>

**No**, the page split should NOT be undone, even if the inserting transaction aborts.

Reasons:
1. Other transactions may have already inserted keys into the new page created by the
   split. Undoing the split would lose those insertions.
2. The split is a **structural** modification to the tree, not a logical data change. The
   only thing that should be undone is the insertion of the specific key.

ARIES handles this with **nested top actions**: the split is logged normally, but a CLR
is written at the end whose `undoNextLSN` skips over the entire split operation. During
undo, the undo chain jumps past the split, and only the key insertion is undone logically.
</details>

---

### Question 21: recLSN vs pageLSN

What is the difference between `recLSN` (in the Dirty Page Table) and `pageLSN` (on the
data page)? When is each set?

<details>
<summary>Answer</summary>

**recLSN** (recovery LSN):
- Stored in the DPT (in-memory structure)
- Set to the LSN of the FIRST log record that dirtied the page after it was last flushed
  to disk
- Purpose: tells recovery the earliest LSN from which redo might be needed for this page
- Never updated after initial set (until the page is flushed and re-dirtied)

**pageLSN**:
- Stored on the physical page itself (page header)
- Set to the LSN of the MOST RECENT log record that modified this page
- Updated on every modification
- Purpose: tells recovery whether a specific redo is needed (if pageLSN >= record's LSN,
  the change is already on the page)

Example: Page P is clean. T1 modifies it (LSN=50), then T2 modifies it (LSN=80).
- recLSN = 50 (first modification since last flush)
- pageLSN = 80 (most recent modification)
</details>

---

### Question 22: WAL Segment Management

PostgreSQL splits the WAL into 16MB segment files. When can a WAL segment be recycled
(reused)? What conditions must be met?

<details>
<summary>Answer</summary>

A WAL segment can be recycled when ALL of the following conditions are met:

1. **All log records in the segment have been applied to the data pages on disk**: the
   checkpoint has advanced past all records in the segment (i.e., the checkpoint's redo
   point is beyond the segment).

2. **The segment has been archived** (if WAL archiving is enabled for PITR): the archiver
   must have successfully copied the segment to the archive location.

3. **No replication slot requires the segment**: if streaming replication is configured,
   standby servers may still need the segment.

Once recycled, the segment file is renamed (not deleted) to a future segment name and
reused, avoiding the overhead of creating new files.
</details>

---

### Question 23: PITR Recovery

You accidentally run `DROP TABLE customers` at 3:15 PM. You have a base backup from
2:00 AM and continuous WAL archiving. Describe the steps to recover the `customers` table.

<details>
<summary>Answer</summary>

1. **Stop the database server** to prevent further damage.

2. **Restore the base backup** from 2:00 AM to a new directory (do not overwrite the
   current data directory).

3. **Configure recovery**: create a `recovery.conf` (or set recovery parameters in
   `postgresql.conf` for PG 12+):
   - `restore_command = 'cp /archive/%f %p'` (to fetch archived WAL segments)
   - `recovery_target_time = '2026-03-28 15:14:59'` (one second before the DROP)
   - `recovery_target_action = 'promote'`

4. **Start the server**: PostgreSQL will:
   - Replay all archived WAL segments from 2:00 AM to 3:14:59 PM
   - Stop just before the DROP TABLE command
   - Promote to a new timeline

5. **Export the customers table** from this recovered instance using `pg_dump`.

6. **Import the table** back into the production database.
</details>

---

### Question 24: Multiple Undo Transactions

During the undo phase, transactions T1 (lastLSN=100) and T2 (lastLSN=90) both need undo.
T1's chain: 100 -> 70 -> 40 -> 10. T2's chain: 90 -> 60 -> 30. In what order are the
undo operations performed?

<details>
<summary>Answer</summary>

ARIES processes undo in **reverse LSN order** across all transactions simultaneously using
a single backward pass. The ToUndo set is a max-heap.

| Step | Pick (max LSN) | Txn | Action | ToUndo after |
|------|---------------|-----|--------|-------------|
| 1 | 100 | T1 | Undo LSN=100, add 70 | {90, 70} |
| 2 | 90 | T2 | Undo LSN=90, add 60 | {70, 60} |
| 3 | 70 | T1 | Undo LSN=70, add 40 | {60, 40} |
| 4 | 60 | T2 | Undo LSN=60, add 30 | {40, 30} |
| 5 | 40 | T1 | Undo LSN=40, add 10 | {30, 10} |
| 6 | 30 | T2 | Undo LSN=30, T2 fully undone | {10} |
| 7 | 10 | T1 | Undo LSN=10, T1 fully undone | {} |

Order: 100, 90, 70, 60, 40, 30, 10

This single-pass approach is more efficient than undoing each transaction separately.
</details>

---

### Question 25: Redo Idempotency

Why must redo be idempotent? Give a concrete example of what goes wrong if redo is not
idempotent.

<details>
<summary>Answer</summary>

Redo must be idempotent because the system may crash during redo itself, causing the same
redo to be applied multiple times across multiple recovery attempts.

Example of non-idempotent redo: suppose a log record says "increment balance by 100" (a
logical redo). If this redo is applied twice (crash during recovery, then recovery again),
the balance is incremented by 200 instead of 100.

ARIES ensures idempotency through the **pageLSN check**: before redoing a record, it
checks whether `pageLSN < record's LSN`. If the pageLSN is already >= the record's LSN,
the change was already applied and the redo is skipped. This makes redo a no-op when
applied to a page that already has the change -- which is the definition of idempotent.
</details>

---

### Question 26: Force Policy Trade-off

A system uses the FORCE policy (all dirty pages are flushed at commit time). Transaction
T1 modifies 500 pages and then commits. What is the performance cost compared to
NO-FORCE?

<details>
<summary>Answer</summary>

With FORCE: At commit time, all 500 dirty pages must be written to disk before the commit
can be acknowledged to the client. This means:
- 500 random page writes (each 8-16KB)
- With 5ms average I/O latency: 500 * 5ms = 2.5 seconds for the commit
- Even with some parallelism, this is catastrophically slow

With NO-FORCE: At commit time, only the COMMIT log record needs to be flushed -- a single
sequential write of a few bytes followed by one fsync. Total latency: ~5ms.

The 500 dirty pages are written lazily in the background by the background writer or
checkpointer, amortized over time and possibly combined with other transactions' writes.

The performance difference is roughly **500x** for this transaction's commit latency.
</details>

---

### Question 27: Checkpoint Frequency

What happens if checkpoints are taken too frequently? Too infrequently?

<details>
<summary>Answer</summary>

**Too frequently**:
- Increased I/O load from writing checkpoint records and (for non-fuzzy checkpoints)
  flushing pages
- Increased WAL volume from full-page writes (PostgreSQL writes a full page image after
  each checkpoint for the first modification to each page)
- CPU overhead of capturing ATT and DPT snapshots

**Too infrequently**:
- Recovery takes longer because the redo phase must scan more of the log (from an older
  checkpoint)
- More WAL segments must be retained (cannot recycle segments before the checkpoint)
- Increased disk space usage for WAL

Production systems typically checkpoint every few minutes (e.g., PostgreSQL defaults to
every 5 minutes or every `max_wal_size` of WAL generated, whichever comes first).
</details>

---

### Question 28: WAL and Replication

How does WAL enable database replication? What is the relationship between WAL-based
recovery and WAL-based replication?

<details>
<summary>Answer</summary>

WAL-based replication works by shipping WAL records from the primary server to standby
servers, which then replay (redo) the records to maintain a copy of the database.

The relationship is direct: **replication uses the exact same redo mechanism as crash
recovery**. The standby is perpetually in "recovery mode," continuously replaying WAL
records as they arrive from the primary.

This is called **physical replication** (or log shipping / streaming replication):
- The standby applies the same physical-level changes as the primary
- The standby's data files are byte-for-byte identical to the primary's
- No logical interpretation of the changes is needed
- This is the foundation of PostgreSQL's streaming replication and most RDBMS HA solutions
</details>

---

## Section 7: Conceptual Questions

### Question 29: Why Not Just Use Snapshots?

Some systems use copy-on-write (COW) snapshots instead of WAL. What are the trade-offs?

<details>
<summary>Answer</summary>

**COW/Snapshot approach** (used by SQLite in rollback journal mode, LMDB, etc.):
- Pro: Simpler implementation -- no redo needed, just restore from snapshot on crash
- Pro: Instant recovery -- just discard the incomplete write
- Con: Write amplification -- every modification copies the original page first
- Con: Cannot support PITR (no incremental log of changes)
- Con: Cannot support streaming replication easily
- Con: Random write I/O for the copies (vs sequential for WAL)

**WAL approach**:
- Pro: High write throughput (sequential I/O)
- Pro: Enables PITR, replication, and logical decoding
- Pro: Flexible buffer management (STEAL/NO-FORCE)
- Con: More complex recovery (three phases)
- Con: Requires careful implementation (fsync, CRC, CLRs)

For high-throughput OLTP databases, WAL wins decisively. For embedded databases with
simpler requirements, COW can be a valid choice.
</details>

---

### Question 30: Design Question

You are designing a new database engine. The maximum acceptable recovery time after a
crash is 10 seconds. The WAL write rate is 50 MB/second. Redo can process WAL at
200 MB/second. How frequently should you checkpoint?

<details>
<summary>Answer</summary>

We need recovery (primarily redo) to complete within 10 seconds.

Redo speed: 200 MB/second, so in 10 seconds we can redo 2000 MB of WAL.

WAL accumulation between checkpoints: at 50 MB/s, to accumulate 2000 MB takes 40 seconds.

So checkpoints should occur **at most every 40 seconds** to ensure recovery completes
within 10 seconds.

In practice, you would checkpoint more frequently (e.g., every 20-30 seconds) to leave
margin for the analysis and undo phases, disk warm-up, and variance in redo speed.

This is why PostgreSQL's `max_wal_size` parameter (which triggers a checkpoint) is tuned
based on recovery time objectives, not just disk space.
</details>
