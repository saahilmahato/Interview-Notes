# System Design Interview Questions & Model Answers

---

## FUNDAMENTALS & CORE CONCEPTS

---

**Q1. What is the difference between horizontal and vertical scaling? When would you choose one over the other?**

**Model Answer:**

Vertical scaling (scaling up) means adding more resources — CPU, RAM, disk — to an existing machine. Horizontal scaling (scaling out) means adding more machines to your pool and distributing load across them.

**Vertical scaling** is simpler — no code changes required, no distribution complexity, no network hops between machines. It works well for stateful workloads like relational databases where sharding adds enormous complexity. The hard limits are the physical ceiling of available hardware and the fact that you have a single point of failure.

**Horizontal scaling** is theoretically unbounded, fault-tolerant, and cost-efficient at scale since commodity hardware is cheap. The tradeoff is architectural complexity: you now need load balancers, you need to reason about distributed state, CAP theorem constraints kick in, and your application must be stateless or externalize its state.

In practice, most large systems do both. Databases are often scaled vertically until it hurts, then horizontally sharded. Stateless application servers are scaled horizontally behind load balancers almost by default. I generally recommend starting vertical for simplicity, instrumenting well, and moving to horizontal only when you can quantify the bottleneck demanding it — premature horizontal scaling is one of the most expensive mistakes a team can make.

---

**Q2. Explain CAP theorem. Is it still relevant today? What are its limitations?**

**Model Answer:**

CAP theorem states that a distributed system can only guarantee two of three properties simultaneously: **Consistency** (every read receives the most recent write), **Availability** (every request receives a response, not necessarily the most recent data), and **Partition tolerance** (the system continues operating despite network partitions).

The critical insight is that network partitions are not optional in a distributed system — they *will* happen. So the real choice is CP vs AP: do you sacrifice availability or consistency when a partition occurs?

**CP systems** (HBase, Zookeeper, etcd) refuse to serve stale reads. During a partition, they'll return errors rather than potentially wrong data. Good for financial systems, leader election, configuration stores.

**AP systems** (Cassandra, DynamoDB, CouchDB) continue serving requests but may return stale data. Good for shopping carts, DNS, social feeds, anything where eventual consistency is acceptable.

**Limitations and modern nuance:** Eric Brewer himself has noted that CAP is often misapplied. It's a binary model but real systems operate on spectrums. PACELC (by Daniel Abadi) is a more complete model — it asks: even when the system is running *without* partitions, what is the tradeoff between latency and consistency? Many systems like DynamoDB offer tunable consistency, letting you pick per-request. Also, "consistency" in CAP is linearizability, which is much stronger than what most developers mean when they say "consistent." People conflate it with eventual consistency, serializability, etc. A truly nuanced answer understands that CAP is a useful mental model but a poor specification for real system behavior.

---

**Q3. What is eventual consistency and what are the different models of consistency between strong and eventual?**

**Model Answer:**

Consistency models form a spectrum, roughly from strongest to weakest:

**Linearizability (Strong Consistency):** Every operation appears to take effect atomically at some point between its invocation and completion. The system behaves as if there's a single copy of the data. Very expensive — requires coordination on every operation. Example: etcd, Zookeeper with ZAB protocol.

**Sequential Consistency:** All processes see operations in the same order, but that order doesn't need to match real-time. Weaker than linearizability. Used in some CPU memory models.

**Causal Consistency:** Operations that are causally related are seen by all nodes in the same order. Concurrent operations may be seen in different orders. This is practically powerful — you can guarantee that "Bob's reply to Alice" is always seen after "Alice's message." COPS and Cassandra's lightweight transactions approximate this.

**Read-your-writes Consistency:** After you write, your subsequent reads will reflect that write. Others might not see it yet. Very useful for user-facing systems — you fill out a form, refresh the page, your data is there. Implemented with sticky sessions or routing reads to the primary.

**Monotonic Read Consistency:** Once you read a value, you'll never read an older value. Prevents the jarring experience of data appearing to go backward.

**Eventual Consistency:** Given no new updates, all replicas will converge to the same value — eventually. No time bound guaranteed. The weakest useful model. Works well for high-availability systems where temporary inconsistency is acceptable: DNS propagation, social media likes, shopping cart state.

The art is in choosing the right model per data type in your system. A single system might use linearizability for inventory counts, read-your-writes for user profile updates, and eventual consistency for view counters.

---

**Q4. Walk me through what happens when you type a URL into a browser and press Enter — from a systems design perspective.**

**Model Answer:**

This is a tour across multiple system layers:

**1. DNS Resolution:** The browser checks its local cache. If not found, it queries the OS resolver, which checks `/etc/hosts` and then the configured DNS resolver (usually your ISP or 8.8.8.8). The resolver walks the DNS hierarchy: root nameservers → TLD nameservers (.com) → authoritative nameserver for the domain. Returns an A record (IPv4) or AAAA record (IPv6). TTLs control caching duration. Large companies use GeoDNS to return different IPs based on client location, routing users to the nearest data center.

**2. TCP Connection / TLS Handshake:** Client sends SYN. Server replies SYN-ACK. Client replies ACK — three-way handshake. For HTTPS, TLS negotiation follows: client hello with supported cipher suites, server hello with certificate, client verifies the certificate chain against trusted CAs, key exchange (ECDHE for forward secrecy), symmetric session key derived. This is 2 round trips before a single byte of HTTP is sent — which is why HTTP/2 and QUIC (HTTP/3 over UDP) are such improvements.

**3. HTTP Request:** Browser sends GET / HTTP/1.1 with headers: Host, User-Agent, Accept, Cookie, etc. If HTTP/2, the connection is multiplexed and headers are compressed (HPACK).

**4. Load Balancer:** The request hits a load balancer (hardware like F5, or software like NGINX, HAProxy, or cloud-native like AWS ALB). L4 LBs route based on TCP/IP. L7 LBs can route based on HTTP headers, paths, host names. The LB selects a backend server — round-robin, least connections, consistent hashing for sticky sessions, etc. It may also terminate TLS here.

**5. CDN / Cache Layer:** If static assets are involved, they may be served from a CDN PoP close to the user without hitting origin. For dynamic content, a reverse proxy cache (Varnish, Nginx cache) might serve cached responses.

**6. Application Server:** Receives the request. Authenticates the user (validate session token or JWT). Processes business logic. May fan out to multiple microservices or make database calls.

**7. Database:** Query hits a connection pool (PgBouncer, HikariCP). Query planner executes the SQL. If the data is in the buffer pool (in-memory page cache), it's fast. Otherwise, it reads from disk (random I/O). Indexes reduce the search space from O(n) to O(log n) or O(1).

**8. Cache (Redis/Memcached):** Frequently-read data is served from an in-memory cache, bypassing the DB entirely. Cache-aside pattern: check cache first, on miss populate from DB.

