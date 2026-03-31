# Module 8: Join Algorithms & Query Execution -- Questions

## Conceptual Questions

### Q1: Nested Loop Join Cost Analysis

You have two relations:
- **R**: 2000 pages, 100 tuples per page (200,000 tuples total)
- **S**: 500 pages, 80 tuples per page (40,000 tuples total)
- **Buffer pool**: 52 pages available

Calculate the I/O cost for:
a) Simple (tuple-level) nested loop join with R as the outer relation
b) Simple nested loop join with S as the outer relation
c) Block nested loop join with R as outer (using all 52 buffer pages)
d) Block nested loop join with S as outer (using all 52 buffer pages)
e) Which configuration is cheapest?

<details>
<summary>Answer</summary>

a) **R outer, simple NLJ:** M + p_R * N = 2000 + 200,000 * 500 = **100,002,000 I/Os**

b) **S outer, simple NLJ:** N + p_S * M = 500 + 40,000 * 2000 = **80,000,500 I/Os**

c) **R outer, block NLJ (B=52):** M + ceil(M/(B-2)) * N = 2000 + ceil(2000/50) * 500 = 2000 + 40 * 500 = **22,000 I/Os**

d) **S outer, block NLJ (B=52):** N + ceil(N/(B-2)) * M = 500 + ceil(500/50) * 2000 = 500 + 10 * 2000 = **20,500 I/Os**

e) **S outer, block NLJ is cheapest** at 20,500 I/Os. The smaller relation should always be the outer in block NLJ.
</details>

---

### Q2: Sort-Merge Join Feasibility

You have 20,000 pages of data and 150 buffer pages. Can you sort this data in two passes? If not, how many passes do you need?

<details>
<summary>Answer</summary>

**Two-pass external sort** can handle up to B * (B-1) = 150 * 149 = **22,350 pages**. Since 20,000 < 22,350, **yes, two passes suffice**.

Phase 1 creates ceil(20,000/150) = 134 sorted runs. Phase 2 merges all 134 runs using 134 input buffers + 1 output buffer = 135 buffers needed. Since 135 <= 150, we can merge in one pass.

If we had more than 22,350 pages, we would need three passes:
- Phase 1: Create sorted runs of B pages each
- Phase 2: Merge B-1 runs at a time, producing larger runs
- Phase 3: Final merge of remaining runs
</details>

---

### Q3: Hash Join Memory Requirements

For Grace Hash Join on a build relation S with 1,000 pages:
a) What is the minimum number of buffer pages needed?
b) What happens if the data is heavily skewed (90% of tuples have the same join key)?
c) How does Hybrid Hash Join improve upon Grace Hash Join when extra memory is available?

<details>
<summary>Answer</summary>

a) We need B > sqrt(N_S) + 1 = sqrt(1000) + 1 approximately 33 buffer pages. We create B-1 = 32 partitions, each of about 1000/32 approximately 31 pages, which fits in the remaining buffer space.

b) With heavy skew, one partition will contain 90% of the tuples (900 pages), which does not fit in memory. The solution is **recursive partitioning**: re-partition the oversized partition with a different hash function. In extreme cases (all tuples have the same key), this degenerates and requires a nested loop join for that partition.

c) Hybrid Hash Join keeps **one partition in memory** during the partitioning phase. If we have B=200 buffer pages and 32 partitions, partition 0 stays in memory (~31 pages for the hash table), while partitions 1-31 are written to disk. R tuples matching partition 0 are joined immediately. This saves 2 * (size of partition 0) I/Os compared to Grace Hash Join.
</details>

---

### Q4: Why Can't Hash Join Handle Inequality Predicates?

Explain why hash join only works for equi-join predicates (=) and not for inequality predicates (<, >, <=, >=, BETWEEN).

<details>
<summary>Answer</summary>

Hash join works by hashing the join key to find matching tuples in O(1). Two equal values hash to the same bucket, so all matching tuples are co-located. However, for inequality predicates:

- Values that satisfy `a < b` do NOT hash to the same bucket. A hash function is designed to **scatter** similar values across different buckets.
- To find all tuples in S where `s.key < r.key`, we would need to probe **every** bucket in the hash table, defeating the purpose of hashing.
- There is no way to organize a hash table so that "all values less than X" are in a predictable set of buckets.

**Sort-merge join** handles inequality predicates because sorted order preserves the relational comparison. Similarly, index nested loop join with a B+ tree supports range predicates because B+ trees maintain sorted order.
</details>

