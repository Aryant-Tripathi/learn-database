# Module 4: Query Processing & Optimization -- Deep Dive

## Introduction

This document goes deeper into query execution models, optimizer internals, cardinality estimation challenges, and modern techniques like JIT compilation and adaptive query processing. While the core teaching covers *what* happens, this document explains *why* systems are designed this way and where the hard problems lie.

---

## 1. Query Execution Models

Once the optimizer produces a physical plan, the executor must actually run it. There are three major execution models, each with distinct performance characteristics.

### 1.1 The Volcano / Iterator Model

The Volcano model (also called the Iterator model or demand-driven model) was introduced by Goetz Graefe in 1994. It is the most widely used execution model in traditional row-oriented databases including PostgreSQL, MySQL, and SQLite.

**Core idea**: Every operator in the plan tree implements a simple interface:

```
open()   -- Initialize the operator
next()   -- Return the next tuple (or EOF)
close()  -- Clean up resources
```

Execution is *pull-based*: the root operator calls `next()` on its child, which calls `next()` on *its* child, and so on down to the leaf scan operators.

```mermaid
sequenceDiagram
    participant Root as Projection
    participant Join as Hash Join
    participant Left as Seq Scan (users)
    participant Right as Seq Scan (orders)

    Root->>Join: next()
    Join->>Left: next()
    Left-->>Join: tuple (id=1, name="Alice")
    Join->>Right: next()
    Right-->>Join: tuple (user_id=1, total=50)
    Join-->>Root: tuple (name="Alice", total=50)
    Root->>Join: next()
    Join->>Right: next()
    Right-->>Join: tuple (user_id=1, total=75)
    Join-->>Root: tuple (name="Alice", total=75)
    Root->>Join: next()
    Join->>Left: next()
    Left-->>Join: tuple (id=2, name="Bob")
    Note right of Join: Probe hash table for id=2...
```

**Advantages**:
- Simple and elegant: each operator is independent and composable.
- Memory-efficient: only one tuple is in flight at a time per pipeline.
- Supports pipelining: no need to materialize intermediate results (in many cases).
- Easy to implement and debug.

**Disadvantages**:
- High per-tuple overhead: virtual function calls for every single tuple.
- Poor CPU cache utilization: processing one tuple at a time means frequent cache misses.
- Branch prediction failures: the `next()` dispatch varies between operators.
- For OLTP with small result sets, the overhead is acceptable. For OLAP scanning millions of rows, it becomes a bottleneck.

### 1.2 The Materialization Model

The materialization model (also called the "push" model) processes one operator at a time, producing the *entire* output before passing it to the parent.

```mermaid
flowchart TD
    A["Step 1: Seq Scan users<br>Materialize all matching rows"] --> B["Step 2: Seq Scan orders<br>Materialize all matching rows"]
    B --> C["Step 3: Hash Join<br>Build hash table from users<br>Probe with orders<br>Materialize result"]
    C --> D["Step 4: Projection<br>Process all rows<br>Produce final result"]

    style A fill:#e1f5fe,stroke:#03A9F4
    style B fill:#e1f5fe,stroke:#03A9F4
    style C fill:#fff3e0,stroke:#FF9800
    style D fill:#c8e6c9,stroke:#4CAF50
```

**Advantages**:
- Lower per-tuple overhead: tight loops over arrays of data.
- Better cache locality: processes all data for one operator before moving on.
- Good for in-memory databases where I/O is not the bottleneck.

**Disadvantages**:
- High memory usage: entire intermediate results must be materialized.
- No pipelining: cannot produce output until the entire input is consumed.
- Latency to first row is high.

### 1.3 The Vectorized / Batch Execution Model

The vectorized model is a hybrid that processes data in *batches* (vectors) of typically 1000-4096 tuples rather than one at a time. Pioneered by MonetDB/X100 (now the basis for DuckDB, ClickHouse, and Velox).