**9. Response path:** Data flows back through the stack, serialized (JSON, protobuf), compressed (gzip/brotli), and returned. Browser receives the HTML, parses it, discovers sub-resources (CSS, JS, images), fires parallel requests (limited by browser connection limits), renders the page via DOM construction + CSSOM + layout + paint.

The reason I love this question is it tests whether a candidate truly understands the layers — not just the happy path but all the failure modes: DNS poisoning, certificate pinning, connection pool exhaustion, cache stampede, slow query, thundering herd on application restart, etc.

---

## DATABASES

---

**Q5. When would you use a relational database vs. a NoSQL database? Be specific.**

**Model Answer:**

The framing of "SQL vs NoSQL" is somewhat outdated — the real question is matching data access patterns to storage engines.

**Use a relational database (PostgreSQL, MySQL) when:**
- Your data has complex, well-defined relationships that you query across (JOINs). User → Orders → OrderItems → Products is a classic OLTP star.
- You need ACID transactions — especially multi-row, multi-table operations that must be atomic. Transferring money between accounts. Placing an order that decrements inventory.
- Your schema is relatively stable and the domain benefits from schema enforcement — it catches bugs at the DB layer.
- You need complex ad-hoc queries that aren't known at design time.
- Team size is small to medium and the operational simplicity of a single Postgres instance is a real advantage.

**Use a document store (MongoDB, Firestore) when:**
- Your data is naturally hierarchical/nested and you always read it together. A user's profile with addresses and preferences is a single document read — no JOINs needed.
- Schema flexibility matters — fields vary across documents, schema evolves rapidly.
- You're doing high-velocity writes of self-contained records (logs, events, user-generated content).

**Use a wide-column store (Cassandra, HBase) when:**
- You need massive write throughput (Cassandra can sustain millions of writes/sec across a cluster).
- Your access patterns are known and narrow — you almost always query by a specific partition key. Time-series data, IoT telemetry, user activity logs.
- You need geo-distributed multi-master replication.
- You do NOT need flexible queries — Cassandra is hostile to anything it wasn't designed for.

**Use a key-value store (Redis, DynamoDB) when:**
- Access is always by a single key. Session stores, caches, rate limiting counters, feature flags.
- You need sub-millisecond latency at extreme scale.
- DynamoDB specifically: when you need a managed, globally distributed key-value/document store with predictable single-digit millisecond latency.

**Use a graph database (Neo4j, Amazon Neptune) when:**
- Relationships are the primary query target, not the data itself. Friend-of-friend traversals, fraud ring detection, recommendation engines based on social graph. SQL JOINs become exponentially expensive for deep graph traversals — graph DBs represent edges natively.

**Use a time-series database (InfluxDB, TimescaleDB) when:**
- Data is always appended, never updated, and queried by time ranges. Metrics, monitoring, financial tick data. These engines have purpose-built compression (run-length encoding, delta encoding) and time-window aggregation functions.

The worst mistake is choosing a database based on familiarity rather than access patterns. The second-worst is designing a data model first and hoping any database will serve it.

---

**Q6. Explain database indexing in depth. What types of indexes exist and what are the tradeoffs?**

**Model Answer:**

An index is a separate data structure that the database maintains to speed up lookups, trading write overhead and storage for read performance.

**B-Tree Index (default in Postgres/MySQL):**
The workhorse. A balanced tree where leaf nodes contain the indexed values plus pointers to heap rows. Supports equality (`=`), range (`>`, `<`, `BETWEEN`), prefix matching on strings (`LIKE 'foo%'`), and `ORDER BY` without a sort step. Height is O(log n) — for a billion rows, 30 levels. Each node is typically a 8KB page, filling 3-4 levels in practice.

Tradeoffs: Write amplification — every INSERT/UPDATE/DELETE must update the B-tree, and if a page is full, a page split occurs, which is expensive. Bloat over time requires `VACUUM` / index rebuilds.

**Hash Index:**
Perfect for equality lookups only — `WHERE id = 42`. O(1) average. Cannot do ranges or sorting. In Postgres, hash indexes are now WAL-logged (safe for crash recovery post-9.x). In MySQL/InnoDB, hash indexes are adaptive and built in-memory automatically by the engine for hot B-tree lookups.