---

### Q5: Execution Model Tradeoffs

Compare the three execution models (Iterator, Materialization, Vectorized) for the following query on a table with 100 million rows:

```sql
SELECT department, AVG(salary) FROM employees WHERE age > 30 GROUP BY department;
```

Which model would perform best and why?

<details>
<summary>Answer</summary>

**Iterator model:** Each tuple goes through Filter -> Project -> HashAggregate one at a time. For 100M rows, this means ~100M virtual function calls per operator boundary, plus constant L1 cache thrashing. The CPU spends most of its time on overhead, not computation.

**Materialization model:** SeqScan materializes all 100M rows. Filter materializes all qualifying rows (say 50M). Then HashAggregate processes the materialized result. This uses enormous memory for intermediate results but avoids per-tuple function call overhead.

**Vectorized model (best):** Processes batches of 1024-4096 tuples. The filter evaluates `age > 30` on a vector of age values using a tight SIMD-friendly loop. The hash aggregate updates accumulators in batch. Function call overhead is amortized over 1024 tuples. Cache utilization is excellent because columnar batches fit in L1/L2 cache.

The vectorized model is best because it combines the pipelining benefits of the iterator model (no full materialization) with the cache efficiency and SIMD potential of batch processing.
</details>

---

### Q6: Pipeline Breakers

Identify all pipeline breakers in this query plan and draw the pipeline boundaries:

```sql
SELECT c.name, SUM(o.total) as revenue
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN lineitem l ON o.id = l.order_id
WHERE l.quantity > 5
GROUP BY c.name
ORDER BY revenue DESC
LIMIT 10;
```

<details>
<summary>Answer</summary>

Assuming the optimizer chooses hash joins:

**Pipeline breakers:**
1. **Hash Join 1 (build side):** Building hash table on `customers` -- must consume all customer tuples before probing
2. **Hash Join 2 (build side):** Building hash table on the result of join 1 -- must consume all join results before probing
3. **HashAggregate:** Must consume all input to compute complete group aggregates
4. **Sort:** Must consume all aggregate results before producing sorted output

**Pipelines:**
- Pipeline 1: SeqScan(customers) -> Hash Build for Join 1
- Pipeline 2: SeqScan(orders) -> Hash Probe Join 1 -> Hash Build for Join 2
- Pipeline 3: SeqScan(lineitem) -> Filter(qty>5) -> Hash Probe Join 2 -> HashAggregate
- Pipeline 4: Sort(revenue DESC) -> Limit 10 -> Output

LIMIT is NOT a pipeline breaker -- it simply stops pulling from the sort operator after 10 tuples.
</details>

---

### Q7: Bloom Filter Benefit

A hash join has a build side with 10,000 distinct join keys. The probe side has 1 million tuples, but only 5% have matching keys in the build side. A Bloom filter with 1% false positive rate is constructed.

a) How many probe tuples are eliminated by the Bloom filter?
b) How much memory does the Bloom filter use?
c) What is the speedup if probing the hash table costs 100ns per tuple?

<details>
<summary>Answer</summary>

a) The 95% non-matching tuples are eliminated. The 1% FPR means 1% of non-matching tuples pass through falsely.
- True matches: 50,000 (5% of 1M)
- False positives: 0.01 * 950,000 = 9,500
- Tuples reaching hash join: 50,000 + 9,500 = **59,500**
- Tuples eliminated: 1,000,000 - 59,500 = **940,500** (~94%)

b) At ~10 bits per element for 1% FPR: 10,000 * 10 bits = 100,000 bits = **12.5 KB** (fits in L1 cache!)

c) Without Bloom filter: 1,000,000 * 100ns = 100ms
   With Bloom filter: Bloom check (~5ns) * 1M + Hash probe * 59,500 = 5ms + 5.95ms approximately **11ms**
   Speedup: approximately **9x**
</details>

---

### Q8: Sort-Merge Join vs. Hash Join

Under what circumstances would you prefer sort-merge join over hash join? List at least four scenarios.

<details>
<summary>Answer</summary>

1. **Input already sorted:** If both inputs come from index scans on the join key, the sort phase is free, making merge join cost just M + N (same as hash join but without hash table overhead).

2. **Output must be sorted:** If the query has `ORDER BY` on the join key, merge join produces sorted output naturally. Hash join would require an additional sort.