```mermaid
flowchart LR
    A["Scan: produce<br>batch of 1024 rows"] --> B["Filter: process<br>batch, output<br>qualifying rows"]
    B --> C["Join: process<br>batch against<br>hash table"]
    C --> D["Project: compute<br>expressions on batch"]

    style A fill:#e1f5fe,stroke:#03A9F4
    style B fill:#fff3e0,stroke:#FF9800
    style C fill:#fce4ec,stroke:#E91E63
    style D fill:#c8e6c9,stroke:#4CAF50
```

**How it works**:
- Operators still use the iterator interface: `open()`, `next_batch()`, `close()`.
- But `next_batch()` returns a *vector* of tuples rather than a single tuple.
- Within each operator, tight loops process the entire batch.
- Data is typically stored in columnar format within the batch for SIMD-friendly processing.

**Advantages**:
- Amortizes function call overhead over many tuples.
- Enables SIMD (Single Instruction, Multiple Data) processing.
- Excellent cache utilization: data fits in L1/L2 cache within a batch.
- Retains the composability of the iterator model.
- 10-100x faster than tuple-at-a-time for analytical queries.

**Disadvantages**:
- More complex implementation.
- Batch size tuning matters.
- Not as beneficial for point queries (OLTP).

### 1.4 Comparison

```mermaid
graph TD
    subgraph "Per-tuple overhead"
        A1["Volcano: HIGH<br>virtual call per tuple"]
        A2["Materialization: LOW<br>tight loop"]
        A3["Vectorized: LOW<br>amortized over batch"]
    end

    subgraph "Memory usage"
        B1["Volcano: LOW<br>one tuple in flight"]
        B2["Materialization: HIGH<br>full intermediate results"]
        B3["Vectorized: MEDIUM<br>batch-sized buffers"]
    end

    subgraph "Best for"
        C1["Volcano: OLTP<br>small results"]
        C2["Materialization: In-memory<br>simple queries"]
        C3["Vectorized: OLAP<br>large scans"]
    end

    style A1 fill:#ffcdd2
    style A2 fill:#c8e6c9
    style A3 fill:#c8e6c9
    style B1 fill:#c8e6c9
    style B2 fill:#ffcdd2
    style B3 fill:#fff9c4
    style C1 fill:#e1f5fe
    style C2 fill:#e1f5fe
    style C3 fill:#e1f5fe
```

---

## 2. How PostgreSQL's Planner Works

PostgreSQL's optimizer (in `src/backend/optimizer/`) follows a well-defined architecture.

### 2.1 Overview

```mermaid
flowchart TD
    A["Query (after rewrite)"] --> B["subquery_planner()"]
    B --> C["Pull up sublinks/subqueries"]
    B --> D["Flatten join tree"]
    B --> E["Reduce WHERE/HAVING"]
    B --> F["grouping_planner()"]
    F --> G["query_planner()"]
    G --> H["make_one_rel()"]
    H --> I["Generate Access Paths<br>for each base relation"]
    I --> J["Generate Join Paths<br>(dynamic programming)"]
    J --> K["Set of Paths for<br>entire join tree"]
    K --> L["Pick cheapest path"]
    L --> M["create_plan()"]
    M --> N["Physical Plan Tree"]

    style A fill:#e1f5fe,stroke:#03A9F4
    style L fill:#fff3e0,stroke:#FF9800
    style N fill:#c8e6c9,stroke:#4CAF50
```

### 2.2 Paths vs Plans

PostgreSQL separates the concepts of **paths** and **plans**:

- A **Path** represents a possible way to compute a relation. It includes the execution strategy and estimated cost, but *not* the full execution details. Multiple paths exist for each relation.
- A **Plan** is the final executable tree, created from the cheapest path. Only one plan is created.

This separation allows the optimizer to explore many alternatives cheaply (paths are lightweight) before committing to the expensive plan construction.

### 2.3 Access Path Generation

For each base relation, PostgreSQL generates multiple access paths:

1. **Sequential scan path**: Always generated.
2. **Index scan paths**: One per usable index with matching WHERE clauses.
3. **Bitmap scan paths**: For conditions matching multiple index entries.
4. **TID scan path**: If the query filters on `ctid`.

Each path has an estimated `startup_cost` and `total_cost`.

### 2.4 Join Path Generation

For each pair (or set) of relations that can be joined, PostgreSQL considers:

