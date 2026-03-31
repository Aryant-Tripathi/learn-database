# Module 4: Query Processing & Optimization -- Quiz Questions

## Instructions

Answer the following questions to test your understanding of query processing, optimization, and execution. Questions range from foundational to advanced. Try to answer from memory before checking the teaching materials.

---

## Section 1: Query Pipeline Fundamentals

### Question 1: Pipeline Stages

List the stages a SQL query passes through from raw text to result set, in order. Briefly describe the responsibility of each stage.

<details>
<summary>Answer</summary>

1. **Lexer/Tokenizer**: Breaks raw SQL string into a stream of typed tokens (keywords, identifiers, literals, operators, punctuation).
2. **Parser**: Validates the token stream against the SQL grammar and produces an Abstract Syntax Tree (AST).
3. **Semantic Analyzer**: Resolves names (tables, columns), checks types, validates scopes (GROUP BY correctness, etc.). Annotates the AST with resolved metadata.
4. **Query Rewriter**: Transforms the AST -- expands views, flattens subqueries, folds constants, simplifies predicates.
5. **Logical Planner**: Converts the AST into a tree of relational algebra operators (scan, filter, project, join, aggregate, sort).
6. **Optimizer**: Explores equivalent logical plans, estimates their cost, and selects the cheapest one. May apply heuristic rules and cost-based join ordering.
7. **Physical Planner**: Converts the optimized logical plan into a physical plan by choosing concrete algorithms (e.g., hash join vs nested loop, seq scan vs index scan).
8. **Executor**: Executes the physical plan and produces result tuples.
</details>

### Question 2: AST vs Parse Tree

What is the difference between a parse tree (concrete syntax tree) and an abstract syntax tree (AST)? Why do databases use the AST rather than the parse tree?

<details>
<summary>Answer</summary>

A **parse tree** (concrete syntax tree) includes every token from the input, including punctuation, keywords, and grammar structure nodes. It mirrors the grammar rules directly.

An **AST** (abstract syntax tree) is a simplified, cleaned-up tree that retains only the semantically meaningful structure. It discards syntactic noise like parentheses, commas, and keywords that served only as grammar markers.

Databases use ASTs because:
- They are more compact and easier to traverse.
- The optimizer and planner need the logical structure of the query, not its syntactic details.
- Subsequent transformations (rewriting, planning) are simpler on ASTs.
</details>

### Question 3: Tokenizer Edge Cases

Given the SQL fragment `WHERE name <> 'O''Brien' AND age >= 21`, what tokens should the tokenizer produce? Pay attention to the escaped single quote.

<details>
<summary>Answer</summary>

```
Token(KEYWORD, "WHERE")
Token(IDENTIFIER, "name")
Token(NOT_EQUALS, "<>")
Token(STRING_LITERAL, "O'Brien")    -- escaped quote becomes single quote
Token(KEYWORD, "AND")
Token(IDENTIFIER, "age")
Token(GREATER_EQ, ">=")
Token(INTEGER_LITERAL, 21)
```

The `''` inside a SQL string literal is the standard way to escape a single quote. The tokenizer should recognize this and produce the string `O'Brien` (with one quote, not two).
</details>

---

## Section 2: Semantic Analysis

### Question 4: Name Resolution

Consider the query:
```sql
SELECT name, total FROM users JOIN orders ON id = user_id;
```
What specific error(s) should the semantic analyzer detect?

<details>
<summary>Answer</summary>

The semantic analyzer should report an **ambiguous column reference** for `id`. Both `users` and `orders` likely have an `id` column, and the reference is unqualified. Similarly, `name` and `total` might exist in both tables (depending on schema). The `user_id` reference is likely unambiguous if only `orders` has that column.

The fix is to qualify the references: `users.id = orders.user_id`, `users.name`, `orders.total`.
</details>

### Question 5: Type Checking

What type errors should the semantic analyzer catch in the following query?
```sql
SELECT name + 10, SUM(email) FROM users WHERE age = 'thirty';
```

<details>
<summary>Answer</summary>

1. `name + 10`: Cannot add a string/varchar column to an integer (type mismatch for the `+` operator).
2. `SUM(email)`: `SUM` is a numeric aggregate function; `email` is a varchar column. This is a type error.
3. `age = 'thirty'`: Comparing an integer column to a string literal. The system may either raise an error or attempt an implicit cast (which would fail at runtime if 'thirty' cannot be converted to an integer).
</details>

---

## Section 3: Query Rewriting

### Question 6: View Expansion

