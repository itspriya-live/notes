# Database Indexing (System Design) — Elaborate Edition

**🔗 Source article:** https://www.hellointerview.com/learn/system-design/deep-dives/db-indexing
_Hello Interview — Deep Dives → Database Indexing (studied from the premium/full version)_

> Style: deep, teaching-level notes — the *why* behind everything, analogies, worked examples, and interview guidance. This is a **complete** capture of the article (every index type, mechanism, and tradeoff), not a summary. Where a precise number, name, or edge case goes one level deeper than the article's prose (e.g. exact B-tree fan-out, bloom-filter false-positive property, partial indexes, TWCS), it's flagged as **elaboration** so you know what's verbatim-source vs. interview-depth built on top of it.

### Full section map (nothing skipped)
1. How Database Indexes Work — physical storage (heap files, pages, disk vs memory), the **cost** of indexing (space, write amplification)
2. B-Tree Indexes — structure (balanced, high fan-out, node = page), why they're the **default**
3. LSM Trees — memtable → SSTables → compaction, bloom filters, the read/write tradeoff
4. Hash Indexes — O(1) exact match, and the range/sort limitation
5. Geospatial Indexes — the 2D problem, Geohash / Quadtree / R-Tree
6. Inverted Indexes — full-text search (Elasticsearch/Lucene)
7. Index Optimization Patterns — Composite (leftmost-prefix) · Covering indexes
8. When NOT to index — cardinality vs **selectivity**

> **The one-sentence thesis:** an index trades **write speed + storage** for **read speed**. Every index you add is a duplicate structure the DB must keep in sync on every write. So the whole topic reduces to one question: *does this specific query read a small enough slice of the table to be worth the write tax?*

---

## 1. How Database Indexes Work

### 1.1 Physical storage — what the DB is actually doing
- Table rows live in **heap files** on disk, grouped into fixed-size **pages** (commonly **8 KB** in Postgres; 16 KB in some engines). The page is the **unit of I/O** — the DB reads/writes a whole page at a time, never a single byte.
- **Disk vs memory is the entire game.** A random disk seek is ~**milliseconds**; an in-memory operation is ~**nanoseconds** — a difference of ~10⁴–10⁶×. So the metric that matters for index performance is **how many page reads (disk seeks)** a query costs — **not** the CPU comparison count.
- Without an index, finding rows matching a `WHERE` clause means a **full table scan**: read every page. An index is a **separate, ordered data structure** that lets the DB jump near the answer without scanning everything — like a book's index pointing you to a page number.

### 1.2 The cost of indexing (this is the whole tradeoff)
An index is not free. Three distinct costs:

1. **Storage** — the index is a **duplicate structure** on disk (the indexed column(s) + a pointer back to the row). More indexes = more disk.
2. **Write amplification** — **every** INSERT/UPDATE/DELETE that touches an indexed column must also **update every index** on that column. 4 indexes → up to 4 extra index writes per row write.
   - *Elaboration:* on a **B-tree** those writes can trigger **node splits / rebalancing** (random-access writes), not a cheap append. On an **LSM** they add to **compaction** load and (for deletes) **tombstones**.
3. **Not free even when unused** — a poorly-chosen index the query planner ignores is **pure cost, zero benefit**.

> **Worked example (write-heavy, read-rare):** an audit-log table doing ~50,000 INSERTs/sec, read only by rare overnight investigations. Adding 4 B-tree indexes = up to **200,000 extra index maintenance ops/sec** on the hot write path, to speed up reads that almost never happen. **Wrong trade.** The best answer is often **no index** (accept an overnight table scan); if you *must* index a write-heavy table, reach for an **LSM-based** store that turns writes into sequential appends.

---

## 2. B-Tree Indexes — the default

### 2.1 Structure
- A B-tree (in practice a **B+ tree**) is a **balanced tree with high fan-out** — **NOT a binary tree**. Each node has **many** children, and **each node = one disk page**.
- Balanced ⇒ all leaves are at the same depth ⇒ **every lookup costs the same** number of page reads.
- **Why "not binary" is the key insight:** the number of **page reads ∝ tree depth**. A binary tree (fan-out 2) is deep; a high-fan-out tree is **shallow** → far fewer disk seeks.