3. **Non-equi join predicates:** Merge join handles range predicates like `a.x BETWEEN b.lo AND b.hi` (with some modifications). Hash join cannot.

4. **Insufficient memory for hash table:** Sort-merge join's memory requirement during the merge phase is proportional to the number of sorted runs, not the size of a relation. External sort gracefully handles data larger than memory.

5. **Data skew:** Hash join can degrade with skewed data (large partitions). Sort-merge join is unaffected by key distribution -- it always does the same amount of work.

6. **Many duplicate join keys:** When many tuples share the same join key, the merge phase may need to "back up" the inner cursor, but this is bounded. Hash join may suffer from long chains in the hash table.
</details>

---

### Q9: Parallel Hash Join Design

You need to implement a parallel hash join for a 16-core machine. Compare the trade-offs of:
a) Partitioned parallel hash join
b) Shared hash table with lock-free insertion

<details>
<summary>Answer</summary>

**a) Partitioned parallel hash join:**
- Partition both relations into 16 (or more) partitions using radix partitioning
- Each thread independently processes one partition: builds hash table and probes
- **Pros:** Zero synchronization during join; each partition fits in L2 cache (cache-friendly); no contention
- **Cons:** Partitioning phase itself has random memory access patterns; extra I/O to write/read partitions; load imbalance if partitions are uneven; two full passes over the data

**b) Shared hash table with CAS:**
- All threads concurrently build into one hash table using compare-and-swap for insertions
- After a barrier, all threads concurrently probe (read-only -- no synchronization needed)
- **Pros:** Single pass over data; no partitioning overhead; works naturally with morsel-driven execution; handles skew
- **Cons:** CAS contention during build phase; false sharing on cache lines; hash table must be pre-allocated; memory bandwidth becomes the bottleneck

**Best practice:** Use shared hash table for moderate-sized build sides (fits in LLC) and partitioned join for very large build sides where cache locality is critical. DuckDB and Umbra use shared hash tables with morsel-driven parallelism.
</details>

---

### Q10: Adaptive Join Switching

SQL Server's Adaptive Join starts as a nested loop join and can switch to hash join at runtime. Describe:
a) What triggers the switch?
b) How is the already-processed data handled?
c) What is the overhead of this adaptivity?

<details>
<summary>Answer</summary>

a) The switch is triggered by an **adaptive threshold** based on the number of rows from the outer (probe) side. The optimizer pre-computes a crossover point where hash join becomes cheaper than nested loop join. If the actual row count exceeds this threshold during execution, the switch occurs.

b) Tuples already processed by the nested loop are used to build the hash table. The inner (build) side has already been materialized (since NLJ re-scans it), so we hash those tuples into a hash table. The remaining outer tuples then probe the hash table.

c) The overhead includes:
- Maintaining a row counter during NLJ execution
- The cost of building the hash table from already-materialized inner tuples (this work was "wasted" on NLJ re-scans)
- A brief pause during the switch
- Slightly larger plan representation

The overhead is minimal compared to the catastrophic cost of running a full NLJ when hash join would have been 100x faster.
</details>

---

### Q11: Iterator Model and LIMIT

Explain why the iterator model handles LIMIT clauses efficiently, while the materialization model does not.

<details>
<summary>Answer</summary>

In the **iterator model**, the LIMIT operator simply stops calling `Next()` on its child after receiving *k* tuples. This cascades down -- if the child is a sort, the sort still must consume all input (pipeline breaker), but if the child is a filter over a scan, only as many tuples as needed are scanned. For `SELECT * FROM large_table WHERE id < 100 LIMIT 10`, as soon as 10 matching rows are found, scanning stops.

In the **materialization model**, each operator fully produces its output before the parent starts. The filter would process ALL qualifying rows, materialize them, and then LIMIT would take the first 10 from the materialized result. If 1 million rows match the filter, all 1 million are materialized just to keep 10.

This is why OLTP databases (which frequently use LIMIT) universally use the iterator model or a variant of it.
</details>

---

### Q12: Join Ordering Problem

Given four tables A(100 rows), B(10,000 rows), C(1,000 rows), D(50,000 rows) with join predicates:
- A.id = B.a_id (selectivity 0.01)
- B.id = C.b_id (selectivity 0.001)
- C.id = D.c_id (selectivity 0.0001)

a) How many possible left-deep join orderings exist?
b) Why can't we try all possible orderings for 15 tables?
c) What heuristic would you use to pick a good ordering?

<details>
<summary>Answer</summary>