- **Nested Loop Join**: Always considered. Best when inner side is small or has a fast index.
- **Hash Join**: Considered when there is an equijoin condition. Best for large unsorted inputs.
- **Merge Join**: Considered when inputs can be sorted on join keys. Best when both sides are large and sorted.

For each join method, PostgreSQL also considers parameterized paths (where the inner scan depends on values from the outer scan).

### 2.5 Join Ordering

- For up to `geqo_threshold` tables (default 12), PostgreSQL uses **dynamic programming** (System R style).
- For more tables, it switches to **GEQO** (Genetic Query Optimization), a genetic algorithm that avoids exponential blowup.

---

## 3. System R Optimizer and Selinger-Style Optimization

The System R optimizer, published in the seminal 1979 paper by Selinger et al., established the framework that most relational optimizers still follow.

### 3.1 Key Innovations

1. **Cost-based optimization**: Use statistics rather than just heuristics.
2. **Dynamic programming for join ordering**: Optimal substructure -- the best plan for {A, B, C} can be constructed from the best plan for {A, B} joined with C (or other decompositions).
3. **Interesting orders**: A plan that produces sorted output may not be the cheapest, but could avoid a later sort. Track "interesting orders" and keep the cheapest plan for each interesting order.

### 3.2 Interesting Orders

Consider:
```sql
SELECT * FROM users u JOIN orders o ON u.id = o.user_id ORDER BY u.id;
```

Plan A: Hash Join (cost 100) + Sort (cost 50) = 150
Plan B: Merge Join (cost 130, output sorted by u.id) = 130

Even though the Hash Join is cheaper in isolation, the Merge Join is better overall because it produces output in the needed order.

```mermaid
flowchart TD
    subgraph "Plan A: Total cost 150"
        A1["Sort by u.id<br>cost: 50"] --> A2["Hash Join<br>cost: 100"]
        A2 --> A3["Scan users"]
        A2 --> A4["Scan orders"]
    end

    subgraph "Plan B: Total cost 130"
        B1["Merge Join<br>cost: 130<br>(output sorted by u.id)"] --> B2["Index Scan users<br>(sorted by id)"]
        B1 --> B3["Sort orders by user_id<br>cost: 30"]
        B3 --> B4["Scan orders"]
    end

    style A1 fill:#ffcdd2,stroke:#f44336
    style B1 fill:#c8e6c9,stroke:#4CAF50
```

### 3.3 The DP Table

The optimizer maintains a table mapping each *set of relations* to the cheapest plan(s) for that set. For each interesting order, it keeps the cheapest plan producing that order.

```
{users}:
  - any order: SeqScan (cost 10)
  - sorted by id: IndexScan on pk_users (cost 12)

{orders}:
  - any order: SeqScan (cost 20)
  - sorted by user_id: IndexScan on idx_orders_user_id (cost 22)

{users, orders}:
  - any order: HashJoin(SeqScan users, SeqScan orders) (cost 35)
  - sorted by users.id: MergeJoin(IdxScan users, Sort(SeqScan orders)) (cost 40)
```

---

## 4. Cardinality Estimation: The Hard Problem

Cardinality estimation is widely considered the hardest problem in query optimization. Getting it wrong can cause the optimizer to choose catastrophically bad plans.

### 4.1 The Uniformity Assumption

The optimizer assumes values are uniformly distributed unless statistics say otherwise.

For `WHERE age > 25` on a column with range [18, 80]:
- Selectivity = (80 - 25) / (80 - 18) = 0.887

Reality: Age distributions are not uniform. There may be far more 25-35 year olds than 65-80 year olds. Histograms help, but only for single-column predicates.

### 4.2 The Independence Assumption

For multiple predicates, the optimizer assumes they are statistically independent:

```sql
WHERE city = 'San Francisco' AND state = 'California'
```

- sel(city = 'SF') = 0.01
- sel(state = 'CA') = 0.12
- Combined: 0.01 * 0.12 = 0.0012

But in reality, if city is SF then state is *always* CA. The true selectivity is 0.01, not 0.0012. The optimizer underestimates by 8.3x.

### 4.3 Why Estimation Errors Compound