**Composite Index:**
An index on multiple columns: `(last_name, first_name, dob)`. Critically, this index satisfies queries that use a *prefix* of the columns in order. A query on `last_name` uses it. A query on `last_name, first_name` uses it. A query on just `first_name` does NOT use it (unless it's a skip scan). Understanding this "leftmost prefix rule" is essential for query optimization.

**Covering Index:**
An index that includes all columns needed by a query — so the database never needs to touch the heap table ("index-only scan"). `CREATE INDEX ON orders(user_id) INCLUDE (amount, status)` means a query `SELECT amount, status FROM orders WHERE user_id = ?` is served entirely from the index. Massive performance win for hot queries. Tradeoff: larger index, more write overhead.

**Partial Index:**
An index with a `WHERE` clause: `CREATE INDEX ON orders(user_id) WHERE status = 'pending'`. If 95% of orders are completed and you only query pending ones, this index is dramatically smaller and faster. Underused and underappreciated.

**GIN (Generalized Inverted Index):**
Used for full-text search, JSONB containment, array operators. It inverts the structure: for each element (word, JSONB key, array element), it stores the list of rows containing it. Very fast for containment queries (`@>`, `@@`). Expensive to update — writes accumulate in a "pending list" and are merged in batch.

**GiST (Generalized Search Tree):**
Flexible index type for geometric data, nearest-neighbor, range types, full-text. Used in PostGIS for spatial queries.

**BRIN (Block Range Index):**
Ultra-compact index that stores just the min/max value for each block range (128 pages by default). Works beautifully for naturally ordered data like timestamps in append-only tables. A 1TB log table's BRIN index might be 1MB. For a query `WHERE created_at BETWEEN x AND y`, it eliminates most blocks instantly. Useless for random-access patterns.

**Key design principles I apply:** Index columns in WHERE clauses that are high cardinality and selective. Never index boolean columns alone — two values means 50% selectivity, the planner will just scan. Watch write amplification — tables with 15 indexes will crawl on writes. Use `EXPLAIN ANALYZE` obsessively. Indexes on foreign keys are critical — without them, every FK constraint check is a sequential scan on the child table.

---

**Q7. What is the N+1 query problem and how do you solve it?**

**Model Answer:**

The N+1 problem occurs when you execute 1 query to fetch a list of N records, then N additional queries to fetch related data for each record. Classically:

```
1 query: SELECT * FROM posts LIMIT 100
100 queries: SELECT * FROM users WHERE id = ? (once per post's author_id)
```

Total: 101 queries. At 1ms per query, that's 100ms of pure database round-trip time that scales linearly with result size.

**Solutions:**

**1. Eager Loading / JOINs:** Fetch everything in one query:
```sql
SELECT posts.*, users.name, users.avatar 
FROM posts 
JOIN users ON posts.author_id = users.id 
LIMIT 100
```
One round trip. The ORM equivalent is `.includes(:author)` in Rails, `select_related()` in Django, `JOIN FETCH` in JPA.

**2. Batch Loading (DataLoader pattern):** Instead of one query per record, collect all needed IDs and fire one `WHERE id IN (...)` query. This is how Facebook's DataLoader works for GraphQL, and it's architecturally elegant for cases where JOINs are impractical (e.g., across microservice boundaries or when the related data is in a different database).

```
Collect: [1, 2, 3, 4, ..., 100]
Execute: SELECT * FROM users WHERE id IN (1, 2, 3, ..., 100)
```

2 total queries regardless of N.

**3. Denormalization / Caching:** For truly hot paths, embed frequently needed fields (like author name) directly in the post record, accepting that it's stale until updated. Works when the referenced data changes rarely.

**4. Query analysis tooling:** In production I always recommend enabling slow query logging, using tools like `pg_stat_statements`, Bullet gem (Rails), or Hibernate's SQL logging to catch N+1 problems before they become incidents. An N+1 on a table with 10 rows is harmless in dev. On a table with 10,000 rows in prod, it's a 5-second page load.

The deeper lesson: ORMs are wonderfully productive but they abstract away the queries they generate. You must always be aware of what SQL your ORM is producing. "Convenient" chained accessors in a template are one of the most common sources of accidental N+1s in production systems.

---

**Q8. Design a database schema for a ride-sharing application like Uber.**

**Model Answer:**

I think about this domain-first, then normalize to 3NF, then denormalize strategically for performance.

**Core entities:**

```sql
users (
  id UUID PK,
  phone VARCHAR UNIQUE NOT NULL,
  email VARCHAR UNIQUE,
  name VARCHAR NOT NULL,
  created_at TIMESTAMPTZ,
  stripe_customer_id VARCHAR  -- external ref to payment processor
)

drivers (
  id UUID PK,
  user_id UUID FK → users,
  license_number VARCHAR UNIQUE NOT NULL,
  rating NUMERIC(3,2),   -- 4.87 etc
  is_active BOOLEAN DEFAULT FALSE,
  vehicle_id UUID FK → vehicles
)

vehicles (
  id UUID PK,
  driver_id UUID FK → drivers,
  make VARCHAR,
  model VARCHAR,
  year SMALLINT,
  plate VARCHAR UNIQUE,
  vehicle_type ENUM('standard','xl','black','pool')
)

rides (
  id UUID PK,
  rider_id UUID FK → users,
  driver_id UUID FK → drivers (nullable until matched),
  status ENUM('requested','matching','driver_assigned','in_progress','completed','cancelled'),
  origin_lat NUMERIC(9,6),
  origin_lng NUMERIC(9,6),
  destination_lat NUMERIC(9,6),
  destination_lng NUMERIC(9,6),
  pickup_address TEXT,
  dropoff_address TEXT,
  requested_at TIMESTAMPTZ,
  accepted_at TIMESTAMPTZ,
  picked_up_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  estimated_fare_cents INT,
  final_fare_cents INT,
  distance_meters INT,
  duration_seconds INT,
  surge_multiplier NUMERIC(4,2)
)

ride_locations (
  id BIGSERIAL PK,
  ride_id UUID FK → rides,
  lat NUMERIC(9,6),
  lng NUMERIC(9,6),
  recorded_at TIMESTAMPTZ,
  speed_kmh SMALLINT
)
-- This is append-only, high volume. Consider TimescaleDB or partitioning by ride_id/time.

payments (
  id UUID PK,
  ride_id UUID FK → rides UNIQUE,
  rider_id UUID FK → users,
  amount_cents INT NOT NULL,
  currency CHAR(3) DEFAULT 'USD',
  status ENUM('pending','authorized','captured','refunded','failed'),
  payment_method_id VARCHAR,   -- Stripe PaymentMethod ID
  stripe_payment_intent_id VARCHAR UNIQUE,
  created_at TIMESTAMPTZ
)

ratings (
  id UUID PK,
  ride_id UUID FK → rides UNIQUE,
  rider_rating SMALLINT CHECK(rider_rating BETWEEN 1 AND 5),
  driver_rating SMALLINT CHECK(driver_rating BETWEEN 1 AND 5),
  rider_comment TEXT,
  driver_comment TEXT,
  created_at TIMESTAMPTZ
)

driver_locations (
  driver_id UUID PK FK → drivers,  -- one current record per driver
  lat NUMERIC(9,6),
  lng NUMERIC(9,6),
  bearing SMALLINT,  -- direction 0-359
  updated_at TIMESTAMPTZ
)
-- This is upserted every 4 seconds per active driver.
-- In practice, this lives in Redis as a geo-hash, not Postgres.
```

**Key design decisions I'd highlight:**

- **Money in cents as integers** — never floats for currency.
- **UUIDs vs auto-increment:** UUIDs prevent enumeration attacks and work better in distributed systems. Tradeoff: larger index footprint, slightly slower B-tree. Use ULIDs for sortable UUIDs if ordering matters.
- **driver_locations lives in Redis** using `GEOADD` / `GEORADIUS` — you need sub-second range queries like "find all drivers within 2km" and you're upsertng millions of rows per minute. A relational table would buckle.
- **ride_locations is a hot append-only time series** — partition by time, or push to a time-series store, or use Kafka → object storage for historical replay.
- **Surge multiplier stored on ride** — denormalization, but you need a historical record of what multiplier applied when the fare was calculated.
- **Status enum with timestamps** — rather than a status_history table, I embed key timestamps directly on rides for the most common queries ("how long did pickup take?") and add a `ride_events` audit table for full state-machine history.

**Indexes I'd add:**
```sql
CREATE INDEX ON rides(rider_id, requested_at DESC);
CREATE INDEX ON rides(driver_id, requested_at DESC);
CREATE INDEX ON rides(status) WHERE status NOT IN ('completed','cancelled');  -- partial
CREATE INDEX ON payments(stripe_payment_intent_id);
```

---

## CACHING

---

**Q9. Explain caching strategies — cache-aside, read-through, write-through, write-back, and write-around. When do you use each?**

**Model Answer:**

**Cache-Aside (Lazy Loading):**
The application manages the cache explicitly. On read: check cache → if miss, read from DB → populate cache → return. On write: write to DB → invalidate (or update) cache.

*Pros:* Only requested data is cached (working set naturally fits in cache). Cache failure is non-fatal — app falls back to DB. Simple to reason about.
*Cons:* Cache miss always incurs two round trips (cache + DB). Initial load or after a cache flush causes a stampede. Risk of stale data if invalidation is missed.
*Use when:* Read-heavy workloads with unpredictable access patterns. The most common pattern — used by almost every web application.

**Read-Through:**
The cache sits in front of the DB. On miss, the *cache* (not the application) fetches from the DB and populates itself. Application only talks to cache.

*Pros:* Simpler application code. Cache always has the "answer."
*Cons:* First request always misses. Tightly couples cache to DB schema.
*Use when:* Managed caching layers (some CDN edge caches work this way). When you want to push DB logic out of application code.

**Write-Through:**
Every write goes to the cache AND the DB synchronously before returning success. Cache is always up to date.

*Pros:* Cache is never stale. No invalidation needed.
*Cons:* Write latency is doubled. Caches data that may never be read ("write amplification" — you write every user's last-login timestamp to cache even if it's never queried from cache).
*Use when:* Combined with read-through. Data that's written and immediately read. Financial balances, inventory counts.

**Write-Back (Write-Behind):**
Writes go to cache only, immediately returning success. The cache asynchronously flushes to the DB later (on eviction, on a timer, in batches).

*Pros:* Extremely low write latency. Batching reduces DB load — 1000 writes to the same key become 1 DB write.
*Cons:* Risk of data loss if cache node fails before flush. Complex to implement correctly. Very difficult to reason about consistency.
*Use when:* High-frequency writes to the same key where some loss is acceptable (ad impression counters, like counts, analytics). Or when batching writes to a DB is a deliberate optimization (Redis → DynamoDB bulk writes).

**Write-Around:**
Writes go directly to DB, bypassing the cache. Cache is populated only on read.

*Pros:* Prevents cache pollution for data that's written once and rarely re-read (log entries, bulk imports).
*Cons:* First read after write always misses cache.
*Use when:* Bulk uploads, one-time writes, large files. Anything you don't expect to serve from cache.

**In practice**, most systems use cache-aside as the default, layered with write-through for highly read-sensitive data and write-back only where the latency benefit justifies the complexity and loss risk. The hardest part of caching isn't the strategy — it's cache invalidation, which Phil Karlton famously called one of the two hard problems in CS.

---

**Q10. What is a cache stampede (thundering herd) and how do you prevent it?**

**Model Answer:**

A cache stampede occurs when a popular cache key expires and many concurrent requests simultaneously find a cache miss, all racing to the database to regenerate the value. If 10,000 requests/sec were being served from cache and the key expires, you suddenly have thousands of concurrent DB queries for the same value, potentially overwhelming the database.

**Solutions:**

**1. Locking / Mutex:**
When a cache miss is detected, only one process acquires a distributed lock (e.g., `SET lock:key NX PX 5000` in Redis) and regenerates the value. All others either wait or serve a slightly stale value while the winner regenerates. The tradeoff is lock contention and latency for the waiters.

**2. Probabilistic Early Expiration (XFetch):**
Rather than expiring exactly at TTL, recompute the value with increasing probability as it approaches expiration. The formula is roughly: recompute if `current_time - beta * log(random())` > expiry time. This spreads out cache regeneration over time rather than having a cliff edge. Elegant, stateless, and effective.

**3. Stale-while-revalidate:**
Serve the stale cached value while a background process regenerates it asynchronously. The current request gets an answer immediately. The next request will get the fresh value. This is the model used by HTTP `Cache-Control: stale-while-revalidate` and many CDNs. Works beautifully for data where short staleness is acceptable (social feeds, product listings).

**4. Jitter on TTL:**
When setting cache keys, add random jitter to TTL: `TTL = base_ttl + random(0, base_ttl * 0.1)`. If 10,000 keys were all set at the same time with the same TTL, they'd all expire simultaneously. Jitter staggers expiration. Simple and underrated.

**5. Event-driven invalidation:**
Don't use TTL-based expiration at all — invalidate cache keys explicitly when the underlying data changes (via CDC, Kafka events, or application-level invalidation). No expiry → no stampede. The difficulty is ensuring invalidation is reliable and handling cache invalidation bugs.

The best production systems layer these: jitter to prevent synchronized expiration + stale-while-revalidate to absorb traffic during regeneration + circuit breaker at the DB layer as a last resort.

---

## SYSTEM DESIGN CLASSICS

---

**Q11. Design a URL shortener like bit.ly.**

**Model Answer:**

**Requirements clarification (I always do this first):**
- 100M URLs shortened per day? 10 billion total?
- Reads vs writes ratio? (Typically 100:1 read-heavy)
- Custom aliases? Expiration? Analytics?
- Global, multi-region? SLA?

**Assumptions:** 100M new URLs/day, 10B reads/day, 5-year retention, analytics required.

---

**Core Algorithm — Short Code Generation:**

Option A: **Base62 encoding of a counter.** An auto-incrementing ID from a distributed counter (or DB sequence) encoded in base62 (a-z, A-Z, 0-9). ID 12345678 → `5BKpb`. 7 characters covers 62^7 ≈ 3.5 trillion URLs. Simple, but sequential IDs are guessable.

Option B: **Random string + collision check.** Generate 7 random base62 chars, check for collision in DB. At small scale fine, at large scale probability of collision rises (birthday problem) and each write requires a read.

Option C: **MD5/SHA hash of the long URL, take first 7 chars.** Fast, deterministic (same URL = same short code, natural deduplication). Collision probability is low but nonzero, requires handling.

**Best choice: Snowflake-style distributed ID → Base62.** Combine machine ID + timestamp + sequence number. Gives unique, roughly time-sortable, non-guessable IDs without coordination. Twitter uses this, Discord uses this.

---

**Data Storage:**

```
url_mappings:
  short_code  VARCHAR(8) PK
  long_url    TEXT NOT NULL
  user_id     UUID (nullable, for registered users)
  created_at  TIMESTAMPTZ
  expires_at  TIMESTAMPTZ (nullable)
  
clicks:
  id          BIGSERIAL
  short_code  VARCHAR(8)
  clicked_at  TIMESTAMPTZ
  ip          VARCHAR
  user_agent  TEXT
  referer     TEXT
  country     CHAR(2)  -- from IP geolocation
```

`url_mappings` fits comfortably in PostgreSQL. At 100M rows/day × 365 × 5 years = 182B rows. That's too large for a single Postgres instance. Options:

- **Shard by short_code prefix** (consistent hashing)
- **Use Cassandra** — primary key = short_code, naturally partitioned, excellent for key lookups
- **DynamoDB** — managed, scales horizontally, single-digit ms reads

For `clicks`, this is time-series write-heavy (10B rows/day). Use:
- Kafka for ingestion (fire and forget from redirect service)
- ClickHouse, Redshift, or BigQuery for analytics

---

**Redirect Service (the hot path):**

```
Client → DNS → Load Balancer → Redirect Service → Cache (Redis) → DB
                                                          ↓
                                                  301/302 redirect
```

Redis stores `short_code → long_url` with TTL. 99%+ of redirects served from cache. Cache miss hits DB, populates cache.

**301 vs 302:** 301 is permanent — browsers cache it and never re-request. Great for reducing server load. Bad for analytics (you lose click data). **302 (temporary) is the right choice for analytics** — every redirect hits your server.

---

**Scale numbers:**

- 10B reads/day = 115K reads/sec average, maybe 500K peak
- Redis cluster with 6 nodes (3 primary, 3 replica): each can do 100K+ ops/sec. Comfortable.
- Redirect service: stateless, horizontally scalable, maybe 50 nodes at peak
- 100M writes/day = 1,157 writes/sec. A single Postgres primary handles this easily. Cassandra/DynamoDB even more so.

**Analytics pipeline:**

```
Redirect Service → Kafka → Stream processor (Flink/Spark Streaming) → ClickHouse
                                                                      ↓
                                                              Dashboard / API
```

Pre-aggregate counts per short_code per hour into a summary table for the common case (clicks in last 24h, last 7 days). Full-scan analytics on ClickHouse.

---

**Q12. Design a rate limiter.**

**Model Answer:**

**Algorithms:**

**Token Bucket:** A bucket holds up to N tokens. Tokens are added at rate R per second. Each request consumes one token. If the bucket is empty, requests are rejected or queued. Allows bursting up to N. Used by AWS, Stripe.

**Leaky Bucket:** Requests enter a queue (the bucket) and are processed at a fixed rate. Excess requests overflow and are dropped. Smooths out bursts — output rate is constant. Good for protecting downstream services from spikes.

**Fixed Window Counter:** Count requests in the current time window (e.g., current minute). Reset counter at window boundary. Simple but has a boundary problem: 100 requests at 0:59 + 100 requests at 1:01 = 200 requests in 2 seconds, both windows satisfied.

**Sliding Window Log:** Store a timestamp log of each request. Count timestamps within the last N seconds. Accurate but memory-heavy — O(requests) storage per user.

**Sliding Window Counter:** Hybrid. Current window count + (previous window count × overlap fraction). Approximates the sliding window with O(1) storage. Used by Cloudflare — 99.97% accuracy vs true sliding window.

---

**Implementation with Redis:**

```python
# Token bucket in Redis using Lua script (atomic)
local key = KEYS[1]
local rate = tonumber(ARGV[1])      -- tokens per second
local capacity = tonumber(ARGV[2])  -- bucket size
local now = tonumber(ARGV[3])       -- current timestamp (ms)
local requested = tonumber(ARGV[4]) -- tokens needed

local last_tokens = tonumber(redis.call("hget", key, "tokens") or capacity)
local last_refreshed = tonumber(redis.call("hget", key, "ts") or 0)

local elapsed = math.max(0, now - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (elapsed * rate / 1000))
local allowed = filled_tokens >= requested
local new_tokens = allowed and (filled_tokens - requested) or filled_tokens

redis.call("hset", key, "tokens", new_tokens, "ts", now)
redis.call("expire", key, math.ceil(capacity / rate) + 1)
return allowed and 1 or 0
```

Using a Lua script is critical — it executes atomically on the Redis server, eliminating race conditions without needing WATCH/MULTI/EXEC.

---

**Distributed rate limiting:**

Each application server can't maintain its own counter (they'd each allow the full rate, multiplying by N servers). Options:

