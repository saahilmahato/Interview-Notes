# Database Interview Questions & Sample Answers
### For Senior/Principal Software Engineers

---

## SECTION 1: Foundational Concepts

---

**Q1: What is the difference between a database and a DBMS? Why does the distinction matter?**

**Sample Answer:**

A database is the organized collection of data itself — the actual stored information. A DBMS (Database Management System) is the software layer that manages, retrieves, stores, and manipulates that data. PostgreSQL, MySQL, and Oracle are DBMSs. The data they manage is the database.

The distinction matters because it separates concerns. The DBMS handles concurrency, durability, query optimization, and access control — complexities that would otherwise fall to application developers. When we talk about choosing a database, we're really choosing a DBMS plus a data model (relational, document, graph, etc.). Understanding this distinction prevents engineers from conflating storage format with query semantics, which becomes important when designing polyglot persistence architectures where a single application talks to multiple DBMSs, each managing its own database for different workloads.

---

**Q2: Explain the ACID properties. Don't just define them — tell me why each one is hard to implement.**

**Sample Answer:**

**Atomicity** — All operations in a transaction succeed or none do. Difficult to implement because systems can fail mid-transaction (power loss, crash). The engine must maintain a write-ahead log (WAL) or undo log so it can roll back partial changes on recovery. The hard part is making rollback itself reliable and ensuring the log is flushed to disk before the transaction commits.

