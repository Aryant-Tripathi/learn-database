# Module 10: Distributed Databases & Consensus - Quiz Questions

## Instructions

Answer each question thoroughly. For multiple-choice questions, select the best answer and explain your reasoning. For open-ended questions, provide detailed explanations.

---

## Section 1: CAP Theorem and Consistency Models

### Question 1
**What does the CAP theorem actually state, and why is it commonly misunderstood?**

<details>
<summary>Answer</summary>

The CAP theorem states that a distributed system can provide at most two of three guarantees simultaneously: **Consistency** (linearizability), **Availability** (every non-failing node returns a response), and **Partition Tolerance** (the system continues operating despite network partitions).

Common misunderstandings:
- **Partition tolerance is not optional.** Network partitions will happen in any distributed system, so the real choice is between CP and AP *during a partition*.
- **"Consistency" means linearizability**, not ACID consistency. Many people confuse the two.
- **It is not a permanent, system-wide choice.** Systems can make different trade-offs for different operations, and when the network is healthy, all three properties can coexist.
- **The choice is not binary.** There is a spectrum between strong consistency and high availability.
</details>

### Question 2
**A system uses quorum reads and writes with N=5, W=3, R=3. During a network partition, two nodes become unreachable. Can the system still process reads and writes? Explain.**

<details>
<summary>Answer</summary>

With 3 out of 5 nodes reachable:
- **Writes:** W=3 requires 3 nodes. Exactly 3 are available, so writes can succeed (barely).
- **Reads:** R=3 requires 3 nodes. Same situation, reads can succeed.
- **Overlap guarantee:** W + R = 6 > N = 5, so reads and writes overlap by at least 1 node, maintaining consistency.

However, if a third node fails, both reads and writes would fail (only 2 nodes available, need 3). The system trades availability for consistency in this scenario.
</details>

### Question 3
**Explain the PACELC theorem. Why does it provide more useful information than CAP alone?**

<details>
<summary>Answer</summary>

PACELC: **If Partition (P), choose Availability (A) or Consistency (C). Else (E), choose Latency (L) or Consistency (C).**

CAP only describes behavior during partitions, but partitions are rare. Most of the time the system operates normally, and the latency vs. consistency trade-off is what users actually experience day-to-day. PACELC captures this.

For example, Dynamo is PA/EL: during partitions it favors availability, and during normal operation it favors low latency over consistency. Spanner is PC/EC: during partitions it blocks (consistency), and during normal operation it pays the latency cost for strong consistency (commit-wait).
</details>

### Question 4
**Rank these consistency models from strongest to weakest: eventual consistency, sequential consistency, linearizability, causal consistency. Explain what each guarantees.**

<details>
<summary>Answer</summary>

From strongest to weakest:

1. **Linearizability** - Operations appear atomic and ordered by real-time. If operation A completes before B starts, B sees A's effect. Equivalent to a single-copy system.
2. **Sequential consistency** - All operations appear in some total order consistent with each process's program order. But this order need not match real-time (two concurrent operations could be ordered either way).
3. **Causal consistency** - Causally related operations are seen in order by all nodes. Concurrent (unrelated) operations may be seen in different orders by different nodes.
4. **Eventual consistency** - If no new writes occur, all replicas will eventually converge. No ordering guarantees during convergence.
</details>

### Question 5
**A social media application shows a user's post feed. User A posts a photo, then User B comments on it. Under eventual consistency, what anomaly might a third user C observe? Under causal consistency?**

<details>
<summary>Answer</summary>

**Under eventual consistency:** User C might see B's comment before A's photo (or see the comment without the photo at all). This is because eventual consistency provides no ordering guarantees - replicas may receive and apply updates in any order.

**Under causal consistency:** User C is guaranteed to see A's photo before B's comment, because B's comment is causally dependent on A's photo (B saw the photo, then commented). However, if User D independently posts at the same time, C might see D's post in a different position relative to A's post than another user would.
</details>

---

## Section 2: Replication

### Question 6
**Compare single-leader, multi-leader, and leaderless replication. For each, give a real-world system and a scenario where it is the best choice.**

<details>
<summary>Answer</summary>