**1. Centralized Redis:** All servers talk to the same Redis cluster. Fast (sub-millisecond), accurate. Single point of failure mitigated by Redis Sentinel/Cluster. Network round trip adds latency to every request.

**2. Sliding window at edge:** Use a CDN or API gateway (Nginx, Kong, Cloudflare) that rate limits before traffic hits your servers. Offloads the problem entirely.

**3. Approximate local + async sync:** Each server maintains a local counter. Periodically (every 100ms) syncs with Redis, applying a fraction of the global limit locally. Trades some accuracy for lower latency. Twitter uses this approach.

---

**Rate limit response:**

Return `429 Too Many Requests` with headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714521600
Retry-After: 60
```

**Multi-dimensional rate limiting:** Different limits per user tier (free: 100/min, paid: 10K/min), per endpoint (login: 5/min, search: 1000/min), per IP as a last defense against unauthenticated abuse. Use a composite key: `ratelimit:{user_id}:{endpoint}:{window}`.

---

**Q13. Design a distributed message queue like Kafka.**

**Model Answer:**

**Core Concepts:**

A message queue decouples producers from consumers. Producers write messages without knowing who reads them. Consumers read at their own pace. Kafka's differentiation from traditional queues (RabbitMQ): messages are **retained** (not deleted on consumption), **replayable**, and **partitioned** for parallel consumption.

---

**Architecture:**

**Broker:** A server that stores and serves messages. A Kafka cluster has many brokers.

**Topic:** A logical channel. "user-signups", "payment-events". Producers write to topics, consumers read from topics.

**Partition:** Each topic is split into N partitions. A partition is an ordered, immutable, append-only log stored on disk. Partitions are the unit of parallelism — you can have at most N consumers in a consumer group reading a topic with N partitions concurrently.

**Offset:** A sequential integer ID for each message within a partition. Consumers track their current offset, committing it to Kafka after processing. This allows replaying, exactly-once processing, and consumer restarts.

**Consumer Group:** A group of consumers that collectively read all partitions of a topic. Each partition is assigned to exactly one consumer in the group. If a consumer dies, its partitions are rebalanced to remaining consumers.

**Replication:** Each partition has 1 leader and N-1 followers on different brokers. Producers write to leader. Followers replicate asynchronously. If leader dies, a follower becomes leader. `replication.factor = 3` for production durability.

**ZooKeeper / KRaft:** Historically Kafka used ZooKeeper for cluster metadata and leader election. Kafka 3.x introduced KRaft (Kafka Raft), eliminating ZooKeeper for simpler operations.

---

**Storage Engine:**

Each partition is a directory of **segment files** on disk: `00000000000000000000.log`, `00000000000012345678.log` etc. Each segment has an accompanying `.index` file (offset → byte position) and `.timeindex` file (timestamp → offset) for fast seeking.

Writes are sequential appends to the active segment — extremely fast, optimized for disk I/O (sequential disk access rivals memory bandwidth). Old segments are deleted or compacted based on retention policy.

**Page cache:** Kafka relies heavily on the OS page cache rather than maintaining its own. Writes go to page cache (fast), fsync to disk periodically. Reads of recent messages are served from page cache without disk I/O at all. This is why Kafka on a machine with lots of RAM is dramatically faster.

**Zero-copy:** Kafka uses `sendfile()` syscall for consumer reads, transferring data from page cache to network socket without copying to user space. Dramatically reduces CPU overhead for consumer throughput.

---

**Producer durability settings:**

- `acks=0`: Fire and forget. Fastest, data loss possible.
- `acks=1`: Leader acknowledges. Leader failure before replication = data loss.
- `acks=all` (`-1`): All in-sync replicas acknowledge. Strongest durability guarantee. Higher latency.

**Consumer delivery semantics:**

- **At-most-once:** Commit offset before processing. If crash after commit but before processing, message is lost.
- **At-least-once:** Commit offset after processing. If crash after processing but before commit, message is reprocessed. Most common. Consumers must be idempotent.
- **Exactly-once:** Kafka transactions + idempotent producer. Producer deduplication + atomic offset commit + topic write in a transaction. Powerful but complex. Used for Kafka-to-Kafka pipelines and financial systems.

---

**Partition key selection:**

Producers choose a partition by `hash(key) % num_partitions`. Messages with the same key always go to the same partition → guaranteed ordering per key. Use `user_id` as key if you need all events for a user to be ordered. If `key=null`, round-robin across partitions for maximum throughput.

The gotcha: you can't easily increase partition count without disrupting key-to-partition mapping. Choose an initial count that leaves headroom (start with 12-24 for a moderate-scale topic).

---

**Compacted topics:**

Instead of time-based retention, Kafka can compact topics: retain only the **latest message per key**. Deleted keys are represented by a tombstone (null value). Excellent for event sourcing / materialized views — a changelog topic for a database table, where each message is a row update, compacted to current state.

---

**Q14. Design Instagram's news feed.**

**Model Answer:**

**Two fundamental approaches:**

**Pull (Fan-out on Read):** When User A opens their feed, query all accounts they follow, fetch recent posts from each, merge and sort. Also called "pull model."

*Pros:* No precomputation. Writes are simple — just store the post. Works fine for users following few accounts.
*Cons:* On read, you might query 1000 followed accounts, fetch and merge their posts. 1000 DB queries per feed load. Even with caching per-user-timeline this is expensive at scale.

**Push (Fan-out on Write):** When a user posts, the system immediately pushes that post ID to the feed of every follower. Feed is precomputed, stored as a sorted list (e.g., Redis Sorted Set keyed by user ID).

*Pros:* Feed reads are O(1) — just read your precomputed list. Very fast.
*Cons:* Celebrities (Kylie Jenner has 400M followers) writing one post → 400M write operations. This is the "hot celebrity problem."

**Instagram's hybrid approach:**

- **Regular users** (< ~1M followers): Fan-out on write. Push post IDs to all follower feeds in Redis.
- **Celebrities / high-follower accounts**: Fan-out on read. Don't precompute. At read time, fetch celebrity posts and merge with your precomputed feed.

Most users follow ~500 accounts, celebrities among them. On read:
1. Load precomputed feed from Redis (handles 490 of your follows)
2. For the 10 celebrities you follow, fetch their recent posts directly
3. Merge, deduplicate, sort by score (recency + engagement)

---

**Storage:**

```
posts:
  post_id      BIGINT (Snowflake ID, sortable by time)
  user_id      UUID
  caption      TEXT
  media_keys   TEXT[]  -- S3 keys for photos/videos
  created_at   TIMESTAMPTZ
  like_count   INT
  comment_count INT