A view is defined as:
```sql
CREATE VIEW high_value_orders AS
    SELECT * FROM orders WHERE total > 1000;
```
What does the query `SELECT * FROM high_value_orders WHERE status = 'shipped'` look like after view expansion? Can the rewriter further simplify it?

<details>
<summary>Answer</summary>

After view expansion:
```sql
SELECT * FROM (SELECT * FROM orders WHERE total > 1000) AS high_value_orders
WHERE status = 'shipped';
```

The rewriter can further simplify by merging the subquery:
```sql
SELECT * FROM orders WHERE total > 1000 AND status = 'shipped';
```

This is a form of subquery flattening combined with predicate merging.
</details>

### Question 7: Predicate Pushdown

Given the query:
```sql
SELECT u.name, o.total
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.age > 25 AND o.total > 100;
```
Show the logical plan tree before and after predicate pushdown. Which predicates can be pushed and where?

<details>
<summary>Answer</summary>

**Before pushdown:**
```
Project(u.name, o.total)
  Filter(u.age > 25 AND o.total > 100)
    Join(u.id = o.user_id)
      Scan(users u)
      Scan(orders o)
```

**After pushdown:**
```
Project(u.name, o.total)
  Join(u.id = o.user_id)
    Filter(u.age > 25)
      Scan(users u)
    Filter(o.total > 100)
      Scan(orders o)
```

- `u.age > 25` references only table `u`, so it is pushed below the join to the `users` scan.
- `o.total > 100` references only table `o`, so it is pushed below the join to the `orders` scan.
- The conjunction `AND` is split so each predicate can be pushed independently.
</details>

### Question 8: Subquery Flattening

Rewrite this correlated subquery as a join:
```sql
SELECT * FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);
```

<details>
<summary>Answer</summary>

```sql
SELECT e.*
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary;
```

The correlated subquery (which would execute once per row in `employees`) is converted to a single aggregation subquery joined to the outer table. This is dramatically more efficient.
</details>

---

## Section 4: Relational Algebra

### Question 9: SQL to Algebra

Express the following SQL in relational algebra using the operators: selection (sigma), projection (pi), join (bowtie), and rename (rho):

```sql
SELECT e.name, d.dept_name
FROM employees e JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 50000;
```

<details>
<summary>Answer</summary>

```
pi_{name, dept_name} (
    sigma_{salary > 50000} (
        employees bowtie_{dept_id = id} departments
    )
)
```

Or, with predicate pushdown applied:
```
pi_{name, dept_name} (
    (sigma_{salary > 50000} employees) bowtie_{dept_id = id} departments
)
```
</details>

### Question 10: Algebraic Equivalences

Which of the following transformations are always valid?

A) `sigma_{A AND B}(R) = sigma_A(sigma_B(R))`
B) `sigma_A(R bowtie S) = sigma_A(R) bowtie S` (where A references only columns of R)
C) `pi_X(sigma_A(R)) = sigma_A(pi_X(R))` (where A references only columns in X)
D) `R bowtie S = S bowtie R`
E) `sigma_A(R UNION S) = sigma_A(R) UNION sigma_A(S)`

<details>
<summary>Answer</summary>

**All five are valid:**

A) **Valid**: Selection is conjunctive -- cascading selections is equivalent to a single selection with AND.
B) **Valid**: If predicate A only references columns from R, it can be pushed below the join to filter R first.
C) **Valid**: If A only uses columns in projection set X, the projection can be done before or after filtering. However, this only works if all columns needed by A are in X.
D) **Valid**: Join is commutative (for inner joins). The result set is the same regardless of order.
E) **Valid**: Selection distributes over union. Filtering each side of a union independently produces the same result as filtering after the union.
</details>

---

## Section 5: Cost Model and Statistics

### Question 11: Selectivity Estimation

A table `products` has 10,000 rows. The column `price` has a minimum of 5, maximum of 500, and 200 distinct values. Estimate the selectivity and result cardinality for each predicate:

A) `WHERE price = 99`
B) `WHERE price > 400`
C) `WHERE price BETWEEN 100 AND 200`
D) `WHERE price = 99 AND category = 'Electronics'` (category has 20 distinct values)

<details>
<summary>Answer</summary>

A) **Equality selectivity**: 1 / n_distinct = 1/200 = 0.005. Cardinality = 10,000 * 0.005 = **50 rows**.

B) **Range selectivity**: (max - value) / (max - min) = (500 - 400) / (500 - 5) = 100/495 = 0.202. Cardinality = 10,000 * 0.202 = **~2,020 rows**.