In a plan with multiple joins, estimation errors multiply at each step:

```mermaid
graph TD
    A["Scan A<br>Estimated: 1000<br>Actual: 1000"] --> J1["Join A-B<br>Estimated: 100<br>Actual: 500<br>(5x error)"]
    B["Scan B<br>Estimated: 2000<br>Actual: 2000"] --> J1
    J1 --> J2["Join AB-C<br>Estimated: 10<br>Actual: 2500<br>(250x error!)"]
    C["Scan C<br>Estimated: 500<br>Actual: 500"] --> J2

    style J1 fill:#fff3e0,stroke:#FF9800
    style J2 fill:#ffcdd2,stroke:#f44336
```

A 5x error at the first join becomes a 250x error at the second join. With such errors, the optimizer may choose nested loop joins where hash joins would be orders of magnitude faster.

### 4.4 Mitigations

- **Multivariate statistics**: PostgreSQL 10+ supports `CREATE STATISTICS` for correlated columns.
- **Join sampling**: Sample from the actual join to estimate cardinality.
- **Bayesian estimation**: Use conditional probabilities rather than independence.
- **Learned cardinality estimation**: Use machine learning models trained on query feedback (active research area).

```sql
-- PostgreSQL extended statistics for correlated columns
CREATE STATISTICS city_state_stats (dependencies) ON city, state FROM addresses;
ANALYZE addresses;
```

---

## 5. Adaptive Query Processing

Traditional optimizers commit to a plan before execution begins. Adaptive techniques adjust *during* execution.

### 5.1 Adaptive Join Methods

If the optimizer estimated 100 rows but sees 100,000 during execution, it could switch from nested loop to hash join mid-query.

### 5.2 Adaptive Cursor Prefetching

Start fetching additional pages when actual cardinality exceeds estimates.

### 5.3 PostgreSQL's Approach

PostgreSQL does not (as of v17) support mid-execution plan changes. However:

- It uses **parameterized paths** that adapt to runtime values.
- **Custom plans vs generic plans** for prepared statements adapt based on observed performance.
- The **Incremental Sort** operator (v13+) adapts sorting strategy based on pre-existing order.

### 5.4 Oracle's Approach

Oracle 12c introduced **Adaptive Plans**:

```mermaid
flowchart TD
    A["Start Execution"] --> B{"Actual rows > threshold?"}
    B -->|No| C["Continue with Nested Loop"]
    B -->|Yes| D["Switch to Hash Join"]
    D --> E["Continue execution"]
    C --> E

    style B fill:#fff3e0,stroke:#FF9800
    style C fill:#e1f5fe,stroke:#03A9F4
    style D fill:#fce4ec,stroke:#E91E63
```

Oracle also has **Adaptive Statistics** that feed information from one execution into future plan choices.

### 5.5 SQL Server's Approach

SQL Server has **Adaptive Joins** (2017+) and **Interleaved Execution** for multi-statement table-valued functions, where it pauses optimization, executes part of the plan to get actual cardinalities, and resumes optimization with better estimates.

---

## 6. Prepared Statements and Plan Caching

### 6.1 The Problem

Planning is expensive -- for a complex query, it can take milliseconds. For OLTP workloads executing the same query thousands of times per second with different parameters, replanning every time is wasteful.

### 6.2 PostgreSQL's Strategy

```mermaid
flowchart TD
    A["PREPARE stmt AS SELECT ..."] --> B["First 5 executions:<br>Create custom plan<br>with known parameters"]
    B --> C{"Is generic plan<br>consistently not worse<br>than custom plans?"}
    C -->|Yes| D["Switch to generic plan<br>(plan once, reuse forever)"]
    C -->|No| E["Keep using custom plans<br>(replan each time)"]

    style D fill:#c8e6c9,stroke:#4CAF50
    style E fill:#fff3e0,stroke:#FF9800
```

- **Custom plan**: Planned with actual parameter values. Can use value-specific optimizations (e.g., choose index scan for rare values, seq scan for common values).
- **Generic plan**: Planned with unknown parameter values (using average statistics). Reused for all parameter values.

PostgreSQL automatically decides which strategy to use based on comparing estimated costs.

