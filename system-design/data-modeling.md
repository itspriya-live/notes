# Data Modeling (System Design) — Elaborate Edition

**🔗 Source article:** https://www.hellointerview.com/learn/system-design/core-concepts/data-modeling
_Hello Interview — System Design in a Hurry → Core Concepts → Data Modeling_

> Style: deep, teaching-level notes — the *why* behind everything, analogies, worked examples, and interview guidance. This is a **complete** capture of the article (every section, table, and example), not a summary.

### Full section map (nothing skipped)
1. What data modeling is + where it shows up in the interview
2. Database Model Options — Relational (default) · Document · Key-Value · Wide-Column · Graph
3. Schema Design Fundamentals — Start with Requirements (3 drivers) · Entities/Keys/Relationships · Indexing · Normalization vs Denormalization · Scaling & Sharding
4. Conclusion — the 6-step whiteboard checklist

> **Interviewer reality check:** the bar is *much* lower than a dedicated data-modeling interview. You are **not** expected to fully normalize or produce a complete schema diagram — just something **clear, functional, and aligned with requirements**. It shows up **twice** in the delivery framework: (1) **core entities** during requirements (≈ 1:1 with tables), and (2) a **basic schema** during high-level design (key fields, relationships, a note on indexing/partitioning). A sloppy model causes pain later; a "good enough" one keeps the conversation where it belongs.

---

## 1. Database Model Options

**Golden rule: default to a relational database (PostgreSQL).** Resist the temptation to show off with exotic DB types. Only pick a specialized model if requirements **clearly** signal it. Knowing *when* the alternatives fit shows you think in tradeoffs — but SQL is the star.

### 1.1 Relational (SQL) — the default
- Data in **tables with fixed schemas**: rows = entities, columns = attributes.
- Relationships enforced via **foreign keys**; **ACID** guarantees for transactions.
- Most problems map naturally: social app (users, posts, comments, likes), e-commerce (users, products, orders, payments).

Example (social app):
```
users:  id (PK) | username | email | created_at
posts:  id (PK) | user_id (FK→users.id) | content | created_at
likes:  id (PK) | user_id (FK→users.id) | post_id (FK→posts.id) | created_at
```
- **Great at complex queries** (joins): "all posts by users a given user follows, ordered by recency." *But* be careful — multi-table joins can become **performance traps at scale**. Reporting-style queries raise yellow flags; think denormalized views / caching / precomputed results if needed.
- **Strong consistency** (ACID) is the right tool when it's a non-functional requirement — payments not double-charging, inventory not overselling.
- **Scalability knock is exaggerated:** modern SQL scales via **read replicas, sharding, connection pooling, caching**. Facebook & Airbnb run on relational foundations. Scaling is about how you architect *around* the DB, not just which DB.
- **Tech:** PostgreSQL, MySQL, SQLite.

### 1.2 Document Databases
- Store **JSON-like documents** with **flexible schemas**. Modeling = **nesting/embedding** related data rather than normalizing across tables.
- Example: embed a user's posts directly inside the user document → **eliminates joins**, but updating a post means finding + modifying the **entire user document**.
- **When over SQL:** frequently-changing schemas, deeply-nested data that would need many joins, or records with **vastly different structures** (e.g. user profiles where some have huge work histories, others minimal).
- **Interview caveat:** interviews **scope requirements to a clear, fixed set**, so you're unlikely to have "evolving schemas" — which removes the main reason to pick document DBs. Only choose if the interviewer **explicitly** mentions rapidly-changing structures.
- **Data-modeling impact:** denormalize aggressively, embedding related data → trades storage + update complexity for read performance.
- **Tech:** MongoDB, Firestore, CouchDB.

### 1.3 Key-Value Stores
- **Simple lookups by exact key.** Extremely fast, but limited querying beyond that.
- **When over SQL:** caching, session storage, feature flags, or lookups by a single identifier; high-write scenarios needing max performance without complex queries.
- **"Over SQL" is misleading** — you usually use **both together**: SQL as source of truth + a KV cache (Redis) in front for hot data → fast access **without** sacrificing durability/complex queries.
- **Data-modeling impact:** very **flat** schema; **denormalize heavily** and duplicate across keys to support different access patterns (no joins). Great for reads, **terrible for consistency** when data changes.
- **Tech:** Redis, DynamoDB, Memcached.