C) **Range selectivity**: (200 - 100) / (500 - 5) = 100/495 = 0.202. Cardinality = **~2,020 rows**.

D) **Combined selectivity** (independence assumption): sel(price=99) * sel(category='Electronics') = (1/200) * (1/20) = 0.00025. Cardinality = 10,000 * 0.00025 = **~2.5, rounded to 3 rows**.

Note: In practice, correlation between price and category would make the independence assumption inaccurate.
</details>

### Question 12: Cost Comparison

Given a table with 1,000 pages and 100,000 rows, and a B-tree index on column `status`:

- `seq_page_cost = 1.0`
- `random_page_cost = 4.0`
- `cpu_tuple_cost = 0.01`

Compare the cost of a sequential scan versus an index scan for `WHERE status = 'active'` when:
A) 1% of rows match (1,000 rows)
B) 50% of rows match (50,000 rows)

<details>
<summary>Answer</summary>

**Sequential scan cost** (same regardless of selectivity):
- I/O: 1,000 pages * 1.0 = 1,000
- CPU: 100,000 * 0.01 = 1,000
- **Total: 2,000**

**Index scan cost:**

A) 1% selectivity (1,000 matching rows):
- Index traversal: ~3 levels * 4.0 = 12 (negligible)
- Heap fetches: ~1,000 * 4.0 = 4,000 (worst case: each row on different page)
- With correlation/clustering: maybe ~100 pages * 4.0 = 400
- CPU: 1,000 * 0.01 = 10
- **Worst case: ~4,012; Best case: ~412**

B) 50% selectivity (50,000 matching rows):
- Heap fetches: up to 50,000 * 4.0 = 200,000 (random I/O)
- Even with clustering: ~500 pages * 4.0 = 2,000
- CPU: 50,000 * 0.01 = 500
- **Worst case: ~200,500; Best case: ~2,500**

**Conclusion**: Index scan wins easily for 1% selectivity. For 50% selectivity, sequential scan (cost 2,000) is almost always better than index scan (cost 2,500-200,500) due to random I/O overhead.
</details>

### Question 13: Histograms

Why are equi-depth histograms preferred over equi-width histograms for cardinality estimation?

<details>
<summary>Answer</summary>

**Equi-width histograms** use buckets of equal value range (e.g., 0-10, 10-20, ...). If data is skewed, some buckets may contain most of the data while others are nearly empty, leading to poor estimates within dense buckets.

**Equi-depth histograms** use buckets containing roughly the same number of rows. This adapts to data distribution: dense regions get more, narrower buckets while sparse regions get fewer, wider buckets. This provides:
- More uniform error bounds across the value range.
- Better estimates for range queries in skewed distributions.
- More efficient use of limited bucket space.

PostgreSQL uses equi-depth histograms (stored in `pg_stats.histogram_bounds`).
</details>

---

## Section 6: Join Ordering

### Question 14: Join Order Enumeration

For a query joining 4 tables A, B, C, D with the following estimated sizes after filtering:
- A: 100 rows
- B: 10,000 rows
- C: 500 rows
- D: 50 rows

And the following join predicates: A-B, B-C, C-D (a linear join graph).

What is the optimal left-deep join order, and why?

<details>
<summary>Answer</summary>

The optimal left-deep join order minimizes intermediate result sizes. The general heuristic is to start with the smallest tables and most selective joins:

**Best order: D -> C -> A -> B** (or variations, depending on join selectivities)

Reasoning:
1. Start with `D` (50 rows) joined to `C` (500 rows) via C-D predicate. Estimated result: small.
2. Next join with `A` (100 rows). Still manageable intermediate result.
3. Finally join with `B` (10,000 rows).

The key principle: keep intermediate results small by joining smaller tables first and using the most selective joins first. The worst order would be starting with B (10,000 rows) cross-joined or loosely joined with another large table.

In practice, the optimizer uses cost-based dynamic programming considering actual join selectivities, not just table sizes.
</details>

### Question 15: Dynamic Programming Complexity

How many subsets does the dynamic programming join enumerator consider for N tables? Why does PostgreSQL switch to GEQO for more than 12 tables?

<details>
<summary>Answer</summary>

The DP enumerator considers all non-empty subsets of N tables: **2^N - 1** subsets.

For each subset of size k, it considers all ways to partition it into two non-empty subsets: roughly 2^(k-1) partitions.

Total work is approximately O(3^N) when accounting for all partitions across all subsets.