| Strategy | System | Best Scenario |
|----------|--------|---------------|
| **Single-leader** | PostgreSQL streaming replication | Traditional OLTP with strong consistency needs. One write location, read replicas for scaling. |
| **Multi-leader** | CouchDB, MySQL Group Replication | Multi-datacenter deployment where each DC needs local write capability and cross-DC latency is high. |
| **Leaderless** | Cassandra, DynamoDB | High-availability systems that can tolerate eventual consistency. Shopping carts, sensor data, DNS. |

Key trade-offs:
- Single-leader: No write conflicts, but leader is a bottleneck and SPOF.
- Multi-leader: Write to any DC, but must resolve conflicts.
- Leaderless: Highest availability, but weakest consistency without careful quorum tuning.
</details>

### Question 7
**In a leaderless system with N=3, W=2, R=2, a client writes value "X" to two nodes. Before the third node receives the value, it fails. When it recovers, how does it get the latest value? Describe two mechanisms.**

<details>
<summary>Answer</summary>

**Read repair:** When a client reads from this node and another up-to-date node, it detects the stale value and writes the latest value back to the stale node.

**Anti-entropy:** A background process periodically compares Merkle tree hashes between replicas. When a difference is detected, only the divergent key ranges are synchronized. This catches values that are rarely read (which read repair would miss).
</details>

### Question 8
**What is the "split brain" problem in single-leader replication? How do systems like Raft prevent it?**

<details>
<summary>Answer</summary>

Split brain occurs when two nodes both believe they are the leader, potentially accepting conflicting writes. This can happen during network partitions or when a leader is slow and declared dead prematurely.

Raft prevents split brain through:
1. **Term numbers:** Each election increments the term. A leader with a lower term steps down when it sees a higher term.
2. **Majority voting:** A leader must receive votes from a majority of nodes. Since two majorities always overlap, at most one leader can be elected per term.
3. **Heartbeats:** The leader sends periodic heartbeats. Followers only start elections if they stop receiving heartbeats.
</details>

### Question 9
**Explain the difference between synchronous and semi-synchronous replication. Why do most production systems use semi-synchronous rather than fully synchronous?**

<details>
<summary>Answer</summary>

- **Synchronous:** The leader waits for ALL followers to acknowledge before confirming the write. Guarantees all replicas are up-to-date but is slow (limited by the slowest follower) and blocks if any follower is down.
- **Semi-synchronous:** The leader waits for ONE follower to acknowledge, then confirms. Replicates asynchronously to remaining followers. Guarantees at least two copies exist.

Semi-synchronous is preferred because:
- It ensures durability beyond the leader without the latency penalty of waiting for all replicas.
- It does not block on a single slow or failed follower.
- It provides a good balance: you tolerate one failure (two copies) without the overhead of full synchronous replication.
</details>

---

## Section 3: Partitioning and Sharding

### Question 10
**A database uses hash-based partitioning with `partition = hash(key) % 4`. If the cluster grows from 4 to 5 nodes, approximately what percentage of keys must be moved? How does consistent hashing improve this?**

<details>
<summary>Answer</summary>

**Hash modulo:** When going from 4 to 5 partitions, nearly all keys change their partition assignment (approximately 80% of keys must move). Only keys where `hash(key) % 4 == hash(key) % 5` stay in place.

**Consistent hashing:** On average, only `K/N` keys must move (where K is the total number of keys and N is the new number of nodes). With 5 nodes, approximately 20% of keys move (1/5). This is the theoretical minimum for any rebalancing scheme.

Consistent hashing achieves this by mapping both nodes and keys to a ring and only reassigning keys between the new node and its immediate neighbor on the ring.
</details>

### Question 11
**What are virtual nodes (vnodes) in consistent hashing? Why are they necessary?**

<details>
<summary>Answer</summary>

Virtual nodes map each physical node to multiple positions on the hash ring (e.g., 256 positions per node). Each position acts as a separate "virtual" node.

They are necessary because:
1. **Load balancing:** With few physical nodes, the ring can be uneven (one node might get 50% of keys). Virtual nodes spread each physical node's responsibility across many ring segments, evening out the distribution.
2. **Smooth rebalancing:** When a node joins or leaves, its virtual nodes are spread across the ring, so the load is redistributed evenly across all remaining nodes rather than being dumped on one neighbor.
3. **Heterogeneous hardware:** A more powerful node can have more virtual nodes, receiving a proportionally larger share of the data.
</details>

### Question 12
**Compare hash-based and range-based partitioning for a time-series database storing IoT sensor readings keyed by `(sensor_id, timestamp)`. Which is better and why?**