> **The fan-out math (elaboration — memorize this, interviewers love it):**
> One 8 KB page holds ~**hundreds** of (key + pointer) entries — call it fan-out ≈ **500**.
> ```
> Depth 3:  500³  ≈ 125 million rows
> Depth 4:  500⁴  ≈ 62 billion rows
> ```
> So a point lookup on a **billion-row** table is only **~3–4 page accesses**. And because the **root and upper levels are hot, they stay cached in RAM** → often just **1–2 real disk seeks**. A binary tree over 62B rows would be ~**36 levels** — same Big-O *shape* (logarithmic), but the fan-out sits in the **base of the log**, turning 36 seeks into 4. That base is the whole point.

### 2.2 Why B-trees are the default
- **Balanced reads for point lookups AND range queries** — because keys are stored **in sorted order**, a B-tree efficiently serves `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, and **leading-anchored prefix** matches (`LIKE 'abc%'`).
- **Real-world:** PostgreSQL, MySQL/InnoDB, MongoDB all default to B+ trees.
- **Weakness:** writes do **in-place updates** with possible **node splits/rebalancing** → random writes. Fine for balanced/read-heavy workloads; costly under extreme write volume (→ LSM).

---

## 3. LSM Trees (Log-Structured Merge Trees)

### 3.1 How they work — the write path
- Writes go to an **in-memory sorted structure, the memtable** → **fast writes** (no disk seek, just an in-memory insert).
- When the memtable fills (size/time threshold) it is **flushed to disk as an immutable, sorted file: an SSTable** (Sorted String Table).
- **SSTables are immutable** — never modified in place. This is the design's core: a flush is a **sequential append** of a whole sorted file (fast sequential I/O), vs a B-tree's random in-place updates.
- **Consequence: SSTables accumulate.** An update to key `X` doesn't overwrite the old value — it writes a **new** value into the memtable that later flushes to a **newer** SSTable. A delete writes a **tombstone** (§3.4). So key `X` can physically exist in **several** SSTables; **the newest version wins**.

### 3.2 The read path — why reads can be *slower* than a B-tree
To answer a point query, the engine checks, **newest → oldest**, stopping at the first hit:
1. the **active memtable** (in memory),
2. then **SSTables on disk**, newest first (the key might be in any of them).

Worst case (a key that doesn't exist, or lives only in the oldest SSTable) → touch **many** SSTables. This is **read amplification** — LSM's price for cheap writes.

### 3.3 The two mechanisms that keep reads sane ⭐ (weak spot — name both)
1. **Bloom filters** — a small **in-memory** probabilistic structure kept **per SSTable**. Before touching an SSTable on disk it answers *"could key X be here?"*
   - **"no" is definitive** → skip that SSTable, **no disk read**. (No false negatives.)
   - **"maybe"** → read it (false positives possible).
   - Effect: turns *"check every SSTable"* into *"check the ~1–2 that might actually hold the key."*
2. **Compaction** — the **background** process that **merges** many small SSTables into fewer larger ones, discarding overwritten/tombstoned versions. Keeps the **SSTable count small** → bounds worst-case read amplification.
   - *(Supporting: each SSTable has a sparse/sorted index, so once inside the right SSTable you binary-search rather than scan.)*

### 3.4 Deletes = tombstones ⭐ (weak spot)
- A DELETE can't erase from an immutable SSTable, so it **writes a tombstone marker** (a "deleted" entry) to the memtable → new SSTable. Reads see the newest version = tombstone → return "not found."
- **Counterintuitive:** right after a delete, **disk usage goes UP, not down** — you now store the original row (old SSTable) **plus** the tombstone. **A delete is a write.** Space is reclaimed **only later, during compaction**, when the tombstone + shadowed value are both dropped.
- **The tombstone problem (a READ cost, not just compaction):** tombstones must **linger** (until compaction / `gc_grace_seconds` in Cassandra) so deletes propagate. A **range read** over a heavily-deleted key range must **read and skip every tombstone** to find the few live rows. A queue-like "delete old, read head" table can scan **thousands of tombstones** to return a handful of rows — Cassandra even aborts scans past a tombstone threshold. This is why **LSM handles high-delete / queue workloads poorly**.

### 3.5 When to use
- **Write-heavy, append-heavy, timestamp-ordered** workloads where reads are often recent. **Cassandra, RocksDB, DynamoDB, LevelDB, HBase.**
- Accept slower/read-amplified reads in exchange for high write throughput.

> **⚠️ Repeated updates to the same row are hostile to LSM** ⭐ (weak spot — don't reverse this): a hot counter (`like_count += 1` thousands of times) writes a **new version each time**, piling up stale copies across SSTables → burdens reads + compaction. A **B-tree updates in place** — one location, overwritten each time — so **B-tree is the better fit for repeated same-row updates.**

---

## 4. Hash Indexes

- **How:** hash the key → bucket holding a pointer to the row. **O(1) average** exact-match lookup.
- **When over a B-tree:** pure **exact-match, read-heavy** workloads with **no range/sort** need — e.g. a **URL shortener** (`WHERE short_code = ?` → the long URL). O(1) beats the B-tree's O(log n).
- **The limitation that makes it the wrong *default*:** hashing **destroys order**, so hash indexes support **only equality**. **No range queries, no sorting, no `ORDER BY`, no prefix scans.** A B-tree keeps keys ordered, so it's the sensible general default; use a hash index only when the workload is exclusively exact-match.
- *Elaboration:* O(1) is **average** — **collisions** (multiple keys → same bucket) degrade the worst case; high-entropy keys keep collisions rare.

---

## 5. Geospatial Indexes

**The problem:** proximity queries — *"restaurants within 5 km of me"* on `(latitude, longitude)`.

**Why a composite B-tree `(latitude, longitude)` performs poorly:** it sorts **primarily by latitude, then longitude only as a tiebreaker** within the same latitude. That's a **1-dimensional order over 2-dimensional data** — it can narrow the latitude band but within it the longitude ordering doesn't preserve nearness. Two points at the same latitude but opposite ends of the world sort adjacently; two genuinely close points can be far apart in the index. Result: scan a wide latitude strip and filter → slow.

**Specialized approaches:**
- **Geohash** — recursively subdivide the map (base-32 encoding); each added character = a smaller cell. **Nearby locations share a common prefix**, so proximity search becomes a **prefix match** — which a **plain B-tree can serve** (store the geohash string, query by prefix).
  - *Boundary gotcha (elaboration):* two close points can straddle a cell boundary and get **different prefixes** → also query the **8 neighboring cells**, then compute exact distances and filter.
- **Quadtree** — recursively splits space into 4 quadrants, subdividing dense regions further.
- **R-Tree** — groups nearby objects into **bounding boxes** arranged in a tree.

> Interview move: default to **Geohash on a B-tree** — it converts a hard 2D problem into a 1D prefix problem your existing index already handles.

---

## 6. Inverted Indexes

**The problem:** full-text search — *"find every article whose body contains 'database indexing'"* → `WHERE body LIKE '%database indexing%'`.

**Why a B-tree can't help:** a B-tree indexes the **whole column value as one sorted string**, anchored at the first character — it has no knowledge of the **words inside** the text. A **leading `%`** wildcard defeats the sort order entirely → **full table scan**, substring-matching every body. Hopeless at millions of rows.

**How an inverted index solves it:**
- It **inverts** the mapping: instead of *document → its words*, it stores **word (term) → the list of documents containing it** (a **postings list**).
- A search for `database indexing` = look up `"database"` → doc list; look up `"indexing"` → doc list; **intersect** the two. Each term lookup is fast (the term dictionary is itself indexed); **no body scanning**.
- Postings lists often store **term frequency / positions**, powering **relevance ranking** (TF-IDF / BM25).
- **Real system (from the article): Elasticsearch / Lucene** (Lucene is the inverted-index engine).

---

## 7. Index Optimization Patterns

### 7.1 Composite indexes & the leftmost-prefix rule ⭐
A composite index on `(A, B)` is sorted by **A first, then B only within each A**. It can be used efficiently **only for a leftmost prefix** of its columns.

Worked example — index on `(last_name, first_name)`:

| Query | Uses index? | Why |
|---|---|---|
| `last_name='X' AND first_name='Y'` | ✅ Fully | both columns, in order — jumps to exact spot |
| `last_name='X'` | ✅ | leftmost column — contiguous range scan |
| `first_name='Y'` | ❌ | **skips the leftmost column** → `Y` rows scattered across every last_name → **full scan** |

- **Analogy:** a phone book sorted by last then first name. Finding "Sharma, Priya" or all "Sharma"s is instant; finding *everyone named Priya* means flipping the whole book.
- **Column-order rule:** **equality/filter column first, range/sort column second.**
- **To serve `first_name` alone:** add a **separate `(first_name)` index** — the composite can't be reused. **Tradeoff:** another index = more **write amplification + storage**; only worth it if that query is frequent enough.

### 7.2 Covering indexes ⭐
A **covering index** includes **all columns a query needs**, so the query is answered **entirely from the index** — no trip back to the table.

Worked example — `SELECT order_id, status FROM orders WHERE customer_id = 12345`:
- **With only `(customer_id)`:** two steps — (1) **index seek** to find matching entries, then (2) a **heap fetch** back to the table for each match to read `order_id`/`status` (the index only stores `customer_id` + a row pointer). Each heap fetch is a potential **random disk seek per row** — that's the cost to eliminate.
- **Covering index:** `CREATE INDEX ... ON orders (customer_id, order_id, status);` — now the leaf entries carry `order_id` and `status`, so the query is served from the index alone. **Zero heap fetches.**
- *Elaboration:* Postgres/SQL Server support `INCLUDE (order_id, status)` — extra columns stored in the leaf **but not in the sort key**, slightly leaner than making them key columns.
- **Tradeoff:** a **wider** index → more storage + write cost. Reserve for **hot, high-frequency** queries.

---

## 8. When NOT to index — cardinality vs selectivity ⭐ (weak spot)

- **Cardinality** = a property of the **column**: how many **distinct values** it has.
- **Selectivity** = a property of the **query**: what **fraction of rows** a specific `WHERE` returns. **Low fraction → highly selective → good for an index.**
- Cardinality is the usual **proxy** for selectivity, but it **breaks on skewed data** — and when they diverge, **selectivity wins.**

**Rule of thumb:** an index helps when a query returns **< ~5–10%** of the table. Above that, a full scan is cheaper than index-seek + heap-fetches.

| Column / query | Index? | Reason |
|---|---|---|
| `gender` 50/50, `WHERE gender='F'` | ❌ | low cardinality → matches ~half the table → unselective |
| `email` unique, `WHERE email=?` | ✅ | high cardinality → 1 row → highly selective (login lookups) |
| `status` 99% done / 1% pending, `WHERE status='pending'` | ✅ | **low cardinality but the query is highly selective (1%)** — use a **partial index** |
| `country` 92% US, `WHERE country='Iceland'` | ✅ | same column is **useless** for `='US'` (92%) but **great** for a rare value |

> **Partial / filtered index** (the tool for skew): `CREATE INDEX idx_pending ON orders (status) WHERE status='pending';` — indexes **only** the rare, selective slice. Tiny, cheap to maintain, ignores the common value entirely. Best of both worlds.

---

## 9. Quick Reference

| Topic | Key point |
|---|---|
| Core tradeoff | index = faster reads, at cost of **storage + write amplification** |
| What matters | **page reads (disk seeks)**, not CPU comparison count |
| B-tree | balanced, **high fan-out (not binary)**, node = page; ~**3–4 reads for billions of rows**; the **default** (point + range + sort) |
| LSM | memtable → **immutable SSTables** → compaction; fast **sequential writes**, **read amplification** |
| Bloom filter | per-SSTable, in-memory; **"no" is definitive** → skips SSTables (no false negatives) |
| Compaction | merges SSTables, drops stale/tombstoned versions → bounds read cost |
| LSM delete | **tombstone**; disk **goes up** first; **tombstone problem = slow *reads*** over deleted ranges |
| Repeated same-row update | **B-tree in-place = good; LSM = bad** (piles up versions) |
| Hash index | **O(1)** exact match; **no range/sort** → wrong default |
| Geospatial | composite B-tree fails (1D order, 2D data); **Geohash** (prefix on B-tree), Quadtree, R-Tree |
| Inverted index | **term → doc list** (postings); Elasticsearch/Lucene; beats `LIKE '%...%'` (leading wildcard = scan) |
| Composite index | **leftmost-prefix**; **equality col first, range/sort col second** |
| Covering index | includes all queried cols → **no heap fetch**; wider index |
| Index decision | **selectivity** (query), not just **cardinality** (column); **partial index** for skew |

---

## 10. Self-Test Q&A (tricky — weak spots baked in)

**Q1. 50k INSERT/s, read-rare audit log — 4 B-tree indexes: good idea? Cost mechanism?**
No. **Write amplification** — each write updates up to 4 indexes (~200k extra ops/s), plus B-tree node splits, to speed reads that almost never run. Best answer: **no index** (accept overnight scan); if forced, an **LSM** store (sequential appends).

**Q2. Why is calling a B-tree a "balanced binary search tree" wrong, and why does it matter?**
It's **n-ary, high fan-out**, not binary. Each node = a page, and **page reads ∝ depth**. High fan-out (~500/page) keeps a billion-row tree ~3–4 levels deep → ~3–4 seeks (often 1–2 with caching); binary would be ~36. Disk seeks dominate, so the log **base** is everything.

**Q3. ⭐ Why can an LSM read be slower than a B-tree read, and what are the TWO mechanisms that fix it?**
A key may be in the memtable or **any** SSTable, so worst case you check them all newest→oldest (**read amplification**). **(1) Bloom filters** skip SSTables that can't hold the key ("no" is definitive); **(2) compaction** merges SSTables to keep the count small.

**Q4. Hash index vs B-tree for a URL shortener — advantage and the disqualifying limitation?**
Hash = **O(1)** exact match vs B-tree O(log n); the shortener only does exact lookups. Limitation: **no range/sort/prefix** — hashing destroys order — which is why it's the wrong **default** for general tables.

**Q5. Why does composite B-tree `(lat, long)` fail for "near me"? Fix?**
It sorts by latitude, then longitude as a tiebreaker — a **1D order over 2D data**, so it can't preserve proximity. **Geohash**: recursive base-32 cells where nearby points share a **prefix** → prefix scan on a B-tree (also check the 8 neighbor cells for boundary points).

**Q6. Why can't a B-tree serve `body LIKE '%database%'`? What does an inverted index map?**
B-tree indexes the whole string anchored at char 1; a **leading `%`** → full scan. An inverted index maps **each word → list of documents** (postings), so search = term lookups + list intersection. (Elasticsearch/Lucene.)

**Q7. ⭐ Index `(last_name, first_name)` — which of `{last+first, last, first}` use it, and the rule?**
`last+first` ✅, `last` ✅, `first` ❌ (skips the leftmost column → scattered → scan). **Leftmost-prefix rule**: equality col first, range/sort col second. Serve `first` alone with a separate `(first_name)` index (extra write cost).

**Q8. ⭐ `SELECT order_id, status WHERE customer_id=?` with a `(customer_id)` index — the two steps + the cost a covering index kills?**
Index seek → **heap fetch** per matching row (random disk seek) to read `order_id`/`status`. Covering index `(customer_id, order_id, status)` serves it from the index → **no heap fetch**. Cost: wider index / more write overhead.

**Q9. ⭐ Read-heavy posts table with hammered `like_count` — LSM or B-tree, and why (both parts)?**
**B-tree.** It's read-heavy, so LSM's read amplification is backwards. And the repeated `like_count` updates are **hostile to LSM** (immutable → a new version per increment piles up); a **B-tree updates in place**. Don't index `like_count`; index `(user_id, created_at)` for timelines.

**Q10. ⭐ DELETE in an LSM: mechanism, disk-usage direction, and the tombstone *problem*?**
Writes a **tombstone** (a delete = a write) → disk usage **goes UP** until compaction drops the tombstone + shadowed value. **Tombstone problem = a READ cost**: range scans over heavily-deleted ranges must read/skip thousands of lingering tombstones to find live rows (bad for queue-like workloads).

**Q11. ⭐ `status` 99% done / 1% pending, only query is `='pending'` — index? Cardinality vs selectivity?**
**Yes.** Low **cardinality** but the query is highly **selective** (1%). Selectivity (query) beats cardinality (column) on skewed data. Use a **partial index** `WHERE status='pending'` — tiny, cheap, ignores the 99%.

**Q12. Time-series metrics (millions of writes/s, recent reads, 30-day expiry) — engine, index, expiry?**
**LSM** (write-heavy, append-only, recent reads). Key **`(server_id, metric_name, timestamp)`** — leftmost-prefix filter + sorted time-range scan. Expiry: **NOT per-row deletes** (tombstone flood) — use **TTL + time-windowed partitions/SSTables (e.g. TWCS)** so whole old files are **dropped wholesale**, zero tombstones.

**Q13. In one minute: the costs of adding an index + the one question you always ask.**
Costs: **(1) storage** (duplicate structure), **(2) write amplification** (every write maintains it; B-tree splits / LSM compaction+tombstones), **(3) dead weight** if the planner ignores it. The question: **"How selective is the query this index serves?"** < ~5–10% of rows → worth it; most of the table → skip. Weigh against the write load.