For N tables:
- N=5: 243 operations (trivial)
- N=10: 59,049 operations (fast)
- N=12: 531,441 operations (still manageable)
- N=15: 14,348,907 operations (getting slow)
- N=20: 3,486,784,401 operations (too slow)

PostgreSQL uses `geqo_threshold = 12` because beyond 12 tables, the exponential growth makes exhaustive DP too expensive. GEQO (Genetic Query Optimization) uses a genetic algorithm to heuristically search the plan space in polynomial time.
</details>

---

## Section 7: Execution Models

### Question 16: Volcano Model

In the Volcano (iterator) model, describe the flow of control and data when executing this plan:

```
Limit(5)
  Sort(age DESC)
    Filter(active = true)
      Seq Scan(users)
```

<details>
<summary>Answer</summary>

1. The client calls `next()` on the **Limit** operator.
2. Limit calls `next()` on **Sort**. Since Sort needs all input before producing output, Sort calls `next()` repeatedly on **Filter** until Filter returns EOF.
3. For each call, Filter calls `next()` on **Seq Scan**.
4. Seq Scan reads the next tuple from the `users` table and returns it to Filter.
5. Filter evaluates `active = true`. If the tuple matches, it returns it to Sort. If not, Filter calls `next()` on Seq Scan again.
6. Once Filter returns EOF, Sort has all qualifying tuples. It sorts them by `age DESC`.
7. Sort returns the first tuple (highest age) to Limit.
8. Limit counts it (1 of 5) and returns it to the client.
9. The client calls `next()` on Limit again. Limit calls Sort's `next()` to get the second tuple.
10. This repeats until Limit has returned 5 tuples, after which it returns EOF.

Key point: Sort is a **blocking operator** (also called a *pipeline breaker*) -- it must consume all input before producing any output. Limit, Filter, and Seq Scan are **non-blocking** (pipelining).
</details>

### Question 17: Vectorized vs Tuple-at-a-Time

Explain why vectorized execution is significantly faster than tuple-at-a-time execution for a query scanning 10 million rows with a simple filter. Focus on CPU-level effects.

<details>
<summary>Answer</summary>

With 10 million rows in tuple-at-a-time (Volcano):
- **10 million virtual function calls** through the `next()` interface for each operator.
- Each call involves function pointer dispatch, which causes **branch mispredictions**.
- Each tuple is processed individually, meaning the CPU cannot keep data in **L1/L2 cache** effectively.
- The tight loop within each operator cannot be optimized well by the compiler because the operator only processes one value.

With vectorized execution (batches of, say, 1024 tuples):
- Only **~10,000 calls** to `next_batch()` per operator (10M / 1024).
- Within each batch, a tight loop processes 1024 values. This loop is:
  - **SIMD-friendly**: The compiler can auto-vectorize, processing 4-8 values per instruction.
  - **Cache-friendly**: 1024 values fit in L1 cache.
  - **Branch-prediction friendly**: The loop pattern is predictable.
- Function call overhead is amortized over 1024 tuples instead of per-tuple.

The result is typically a **10-100x speedup** for scan-heavy analytical queries.
</details>

---

## Section 8: Advanced Optimization

### Question 18: Cardinality Estimation Failure

Consider a table with columns `country` and `city`. Explain why the independence assumption leads to massive estimation errors for the predicate `WHERE country = 'Japan' AND city = 'Tokyo'`. What can be done about it?

<details>
<summary>Answer</summary>

The independence assumption estimates:
- sel(country = 'Japan') = 1/200 (if 200 countries) = 0.005
- sel(city = 'Tokyo') = 1/10000 (if 10,000 cities) = 0.0001
- Combined: 0.005 * 0.0001 = 0.0000005

But in reality, `city = 'Tokyo'` is highly correlated with `country = 'Japan'`. The actual selectivity of the combined predicate is close to sel(city = 'Tokyo') = 0.0001, not the much smaller 0.0000005.

The optimizer underestimates by a factor of ~200x. This can cause it to choose nested loop joins (expecting very few rows) when hash joins would be appropriate.

**Solutions:**
1. **Multivariate statistics**: `CREATE STATISTICS (dependencies) ON country, city FROM locations;` (PostgreSQL 10+)
2. **Extended statistics with MCV lists**: PostgreSQL 12+ can create multivariate MCV lists.
3. **Column group statistics** tracking joint distributions.
4. **Learned cardinality estimation** using ML models trained on actual query results.
</details>

### Question 19: Plan Caching Pitfalls