### 6.3 The Plan Cache Problem

Caching plans can cause issues:
- **Parameter sensitivity** ("parameter sniffing" in SQL Server): A plan optimal for one parameter value may be terrible for another.
- **Schema changes**: Cached plans must be invalidated when tables/indexes change.
- **Statistics changes**: After ANALYZE, old cached plans may be suboptimal.

---

## 7. JIT Compilation in Query Engines

### 7.1 The Problem with Interpretation

In the iterator model, executing each tuple involves:
1. Calling virtual functions for each operator.
2. Evaluating expressions via a tree-walking interpreter.
3. Type dispatching for each value.

For analytical queries processing millions of rows, this interpretation overhead dominates.

### 7.2 JIT Compilation Approach

Instead of interpreting the plan, compile it to native machine code:

```mermaid
flowchart LR
    A["Query Plan"] --> B["Code Generation<br>(emit LLVM IR)"]
    B --> C["LLVM Optimizer<br>(optimize IR)"]
    C --> D["LLVM Backend<br>(IR -> machine code)"]
    D --> E["Execute native<br>function"]

    style A fill:#e1f5fe,stroke:#03A9F4
    style B fill:#fff3e0,stroke:#FF9800
    style C fill:#f3e5f5,stroke:#9C27B0
    style D fill:#fce4ec,stroke:#E91E63
    style E fill:#c8e6c9,stroke:#4CAF50
```

### 7.3 PostgreSQL JIT (v11+)

PostgreSQL uses LLVM for JIT compilation. It JIT-compiles:

- **Expression evaluation**: WHERE clauses, computed columns, aggregations.
- **Tuple deforming**: Extracting column values from heap tuples.
- **Tuple projection**: Constructing output tuples.

It does *not* JIT-compile entire operators (the iterator framework remains interpreted).

Configuration:
```sql
SET jit = on;                    -- Enable JIT (default: on)
SET jit_above_cost = 100000;     -- Cost threshold to trigger JIT
SET jit_inline_above_cost = 500000;  -- Threshold for inlining
SET jit_optimize_above_cost = 500000; -- Threshold for LLVM optimization
```

When to use JIT:
- Beneficial for long-running analytical queries (seconds to minutes).
- Harmful for short OLTP queries (JIT compilation time exceeds query time).

### 7.4 DuckDB's Approach

DuckDB combines vectorized execution with a form of JIT. It uses:
- **Vectorized operators** for data-intensive operations.
- **Compiled pipelines** that fuse multiple operators.
- **Morsel-driven parallelism** for multi-core scaling.

### 7.5 Compilation vs Vectorization

This is an active debate in the database community:

| Aspect | Vectorized | Compiled (JIT) |
|--------|-----------|-----------------|
| Compilation time | None | Can be significant |
| Debugging | Easier (standard code) | Harder (generated code) |
| SIMD usage | Explicit, controlled | Compiler-dependent |
| Code complexity | Lower | Higher |
| Adaptability | Easy to switch strategies | Must recompile |
| Best case | Scan-heavy queries | Expression-heavy queries |

Modern systems increasingly combine both approaches.

---

## 8. Advanced Optimizer Topics

### 8.1 Subquery Optimization

Subqueries come in several forms, each with different optimization strategies:

```mermaid
graph TD
    A["Subquery Types"] --> B["Scalar Subquery<br>Returns one value"]
    A --> C["EXISTS Subquery<br>Returns boolean"]
    A --> D["IN/ANY Subquery<br>Set membership"]
    A --> E["Derived Table<br>Subquery in FROM"]

    B --> F["Execution: run once or<br>convert to join"]
    C --> G["Execution: semi-join<br>(stop at first match)"]
    D --> H["Execution: semi-join<br>or hash lookup"]
    E --> I["Execution: flatten into<br>outer query"]

    style A fill:#e1f5fe,stroke:#03A9F4
    style F fill:#c8e6c9,stroke:#4CAF50
    style G fill:#c8e6c9,stroke:#4CAF50
    style H fill:#c8e6c9,stroke:#4CAF50
    style I fill:#c8e6c9,stroke:#4CAF50
```