**Consistency** — The database moves from one valid state to another, honoring all defined constraints. The difficulty is that "consistency" is partly the database's job (enforcing foreign keys, check constraints) and partly the application's job (business rules the DB doesn't know about). It's hard because you can define constraints that are individually sensible but collectively impossible to satisfy without careful ordering — think two tables that each have foreign keys pointing at each other.

**Isolation** — Concurrent transactions behave as if they were serialized. This is the hardest one. True serializability is expensive, so databases offer weaker isolation levels (read committed, repeatable read, snapshot isolation) that allow specific anomalies. Implementing even snapshot isolation requires multi-version concurrency control (MVCC), where the engine maintains multiple versions of rows so readers don't block writers. The tricky part is garbage collection — knowing when old row versions are safe to discard.

**Durability** — Once committed, data survives crashes. The challenge is the gap between writing to memory and writing to disk. Even "flushing" to disk is complicated by OS page caches, disk write caches, and the fact that fsync semantics vary across hardware. SSDs with volatile write caches can silently violate durability guarantees. Database engines use group commit to batch fsync calls, balancing throughput against latency.

---

**Q3: What is normalization? Walk me through the normal forms and tell me when you'd intentionally violate them.**

**Sample Answer:**

Normalization is the process of organizing a relational schema to reduce data redundancy and improve data integrity by following a set of progressive rules called normal forms.

**1NF** — Eliminate repeating groups. Every column holds atomic values; no arrays or comma-separated lists in a cell. Hard to violate accidentally in modern RDBMSs since they enforce column-level atomicity, but conceptually it means one fact per row.

**2NF** — No partial dependencies. In tables with composite primary keys, every non-key column must depend on the *entire* key, not just part of it. Violations typically mean you have entities mixed into one table.

**3NF** — No transitive dependencies. Non-key columns must depend only on the primary key, not on other non-key columns. Classic violation: storing `zip_code` and `city` in the same table when city is determined by zip.

**BCNF (Boyce-Codd)** — A stricter version of 3NF. Every determinant must be a candidate key. BCNF can sometimes eliminate 3NF-compliant schemas that still contain subtle anomalies.

**4NF and 5NF** — Deal with multi-valued dependencies and join dependencies. These are rarely discussed in practice but matter in academic rigor and very complex schemas.

**When to intentionally violate (denormalize):**

- **Read-heavy analytics workloads** — Joins are expensive at scale. A denormalized star schema in a data warehouse avoids joining dozens of tables for every query.
- **Caching derived values** — Storing a computed `order_total` column avoids summing line items on every read.
- **Reducing join depth** — In extremely high-throughput OLTP systems, even a single join can matter; embedding a frequently-read field from a foreign table avoids it.
- **Document storage patterns** — In Postgres with JSONB, you might intentionally embed nested data that would otherwise be normalized into child tables.

The key is: denormalize *deliberately*, *document why*, and *build safeguards* so writes keep derived data consistent.

---

**Q4: What is the difference between OLTP and OLAP systems? How does this distinction drive architectural decisions?**

**Sample Answer:**

**OLTP (Online Transaction Processing)** handles many short, concurrent, write-heavy operations — think inserting an order, updating account balance, recording a login. Queries are highly selective (fetch one or a few rows by primary key). Row-oriented storage is ideal because you need all columns for a given row quickly.

**OLAP (Online Analytical Processing)** handles complex aggregations over large datasets — think "revenue by region for the last 12 months." Queries scan millions of rows but only need a few columns. Column-oriented storage is ideal because you can scan just the relevant columns, skip irrelevant ones, and compress them heavily since values in a column are often similar.

**Architectural consequences:**

- OLTP DBs (PostgreSQL, MySQL, Oracle) are row-oriented and optimized for low-latency individual reads/writes with strong ACID guarantees.
- OLAP systems (BigQuery, Snowflake, Redshift, ClickHouse) are column-oriented, optimize for scan throughput, and often relax strict transaction semantics.
- Running analytics queries on an OLTP database destroys performance for transactional workloads — full-table scans lock buffer pools and compete with index lookups.
- This drives the classic **ETL pipeline**: data flows from OLTP → transformation → data warehouse/OLAP system, often on a schedule or via CDC (Change Data Capture).
- More modern architectures use **HTAP** (Hybrid Transactional/Analytical Processing) systems like TiDB or CockroachDB, or dual-engine approaches like MySQL + HeatWave, that attempt to serve both workloads from one system by maintaining both row and column stores.

---

## SECTION 2: Indexing

---

**Q5: Explain how a B-tree index works and why it's the default index type in most databases.**

**Sample Answer:**

A B-tree (Balanced Tree) is a self-balancing tree data structure where every node contains sorted keys and pointers to child nodes or data pages. The tree is kept balanced so all leaf nodes are at the same depth, guaranteeing O(log n) lookups, insertions, and deletions regardless of data distribution.

In a database context, the leaf nodes of a B-tree index contain the indexed column values along with pointers (row IDs or primary key values) to the actual heap rows. Non-leaf (internal) nodes contain separator keys to guide traversal. Because the tree fans out significantly at each level (nodes are sized to fill a disk page, often holding hundreds of keys), even billion-row tables are only 3-4 levels deep.

**Why it's the default:**

1. **Range queries** — Because keys are sorted, range scans (`WHERE age BETWEEN 25 AND 35`) are efficient: find the start point with one traversal, then scan leaf nodes sequentially.
2. **Equality queries** — O(log n) point lookups are fast.
3. **ORDER BY** — If the query orders by the indexed column, the index already provides sorted output, eliminating a sort step.
4. **Predictable performance** — Unlike hash indexes, which are O(1) for equality but O(n) for ranges and can degrade with collisions, B-trees have consistent, predictable performance across all access patterns.
5. **Concurrency-friendly** — B-trees have well-understood locking and MVCC integration stories.

**Tradeoffs:** Write amplification on insert/update because the tree must be rebalanced and potentially restructure pages. This is why write-heavy workloads sometimes benefit from LSM trees (Log-Structured Merge-trees) used by RocksDB and LevelDB.

---

**Q6: What is the difference between a clustered and a non-clustered index? What are the implications of each?**

**Sample Answer:**

A **clustered index** dictates the physical order of rows on disk. The table's data is stored in the leaf nodes of the index itself. There can only be one clustered index per table because the data can only be physically ordered one way. In InnoDB (MySQL), the primary key is always the clustered index. In SQL Server, you explicitly define one.

A **non-clustered index** is a separate structure that contains the indexed column values plus a pointer back to the actual row (either a heap row ID or the clustered index key). There can be many non-clustered indexes per table.

**Implications of clustered indexes:**

- Range scans on the clustered key are extremely fast because matching rows are physically adjacent on disk — one sequential read retrieves many rows.
- Inserts in random clustered key order (e.g., using a random UUID as a primary key in InnoDB) cause **page splits** and fragmentation, wrecking write performance. This is a major reason to use sequential IDs or ULIDs.
- In InnoDB, all non-clustered indexes contain the primary key value as the row pointer. If the primary key is large (e.g., a UUID), every non-clustered index gets bloated.

**Implications of non-clustered indexes:**

- A lookup that uses a non-clustered index may require a **bookmark lookup** (or key lookup): find the indexed value, then follow the pointer to the heap row to fetch remaining columns. This is expensive if done for many rows.
- This is why **covering indexes** matter so much — if all columns the query needs are in the index itself, the engine never touches the heap.
- Non-clustered indexes have write overhead: every insert/update/delete must update each non-clustered index.

---

**Q7: What is a covering index and when would you use one?**

**Sample Answer:**

A covering index is an index that contains all the columns a particular query needs — both the columns used for filtering/joining and the columns being selected. When the query engine can satisfy the entire query from the index alone without touching the base table rows, we say the index "covers" the query.

For example, if a query selects `user_id, email` and filters by `status = 'active'`, a composite index on `(status, user_id, email)` would cover it entirely. The engine reads only the index, which is typically much smaller and more cache-friendly than the full table.

**When to use them:**

- **High-frequency queries on large tables** where the extra index size is justified by read performance gains.
- **Reporting queries** that read only a subset of columns from a large table.
- **Avoiding bookmark lookups** — when the query planner's non-clustered index lookup is followed by expensive heap fetches, adding the missing column to the index eliminates the double-read.

**Tradeoffs:**

- Covering indexes tend to be wide, consuming more disk space and memory.
- Every write must update the additional index columns.
- It's easy to over-index — adding too many covering indexes for specific queries creates a maintenance burden and slows writes. You should identify the most critical, highest-frequency queries and optimize those.

A good signal: if EXPLAIN shows an "index scan" (covering) vs. an "index seek + key lookup," that's when adding columns to the index becomes compelling.

---

**Q8: What are partial indexes, expression indexes, and composite indexes? Give a use case for each.**

**Sample Answer:**

**Partial index** — An index built over a subset of rows, defined by a WHERE clause.

*Use case:* You have a `jobs` table with millions of rows, but only a small fraction have `status = 'pending'` — the only status you ever query on for job processing. A partial index on `(created_at) WHERE status = 'pending'` is tiny, fits in memory, and makes the polling query blazing fast. Full-table index would be wasteful.

**Expression index (Functional index)** — An index on the result of a function or expression rather than a raw column value.

*Use case:* You store emails in mixed case but always search case-insensitively (`WHERE lower(email) = lower($1)`). A regular index on `email` won't help because the function transforms the value at query time. An expression index on `lower(email)` indexes the transformed value so the query can use it. Similarly, indexing `(year, month)` extracted from a timestamp column for a query that filters by those derived values.

**Composite index** — An index on multiple columns. Column order matters critically.

*Use case:* A query filters on `tenant_id` and then sorts by `created_at`. A composite index on `(tenant_id, created_at)` lets the engine use the first column to narrow rows to a tenant, then returns them already sorted by date — eliminating a sort operation. The leftmost prefix rule applies: this index also helps queries that filter only on `tenant_id`, but it won't help queries that filter only on `created_at`.

---

## SECTION 3: Transactions & Concurrency

---

**Q9: Walk me through the SQL transaction isolation levels and the anomalies each one permits.**

**Sample Answer:**

SQL defines four isolation levels, each permitting progressively fewer anomalies at the cost of progressively higher concurrency overhead.

**Read Uncommitted** — Transactions can read data written by other transactions that haven't committed yet. Allows **dirty reads**: you read a value that gets rolled back, meaning you acted on data that never officially existed. Almost never used in practice. Only useful in very specific bulk-load diagnostics scenarios.

**Read Committed** — A transaction only sees data committed before each statement begins. Eliminates dirty reads. But allows **non-repeatable reads**: within the same transaction, reading the same row twice may yield different values if another transaction commits between the two reads. This is the default in PostgreSQL and Oracle.

**Repeatable Read** — The transaction sees a consistent snapshot of committed data as of its start. Reading the same row twice always returns the same value. Eliminates dirty reads and non-repeatable reads. But classically allows **phantom reads**: a re-executed range query may return additional rows inserted by another committed transaction. This is MySQL InnoDB's default (though InnoDB actually prevents phantoms via gap locking, making its repeatable read stronger than the SQL standard requires).

**Serializable** — Transactions appear to execute one at a time in some serial order. Prevents all anomalies including phantoms. Modern implementations use either strict two-phase locking (S2PL) or Serializable Snapshot Isolation (SSI), which PostgreSQL uses. SSI detects dangerous concurrency patterns and aborts transactions that would cause a non-serializable outcome, at the cost of some abort rate. The highest correctness guarantee, lowest throughput.

**Important nuance:** Most databases, including PostgreSQL, implement isolation via MVCC (Multi-Version Concurrency Control). Each transaction reads from a snapshot of the database at a specific point in time. This means readers and writers don't block each other — a fundamental advantage over pure lock-based approaches. However, MVCC creates **write-write conflicts** that are handled differently at each isolation level.

---

**Q10: What is a deadlock? How do databases detect and resolve them? What can engineers do to prevent them?**

**Sample Answer:**

A deadlock occurs when two or more transactions are each waiting for a lock held by the other, creating a circular dependency where none can proceed.

Classic example: Transaction A holds a lock on row 1 and wants row 2. Transaction B holds a lock on row 2 and wants row 1. Both wait forever.

**Detection:** Most databases use a **wait-for graph**. Each node is a transaction; a directed edge from A to B means A is waiting for a lock B holds. The engine periodically (or continuously, using a background thread) checks for cycles in this graph. When a cycle is detected, one transaction is chosen as the **victim** and is rolled back, releasing its locks and allowing the others to proceed. The choice of victim is usually based on which transaction is cheapest to roll back — often the one that's done the least work.

**Some databases** use **timeout-based detection** as a simpler alternative — if a transaction waits for a lock beyond a threshold, it's aborted. Less precise but simpler to implement.

**Prevention at the application level:**

1. **Consistent lock ordering** — Always acquire locks in the same global order. If all transactions lock rows in ascending primary key order, circular dependencies can't form. This is the most reliable preventive technique.
2. **Keep transactions short** — The shorter a transaction holds locks, the smaller the window for deadlocks. Do computation outside the transaction, then open a transaction, do the writes, commit immediately.
3. **Use SELECT FOR UPDATE early** — Acquire write locks at the beginning of the transaction rather than upgrading from read locks to write locks mid-transaction. Lock upgrades create deadlock windows.
4. **Reduce lock granularity** — Use row-level locking over table-level locking where possible, reducing contention.
5. **Retry logic** — Deadlocks are often unavoidable under sufficient concurrency. Applications should catch deadlock errors (specific error codes per database) and retry the transaction with exponential backoff.

---

**Q11: What is MVCC and why is it a major architectural advantage?**

**Sample Answer:**

MVCC — Multi-Version Concurrency Control — is a concurrency technique where the database maintains multiple versions of each row simultaneously. Instead of overwriting a row in place, a write creates a new version of the row, annotated with the transaction ID that created it. Old versions are retained until no active transaction could possibly need them.

When a read transaction starts, it receives a **snapshot ID** (typically a transaction ID). When it reads a row, it sees the most recent version of that row committed *before* its snapshot, ignoring any later versions. This means:

- **Readers never block writers** — A writer creating a new row version doesn't prevent readers from reading the old version.
- **Writers never block readers** — Readers don't wait for writers to finish.
- Only **write-write conflicts** can cause blocking (two transactions trying to modify the same row).

This is a massive advantage over pure lock-based systems (like classic Oracle without MVCC, or some older databases), where a read would have to wait for a write lock to be released.

**Consequences of MVCC:**

- **Vacuum/garbage collection** — Old row versions accumulate. PostgreSQL's `VACUUM` process reclaims dead tuples. If vacuuming falls behind (which can happen under heavy write load), tables bloat, and eventually transaction ID wraparound can threaten data integrity — a genuinely serious operational concern in PostgreSQL.
- **Long-running transactions are dangerous** — A transaction with an old snapshot ID prevents MVCC from cleaning up any row version that postdates its snapshot. One long-running OLAP query on a heavily-written table can cause massive table bloat.
- **Write amplification** — Every update is actually an insert of a new version plus marking the old version as dead, doubling I/O in some access patterns.

---

## SECTION 4: Query Execution & Optimization

---

**Q12: What is a query planner/optimizer and what does it do?**

**Sample Answer:**

A query planner (or query optimizer) is the component of a DBMS that takes a parsed SQL query and determines the most efficient physical execution plan for it. SQL is declarative — you say *what* you want, not *how* to get it. The planner figures out the *how*.

The planner's job involves several stages:

**Parsing and rewriting** — The SQL is parsed into an abstract syntax tree, then rewritten by applying logical transformations (e.g., subquery flattening, view expansion, constant folding).

**Plan enumeration** — The planner generates possible execution plans. For a join of N tables, there are N! possible join orderings. For large N, exploring all of them is combinatorially infeasible, so planners use dynamic programming (PostgreSQL), heuristics, or genetic algorithms (PostgreSQL's geqo for many-table joins) to prune the search space.

**Cost estimation** — Each candidate plan is assigned a cost estimate based on statistics about the data: table row counts, column value distributions (histograms), distinct value counts, correlation between columns, and index availability. The planner estimates how many rows each operation will produce and how much CPU and I/O each step will consume.

**Plan selection** — The lowest-cost plan is chosen and executed.

**Why this matters for engineers:**

- Statistics must be up-to-date. Stale statistics (from an un-analyzed table after a bulk load) cause the planner to make terrible choices. Running `ANALYZE` (PostgreSQL) or `UPDATE STATISTICS` (SQL Server) is critical after large data changes.
- The planner is not perfect. Understanding `EXPLAIN` output lets you identify when the planner underestimates cardinality (often due to correlated columns or complex predicates) and choose workarounds: hints (Oracle, SQL Server), partial indexes, rewriting the query, or updating statistics targets.

---

**Q13: What are the common types of joins at the physical execution level? When does the planner choose each one?**

**Sample Answer:**

At the logical level, joins are simple set operations. At the physical level, the query planner must choose an algorithm to execute them. The three main join algorithms are:

**Nested Loop Join**
The outer table is iterated row by row. For each outer row, the engine scans the inner table (or uses an index) to find matching rows. Complexity: O(N × M) without indexes, O(N × log M) with an index on the inner table.

*Best when:* The outer relation is small, and there's an efficient index on the join column of the inner relation. Good for OLTP with highly selective predicates returning few rows.

**Hash Join**
Phase 1 (build): The smaller table is scanned and a hash table is built in memory, keyed on the join column. Phase 2 (probe): The larger table is scanned; each row's join key is hashed and looked up in the hash table. Complexity: O(N + M).

*Best when:* Both tables are large, no useful index exists on the join column, and the build table fits in memory (or the planner can partition it across memory and disk in a grace hash join). Very common in OLAP/data warehouse queries.

*Risk:* If the hash table doesn't fit in memory, the engine spills to disk, causing a dramatic performance drop. `work_mem` in PostgreSQL controls how much memory a sort or hash operation can use before spilling.

**Merge Join (Sort-Merge Join)**
Both input relations are sorted on the join column. Then they're merged in a single linear scan — like merging two sorted arrays. Complexity: O(N log N + M log M) if sorting is needed, O(N + M) if inputs are already sorted.

*Best when:* Both inputs are already sorted (via index scan on the join column, or a preceding ORDER BY). Can be very efficient for large range joins. Also useful when the output itself needs to be sorted on the join key.

**The planner chooses based on:** Row count estimates, available indexes, available memory (`work_mem`), whether the query has additional ordering requirements, and parallelism settings.

---

**Q14: What does EXPLAIN ANALYZE tell you and what are the most important things to look for?**

**Sample Answer:**

`EXPLAIN ANALYZE` (PostgreSQL syntax; other databases have equivalents) actually executes the query and shows the execution plan annotated with real runtime statistics, not just estimates. It reveals:

**Plan nodes** — Each operation in the tree: sequential scan, index scan, hash join, sort, aggregate, etc.

**Estimated vs. actual rows** — The planner's row count estimate vs. what was actually returned. A large discrepancy (say, estimated 10 rows, actual 10,000) is a red flag — it means statistics are stale or the predicate involves correlated columns, causing the planner to make poor downstream decisions (wrong join order, wrong join algorithm). This is the single most important thing to investigate.

**Cost** — Shown as `(cost=startup..total)`. Startup cost is the cost before the first row is returned; total cost is the full cost. These are in arbitrary planner units, useful for comparing plans against each other.

**Actual time** — `(actual time=startup..total rows=N loops=L)`. Actual milliseconds. If a node executes in a loop (e.g., inner side of a nested loop), `loops` shows how many times it was called. Total time for that node is `time × loops`.

**Key things to look for:**

1. **Seq scans on large tables** — If you see a sequential scan on a table you expected to be indexed, either the index doesn't exist, the query isn't written to use it, or the planner decided the index scan would be more expensive (possible when selectivity is low). Investigate which.

2. **Row count estimation errors** — As noted above, a ≥10x difference between estimated and actual rows usually explains a poor plan choice.

3. **Sort operations on large datasets** — Especially if spilling to disk (`Sort Method: external merge Disk: 32MB`). Consider an index to pre-sort, or increase `work_mem`.

4. **Hash batches > 1** — In a hash join, `Batches: 4` means the hash table spilled to disk. Increasing `work_mem` or the hash join might perform differently.

5. **Buffer statistics** (with `EXPLAIN (ANALYZE, BUFFERS)`) — Shows shared buffer hits vs. reads. A high ratio of disk reads vs. buffer hits suggests the working set isn't fitting in cache.

---

## SECTION 5: Schema Design

---

**Q15: What are the tradeoffs between using surrogate keys (auto-increment IDs) vs. natural keys as primary keys?**

**Sample Answer:**

**Surrogate keys** are system-generated identifiers with no business meaning (auto-increment integers, UUIDs, ULIDs). **Natural keys** are attributes from the domain that uniquely identify an entity (email address, SSN, ISBN, composite business identifiers).

**Surrogate key advantages:**
- Immutable — Business attributes change (customers change their email), but a surrogate ID never needs to cascade updates through foreign keys.
- Simple join columns — A single integer is compact, cache-friendly, and fast for joins.
- Decoupled from business logic — The database layer doesn't need to know about domain rules.
- Easy to generate — Auto-increment or sequence generation is well-understood.

**Surrogate key disadvantages:**
- Meaningless in isolation — You always need to join to see what the record represents.
- Can hide data quality issues — Natural key violations (e.g., duplicate emails) aren't caught at the DB level if a surrogate key is the PK unless you add a separate unique constraint.

**Natural key advantages:**
- Self-documenting — You can reason about the data without joining.
- Enforces domain constraints inherently — A `(order_id, line_number)` composite natural key can't have duplicate line items.
- No extra unique constraint needed — The PK itself is the business constraint.

**Natural key disadvantages:**
- Keys can change — Cascading updates are painful and error-prone.
- Often composite — Multi-column natural keys make foreign keys verbose and joins more expensive.
- Can be long — A string natural key (URL, email) as a PK makes every index referencing it larger.

**Best practice:** Use surrogate keys as the PK for most tables, but *always* add unique constraints on natural key columns. Don't let the surrogate key substitute for enforcing business uniqueness rules.

**Special note on UUIDs:** Random UUIDs as clustered index keys in InnoDB are notorious for write amplification and page fragmentation because inserts go to random positions in the index. Sequential UUIDs (ULIDv7, UUID v7, or `gen_random_uuid()` in PostgreSQL with sequential properties) solve this by encoding time in the high bits, ensuring monotonically increasing values.

---

**Q16: How would you design a schema to handle soft deletes? What are the tradeoffs?**

**Sample Answer:**

Soft delete means marking a record as deleted without physically removing it from the database — typically via a `deleted_at` timestamp column (NULL means active, a timestamp means deleted) or a boolean `is_deleted` flag.

**Implementation options:**

**Option 1: `deleted_at` timestamp column**
Preferred over boolean because it captures *when* the deletion happened, which is often needed for auditing, and allows querying "deleted in the last 30 days."

**Option 2: Separate archive/history table**
Move deleted rows to a `_deleted` shadow table. Keeps the main table clean and fast. Restoring is an explicit move-back operation. Good when deleted records are large and rare.

**Option 3: Status enum**
A `status` column with values like `active`, `deleted`, `suspended`, `archived`. More flexible than binary but more complex to query.

**Tradeoffs and pitfalls:**

- **Every query must filter on `deleted_at IS NULL`** — This is the most dangerous pitfall. One forgotten WHERE clause returns deleted records. Mitigations: views that apply the filter, ORM-level default scopes (with care — they can be accidentally bypassed).
- **Unique constraints become complicated** — If `email` must be unique among active users, but deleted users with the same email should be allowed to re-register, a simple UNIQUE on `email` breaks. You need a partial unique index: `CREATE UNIQUE INDEX ON users(email) WHERE deleted_at IS NULL`.
- **Foreign key integrity** — If row B references soft-deleted row A, is that valid? Hard to enforce at the DB level. Application must handle this.
- **Index bloat and query performance** — The `deleted_at IS NULL` filter on a column that's NULL for 99% of rows is a great candidate for a partial index that covers only active rows.
- **GDPR/data regulation tension** — Soft delete is not the same as erasure. If regulations require data to be truly deleted, soft delete can create compliance issues unless combined with a scheduled hard-delete pass.

---

**Q17: What is the N+1 query problem and how do you solve it?**

**Sample Answer:**

The N+1 problem occurs when code fetches a list of N parent records with one query, then issues one additional query *per parent* to fetch associated child records — resulting in N+1 total queries instead of 1 or 2.

Example: Fetch 100 blog posts (1 query), then for each post, fetch the author's name (100 more queries) = 101 queries total. This is catastrophic at scale.

**Solutions:**

**JOIN / Eager loading** — Rewrite the query to JOIN the two tables in one query and return all data together. Most ORMs have eager loading primitives: `include(:author)` in ActiveRecord, `.Include(p => p.Author)` in Entity Framework, etc.

**Batched secondary query** — Fetch the parent IDs from the first query, then fetch all children with `WHERE parent_id IN (...)` in one second query. Two queries total. Good when JOIN would produce too many rows (many-to-many with large fan-out). This is how most ORM eager loading actually works under the hood.

**DataLoader pattern** — Used heavily in GraphQL. Requests for individual records are batched within a single request-response cycle into one IN-clause query. Solves the N+1 problem in graph-style resolution without requiring the resolver to know about its siblings.

**Denormalization** — For read-heavy cases, store frequently-needed child data directly on the parent record to avoid the join entirely. Tradeoff: consistency burden.

**Caching** — If the child records are stable (e.g., category names), caching them in application memory avoids database round trips entirely. Not a real fix for the underlying query pattern, but effective in practice.

The N+1 problem is one of the most common ORM-related performance bugs. A good signal is watching query logs and seeing the same query pattern repeated N times with only a different ID parameter.

---

## SECTION 6: Replication, Scaling, and Distribution

---

**Q18: Explain the difference between synchronous and asynchronous replication. What does each one sacrifice?**

**Sample Answer:**

In **synchronous replication**, the primary waits for the replica to confirm it has received and written (or at least acknowledged) the data before the write is considered committed and the client receives a success response. This guarantees zero data loss on failover — the replica is always current.

*Sacrifice:* Write latency increases because every write must wait for a network round-trip to the replica. If the replica is slow or the network is congested, the primary is slowed too. If the replica becomes unavailable, writes can stall or fail entirely — availability suffers.

In **asynchronous replication**, the primary commits and acknowledges the client without waiting for the replica. The replica applies changes as fast as it can, but there's a **replication lag** — a window where the replica doesn't yet have the latest data.

*Sacrifice:* **Durability on failover.** If the primary crashes, any writes not yet replicated are lost. Additionally, reads from replicas may return stale data — something application developers must explicitly design around.

**Semi-synchronous replication** (MySQL's offering) is a middle ground: the primary waits for at least one replica to acknowledge receipt of the transaction before committing. This ensures at least one copy of data exists outside the primary, but doesn't guarantee the replica has *applied* the write yet.

**Practical design decisions:**
- Financial systems and anything where losing committed data is catastrophic → synchronous replication for critical writes.
- High-throughput systems where write latency matters more than the small risk of losing a few seconds of writes → asynchronous.
- Many systems combine both: synchronous to one standby in the same AZ for HA, asynchronous to replicas in other regions for read scaling.
- **Read-your-writes consistency** is a critical concern with async replicas. If a user writes data and immediately reads it from a replica, they may not see their own write. Solutions: route reads to the primary for the same user session after a write, or track replication position and wait for the replica to catch up.

---

**Q19: What is the CAP theorem? Is it actually useful in practice?**

**Sample Answer:**

The CAP theorem (Brewer, 2000) states that a distributed data system can provide at most two of the following three guarantees simultaneously:

- **Consistency (C)** — Every read returns the most recent write or an error. (Not to be confused with the C in ACID.)
- **Availability (A)** — Every request receives a non-error response, even if it's not the most recent write.
- **Partition tolerance (P)** — The system continues operating when network partitions (message loss or delay between nodes) occur.

The classic conclusion: since network partitions are a fact of life in distributed systems (they *will* happen), you must tolerate them (P is non-negotiable), so you're actually choosing between **CP** (remain consistent, sacrifice availability during a partition) or **AP** (remain available, sacrifice consistency during a partition).

**Is it actually useful?**

Sort of, but it's often oversimplified. The more nuanced critique is that:

1. "Consistency" and "Availability" aren't binary — there's a continuous spectrum.
2. Partition tolerance isn't really optional for any realistic distributed system, making CAP a choice between C and A during partitions, which are rare. Most of the time (when there's no partition), you can have both.
3. CAP doesn't address latency — you can have consistent, available responses but they take 10 seconds. That's often worse than a fast stale response.

**PACELC** (by Daniel Abadi) is a more practical refinement: it says even without partitions, there's a tradeoff between **L**atency and **C**onsistency. Systems make different tradeoffs in both cases (partition and no-partition). This is a much more useful mental model for choosing between systems like DynamoDB (EL: low latency, possible staleness) vs. Spanner (PC: consistent at the cost of cross-region latency).

---

**Q20: What is a write-ahead log (WAL) and what problems does it solve?**

**Sample Answer:**

A Write-Ahead Log is a sequential, append-only log file maintained by the database where every modification is recorded *before* it's applied to the actual data pages. "Write-ahead" means: log the change before you make it.

**Problems it solves:**

**1. Crash recovery (durability)**
If the database crashes after a commit but before flushed data pages have been written to disk, the WAL can be replayed on restart to reconstruct the state. The database is always in a recoverable state. This is the core of ACID durability without requiring every data page to be synced to disk synchronously on every write — syncing the WAL (which is sequential and smaller) is sufficient.

**2. Write performance**
Random writes to data pages are expensive. Sequential writes to the WAL are cheap. The database can write to the WAL (fast, sequential) and then lazily flush dirty data pages in the background (checkpointing). This is one of the primary reasons databases can achieve high write throughput despite disk latency.

**3. Replication**
The WAL is exactly the record of changes needed to replicate data to standby servers. PostgreSQL's streaming replication works by shipping WAL records from primary to standbys, which replay them. This elegantly unifies crash recovery and replication under one mechanism.

**4. Point-in-time recovery (PITR)**
By archiving WAL segments, you can replay them from a base backup up to any specific point in time. This enables recovering to a state right before a catastrophic mistake (accidental DELETE of a table).

**5. Change Data Capture (CDC)**
Systems like Debezium tap into the WAL to stream change events to Kafka and downstream consumers without impacting the primary database's query workload. This powers event-driven architectures and data pipeline ingestion.

---

## SECTION 7: Advanced Topics

---

**Q21: What is a database transaction log vs. an application audit log? How do they complement each other?**

**Sample Answer:**

A **database transaction log (WAL)** is a low-level, internal engine mechanism for crash recovery and replication. It records physical or logical changes to data pages. It's not designed for human consumption or business-level auditing. It's often binary, compact, and periodically vacuumed or truncated.

An **application audit log** is a business-level record of *who did what, when, and potentially why* — it captures meaningful events like "User 42 changed the price of product 17 from $29.99 to $34.99 at 14:32 UTC." It's designed for compliance, debugging, and accountability.

**They complement each other:**
- The WAL gives you recoverability; the audit log gives you accountability.
- The WAL can tell you that a row changed; the audit log tells you why and who authorized it.
- The WAL is ephemeral by design; the audit log is often retained for years for compliance.

**Common implementation patterns for audit logs:**

- **Triggers** — Database triggers on INSERT/UPDATE/DELETE write to an audit table. Automatic and hard to bypass. Downside: performance overhead, and the audit table is in the same database (backup-coupled).
- **Application-level logging** — The application explicitly records events before or after writes. More context (can include request IDs, user intent), but can be bypassed or inconsistent if writes happen via multiple code paths.
- **Temporal tables** — SQL Server and some others support system-versioned temporal tables natively: every row's history is automatically maintained in a history table. Elegant and performant.
- **Event sourcing** — The primary data model is a log of immutable events. Audit is free because the log *is* the data. Tradeoff: architectural complexity.
- **CDC to audit store** — WAL-based CDC streams changes to an external audit datastore (e.g., BigQuery, S3 Parquet). Decoupled, scalable, doesn't impact OLTP performance.

---

**Q22: What are window functions and why are they more powerful than GROUP BY aggregations?**

**Sample Answer:**

Window functions (also called analytic functions) perform calculations across a set of rows related to the current row — a "window" — without collapsing those rows into a single output row as GROUP BY does.

With GROUP BY, aggregation collapses rows: you get one output row per group. The individual row detail is lost. With window functions, you get the aggregation result *alongside* the original row detail.

**Why this is more powerful:**

- **Running totals** — `SUM(amount) OVER (ORDER BY date)` gives a cumulative sum alongside each row. Impossible to express naturally with GROUP BY without self-joins.
- **Rankings** — `RANK() OVER (PARTITION BY dept ORDER BY salary DESC)` ranks employees within each department. GROUP BY can't return both the rank and the employee name in one query.
- **Lag/Lead** — `LAG(value, 1) OVER (ORDER BY date)` accesses the previous row's value from the current row — comparing today's value with yesterday's. Requires a self-join with GROUP BY.
- **Moving averages** — `AVG(price) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` computes a 7-day rolling average.
- **Percentiles within partitions** — `NTILE(4) OVER (PARTITION BY region ORDER BY revenue)` assigns quartile buckets within each region.

**Key clauses:**
- `PARTITION BY` — Defines the grouping (like GROUP BY but doesn't collapse rows).
- `ORDER BY` — Defines the ordering within the window for ordered functions.
- `ROWS/RANGE BETWEEN` — Defines the window frame: which rows relative to the current row are included in the calculation.

Window functions execute *after* WHERE and GROUP BY but *before* the final ORDER BY and LIMIT, meaning you can filter on window function results only by wrapping the query in a CTE or subquery.

---

**Q23: What is the difference between optimistic and pessimistic locking? When would you choose each?**

**Sample Answer:**

**Pessimistic locking** assumes conflicts will happen. It acquires a lock on a resource before reading it, holding the lock for the duration of the operation. In SQL, `SELECT ... FOR UPDATE` acquires a row-level exclusive lock, preventing other transactions from reading (with FOR UPDATE) or writing the rows until the lock is released.

**Optimistic locking** assumes conflicts are rare. It doesn't lock at read time. Instead, it detects conflicts at write time by verifying that the data hasn't changed since it was read. Typically implemented with a `version` column (integer counter) or `updated_at` timestamp. On update, the WHERE clause includes the expected version: `UPDATE ... WHERE id = ? AND version = ?`. If 0 rows are affected, a conflict occurred — another writer modified the row first — and the application must retry.

**When to choose each:**

**Pessimistic locking** is better when:
- Conflicts are frequent — optimistic locking would result in many retries, wasting work.
- The "work" between read and write is significant — if you read data, do 30 seconds of computation, and then write, an optimistic conflict wastes all that computation. Pessimistic locking prevents this.
- The system absolutely cannot tolerate lost updates — e.g., inventory reservation systems where two processes competing for the last unit must not both succeed.

**Optimistic locking** is better when:
- Conflicts are rare — locking overhead is unnecessary.
- The operation is short — retry cost is low.
- High read concurrency is needed — pessimistic locks block reads (or force them to wait), hurting throughput.
- The system is distributed and locks are difficult to coordinate — optimistic locking via CAS (compare-and-swap) or version checks scales better across services.

Many real-world systems combine both: optimistic locking at the application layer for typical operations, with occasional pessimistic locks (`SELECT FOR UPDATE`) for high-conflict critical sections like order fulfillment or seat reservation.

---

**Q24: Explain the concept of database sharding. What are the main strategies and their tradeoffs?**

**Sample Answer:**

Sharding is horizontal partitioning of data across multiple independent database instances (shards), each holding a subset of the total dataset. It's a scale-out strategy — instead of making one server bigger (vertical scaling), you distribute the data and load across many servers.

**Sharding strategies:**

**Range-based sharding** — Rows are assigned to shards based on ranges of the shard key (e.g., user IDs 1-1M on shard 1, 1M-2M on shard 2). Simple to understand, supports range queries efficiently within a shard.

*Tradeoffs:* Hotspot risk — if new users always get the highest IDs and are also the most active, the "latest" shard is overwhelmed while old shards sit idle (monotonic key problem). Rebalancing when a shard fills up requires moving large ranges.

**Hash-based sharding** — A hash function is applied to the shard key; the result modulo N (number of shards) determines the shard. Distributes data evenly, avoids hotspots.

*Tradeoffs:* Range queries are destroyed — rows with adjacent key values are scattered across shards. Adding/removing shards requires rehashing a significant fraction of data (unless using consistent hashing, which limits reshuffling to data on the affected shard).

**Directory-based sharding** — A lookup service maps shard keys to shard locations. Flexible — any mapping scheme can be used; rebalancing is just updating the directory.

*Tradeoffs:* The directory is a single point of failure and performance bottleneck. Must be highly available and cached.

**Geographic/tenant-based sharding** — All data for a region or customer is colocated on one shard. Good for data residency requirements and tenant isolation.

*Tradeoffs:* Tenant imbalance — one large customer fills a shard while others are mostly empty.

**Challenges of sharding in general:**

- **Cross-shard queries/joins** are extremely expensive or impossible at the DB level — must be done at the application layer.
- **Cross-shard transactions** require distributed transaction protocols (2PC, Saga), which add latency and failure complexity.
- **Schema changes** must be applied to all shards — requires careful migration choreography.
- **Rebalancing** (when load shifts) is operationally painful.
- **Operational complexity** — monitoring, backups, failover for N databases instead of one.

Sharding should be a last resort after exhausting vertical scaling, read replicas, caching, and query optimization. Once sharded, it's very hard to un-shard.

---

**Q25: What are the differences between row-oriented and column-oriented storage? How does this affect compression and query performance?**

**Sample Answer:**

**Row-oriented storage** stores all columns of a row together on disk. When you read a row, you read one contiguous block containing all its fields. PostgreSQL, MySQL, and most OLTP databases use this layout.

**Column-oriented storage** stores all values of a column together on disk. The first column's values are contiguous, then the second, etc. BigQuery, Redshift, ClickHouse, Parquet files, and most OLAP systems use this layout.

**How this affects query performance:**

For a query like `SELECT SUM(revenue) FROM sales WHERE year = 2024`:
- Row-oriented: must read every row in full (all columns including customer name, product ID, etc.) even though you only need `revenue` and `year`. Massive I/O waste.
- Column-oriented: reads only the `year` column (to filter) and `revenue` column (to sum). I/O proportional to the columns needed, not the total row width. For wide tables (hundreds of columns), this is a 10-100x I/O reduction.

For a query like `SELECT * FROM orders WHERE id = 12345`:
- Row-oriented: one disk read fetches the entire row efficiently.
- Column-oriented: must read from N separate column files and reassemble the row — N times the seeks.

**How this affects compression:**

Columnar storage achieves dramatically better compression because a column holds many values of the same type, domain, and often similar values. Compression algorithms exploit repetition and limited cardinality:

- **Run-length encoding** — `[NY, NY, NY, NY, CA, CA]` becomes `[(NY, 4), (CA, 2)]`. Perfect for low-cardinality sorted columns (status, country).
- **Dictionary encoding** — Replace string values with integer codes. A `status` column with values "active"/"inactive"/"pending" encodes to integers 0/1/2 across billions of rows.
- **Delta encoding** — Timestamps or sequential IDs stored as differences from the previous value rather than full values.
- **Bitpacking** — If a column's values fit in 4 bits, store 16 values per 64-bit integer.

Row-oriented storage must compress heterogeneous data (name, age, date, amount all in one block) with less leverage. Column data of the same type and domain compresses 5-20x better, reducing both storage cost and I/O (you read fewer actual disk bytes).

---

That's 25 questions covering foundational concepts, indexing, transactions, query execution, schema design, replication, scaling, and advanced topics. Each answer is calibrated to distinguish engineers who have merely *used* databases from those who deeply understand how they work and why — which is exactly what you're looking for in a world-class engineering hire.