### 1.4 Wide-Column Databases
- Data in **column families**; rows can have **different sets of columns**. Optimized for **massive write-heavy** workloads and **time-series** data.
- New post → new row keyed by `(user_id, timestamp)`. Rows with the same partition key (`user_id`) stored together → **fast writes** (append to partition) and **efficient reads** (scan a contiguous range).
- **When over SQL:** enormous write volumes, time-series, append-heavy analytics/aggregations — telemetry, event logging, IoT sensors.
- **Data-modeling impact:** design around **query patterns** even more than SQL; often **duplicate across column families**; **time is a first-class citizen**.
- **Tech:** Cassandra, HBase.
- *(Ties to your paper study: this is the Dynamo/Cassandra world — partition key + clustering by time.)*

### 1.5 Graph Databases
- **Nodes + edges**, optimized for **traversing relationships**.
- **When over SQL:** *"Honestly? Almost never in interviews."* Classic examples (social networks, recommendations) are a **trap** — even **Facebook models its social graph with MySQL**; LinkedIn and Twitter use SQL for core relationship data too.
- Graph DBs **sound sophisticated** but add **unnecessary operational complexity**. Other DBs handle the primary query patterns (friends, friends-of-friends via an indexed join table) without that overhead.
- **Tech:** Neo4j, Amazon Neptune.

> ⚠️ **Terminology:** a **graph database** (nodes/edges storage — Neo4j) is NOT **GraphQL** (an API query language). Don't conflate them in an interview.

---

## 2. Schema Design Fundamentals

### 2.1 Start with Requirements — the 3 drivers
Everything flows from three factors (mostly already determined during requirements + API design). **All the later techniques (entities, keys, normalization, indexes, sharding) are just tools to address these three.**

1. **Data volume** → determines **where data can physically live.** Millions of users may force data across multiple stores → distinct schemas that must reference each other carefully.
2. **Access patterns** → **the most important factor**; drives most decisions. *How will data be queried?* Feed loading "recent posts by followed users" → denormalized data or careful indexes. **Falls out of your APIs** — for each endpoint ask: *"what queries must I support?"*
3. **Consistency requirements** → **how tightly coupled** data can be. Strong consistency (payments — no partial charges) → keep related data in **one ACID DB**. Eventual consistency (a like showing up seconds late is fine) → free to **distribute across systems** with schemas optimized per access pattern.

> **Interview move:** tie schema choices *explicitly* to these. *"Since we load feeds fast and likes can be eventually consistent, I'll denormalize like counts into the posts table."* → shows reasoning, not memorization.

### 2.2 Entities, Keys & Relationships
Map core entities → tables/collections with clear identifiers and relationships.
```
users:    id (PK), username, email
posts:    id (PK), user_id (FK → users.id), content, created_at
comments: id (PK), post_id (FK → posts.id), user_id (FK → users.id), content
likes:    user_id (FK → users.id), post_id (FK → posts.id)
```