a) For 4 tables, there are 4! = **24** left-deep orderings (e.g., A-B-C-D, A-C-B-D, B-A-C-D, ...). Including bushy trees, there are many more.

b) For 15 tables: 15! = 1,307,674,368,000 left-deep orderings. Even at 1 microsecond per cost evaluation, this would take 15 days. Including bushy trees, the number is astronomically larger.

c) Heuristics:
- **Start with the smallest table** (A at 100 rows) to minimize intermediate result sizes
- **Push selective joins first**: The A-B join with selectivity 0.01 produces 100 * 10,000 * 0.01 = 10,000 rows
- Then join with C: 10,000 * 1,000 * 0.001 = 10,000 rows
- Then join with D: 10,000 * 50,000 * 0.0001 = 50,000 rows
- PostgreSQL uses dynamic programming (enumerating subsets) for up to `join_collapse_limit` tables and a genetic algorithm (GEQO) beyond that.
</details>

---

### Q13: Hash Table Sizing

You are implementing an in-memory hash join. The build side has 500,000 tuples, each 64 bytes. Your hash table uses chaining with 8-byte pointers.

a) How much memory does the hash table need?
b) What load factor should you target and why?
c) How does the hash table size affect probe performance?

<details>
<summary>Answer</summary>

a) Tuple storage: 500,000 * 64 = 32 MB. Hash table entries (pointer + next pointer): 500,000 * 16 = 8 MB. With a load factor of 0.75, we need ~667K buckets * 8 bytes = 5.3 MB for the bucket array. **Total: ~45 MB**.

b) Target load factor: **0.5 to 0.75**. Lower load factors mean fewer collisions and faster probes but more memory. Higher load factors save memory but increase chain lengths, causing more cache misses during probing. For join hash tables where probe performance is critical, 0.5-0.7 is typical.

c) At load factor *alpha*, the expected chain length is *alpha* for a successful probe and *alpha* for an unsuccessful probe (with good hashing). At alpha=0.5, on average 1.5 key comparisons per probe. At alpha=2.0, on average 3 comparisons with significant cache miss overhead due to pointer chasing.
</details>

---

### Q14: Vectorized Filter Implementation

Implement a vectorized filter for the predicate `age > 30` on a columnar batch. How does it differ from the iterator approach?

<details>
<summary>Answer</summary>

**Iterator approach (per-tuple):**
```rust
fn next(&mut self) -> Option<Tuple> {
    loop {
        let tuple = self.child.next()?;  // virtual call
        if tuple.age > 30 {              // branch per tuple
            return Some(tuple);
        }
    }
}
```
Each tuple requires one virtual function call and one branch.

**Vectorized approach (per-batch):**
```rust
fn next_batch(&mut self) -> Option<Batch> {
    let batch = self.child.next_batch()?;  // one virtual call per 1024 tuples
    let age_col: &[i32] = batch.column("age");
    let mut sel = Vec::with_capacity(batch.len());

    // Tight loop -- no virtual calls, SIMD-friendly, branch-free with cmov
    for i in 0..batch.len() {
        if age_col[i] > 30 {
            sel.push(i);
        }
    }

    batch.set_selection(sel);
    Some(batch)
}
```

Key differences:
1. Function call overhead is amortized over 1024 tuples
2. The inner loop operates on a contiguous `i32` array -- fits in cache, SIMD-vectorizable
3. The compiler can use branchless conditional moves (cmov)
4. No data copying -- the selection vector acts as a filter mask
</details>

---

### Q15: Grace Hash Join Recursive Partitioning

A Grace Hash Join partitions the build relation into 32 partitions. After partitioning, partition 7 has 10,000 pages (due to skew) but only 500 buffer pages are available. Describe what happens and what the total cost will be.

<details>
<summary>Answer</summary>

Partition 7 (10,000 pages) does not fit in 500 buffer pages. The system performs **recursive partitioning**:

1. Re-read partition 7 (both R_7 and S_7) and repartition using a **different** hash function h2 into 499 sub-partitions.
2. Each sub-partition of S_7 should be about 10,000/499 approximately 20 pages -- fits in memory.
3. For each sub-partition i: build hash table from S_7_i, probe with R_7_i.

Cost for partition 7:
- Read S_7 + R_7: 10,000 + |R_7| pages
- Write sub-partitions: 10,000 + |R_7| pages
- Read sub-partitions for join: 10,000 + |R_7| pages
- Total for partition 7: 3 * (10,000 + |R_7|)