<details>
<summary>Answer</summary>

**Range-based partitioning is better** for this use case, but with a compound partition key:
- Partition by `sensor_id` (range or hash), then within each partition, data is sorted by `timestamp`.
- Range queries like "all readings from sensor X between time T1 and T2" scan a contiguous range within a single partition.

**Pure hash-based** would scatter readings from the same sensor across all partitions, making range queries extremely expensive (must query all partitions).

**Pure range-based on timestamp** would create hot spots: all current writes go to the partition handling the latest time range.

The best approach is a **compound key**: hash on `sensor_id` for distribution, range on `timestamp` for efficient time-range queries within each sensor. This is exactly what Cassandra's partition key + clustering key model supports.
</details>

---

## Section 4: Distributed Transactions

### Question 13
**Describe the Two-Phase Commit (2PC) protocol step by step. What is its biggest weakness?**

<details>
<summary>Answer</summary>

**Phase 1 (Prepare):**
1. Coordinator sends PREPARE to all participants.
2. Each participant writes the transaction to its WAL, acquires locks, and replies VOTE YES or VOTE NO.

**Phase 2 (Commit/Abort):**
1. If all votes are YES: coordinator writes COMMIT to its log, sends COMMIT to all participants.
2. If any vote is NO: coordinator sends ABORT to all participants.
3. Participants execute the decision and release locks.

**Biggest weakness: Blocking.** If the coordinator crashes after receiving votes but before sending the decision, participants that voted YES are stuck. They cannot commit (the coordinator might have decided to abort) or abort (it might have decided to commit). They hold locks indefinitely until the coordinator recovers. This makes 2PC vulnerable to coordinator failure.
</details>

### Question 14
**How does 3PC (Three-Phase Commit) attempt to solve 2PC's blocking problem? Why is it rarely used in practice?**

<details>
<summary>Answer</summary>

3PC adds a PRE_COMMIT phase between voting and final commit:
1. **Can Commit?** - Coordinator asks participants if they can commit (no locks yet).
2. **Pre-Commit** - If all agree, coordinator sends PRE_COMMIT. Participants acquire locks and prepare.
3. **Do Commit** - Coordinator sends final commit.

The key improvement: if a participant has received PRE_COMMIT and then times out waiting for DO_COMMIT, it can safely commit because it knows all participants agreed in Phase 1.

**Why rarely used:**
- 3PC does not work correctly with **network partitions**. A partitioned participant might commit while the coordinator aborts on its side of the partition.
- The extra round-trip adds latency.
- In practice, Paxos/Raft-based commit protocols handle the coordinator failure problem better and work correctly with partitions.
</details>

### Question 15
**Explain the Percolator transaction model used by TiDB. How does it handle write-write conflicts?**

<details>
<summary>Answer</summary>

Percolator uses two phases on top of a timestamped key-value store:

**Prewrite phase:**
1. Get a `start_ts` from the timestamp oracle.
2. For each key in the write set, write a lock and the data value at `start_ts`.
3. One key is designated as the "primary"; others point to it.
4. If any key already has a lock or a write with timestamp > `start_ts`, the transaction aborts (write-write conflict detected).

**Commit phase:**
1. Get a `commit_ts` from the timestamp oracle.
2. Remove the lock on the primary key and write a commit record mapping `commit_ts -> start_ts`.
3. Asynchronously remove locks and write commit records for secondary keys.