follows:
  follower_id  UUID
  followee_id  UUID
  created_at   TIMESTAMPTZ
  PRIMARY KEY (follower_id, followee_id)

-- Feed stored in Redis:
-- Key: feed:{user_id}
-- Type: Sorted Set
-- Member: post_id
-- Score: unix timestamp (for chronological) or ML ranking score
```

**Redis feed structure:**

```
ZADD feed:user123 <timestamp> <post_id>  # add post to feed
ZREVRANGE feed:user123 0 19              # fetch top 20 posts
ZREMRANGEBYRANK feed:user123 0 -1001     # trim to 1000 entries max
```

Cap each feed at ~1000 entries — nobody scrolls further. Older content loaded on demand via pull.

---

**Post creation flow:**

```
1. Client → API → Post Service
2. Store post in Posts DB (Cassandra or Postgres shard)
3. Upload media to S3, process thumbnails asynchronously (Lambda/worker)
4. Publish to Kafka: topic "new-posts", key = user_id
5. Feed Fanout Service consumes Kafka
6. Query Follows DB: SELECT follower_id FROM follows WHERE followee_id = ?
7. For each follower (who is NOT a celebrity-tier threshold):
   ZADD feed:{follower_id} {score} {post_id}
   ZREMRANGEBYRANK feed:{follower_id} 0 -1001