A prepared statement `SELECT * FROM orders WHERE status = $1` uses a generic plan. Explain why this can be problematic if the `status` column is highly skewed (e.g., 90% of rows have status = 'completed').

<details>
<summary>Answer</summary>

A **generic plan** is optimized without knowing the parameter value. It uses average statistics to estimate selectivity.

With skewed data:
- `status = 'completed'` matches 90% of rows -- sequential scan is optimal.
- `status = 'pending'` matches 0.1% of rows -- index scan is optimal.

The generic plan might choose either strategy and use it for all parameter values. If it chooses:
- **Seq scan**: Fine for 'completed' (90%), but wastes effort for 'pending' (0.1%) -- scanning the entire table to find a handful of rows.
- **Index scan**: Fine for 'pending', but terrible for 'completed' -- 90% random I/O through an index is much worse than a sequential scan.

**Solution**: PostgreSQL's adaptive approach compares the estimated cost of the generic plan against custom plans for the first 5 executions. If the generic plan is consistently worse, it keeps using custom plans. But this heuristic is imperfect when parameter distributions vary widely.

In SQL Server, this is known as the **parameter sniffing** problem.
</details>

### Question 20: JIT Compilation Tradeoffs

When would JIT compilation of query expressions hurt performance rather than help?

<details>
<summary>Answer</summary>

JIT compilation hurts when:

1. **Short-running OLTP queries**: If a query takes 1ms to execute but JIT compilation takes 10ms, the total time increases 11x. The compilation cost is not amortized.

2. **Infrequently executed queries**: JIT benefits require amortizing compilation cost over many tuples processed. A one-off query touching 100 rows will not benefit.

3. **Compilation cache misses**: If the system compiles a new function for every query variant, compilation overhead accumulates without reuse.

4. **Complex plans with many expressions**: Compilation time scales with plan complexity. An extremely complex query might spend more time compiling than executing.

This is why PostgreSQL uses cost thresholds (`jit_above_cost = 100000`) -- JIT is only triggered for queries whose estimated cost exceeds the threshold, suggesting they will process enough data to benefit from compiled expressions.
</details>

---

## Section 9: PostgreSQL Specifics

### Question 21: EXPLAIN Interpretation

Interpret this EXPLAIN ANALYZE output and identify the performance problem:

```
Nested Loop  (cost=0.00..225000.00 rows=100 width=72)
             (actual time=0.050..4523.120 rows=95000 loops=1)
  ->  Seq Scan on orders  (cost=0.00..2500.00 rows=100000 width=36)
                          (actual time=0.010..15.340 rows=100000 loops=1)
  ->  Index Scan using pk_users on users  (cost=0.00..2.25 rows=1 width=36)
                                          (actual time=0.040..0.042 rows=1 loops=100000)
        Index Cond: (id = orders.user_id)
Planning Time: 0.150 ms
Execution Time: 4612.500 ms
```

<details>
<summary>Answer</summary>

**The problem**: The optimizer estimated the nested loop would produce **100 rows** but it actually produced **95,000 rows**. This is a 950x cardinality estimation error.

The nested loop performs 100,000 index lookups (one per order row, as shown by `loops=100000`). Each lookup is fast (0.042ms) but 100,000 of them add up to ~4,200ms.

A **Hash Join** would be far better here: build a hash table from `users` (one scan), then probe it while scanning `orders` (one scan). Total would be roughly 2 sequential scans instead of 100,000 random index lookups.

**Why it went wrong**: The optimizer estimated only 100 rows would come from orders (perhaps due to stale statistics or a misestimated filter). With 100 rows, nested loop with index scan is actually optimal. But with 100,000 rows, it is not.

**Fix**: Run `ANALYZE orders;` to update statistics. If the problem persists, investigate whether correlated columns or complex expressions are causing estimation errors.
</details>

### Question 22: Choosing Scan Methods

For each scenario, what scan method would PostgreSQL likely choose, and why?

A) `SELECT * FROM users WHERE id = 42` (primary key lookup)
B) `SELECT * FROM users` (no filter)
C) `SELECT COUNT(*) FROM users WHERE active = true` (active is indexed, 80% of rows are active)
D) `SELECT * FROM users WHERE age BETWEEN 20 AND 25` (age is indexed, 5% of rows match)
E) `SELECT * FROM users WHERE name LIKE '%smith%'` (name is indexed)

<details>
<summary>Answer</summary>