This adds one extra pass over the skewed partition. In the worst case (all tuples have the same key), recursive partitioning never helps and we must fall back to nested loop join for that partition.
</details>

---

### Q16: Interesting Orders

Consider this query:

```sql
SELECT a.id, b.name, c.value
FROM a JOIN b ON a.id = b.a_id
       JOIN c ON b.id = c.b_id
ORDER BY a.id;
```

Table `a` has an index on `id`. Explain how the concept of "interesting orders" affects the optimizer's choice between hash join and merge join for the first join (a JOIN b).

<details>
<summary>Answer</summary>

The optimizer notices that the final `ORDER BY a.id` creates an "interesting order" on `a.id`. It then considers two paths:

**Path 1: Hash Join**
- Cost: hash join cost + sort cost for ORDER BY
- Hash join does not preserve any order, so a full sort is needed at the end

**Path 2: Merge Join**
- Use the index on `a.id` to produce `a` tuples in sorted order
- Sort `b` on `b.a_id`
- Merge join produces output sorted on `a.id` (the interesting order)
- The subsequent ORDER BY is FREE because the data is already sorted

Even if the merge join itself is more expensive than the hash join, the total cost (merge join + no sort) may be lower than (hash join + sort). The optimizer tracks interesting orders through the plan and considers them when comparing plan alternatives.
</details>

---

### Q17: Semi-Join Optimization

Rewrite this correlated subquery as a semi-join and explain why it is faster:

```sql
SELECT * FROM orders o
WHERE o.total > 1000
  AND EXISTS (SELECT 1 FROM premium_customers pc WHERE pc.id = o.customer_id);
```

<details>
<summary>Answer</summary>

**Semi-join rewrite:**
```sql
SELECT o.* FROM orders o
SEMI JOIN premium_customers pc ON pc.id = o.customer_id
WHERE o.total > 1000;
```

**Why it's faster:** A regular inner join would produce duplicate order rows if a customer appears multiple times in premium_customers (unlikely here, but the optimizer must handle it). The correlated subquery, if executed naively, runs the inner query once per outer row.

A hash semi-join:
1. Builds a hash table on `premium_customers.id` (or better, a hash set since we don't need the actual tuples)
2. For each order, probes the hash set. On the **first match**, emits the tuple and moves on (no need to check for more matches)

This is faster than a full join because:
- The hash "table" is actually a hash **set** -- smaller, better cache performance
- We stop probing on first match (early termination)
- No duplicate output tuples
- PostgreSQL's optimizer automatically converts EXISTS subqueries to semi-joins
</details>

---

### Q18: Push vs. Pull Execution

Explain how a `LIMIT 10` clause would be handled differently in a push-based vs. pull-based execution engine. Which is simpler?

<details>
<summary>Answer</summary>

**Pull-based (iterator):** The LIMIT operator simply counts tuples received from `next()` and returns EOF after 10. The child operators are never asked for more tuples. This is simple and natural -- LIMIT is a trivial operator.

```
fn next():
    if count >= 10: return EOF
    tuple = child.next()
    count += 1
    return tuple
```

**Push-based:** The scan operator drives execution by pushing tuples through a pipeline of callbacks. When LIMIT has received 10 tuples, it must **signal the scan to stop**. This requires a cancellation mechanism:
- A return value from the callback (`Continue` or `Stop`)
- An exception/unwind mechanism
- A shared cancellation flag

```
fn consume(tuple):
    count += 1
    parent.consume(tuple)
    if count >= 10:
        return STOP    // Propagate back to scan
```

**Pull-based is simpler** for LIMIT. The scan naturally stops when nobody calls `next()`. In push-based, you need explicit cancellation, which complicates every operator in the pipeline.
</details>

---

### Q19: External Sort Cost

You have a file with 100,000 pages and 256 buffer pages available.

a) How many sorted runs does Phase 1 create?
b) Can the merge be completed in one pass?
c) If not, how many merge passes are needed?
d) What is the total I/O cost?

<details>
<summary>Answer</summary>

a) Phase 1 creates ceil(100,000 / 256) = **391 sorted runs**.

b) One merge pass requires B-1 = 255 input buffers + 1 output buffer. We have 391 runs but only 255 can be merged at once. **No, one pass is insufficient.**

c) Pass 2: Merge 255 runs at a time. ceil(391 / 255) = 2 groups. After pass 2, we have 2 runs. Pass 3: Merge 2 runs. **Total: 3 passes** (1 sort + 2 merge).