```

Step 6 could return millions of follower IDs. The Fanout Service is itself a scalable consumer group on the Kafka topic — you might have 100 instances doing this in parallel, batching Redis writes with pipelining.

---

**ML Ranking:**

Chronological feeds are simple. Algorithmic feeds score posts by:
- Recency (freshness)
- Author relationship strength (how often you interact)
- Post engagement velocity (likes/comments per minute after posting)
- Content type affinity (do you usually engage with video? Reels?)
- Negative signals (hide, not interested)

The ranking score replaces the raw timestamp as the Sorted Set score. The Ranking Service runs a lightweight model (often gradient boosted trees for speed) per candidate post. Instagram uses a multi-stage funnel: coarse retrieval → lightweight ranking → heavy ML ranking → diversity injection.

---

**Q15. Design a distributed key-value store from scratch.**

**Model Answer:**

Think of building something between Redis and DynamoDB.

---

**Core data structure — Nodes:**

Each key is owned by a set of nodes determined by consistent hashing. Place N nodes on a virtual ring. `hash(key) mod 2^32` maps a key to a position on the ring. Walk clockwise to find the responsible node. Virtual nodes (vnodes) per physical node (150-200 per node) ensure even distribution and smooth rebalancing.

**Replication:** The primary node and the next N-1 clockwise nodes on the ring store replicas. `N=3` is standard. Write goes to coordinator (any node via consistent hashing), coordinator propagates to replicas.

---

**Storage engine per node:**

**LSM-Tree (Log-Structured Merge Tree):** Used by Cassandra, RocksDB, LevelDB.

Writes go to an in-memory **MemTable** (sorted). When full, flush to an immutable **SSTable** (Sorted String Table) on disk. SSTables are periodically **compacted** — merging overlapping key ranges, removing deleted keys (tombstones). Very fast writes (sequential I/O). Read amplification: may need to check multiple SSTables + MemTable. Bloom filters per SSTable reduce unnecessary disk reads. Excellent for write-heavy workloads.

**B-Tree Engine:** Used by most RDBMSs. Better read performance, worse write performance than LSM. More appropriate for read-heavy workloads.

I'd choose **RocksDB** (Facebook's LSM implementation) as the embedded storage engine — battle-tested, supports bloom filters, prefix iterators, snapshots, compression.

---

**Consistency model:**

Quorum-based reads/writes:
- `N` = total replicas
- `W` = nodes that must acknowledge a write
- `R` = nodes that must respond to a read
- For strong consistency: `R + W > N`
- `N=3, W=2, R=2` → strong consistency, tolerates 1 failure
- `N=3, W=1, R=1` → eventual consistency, maximum availability

Support tunable consistency per request. Cassandra's `CONSISTENCY ONE / QUORUM / ALL`.

---

**Write path:**

```
Client → Any node (Coordinator)
         → Write to WAL (Write-Ahead Log) for durability
         → Write to MemTable
         → Forward to N-1 replicas (async or sync depending on W)
         → Ack to client when W nodes confirm