### 8.2 Common Table Expressions (CTEs)

PostgreSQL's CTE optimization has evolved:

- **Pre v12**: CTEs were *always* materialized (optimization fence). A CTE was computed once and its result stored, even if it would be better to inline it.
- **v12+**: CTEs are inlined if they are referenced only once and are not recursive. This allows the optimizer to push predicates into the CTE body.

```sql
-- In v12+, this CTE is inlined and the WHERE predicate is pushed down:
WITH active AS (SELECT * FROM users WHERE active = true)
SELECT * FROM active WHERE age > 25;

-- Effectively becomes:
SELECT * FROM users WHERE active = true AND age > 25;
```

### 8.3 Partition Pruning

For partitioned tables, the optimizer eliminates partitions that cannot contain matching rows:

```sql
-- Table orders is partitioned by order_date (monthly partitions)
SELECT * FROM orders WHERE order_date = '2025-06-15';
-- Only the June 2025 partition is scanned
```

PostgreSQL supports:
- **Static pruning** at plan time (constant values).
- **Dynamic pruning** at execution time (parameterized values, subquery results).

### 8.4 Parallel Query Execution

PostgreSQL can parallelize:
- Sequential scans (Parallel Seq Scan)
- Index scans (Parallel Index Scan)
- Hash Joins (Parallel Hash Join, shared hash table)
- Aggregations (Partial Aggregate + Gather + Finalize Aggregate)
- Sorts (currently limited)

```mermaid
graph TD
    A["Gather / Gather Merge"] --> B["Worker 1:<br>Parallel Seq Scan<br>+ Filter"]
    A --> C["Worker 2:<br>Parallel Seq Scan<br>+ Filter"]
    A --> D["Worker 3:<br>Parallel Seq Scan<br>+ Filter"]
    A --> E["Leader also<br>participates"]

    style A fill:#f3e5f5,stroke:#9C27B0
    style B fill:#e1f5fe,stroke:#03A9F4
    style C fill:#e1f5fe,stroke:#03A9F4
    style D fill:#e1f5fe,stroke:#03A9F4
    style E fill:#e1f5fe,stroke:#03A9F4
```

---

## 9. Query Optimization in Distributed Databases

Distributed databases add new dimensions to optimization:

### 9.1 Additional Costs

- **Network transfer cost**: Moving data between nodes.
- **Serialization cost**: Converting data for network transmission.
- **Coordination cost**: Distributed locks, two-phase commit.

### 9.2 Data Placement Awareness

The optimizer must know where data resides:
- **Replicated tables**: Available on all nodes (joins are local).
- **Sharded tables**: Distributed by a shard key. Co-located joins (same shard key) are local; cross-shard joins require data movement.

### 9.3 Strategies

```mermaid
flowchart TD
    A["Distributed Join Strategy"] --> B{"Are join keys<br>co-located?"}
    B -->|Yes| C["Local join on<br>each node"]
    B -->|No| D{"Small table?"}
    D -->|Yes| E["Broadcast small table<br>to all nodes"]
    D -->|No| F["Reshuffle both tables<br>by join key"]

    style C fill:#c8e6c9,stroke:#4CAF50
    style E fill:#fff3e0,stroke:#FF9800
    style F fill:#ffcdd2,stroke:#f44336
```

---

## 10. Summary

| Concept | Key Insight |
|---------|------------|
| Volcano model | Simple, composable, but high per-tuple overhead |
| Vectorized execution | Processes batches for cache efficiency and SIMD |
| Materialization | Full intermediate results; good for in-memory |
| System R optimizer | DP for join ordering + interesting orders |
| Cardinality estimation | Independence assumption fails on correlated columns |
| Adaptive processing | Adjusting plans during execution based on actual data |
| Plan caching | Custom vs generic plans tradeoff |
| JIT compilation | Compiles expressions to native code for analytical queries |
| Distributed optimization | Must account for network costs and data placement |

The key theme is that query optimization is fundamentally about making good decisions under uncertainty. Statistics are imperfect, assumptions are violated, and workloads are unpredictable. Modern systems use a combination of careful engineering, statistical methods, and increasingly, machine learning to navigate this uncertainty.