Wait, let me recalculate. The standard formula is:
- Pass 0 (sort phase): Creates ceil(N/B) = 391 runs
- Pass 1 (merge): Merge 255 at a time: ceil(391/255) = 2 runs
- Pass 2 (merge): Merge 2 remaining runs: 1 run (done)
- Total passes: 3

d) Each pass reads and writes all pages: 2 * 100,000 = 200,000 I/Os per pass. For 3 passes: **600,000 I/Os total** (or equivalently, 2 * 3 * N = 6N = 600,000).
</details>

---

### Q20: Morsel-Driven Parallelism

In a morsel-driven parallel hash join:
a) How is the build phase parallelized?
b) What happens at the boundary between the build pipeline and the probe pipeline?
c) How does this handle NUMA architectures?

<details>
<summary>Answer</summary>

a) **Build phase parallelism:** The build relation is divided into morsels (e.g., 10K tuples each). Each worker thread grabs a morsel and inserts its tuples into the shared hash table using lock-free operations (CAS on the linked list heads). The work is automatically load-balanced because threads grab morsels on demand.

b) **Pipeline boundary (global barrier):** After all build morsels are consumed, ALL threads must synchronize at a barrier. This ensures the hash table is complete before any probing begins. After the barrier, threads switch to the probe pipeline and grab morsels from the probe relation.

c) **NUMA awareness:**
- The morsel dispatcher preferentially assigns morsels to threads on the same NUMA node as the data
- Each NUMA node may have a local portion of the hash table, with the complete table replicated or distributed
- Thread-local aggregation buffers avoid cross-socket writes
- The HyPer paper shows that NUMA-aware morsel scheduling can improve performance by 2-4x on multi-socket machines compared to NUMA-oblivious scheduling
</details>

---

### Q21: Comparing Aggregation Strategies

Given a table with 10 million rows and the query `SELECT city, COUNT(*) FROM customers GROUP BY city`:

a) If there are 100 distinct cities, which aggregation strategy is better?
b) If there are 5 million distinct cities, which is better?
c) How does PostgreSQL decide?

<details>
<summary>Answer</summary>

a) **100 distinct cities -- HashAggregate wins.**
The hash table has only 100 entries -- fits easily in L1 cache. Processing is O(N) with near-perfect cache behavior. Sort-based would require O(N log N) to sort 10M rows, which is far more expensive.

b) **5 million distinct cities -- it depends on memory.**
The hash table needs space for 5M entries (~hundreds of MB). If this exceeds `work_mem`, the hash table must spill to disk (batched hash aggregate). In that case, sort-based aggregation might be better because external sort handles large data gracefully and produces ordered output.

However, if `work_mem` is sufficient for the hash table, HashAggregate is still O(N) vs. O(N log N) for sort.

c) PostgreSQL estimates the number of distinct groups using `pg_statistic` (n_distinct). It computes the hash table memory as `num_groups * width_per_entry`. If this exceeds `work_mem`, it considers GroupAggregate (sort-based). The decision is purely cost-based: whichever has the lower estimated total cost wins. You can see the decision in `EXPLAIN` output.
</details>

---

### Q22: Multi-Way Join Optimization

Consider a star schema query joining a fact table (1 billion rows) with 5 dimension tables (10K-1M rows each). What join strategy would you recommend?

<details>
<summary>Answer</summary>

**Recommended approach: Hash join with Bloom filter pushdown**

1. **Build hash tables** on all 5 dimension tables (they are small enough to fit in memory).
2. **Construct Bloom filters** from each hash table.
3. **Push Bloom filters down** to the fact table scan. Before a fact table tuple enters any join pipeline, test it against all 5 Bloom filters. A tuple that fails ANY filter is immediately discarded.
4. This can eliminate 90%+ of fact table tuples before they reach the first join.
5. Surviving tuples flow through a pipeline of hash join probes.

**Alternative: Semi-join reduction (for distributed systems)**
Send the distinct join keys from each dimension to the fact table's node. Filter the fact table locally before shipping data. This is the standard technique in distributed star-schema joins (e.g., Snowflake, BigQuery).

**Why not sort-merge?** Sorting a billion-row fact table is extremely expensive. Hash join avoids this cost entirely.

**Why not nested loop?** Even with indexes, probing 5 indexes per fact tuple means 5 billion index lookups -- the overhead is prohibitive.
</details>