A) **Index Scan** on the primary key index. This is a point lookup returning at most 1 row. Cost is O(log N) for the index traversal plus 1 heap page fetch.

B) **Sequential Scan**. No filter means all rows are needed. Sequential I/O (reading pages in order) is always faster than reading all pages through an index.

C) **Sequential Scan**. Even though `active` is indexed, 80% of rows match. Using the index would involve reading 80% of heap pages via random I/O, which is more expensive than a single sequential scan. The index is not selective enough.

D) **Index Scan** (or possibly **Bitmap Index Scan**). 5% selectivity is selective enough for an index scan. If the data is not well-correlated with the index order, PostgreSQL may prefer a bitmap scan to sort the heap page accesses and avoid redundant page reads.

E) **Sequential Scan**. A `LIKE '%smith%'` pattern with a leading wildcard cannot use a standard B-tree index (the index is organized by prefix). A sequential scan with a filter is the only option unless a GIN/GiST trigram index (`pg_trgm`) exists.
</details>

---

## Section 10: Conceptual and Design Questions

### Question 23: Why Not Always Use Indexes?

A junior developer adds indexes to every column of every table, reasoning that more indexes always improve query performance. Explain at least three reasons why this is a bad idea.

<details>
<summary>Answer</summary>

1. **Write overhead**: Every INSERT, UPDATE, and DELETE must also update every index. More indexes means slower writes. For write-heavy OLTP workloads, this can be devastating.

2. **Space overhead**: Each index consumes disk space (and memory when cached). Indexing every column of a wide table can double or triple the total storage.

3. **Optimizer confusion**: With many indexes available, the optimizer must evaluate more access paths, increasing planning time. It may also make suboptimal choices if statistics are imperfect.

4. **Maintenance cost**: Indexes need periodic maintenance (REINDEX) to handle bloat. More indexes means more maintenance.

5. **Low selectivity**: Indexes on low-cardinality columns (e.g., boolean `active`) are rarely useful because the selectivity is too low for index scans to beat sequential scans.

6. **Memory pressure**: Each index competes for buffer pool space, potentially evicting more useful data pages from cache.
</details>

### Question 24: Optimizer Limitations

Name three categories of queries where cost-based optimizers commonly make poor decisions, and explain why.

<details>
<summary>Answer</summary>

1. **Queries with correlated predicates**: The independence assumption fails. `WHERE city='Tokyo' AND country='Japan'` produces severe underestimates. The optimizer may choose nested loops expecting very few rows.

2. **Queries with complex expressions or UDFs**: The optimizer has no statistics on the output of `WHERE my_function(col) > 10`. It falls back to a default selectivity (often 0.5 or 1/3), which may be wildly wrong.

3. **Queries over skewed data with parameter sensitivity**: Prepared statements with generic plans choose one strategy for all parameter values. For skewed distributions, no single plan is optimal for all values.

Additionally:
- **Queries with many joins (>12 tables)**: Exhaustive search is abandoned in favor of heuristics (GEQO), which may miss the optimal plan.
- **Queries with CTEs or complex subqueries**: Some optimizers cannot push predicates into CTEs or flatten complex subqueries.
</details>

### Question 25: Design a Cost Function

You are building a simple cost model for a database that runs on SSDs (where random I/O is only 1.5x slower than sequential, not 4x). How would you adjust PostgreSQL's default cost parameters?

<details>
<summary>Answer</summary>

PostgreSQL defaults:
- `seq_page_cost = 1.0`
- `random_page_cost = 4.0`

For SSDs, random reads are much closer to sequential reads in performance. Recommended SSD settings:
- `seq_page_cost = 1.0`
- `random_page_cost = 1.1` to `1.5`

Effects of this change:
- The optimizer becomes **more willing to use index scans**, since the penalty for random I/O is much lower.
- **Bitmap scans become less important** (their main benefit is converting random I/O to sequential I/O).
- **Nested loop joins with index lookups** become more competitive against hash joins.

You might also adjust:
- `effective_cache_size` upward if SSD performance makes caching less critical.
- `work_mem` can be somewhat smaller since spilling to SSD is less painful.

The key insight: the cost model must reflect the actual hardware characteristics. Using HDD-tuned parameters on SSD hardware causes the optimizer to underuse indexes.
</details>

---

## Scoring Guide

| Score | Level |
|-------|-------|
| 0-8 correct | Review the core teaching material |
| 9-15 correct | Good foundational understanding |
| 16-20 correct | Strong knowledge of query processing |
| 21-25 correct | Expert level -- ready to contribute to a database optimizer |