```

WAL ensures durability on crash before MemTable flush.

**Read path:**

```
Client → Coordinator
         → Query R nodes for value + vector clock
         → If R nodes agree → return value
         → If conflict (diverged replicas) → conflict resolution
             - Last-write-wins (LWW) using timestamps (loses updates)
             - Vector clocks (track causal history, return to client for resolution)
             - CRDTs (mathematically merge-able data types — counters, sets, maps)
```

Dynamo uses vector clocks. Cassandra uses LWW by default (simple, lossy). CRDTs are ideal for counters (increment-only), flags, and commutative operations.

---

**Anti-entropy / Eventual Consistency:**

How do replicas re-sync after network partition?

**Merkle Trees:** Each replica builds a Merkle tree over its key range (hashing subranges recursively). Comparing root hashes between replicas is O(1). If roots match, data is identical. If mismatch, traverse the tree to find divergent subtrees, then exchange only the differing keys. Cassandra's `nodetool repair` uses this. Dramatically reduces the data exchanged for synchronization.

**Gossip Protocol:** Nodes periodically exchange state with random peers. Cluster membership, node health, schema versions all propagate through gossip. Eventually consistent, resilient to node failures, no central coordinator needed.

---

**Failure detection:**

**Phi Accrual Failure Detector:** Instead of binary alive/dead, compute a suspicion score φ based on heartbeat interval history. If heartbeats arrive on time, φ is low. If delayed, φ rises. Application sets a threshold (φ > 8 → suspect dead). Adapts to network jitter, avoids false positives under load. Used by Cassandra, Akka.

---

**ADVANCED TOPICS**

---

**Q16. What is the difference between an event-driven architecture and a request-response architecture? What are the tradeoffs?**

**Model Answer:**

**Request-Response (synchronous):**
Service A calls Service B directly (HTTP/gRPC) and waits for a response. Simple mental model, easy to debug, immediate error handling. The caller knows the result synchronously.

Tradeoffs: Temporal coupling — A can't proceed if B is down or slow. Cascade failures — B's latency becomes A's latency. Tight coupling — A must know B's address/API. At high fan-out (A calling 10 services), latency = max(all latencies) or the critical path.

**Event-Driven (asynchronous):**
Service A publishes an event ("OrderPlaced") to a broker (Kafka, RabbitMQ, SNS). Services B, C, D consume and react independently. A doesn't know who consumes or when.

Tradeoffs: Temporal decoupling — B can be down; events queue up and are processed when B recovers. Better fault isolation. Easy to add new consumers without modifying A. Natural audit log of what happened.

The difficulty: debugging is harder (distributed tracing essential — Jaeger, Zipkin). Eventual consistency becomes the default — you can't get a synchronous response from the downstream. Error handling is complex — how does A know if B failed to process? Dead letter queues, idempotent consumers, and retry strategies become essential design elements. The "dual write problem": writing to your DB and publishing to the broker atomically is non-trivial. The Outbox Pattern solves this: write the event to an `outbox` table in the same transaction as the business data, then a separate poller reads the outbox and publishes to the broker, guaranteeing at-least-once delivery.

**When to use each:**
- User-facing read APIs → Request-response (synchronous, low latency required)
- Payment processing confirmation → Request-response with sync response
- Post-payment side effects (send email, update loyalty points, trigger fraud check) → Event-driven (OrderPaid event)
- Data pipelines, ETL, cross-domain integration → Event-driven
- Microservices coordination where services are owned by different teams → Event-driven to minimize coupling

Most mature architectures use both: synchronous for the critical path user-facing calls, asynchronous for background processing, integration, and fan-out.

---

**Q17. How does consistent hashing work and why is it used in distributed systems?**

**Model Answer:**

In a naive sharding scheme, you assign a key to a node via `hash(key) % N` where N is the number of nodes. This is simple but catastrophic for cluster resizing: when N changes, almost every key maps to a different node, requiring a massive reshuffling of data.

**Consistent hashing** maps both nodes and keys onto the same virtual ring (the hash space, e.g., 0 to 2^32-1). To find which node owns a key, hash the key and walk clockwise on the ring until you reach the first node. 

When a node is added, it takes over keys from its clockwise neighbor — only ~K/N keys are moved (where K is total keys, N is total nodes). When a node is removed, its keys transfer to the next clockwise node. Average data moved: only 1/N fraction of total keys.

**Virtual nodes (vnodes):** A single physical node is represented by multiple points on the ring (e.g., 150 virtual nodes). This ensures even load distribution even when nodes have heterogeneous capacity, and dramatically smooths the key distribution that would otherwise be non-uniform with just one point per physical node. A node with double the capacity gets double the vnodes. When a node fails, its vnodes are distributed across all remaining nodes rather than piling onto a single neighbor.

**Where it's used:**
- Cassandra: determines replica assignment for every key
- Memcached/Redis Cluster: routes client requests to correct shards
- Chord DHT (foundational academic work)
- Akamai CDN routing
- DynamoDB (described in the Dynamo paper)
- Load balancers for sticky routing

**Limitations:** With pure consistent hashing, hotspots can still occur if certain keys are dramatically more popular than others (celebrity problem in social networks). This requires additional strategies: virtual nodes with uneven weighting, or application-level hot key splitting.

---

**Q18. Explain the Saga pattern for distributed transactions.**

**Model Answer:**

In a microservices architecture, a business transaction often spans multiple services — and you cannot use a distributed two-phase commit (2PC) at scale because it's slow, blocks resources, and the coordinator is a single point of failure.

The **Saga pattern** breaks a distributed transaction into a sequence of local transactions, each within a single service. If a step fails, **compensating transactions** undo the preceding steps.

**Example — e-commerce order:**
1. Order Service: Create order (status: PENDING)
2. Payment Service: Reserve funds
3. Inventory Service: Reserve items
4. Shipping Service: Create shipment
5. Order Service: Update order (status: CONFIRMED)

If step 3 fails: compensate step 2 (release reserved funds) and step 1 (cancel order).

**Two coordination approaches:**

**Choreography:** Each service publishes events and reacts to events from other services. No central coordinator. Decentralized, loosely coupled, scales well. Hard to visualize the overall flow, hard to debug, risk of circular event loops.

**Orchestration:** A central Saga Orchestrator (a separate service or state machine) directs each step, calls each service, and issues compensation commands on failure. Easier to understand, debug, and monitor. The orchestrator is a potential bottleneck/SPOF but this is mitigated by making it stateless with state persisted to DB.

**Challenges:**

**Idempotency:** Compensating transactions and service calls must be idempotent — they might be retried. Each local transaction needs an idempotency key.

**Semantic atomicity is approximate:** Sagas provide ACD (no Isolation). Between steps, other transactions can see intermediate state (partial saga). This is often acceptable with careful domain design (use "pending" states) but can require compensating for business logic that's hard to undo — you can cancel a payment reservation, but you can't un-send an email. Those side-effects must be delayed until the saga is confirmed.

**Saga log:** Persist the saga's state and which steps completed to a durable store. On crash recovery, the orchestrator can resume from where it left off.

Sagas are the standard pattern for distributed business transactions in microservices, but they require disciplined event design, idempotent consumers, and careful thought about what "undo" means in your domain.

---

**Q19. You have a system experiencing high read load. Walk me through every layer where you'd add optimization.**

**Model Answer:**

I approach this top-down, from the client to the database.

**1. Client-side:**
- HTTP caching: `Cache-Control`, `ETag`, `Last-Modified`. Browsers and clients cache responses and validate with conditional requests. For GET endpoints returning the same data, a proper 304 Not Modified eliminates bandwidth and processing entirely.
- Client-side state: Optimistic UI, local caching in React Query / SWR / Apollo Client. Reduce request volume by not re-fetching what hasn't changed.

**2. CDN (Content Delivery Network):**
- Static assets (JS, CSS, images) served from edge PoPs globally. Origin never sees these requests.
- For dynamic but cacheable responses (product listings, public user profiles), a CDN with smart TTLs and surrogate keys (cache purge by tag) can serve millions of requests from edge.
- Techniques: Vary header handling, edge-side includes (ESI) for partial page caching.

**3. Load Balancer / API Gateway:**
- Response caching at the API gateway layer (Kong, AWS API GW) for idempotent GET endpoints.
- Request coalescing: if 1000 concurrent requests arrive for the same uncached resource, collapse them into one upstream request (Nginx `proxy_cache_lock`).

**4. Application Server:**
- In-process LRU cache (Guava Cache, Caffeine): sub-microsecond, no network hop. Great for config data, reference data that changes rarely.
- Connection pooling: ensure DB connection pool is right-sized. Pool exhaustion causes queuing even if DB has capacity.
- Async processing: ensure reads that don't need to be synchronous aren't blocking threads.
- Efficient serialization: protobuf over JSON where wire size and parse time matter.

**5. Distributed Cache (Redis):**
- Cache hot query results, serialized objects, rendered HTML fragments.
- Read replicas in Redis (Redis Cluster, Sentinel): distribute read traffic across multiple nodes.
- Consistent hashing for Redis Cluster sharding to horizontally scale cache capacity.
- Watch for cache stampede (addressed earlier).

**6. Database Read Replicas:**
- Add one or more read replicas. Route read queries to replicas. Writes go to primary.
- Replication lag is the tradeoff — reads might be slightly stale. Acceptable for most read workloads. For reads-after-writes where consistency is critical (user just updated their profile), route to primary or wait for replica lag.
- Read replica autoscaling: Aurora automatically scales replicas based on load.

**7. Query Optimization:**
- EXPLAIN ANALYZE every slow query. Add missing indexes. Convert sequential scans to index scans.
- Rewrite N+1 queries to JOINs or batch queries.
- Pagination: never `OFFSET 10000` (scans 10,000 rows to discard). Use keyset pagination: `WHERE id > :last_seen_id LIMIT 20`.
- Denormalize for hot queries: embed frequently-joined data to eliminate JOINs.
- Materialized views: precompute expensive aggregations (daily sales totals, user stats) and refresh on schedule.

**8. Database Sharding:**
- Horizontal partitioning: split data by user_id range, geographic region, or consistent hash.
- Route reads to the correct shard. Dramatically reduces per-shard load.
- Tradeoff: cross-shard queries (aggregations, reports) become very complex.

**9. Read-optimized data stores:**
- OLTP reads heavy on aggregations → push to OLAP store (ClickHouse, Redshift, BigQuery). Change Data Capture (Debezium) streams changes from primary DB to the analytical store.
- Full-text search → Elasticsearch, which is optimized for inverted-index lookups that would be slow in Postgres.
- Time-series queries → InfluxDB, TimescaleDB.

**10. Observability:**
Throughout all of this: instrument everything. Latency percentiles (p50, p95, p99), cache hit rates, DB query times, replica lag. You cannot optimize what you cannot measure. A cache with a 40% hit rate is often worse than no cache (added complexity with little benefit) — you need to understand WHY you're missing before tuning.

---

**Q20. What is back pressure and how do you implement it?**

**Model Answer:**

Back pressure is a flow control mechanism that allows a downstream system to signal upstream systems to slow down when it can't keep up. Without it, a fast producer overwhelms a slow consumer, leading to unbounded queue growth, OOM crashes, and cascading failures.

**The problem:**
```
Producer (1M msg/sec) → Queue → Consumer (100K msg/sec)
```
Without back pressure, the queue grows at 900K msg/sec. In minutes you've consumed all memory or disk.

**Implementations:**

**Reactive Streams / Project Reactor / RxJava:**
The consumer requests N items (`request(n)` in the Reactive Streams protocol). The producer only sends N items. When the consumer is ready for more, it requests again. This demand-driven model ensures the consumer is never overwhelmed. Implemented by Java's `Flow` API, Spring WebFlux, and Reactor.

**TCP flow control:**
TCP has back pressure built in via the receive window size. The receiver advertises how many bytes it can buffer. The sender cannot exceed this window. When the receiver is overwhelmed, the window shrinks to zero, and the sender stops. This propagates all the way to the application layer.

**Message queue acknowledgments:**
In RabbitMQ, `prefetch_count` limits how many unacknowledged messages a consumer holds. If the consumer is slow to ack, the broker stops delivering. Consumer's processing rate becomes the throttle.

In Kafka, consumers pull messages at their own rate. The broker doesn't push. If a consumer falls behind, it simply has a growing lag. You monitor consumer group lag (Burrow, built-in Kafka metrics) and alert/scale consumers when lag grows. Kafka retains messages, so you don't lose them.

**HTTP back pressure:**
`429 Too Many Requests` and `503 Service Unavailable` with `Retry-After` headers. The API gateway or load balancer sheds load and tells clients to back off. Clients implement exponential backoff with jitter.

**Load shedding vs back pressure:**
Back pressure propagates the "slow down" signal upstream. Load shedding simply drops excess requests at the entry point. Load shedding is simpler and protects the system faster — but the user experiences errors. Back pressure is more graceful but requires end-to-end protocol support. Production systems often use both: back pressure up to a limit, then load shedding as a last resort.

**Queue depth as a back pressure signal:**
A simple but effective pattern: if a worker queue depth exceeds N (SQS queue depth, Kafka consumer lag, Redis list length), the producer slows its injection rate. Implemented via a feedback loop — the submission rate is a function of current queue depth.

---

These 20 questions cover the core breadth of system design at the level expected of staff-to-principal engineers. The model answers emphasize not just *what* the answer is, but *why* design decisions are made, what the tradeoffs are, and what failure modes exist — which is exactly what separates great candidates from good ones.