**Conflict handling:** During prewrite, if a write record exists with `commit_ts > start_ts`, there is a conflict and the transaction aborts. If a lock exists from another transaction, the current transaction waits or cleans up the stale lock (by checking the primary key's status).
</details>

### Question 16
**How does Google Spanner achieve "external consistency" (linearizability) for transactions? What hardware does it require?**

<details>
<summary>Answer</summary>

Spanner uses **TrueTime**, a clock API backed by GPS receivers and atomic clocks in every data center. TrueTime returns an interval `[earliest, latest]` rather than a single timestamp.

For external consistency:
1. A transaction acquires locks (pessimistic 2PL).
2. Each participant picks a prepare timestamp.
3. The coordinator picks a commit timestamp `s >= max(all prepare timestamps)`.
4. **Commit-wait:** The coordinator waits until `TrueTime.now().earliest > s` before releasing locks and acknowledging the commit.
5. This guarantees that any transaction starting after the commit will get a timestamp strictly greater than `s`.

**Required hardware:** GPS receivers and atomic clocks at every data center, plus time master servers. The uncertainty interval is typically < 7ms. Without this specialized hardware, Spanner's approach cannot be replicated exactly (though HLC-based systems like CockroachDB approximate it).
</details>

---

## Section 5: Consensus Algorithms

### Question 17
**In Raft, why must a candidate's log be "at least as up-to-date" as a voter's log to receive a vote? What would happen without this restriction?**

<details>
<summary>Answer</summary>

This restriction ensures the **Leader Completeness property**: any committed entry must be present in all future leaders' logs.

Without it, a candidate with a stale log (missing committed entries) could be elected. It would then overwrite followers' logs with its own, causing committed entries to be lost. This would violate the fundamental safety guarantee of Raft.

"At least as up-to-date" is defined as: the candidate's last log entry has a higher term, OR the same term with a higher or equal index. This comparison ensures the candidate has all committed entries because any committed entry has been replicated to a majority, and the candidate must get votes from a majority.
</details>

### Question 18
**A 5-node Raft cluster has nodes A, B, C, D, E. Node A is the leader in term 3. A network partition separates {A, B} from {C, D, E}. Describe what happens step by step.**

<details>
<summary>Answer</summary>

1. **Partition occurs.** A can only communicate with B; C, D, E can only communicate with each other.
2. **Leader A continues sending heartbeats** to B (succeeds) and C, D, E (fails/times out).
3. **Leader A cannot commit new entries** because it can only replicate to 2 out of 5 nodes (A and B), which is not a majority. Writes to A's partition will hang.
4. **C, D, or E times out** (no heartbeats from A) and starts an election in term 4.
5. **The new candidate gets votes from 3 of 5 nodes** (e.g., C, D, E). This is a majority, so a new leader is elected in {C, D, E}.
6. **New leader serves reads and writes** with the majority partition.
7. **When the partition heals,** A receives a message with term 4 (> its term 3), immediately steps down to follower. B also steps down.
8. **A and B's uncommitted entries are overwritten** by the new leader's log. Any entries A replicated only to B (not committed) are lost.
9. **The cluster converges** under the new leader.
</details>

### Question 19
**What is a "split vote" in Raft, and how does Raft mitigate it?**

<details>
<summary>Answer</summary>

A split vote occurs when multiple candidates start elections simultaneously and no candidate receives a majority of votes. For example, in a 4-node cluster, two candidates might each get 2 votes (including their own), and neither reaches the majority of 3.

Raft mitigates split votes using **randomized election timeouts**. Each node picks a random timeout between 150-300ms. This makes it unlikely that two nodes time out simultaneously. The node with the shorter timeout starts its election first and typically wins before the other times out.

If a split vote does occur, both candidates start new elections with new random timeouts. The probability of repeated split votes decreases exponentially with each round.
</details>

### Question 20
**Explain the difference between Paxos and Raft at a high level. Why was Raft created when Paxos already existed?**

<details>
<summary>Answer</summary>

**Paxos** (Lamport, 1989):
- Defines consensus for a single value (single-decree Paxos). Multi-Paxos extends it to a log.
- Roles: Proposers, Acceptors, Learners.
- No designated leader (any proposer can propose).
- The paper is notoriously difficult to understand.
- Multi-Paxos (for practical use) is underspecified; implementations vary significantly.

**Raft** (Ongaro & Ousterhout, 2014):
- Designed explicitly for understandability.
- Decomposes consensus into leader election, log replication, and safety.
- Strong leader: all writes go through the leader.
- Clearly specified for a log of commands (not just a single value).
- Membership changes, snapshotting, and log compaction are part of the specification.

Raft was created because Paxos, while correct, was too difficult to understand and implement correctly. Ongaro's user study showed that students learned Raft significantly faster and made fewer mistakes implementing it.
</details>

---

## Section 6: Clocks and Time

### Question 21
**Why can't we just use wall-clock timestamps to order events in a distributed system?**

<details>
<summary>Answer</summary>

Wall clocks in different machines are not perfectly synchronized. Even with NTP, clock skew can be tens of milliseconds. This means:

1. **Event A on Machine 1 at time 100ms** and **Event B on Machine 2 at time 101ms** may actually have occurred in the reverse order (B before A).
2. **Clock drift:** Clocks run at slightly different speeds. A clock synchronized at time 0 may be off by several ms after a few minutes.
3. **NTP jumps:** NTP can adjust clocks backward (or forward), causing timestamps to go backward.
4. **Last-writer-wins with wall clocks** can silently lose data: two concurrent writes to the same key; the one with the "later" timestamp wins, but "later" is meaningless across unsynchronized clocks.

This is why distributed systems use logical clocks (Lamport, vector) or hybrid approaches (HLC, TrueTime).
</details>

### Question 22
**A system uses vector clocks with three nodes. Node A's clock is [3,1,0], Node B's clock is [2,3,1]. Are these events concurrent? Explain how you determined this.**

<details>
<summary>Answer</summary>

Two vector clocks are **concurrent** if neither dominates (is element-wise >=) the other.

- A = [3, 1, 0]
- B = [2, 3, 1]

Comparison:
- A[0]=3 > B[0]=2 (A dominates in position 0)
- A[1]=1 < B[1]=3 (B dominates in position 1)
- A[2]=0 < B[2]=1 (B dominates in position 2)

Since A dominates in one position and B dominates in others, **neither vector dominates the other**. Therefore, these events are **concurrent** -- neither causally precedes the other.

If A were [3, 3, 1], then A would dominate B in all positions, meaning A happened after B.
</details>

### Question 23
**What is a Hybrid Logical Clock (HLC)? How does it differ from a pure Lamport clock and a vector clock?**

<details>
<summary>Answer</summary>

An HLC combines physical time with a logical counter:
- `hlc = (physical_time, logical_counter)`

**How it works:**
- For local events: `hlc = (max(local_physical_time, hlc.physical), 0)` if physical time advanced; otherwise increment the logical counter.
- For message receipt: take the max of physical times from local and received, adjust the logical counter accordingly.

**Compared to Lamport clock:** An HLC preserves causal ordering (like Lamport) but also stays close to physical time. A Lamport clock is an arbitrary counter with no relation to real time.

**Compared to vector clock:** An HLC is compact (2 values regardless of cluster size), while a vector clock grows with the number of nodes. However, HLC cannot detect all concurrent events like vector clocks can.

**Used by:** CockroachDB, YugabyteDB. It provides "good enough" time ordering without specialized hardware.
</details>

---

## Section 7: System Design and Architecture

### Question 24
**Design a distributed key-value store that supports both strong consistency reads and eventual consistency reads. Explain the read path for each.**

<details>
<summary>Answer</summary>

**Architecture:** Use Raft for replication with a leader and followers.

**Strong consistency read (linearizable):**
1. Client sends read to the leader.
2. Leader executes the ReadIndex protocol: records its current commit index, sends heartbeats to confirm it is still leader.
3. Once a majority acknowledge the heartbeat, leader waits until the recorded commit index is applied.
4. Leader reads from the state machine and returns.
5. This guarantees the read reflects all committed writes.

**Eventual consistency read:**
1. Client sends read to any node (including followers).
2. The node reads from its local state machine immediately.
3. The value may be stale if the follower has not yet applied recent committed entries.
4. This is fast (no coordination) but may return outdated data.

**Implementation:** Expose two API endpoints: `GET /strong/:key` (routes to leader with ReadIndex) and `GET /eventual/:key` (reads from any replica's local state). Let clients choose per-request based on their needs.
</details>

### Question 25
**CockroachDB, TiDB, and YugabyteDB all use Multi-Raft. Why not use a single Raft group for the entire database?**

<details>
<summary>Answer</summary>

A single Raft group has severe limitations:

1. **Write throughput bottleneck:** All writes must go through the single leader. The leader's CPU, disk I/O, and network become the bottleneck.
2. **No parallelism for unrelated operations:** Two writes to completely unrelated keys must be serialized through the same Raft log.
3. **Large log and state machine:** The entire database is one state machine. Snapshotting becomes enormous.
4. **No data locality:** All data is on all nodes. Cannot place data close to users geographically.

Multi-Raft solves all of these:
- Each range/region is an independent Raft group with its own leader.
- Different ranges can have leaders on different nodes, distributing write load.
- Unrelated writes to different ranges execute in parallel.
- Each range is small (64-512 MB), making snapshots fast.
- Ranges can be placed in specific geographic regions for latency.
</details>

### Question 26
**What is a sloppy quorum? When would you use one instead of a strict quorum?**

<details>
<summary>Answer</summary>

In a strict quorum (N=3, W=2, R=2), writes must reach 2 of the 3 designated replicas. If one designated replica is down, the write fails.

In a **sloppy quorum**, if a designated replica is down, the write goes to a non-designated node instead. That node stores the data with a "hint" indicating the intended recipient. When the intended node recovers, the data is forwarded (**hinted handoff**).

**Use a sloppy quorum when:**
- Availability is more important than consistency.
- You cannot tolerate write failures due to temporary node outages.
- Example: Amazon's shopping cart (Dynamo). Losing a write (item disappearing from cart) is worse than a stale read (item reappearing after deletion).

**Do NOT use a sloppy quorum when:**
- You need strong consistency. A sloppy quorum violates the R + W > N overlap guarantee because the "W" nodes are not the same as the designated "N" nodes.
</details>

### Question 27
**Explain the concept of Change Data Capture (CDC). How does a log-based CDC system like Debezium work?**

<details>
<summary>Answer</summary>

CDC captures row-level changes (inserts, updates, deletes) from a database and streams them to downstream consumers in real-time.

**Log-based CDC (Debezium):**
1. Debezium connects to the database's **replication log** (PostgreSQL WAL, MySQL binlog, MongoDB oplog).
2. It reads the log as a replication client, exactly like a database replica would.
3. Each change event is converted to a structured format (JSON/Avro) with before and after values.
4. Events are published to Kafka topics (one topic per table).
5. Downstream consumers (Elasticsearch, data warehouse, cache) subscribe and update their state.

**Advantages of log-based CDC:**
- Non-intrusive: no triggers, no polling, no schema changes.
- Complete: captures every change, including deletes.
- Ordered: events are in transaction-commit order.
- Low latency: milliseconds from commit to downstream availability.
</details>

### Question 28
**In a distributed join between two tables on different partitions, explain the "shuffle join" strategy. What determines its performance?**

<details>
<summary>Answer</summary>

In a shuffle join:
1. Both tables are repartitioned (reshuffled) by the join key.
2. Rows from both tables with the same join key hash go to the same node.
3. Each node performs a local join on its partition.

**Performance factors:**
- **Data volume:** The amount of data transferred across the network during repartitioning. This is the dominant cost.
- **Skew:** If the join key has skewed distribution (some keys appear far more than others), some nodes process much more data, creating a bottleneck.
- **Network bandwidth:** Shuffle moves data across the network; limited bandwidth increases latency.
- **Number of partitions:** More partitions mean more parallelism but also more coordination overhead.

**Optimization:** If one table is small, use a **broadcast join** instead (send the small table to all nodes). If both tables are already partitioned on the join key (co-located), no shuffle is needed at all.
</details>

---

## Bonus Questions

### Question 29
**What is the difference between a Raft "committed" entry and an "applied" entry?**

<details>
<summary>Answer</summary>

- **Committed:** The entry has been replicated to a majority of nodes. It is guaranteed to survive any single node failure and will never be overwritten. The leader has advanced its `commitIndex` to include this entry.
- **Applied:** The entry has been executed against the state machine (e.g., the key-value store has been updated). Application happens after commitment.

An entry is always committed before it is applied. There can be a delay between commitment and application (e.g., if the apply worker is busy). A read that requires linearizability must wait for the entry to be applied, not just committed.
</details>

### Question 30
**A Raft leader crashes and restarts. When it comes back, it is no longer the leader. How does it catch up with the new leader's log?**

<details>
<summary>Answer</summary>

1. The old leader restarts and initializes as a follower (it reads its persisted state: current term, voted for, and log entries).
2. It receives an AppendEntries RPC from the new leader with a higher term.
3. It updates its term and acknowledges the new leader.
4. If its log diverges from the new leader's log (it had uncommitted entries from its leadership), the new leader's AppendEntries will detect the mismatch via the `prevLogIndex/prevLogTerm` consistency check.
5. The new leader decrements `nextIndex` for this follower until it finds a matching log entry.
6. The old leader deletes any conflicting entries and appends the new leader's entries.
7. The old leader applies committed entries to its state machine.

Any entries the old leader had that were not committed (not replicated to a majority before the crash) are lost. This is correct because they were never committed and thus never acknowledged to clients.
</details>