**Primary keys** — use **system-generated IDs** (`user_id`, `post_id`), **NOT business data** (email).
> **Why (the real reason — stability, not just uniqueness):** business data **changes**. If `email` is the PK and a user changes their email, **every FK reference breaks** and needs cascading updates. System-generated IDs are **immutable** → references stay valid forever even when business rules change. (Email is unique too — uniqueness isn't the distinguisher; stability is.)

**Relationships (cardinality):**
- **1:N** — a user has many posts; a post has many comments.
- **N:M** — users like many posts; posts liked by many users (needs a join table).
- **1:1** — **rare in practice**, and usually a sign the **two tables should just be merged** into one (a strict 1:1 split buys you nothing but an extra join).

**Foreign keys** enforce **referential integrity** (no orphaned records — a post referencing a deleted user, comments on a deleted post). In SQL via FK constraints; in NoSQL via application logic.
> **The cost + at-scale tradeoff:** FKs make the DB **validate every insert/update** → **write overhead**. At very large (write-heavy) scale, some companies **drop FKs for write performance** and enforce integrity in **application code** — trading a free DB guarantee for throughput (at the risk of orphaned records). Mentioning this signals maturity.

**Constraints** — `NOT NULL`, `UNIQUE`, `CHECK` (emails unique, prices positive). Enforce correctness at the DB level; add some write overhead.

> Keep the schema **grounded in the domain** (users, tweets, follows for Twitter) — not abstract "entities/relationships."

### 2.3 Indexing for Access Patterns
Indexes = data structures that let the DB find rows **without scanning every row** (like a book's index → jump straight to the page). Call out **which columns are indexed and why**, tied to your **API endpoints**.

Social app examples:
- Index on `posts.user_id` → find all posts by a user.
- Index on `posts.created_at` → load recent posts chronologically.
- **Composite index on `(user_id, created_at)`** → efficiently load a **user's recent posts, newest-first**.

> **Why composite beats two single-column indexes** (worked example — `GET /users/{id}/posts` newest-first): the query **filters by `user_id` AND sorts by `created_at`**. A composite index stores rows **grouped by user_id, and within each user pre-sorted by created_at** → the DB jumps to the user's block and reads rows **already in time order** (no separate sort). Two separate indexes force the engine to pick one, then **sort at query time**.
> **Column-order rule:** put the **equality/filter column first** (`user_id`), the **range/sort column second** (`created_at`). `(created_at, user_id)` would NOT help — you can't jump to a user without scanning all timestamps first.

> **Interview move:** *"The `GET /users/{id}/posts` endpoint needs an index on `posts.user_id`"* — connect indexes directly to endpoints.
> (Deeper index mechanics — B-trees, hash indexes — live in the Database Indexing deep dive.)

### 2.4 Normalization vs Denormalization
**Normalization** = each piece of info lives in **exactly one place** (user data only in `users`, not copied elsewhere). Prevents **anomalies** where an update lands in one place but not another.

```
Normalized:                          Denormalized:
users(id, username, email)           posts(id, user_id, username, email, content, created_at)
posts(id, user_id, content, ...)        ^ username/email duplicated on every post row
```

> **The core danger of denormalizing (consistency):** with `username` copied onto every post, a **username change** forces updating **every post that user ever made**. **Miss one → inconsistent data.** And the article's key point: **consistency problems are much harder to solve than the performance problems you were trying to avoid** — so it's often a bad trade.

**Default: start normalized; denormalize only when needed.** Repeating data is wasteful and creates consistency problems.

**Exceptions where denormalization is acceptable** (common thread: **data is write-once / append-only, or consistency is deprioritized → no updates → no drift**):
1. **Analytics & reporting** — aggregating data that **changes infrequently**.
2. **Event logs & audit trails** — a **snapshot at a point in time** (never updated).
3. **Heavily read-optimized systems** (search engines) — **consistency < speed**.

> **The escape hatch:** even when you need denormalized fast reads, **put a cache in front** (Redis) holding the denormalized/precomputed representation (pre-joined, pre-aggregated). Your **source of truth stays clean and normalized**; the cache serves fast reads. Best of both.

### 2.5 Scaling and Sharding
When data outgrows a single machine, **shard** it across machines. Choose a partition strategy that keeps **related data together**.

**Shard by your primary access pattern.** Dominant query "posts by user" → shard by **`user_id`** → a user's posts stay on one shard → the dominant query hits **one shard**, avoiding expensive cross-shard queries.

> **Cost of that choice:** a **timeline** ("posts from the many users a person follows") spans users **scattered across many shards** → you must **query multiple shards and merge** (scatter-gather) — expensive and complex. You can't optimize every access pattern; optimize the dominant one and handle cross-cutting queries separately (e.g. a precomputed feed).

**⚠️ Time-range sharding trap:** sharding by time ("this week's shard") sounds great for "recent posts" — but **all current writes hit the same (latest) shard** → a **hot shard** that bottlenecks writes while older shards idle. **Anti-pattern for write-heavy systems.** Time-range partitioning is appropriate only for **archival/analytics** where recent data is read-heavy but **writes are spread out** (not a firehose landing on "now").

> **Your shard key is effectively permanent and affects every query.** Think hard about primary access patterns before choosing.

---

## 3. Conclusion — the 6-step whiteboard checklist

Data modeling is core but **not the focus** — design a reasonable schema, then move on. Outline **core entities** early; then when introducing a **database component** in high-level design, walk these **6 steps in order**:

1. **Determine the type of database** (default: relational/PostgreSQL).
2. **List the columns** needed to fulfill the functional requirements for each entity.
3. **Specify primary & foreign keys** for each relationship.
4. **Determine which columns need indexes** (if any) — tied to API query patterns.
5. **Decide whether to denormalize** for performance (default: don't).
6. **Consider whether sharding is necessary** — if yes, **choose a shard key matching your main access pattern**.

Memory hook: **Type → Columns → Keys → Indexes → Denormalize? → Shard?**

---

## 4. Quick Reference

| Topic | Default / Key point |
|---|---|
| DB default | **Relational / PostgreSQL** — resist exotic choices |
| Document DB | evolving/nested schemas (rare in interviews — needs explicit signal) |
| Key-Value | cache/session/flags; used **alongside** SQL, not instead |
| Wide-Column | massive writes / time-series (Cassandra, HBase) |
| Graph DB | **almost never** — even Facebook uses MySQL for the social graph |
| 3 drivers | data volume · **access patterns (most important)** · consistency |
| Primary key | **system-generated ID** — immutable (email changes break FKs) |
| 1:1 relationship | rare → usually **merge the two tables** |
| Foreign keys | integrity, but **write overhead**; dropped at extreme write scale (app enforces) |
| Composite index | `(filter_col, sort_col)` e.g. `(user_id, created_at)` — filter + sort, no re-sort |
| Normalization | **start normalized**; denormalize only for analytics/logs/read-optimized, or via a cache |
| Shard key | match **dominant access pattern** (e.g. `user_id`); cost = cross-shard timelines |
| Time-range shard | **hot-shard anti-pattern** for writes; ok for archival/analytics |
| Terminology | graph **database** ≠ **GraphQL** (API layer) |

---

## 5. Self-Test Q&A (tricky, understanding-level)

**Q1. Social network with friend traversal — why is a graph DB usually the wrong interview instinct?**
Graph DBs add operational complexity; SQL handles friends/friends-of-friends via an **indexed join table**, and even **Facebook runs its social graph on MySQL** (LinkedIn/Twitter use SQL too). Default to **PostgreSQL**. (Only true arbitrary-depth multi-hop traversal justifies a graph DB — rare in interviews.) Note: **graph database ≠ GraphQL**.

**Q2. The 3 requirements drivers? Which matters most?**
**Data volume · access patterns · consistency requirements.** **Access patterns** matter most — they drive indexes, denormalization, and shard key, and fall out of your APIs (ask "what queries does each endpoint need?").

**Q3. Payments vs activity feed — consistency + modeling impact?**
Payments = **strong consistency** → one **ACID SQL DB**, keep related data together. Feed = **eventual consistency** → free to **distribute across systems** and **denormalize** (e.g. like counts on posts) for read speed.

**Q4. Why system-generated PKs over email?**
**Stability**, not just uniqueness (email is unique too). Business data changes — if email is the PK, changing it **breaks every FK reference** and needs cascading updates. System IDs are **immutable**.

**Q5. What's special about 1:1 relationships?**
**Rare in practice**, and usually a sign the **two tables should be merged** into one — a strict 1:1 split just adds a needless join.

**Q6. Why do some large companies drop foreign keys? How preserve integrity?**
FKs **validate every write** → overhead. At extreme **write scale**, drop them for throughput and enforce **referential integrity in application code** (risking orphaned records).

**Q7. `GET /users/{id}/posts` newest-first — which index and why composite?**
Composite `(user_id, created_at)`. The query **filters by user AND sorts by time**; the composite stores rows grouped by user, pre-sorted by time → jump + read in order, **no query-time sort**. Two single-column indexes force picking one, then sorting. Order matters: **equality col first, range/sort col second**.

**Q8. Core danger of denormalization, and why "worse"?**
**Consistency/drift**: duplicated `username` means a username change must update every post; miss one → inconsistent. It's worse because **consistency bugs are harder to fix than the read-perf problem** you were solving.

**Q9. When is denormalization acceptable?**
**Analytics/reporting**, **event logs/audit trails**, **read-optimized systems (search)**. Common thread: **write-once/append-only or consistency deprioritized → no updates → no drift**. (Or denormalize in a **cache** while the DB stays clean.)

**Q10. Shard key for "posts by user" — choice, why, and the cost?**
`user_id` — co-locates a user's posts so the dominant query hits **one shard**. Cost: a **follow-timeline** spans many users across many shards → expensive **scatter-gather + merge**. Shard key is near-permanent.

**Q11. Why is time-range sharding a write-heavy anti-pattern? When ok?**
All current writes hit the **latest** shard → **hot shard**, others idle. Ok only for **archival/analytics** where reads dominate and writes are spread out.

**Q12. The 6-step DB whiteboard checklist?**
Type → Columns → Keys (PK/FK) → Indexes → Denormalize? → Shard (key = main access pattern)?
