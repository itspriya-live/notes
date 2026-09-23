# Caching (System Design) — Elaborate Edition

**🔗 Source article:** https://www.hellointerview.com/learn/courses/system-design/lesson/thinking-in-scale/caching
_Hello Interview — System Design course → Thinking in Scale → Caching (full lesson, not paywalled)_

> Style: deep, teaching-level notes — the *why* behind everything, analogies, worked examples, and interview guidance. This is a **complete** capture of the lesson (every location, pattern, eviction policy, and problem), not a summary. Where a precise name, number, or edge case goes one level deeper than the lesson's prose (e.g. probabilistic early expiration, `SETNX` locking, fingerprinted filenames, TWCS-style expiry), it's flagged as **elaboration** so you know what's verbatim-source vs. interview-depth built on top of it.

### Full section map (nothing skipped)
1. When (and when not) to cache — the signals, the **quantify-first** discipline, the over-caching warning
2. Where to cache — the **four locations**: client-side · CDN · in-process · external/distributed
3. Cache architectures (read/write patterns) — cache-aside · read-through · write-through · write-back
4. Eviction policies — LRU · LFU · FIFO · TTL
5. Invalidation & consistency — TTL vs invalidate-on-write, delete-vs-update, the cache-aside race
6. Common problems — cache stampede (thundering herd) · hot keys · cache-node failure (load-bearing)
7. Introducing caching in an interview — the 5-step framework + proactively-raised downsides

> **The one-sentence thesis:** a cache trades **consistency + a new failure mode** for **read latency + reduced datastore load**. So the whole topic reduces to two questions: *does the read pattern justify it (quantify first)?* and *how much staleness can this data tolerate (which sets your strategy)?*

---

## 1. When (and when not) to cache

### 1.1 The signals that should trigger "add a cache"
- **Read-heavy workload** — the same data is read far more often than it's written.
- **Expensive / repeated queries** — a query that's costly to compute (joins, aggregations, fan-out) and runs repeatedly with the same inputs.
- **Latency reduction** — you need responses faster than the datastore can deliver (in-memory ≈ sub-ms vs. disk-backed DB reads).

### 1.2 The discipline that separates strong from weak candidates ⭐
- **Don't reach for a cache by reflex.** The lesson is emphatic: **quantify the problem first** — read/write ratio, the actual latency you're cutting, the load you're shedding — and **verify a cache actually improves it** before adding one.
- **A cache is a targeted optimization, not a first-line design choice.** Weak candidates say "add Redis" the moment reads come up; strong candidates prove the need, then apply it surgically to the hot path.

### 1.3 The over-caching warning + the cheaper alternative ⭐ (weak spot)
- **"Don't cache everything."** A cache adds a whole **failure mode and consistency surface** for what is sometimes a problem you could solve more cleanly. It can also **mask a bad query** you should have fixed at the source.
- **The named alternative to reach for first: fix the underlying system** — add/optimize a **database index**, **tune the query**, or **scale the DB with read replicas** — before bolting on a cache.

> **Read replica vs. cache** (elaboration — a common interview follow-up): a **read replica** is a synced copy of the DB that serves reads only (writes go to the primary), spreading read *load* across machines; it keeps the DB's own consistency model (small **replication lag** → possible read-after-write staleness). A **cache** makes *repeated reads of hot data* near-instant and offloads them entirely, but you own TTL/invalidation. Rule: **replica scales *how many* diverse reads you serve; cache makes *repeated* hot reads instant.** They compose — cache the hot keys, replicate the rest.

---

## 2. Where to cache — the four locations

Match the **location** to *who shares the data* and *where it's read*. (Location is a separate axis from the read/write *pattern* in §3 — don't conflate "cache-aside" with a location.)

| Location | Where it lives | Best for | Note |
|---|---|---|---|
| **Client-side** | Browser cache, localStorage, mobile app storage | Data owned by one client; avoid a round trip entirely | You **can't purge it** — control only future fetches via cache headers |
| **CDN** | Geographically distributed edge nodes | **Global**, static or shareable content (images, CSS/JS, rendered HTML) | Value ∝ **hit rate**; a miss can be *slower* than origin |
| **In-process** | Inside the app server's own memory | Tiny, hot, rarely-changing data (config); lowest latency | **Not shared** across servers → inconsistency + hit-rate decay as you scale |
| **External / distributed** | A separate Redis/Memcached tier all app servers share | Shared dynamic data (sessions, hot query results) | One network hop per read; one consistent shared copy |

### 2.1 CDN latency — why it wins (elaboration)
- Dominated by **network round-trip time (RTT)** — bounded by speed of light + router hops, and you pay it **multiple times** per request (TCP + TLS handshakes before content flows).
- Rough one-way: same-region ~1–5 ms · same-continent ~20–50 ms · cross-ocean ~80–120 ms · antipodal ~200–300 ms. A CDN **collapses the distance** to a nearby edge → typically **5–10× faster** perceived latency on a **hit**.
- **Caveat:** only wins on a **cache hit**. A miss = edge fetches from origin (that slow trip) **+** the edge hop → can be marginally slower. So CDNs shine for **static, high-hit-rate** content.

### 2.2 In-process vs external — the tradeoff (from §9-style reasoning)
- **In-process advantage:** a local-memory read with **no network hop** — nanoseconds vs. sub-ms; effectively free.
- **In-process downsides across a fleet:** (1) **not shared** → the same key cached on N servers can drift; invalidate in N places. (2) **Hit rate decays as you scale** — each server only caches what *it* has seen, so data is re-fetched per server; more servers → smaller traffic slice each → more misses. (Plus memory duplication and cache lost on restart/deploy.)
- **Decision:** shared/global static → **CDN**; per-user or shared dynamic → **external (Redis)** (sessions *must* be reachable from any server, since requests load-balance — an in-process session cache needs fragile **sticky sessions**); tiny read-hot config → **in-process**.

---

## 3. Cache architectures (read/write patterns)

### 3.1 Cache-aside (lazy loading) — the common default
- **Read path:** app checks cache. **Hit** → return from cache. **Miss** → read DB → **write the value into the cache** → return.
- The cache is populated **on demand, by reads** — only requested data ever lands in it (hence "lazy").
- **The consistency problem:** with *pure* cache-aside, **writes go only to the DB and never touch the cache**, so after an update the cache keeps serving the **old** value until it's evicted. Fixes: **TTL** (self-expire) and/or **invalidate-on-write** (delete the key). See §5.

### 3.2 Read-through
- Like cache-aside, but the **cache itself** (not the app) fetches from the DB on a miss and populates. The app only ever talks to the cache. Moves the fill logic into the cache layer.

### 3.3 Write-through
- App writes to the **cache**, which writes **synchronously** to the DB; both updated before the write returns.
- **Tradeoff:** cache + DB stay consistent, at the cost of **higher write latency** (extra hop on every write). Its failure concern is **atomicity** — cache updated but DB write fails (or vice versa) → handle with atomic/ordered writes. ⭐ (This is the write-latency cost §1 warns about.)

### 3.4 Write-back (write-behind)
- App writes to the **cache**, returns **immediately**; the cache flushes to the DB **asynchronously / in batches** later.
- **Tradeoff:** fastest writes, but a **data-loss / durability risk** — if the cache node dies before flushing, unflushed writes are gone; and the DB lags the cache. ⭐

> ⭐ **Don't swap these two:** **write-through → atomicity cost + write latency; write-back → durability / data-loss risk.**

---

## 4. Eviction policies

A cache has finite memory; when full it must pick a victim.

| Policy | Evicts | Best for | Failure mode |
|---|---|---|---|
| **LRU** (Least Recently Used) | the item **least recently accessed** | "recently viewed", recency-biased access | **Scan burst**: a flood of one-off accesses evicts steadily-popular items (they're not the *most recent*) |
| **LFU** (Least Frequently Used) | the item with the **lowest access count** | "trending"/hot set read constantly | Stale high-count items linger (no recency decay) — was popular, now cold, still resident |
| **FIFO** | oldest inserted (ignores access) | simple; rarely ideal | evicts hot items just because they're old |
| **TTL** | any entry whose **time expires** | freshness bounds | orthogonal to the others (see below) |

- **LRU measures recency; LFU measures frequency.** Neither is universally right → real systems use **hybrids** (LRU-K, aged/decayed LFU).
- ⭐ **TTL is different in *kind*:** it's **time-triggered and absolute** (evict when wall-clock expires, *regardless* of memory pressure or popularity) — about **freshness/staleness**. LRU/LFU are **capacity-triggered and relative** (only fire when the cache is **full**, pick a victim *relative to other entries*) — about **memory management**. The axes are **orthogonal**, and caches run **both at once**: TTL bounds staleness, LRU/LFU handles the memory limit.

---

## 5. Invalidation & consistency ⭐ (weak spot — the hard problem)

### 5.1 TTL vs. invalidate-on-write
- **TTL:** stale value is served **until the TTL lapses**, then re-populated fresh. Simple, zero write-path cost, but you accept **bounded staleness** — bad for business-critical data (e.g. price).
- **Invalidate-on-write:** when the DB updates, **delete the cache key** → next read misses and repopulates fresh. Tight consistency, at the cost of **write-path work** + coupling.
- **Framing:** *TTL trades consistency for simplicity; invalidation trades simplicity for consistency.* Many systems use **both** — invalidate for immediacy, TTL as a safety net for missed invalidations.

### 5.2 Delete vs. update the key on write ⭐ (weak spot)
- **Update** (write the new value into the cache): next read is a **hit**, no repopulation miss — good for hot keys. **But** writing values in from multiple paths risks writing a **stale value** that sticks (see the race).
- **Delete** (just evict): next read misses and re-fetches from the DB (the source of truth). Simpler, **safer under concurrency**.
- **Safer default: delete (invalidate), not update.**

### 5.3 The classic cache-aside race ⭐ (weak spot — the #1 caching gotcha)
A read-miss-populate and a concurrent write-invalidate interleave badly:
1. **Reader** checks cache → **miss**.
2. **Reader** reads DB → gets the **old** value (holding it, about to write to cache).
3. **Writer** updates DB → **new** value.
4. **Writer** invalidates the cache key → deletes it (already empty → no-op).
5. **Reader** (still running) now **writes its stale value into the cache**.

**Result:** DB has the new value, cache holds the **old** — and **nothing corrects it** until the TTL lapses or the key is written again. The invalidation happened *before* the slow reader populated, so it "missed" it.
- Why a **TTL safety net** matters even when you invalidate: it bounds how long a stuck value survives.
- Robust fixes (elaboration): **write-through/read-through** (single cache-management path), **versioned keys**, a short **lock / Redis `SETNX`** around populate, or the **delayed double-delete** pattern.

---

## 6. Common problems

### 6.1 Cache stampede / thundering herd
- **What goes wrong:** a hot key with a TTL **expires**; in that instant **all** concurrent requests for it **miss simultaneously** and stampede the DB with the same expensive query at once → DB overloads → **cascading failure**. The cruel irony: the cache did its job perfectly right up to the instant it expired, then dumped all that load on the DB in one spike.
- **Mitigations:**
  - **Request coalescing / locking** — the first request to miss acquires a **lock** (e.g. Redis `SETNX`) and alone recomputes + repopulates; the rest **wait** or serve **stale** → DB sees **1 query, not N**.
  - **Early / probabilistic recomputation** — refresh the key **before** it expires. **Probabilistic early expiration:** as an entry nears its TTL, each request has a small, increasing random chance of being the one to recompute early — so one request refreshes ahead of expiry while the rest hit the warm cache; the randomization prevents a *new* stampede.
  - **Staggered / jittered TTLs** — add a random offset so keys populated together don't all expire in the same instant.

### 6.2 Hot keys
- **The problem (specifically in a sharded/distributed cache):** keys are sharded to nodes by a **hash of the key**, so one super-hot key (e.g. a viral post's score) lives on **one node** that absorbs a huge share of traffic and saturates — while other nodes idle. **Adding nodes doesn't fix it**: hashing routes the key to the *same* node; you scaled total capacity, not that one key's capacity. It's a load-**imbalance**, not a total-capacity, problem.
- **Mitigations:**
  - **Replicate the hot key** across multiple nodes (`key:1..N`, read a random copy) → fan out the read load.
  - **Push it closer to the client** — cache the hot value **in-process** on each app server (or at the CDN) with a short TTL, so the distributed cache barely sees it.
- **Hot key + short TTL is especially dangerous:** when *that* key expires, the *maximum-traffic* key stampedes the DB all at once — the worst possible stampede. Pair hot-key relief (replication / local caching) with stampede mitigation (locking + early recompute).

### 6.3 Cache-node failure — is the cache *load-bearing*? ⭐ (weak spot — distinct from hot keys)
- **Worse than a single key expiring:** when an **entire cache node crashes**, **everything it held vanishes at once**, so *all* the traffic it was absorbing hits the DB **simultaneously** — a stampede across **every** key at once.
- **The architectural question:** **is your cache optional or load-bearing?**
  - If the DB **cannot survive full traffic without the cache**, the cache is **load-bearing** → a cache outage = a **full outage**. Plan for it: **replicate the cache tier** for failover, **warm** a cold cache gradually rather than take full load instantly, and let the DB **degrade gracefully** (rate-limit / shed load).
  - **Ideal:** design the cache as a **performance optimization, not a correctness/availability dependency** — the DB should absorb or shed traffic if the cache vanishes, even if slower.

> ⭐ **Keep these three separate:** **stampede** (one key expires) · **hot key** (one key overloads its shard) · **node crash** (everything gone → load-bearing?). Three problems, three fixes.

---

## 7. Introducing caching in an interview — the framework

### 7.1 The 5-step sequence (present as a deliberate progression)
1. **Justify it first** — point at the quantified bottleneck (read-heavy / expensive-repeated / latency), and rule out the cheaper fix (index / query tune / read replica) first.
2. **Decide *what* and *where*** — the hot/expensive data; the location (client / CDN / in-process / external) matched to who shares it and where it's read.
3. **Choose the read/write pattern** — cache-aside (default) / write-through (sync consistency) / write-back (fast writes).
4. **Handle invalidation & consistency** — TTL to bound staleness, invalidate-on-write for immediacy, usually both.
5. **Call out the failure modes** — stampede, hot keys, node-crash / load-bearing — with mitigations.

### 7.2 Proactively raise the downsides (and why) ⭐
Volunteer the weaknesses of your own design rather than waiting to be poked — it signals **maturity and ownership** (caching is a tradeoff, not free) and steers the conversation onto prepared ground. Any two of: **consistency/staleness**, **cache stampede**, **hot keys**, **the cache becoming load-bearing**, **added write-path latency**, **operational complexity/cost**.

> **The one line to lead with:** *"A cache buys read latency and offloads the datastore, but it costs consistency and a new failure mode — so I only add it once I've quantified that the read pattern justifies it, and I match the strategy to how tolerant the data is of staleness."*

---

## 8. Quick Reference

| Topic | Key point |
|---|---|
| Core tradeoff | cache = faster reads + less DB load, at cost of **consistency + a new failure mode** |
| Discipline | **quantify first**; a cache is a targeted optimization, not a reflex; rule out index/query/**read replica** first |
| Locations | **client · CDN · in-process · external** — matched to who shares the data & where it's read |
| CDN | value ∝ **hit rate**; ~5–10× faster on a hit; a **miss can be slower** than origin |
| In-process vs external | in-process = no network hop (fastest) but **not shared** (drift + hit-rate decay); external = shared copy, one hop |
| Cache-aside | lazy-populate on read miss; stale on writes → fix with TTL / invalidate |
| Write-through | sync to DB → consistent, **write latency**, **atomicity** concern |
| Write-back | async flush → fast writes, **data-loss / durability** risk |
| Eviction | **LRU** = recency (scan-burst weakness), **LFU** = frequency (stale-count weakness); **TTL** = time-triggered/freshness (orthogonal to capacity-triggered LRU/LFU) |
| Invalidation | **delete > update** the key (avoids the stale-write race); TTL as safety net |
| Cache-aside race | slow reader repopulates stale value **after** writer invalidates → sticks until TTL |
| Stampede | key expires → all miss at once → DB overload; fix: **locking/coalescing** + **probabilistic early recompute** + jittered TTL |
| Hot key | one key overloads its **shard**; adding nodes won't help; fix: **replicate the key** + **local/CDN caching** |
| Node crash | **everything gone at once** → is the cache **load-bearing**? replicate tier + graceful DB degradation |
| Interview | **justify → what/where → pattern → invalidation → failure modes**; proactively raise downsides |

---

## 9. Self-Test Q&A (tricky — weak spots baked in)

**Q1. When should you reach for a cache, what MUST you do beyond naming the signal, and what's the cheaper alternative to consider first?** ⭐
Signals: read-heavy, expensive-repeated queries, latency reduction. Beyond naming: **quantify** it (read/write ratio, target latency) and verify a cache actually helps — don't add by reflex. Cheaper alternative first: **fix the DB** — add/optimize an index, tune the query, or add a **read replica** — a cache can *mask* a bad query.

**Q2. Name the four cache LOCATIONS (not patterns) and one fit for each.** ⭐
**Client-side** (data owned by one client), **CDN** (global static/shareable), **in-process** (tiny hot config), **external/distributed** (shared dynamic — sessions, hot query results). *Cache-aside is a **pattern**, not a location.*

**Q3. Walk the cache-aside read path (hit vs miss), and how do write-through and write-back differ (with the correct risk labels)?** ⭐
Hit → serve from cache. Miss → read DB → write into cache → return. **Write-through**: sync to DB → consistent, but **write latency + atomicity** concern. **Write-back**: async flush → fast writes, but **durability / data-loss** risk. (Don't swap those.)

**Q4. Match a policy: "trending posts" cache vs "recently viewed" cache. What does each measure, and LRU's classic failure?**
Trending → **LFU** (frequency keeps the hot set resident). Recently viewed → **LRU** (recency). LRU measures recency, LFU frequency. LRU failure: a **scan burst** of one-off accesses evicts steadily-popular items because they aren't the *most recent*.

**Q5. How is TTL different in kind from LRU/LFU?** 
TTL is **time-triggered/absolute** (evict on wall-clock expiry regardless of memory or popularity — about **freshness**). LRU/LFU are **capacity-triggered/relative** (fire only when full, victim chosen relative to other entries — about **memory**). Orthogonal; run both at once.

**Q6. ⭐ Cached price is stale after an update. TTL vs invalidate tradeoff; delete-vs-update the key; and the cache-aside race.**
TTL serves stale until it lapses (bad for price); invalidate-on-write deletes the key for immediate freshness at write-path cost — often use both. **Delete > update** (updating from multiple paths can write a stale value that sticks). **The race:** reader misses → reads old DB value → writer updates DB + invalidates (deletes empty key) → reader writes its **stale** value into cache → sticks until TTL. TTL is the safety net.

**Q7. ⭐ Homepage feed, 60s TTL, 2s query, 50k rps. What happens when the key expires, and two mitigations?**
All 50k requests **miss at once** and stampede the DB with the same 2s query → overload → cascading failure. Fix: **request coalescing/locking** (one recomputes, rest wait/serve stale → 1 DB query) + **probabilistic early recomputation** (refresh before expiry) (+ jittered TTLs).

**Q8. ⭐ A viral key gets 80% of traffic in a sharded cache. Why is this a problem, why doesn't adding nodes help, and two fixes?**
It hashes to **one node** that saturates while others idle — a load **imbalance**. Adding nodes re-shards but the key still maps to the same node → no help. Fixes: **replicate the hot key** across nodes (read a random copy) + **push it in-process / to the CDN**. (Distinct from a node crash.)

**Q9. ⭐ How is a full cache-NODE crash worse than a key expiring, and what design question does it force?**
A node crash loses **all** its keys at once → **all** that traffic hits the DB simultaneously (stampede across every key). It forces: **is the cache load-bearing?** If the DB can't survive without it, a cache outage = full outage → **replicate the tier**, warm gradually, and let the DB **degrade gracefully**. Ideally the cache is an optimization, not an availability dependency.

**Q10. In-process advantage over external, its two downsides across a 20-server fleet, and where you'd put session data vs config?** ⭐
Advantage: **no network hop** (local memory, fastest). Downsides: **not shared** (drift → invalidate in 20 places) + **hit rate decays as you scale** (each server re-populates the same keys). **Session data → external** (requests load-balance; must be reachable anywhere; in-process needs fragile sticky sessions). **Config → in-process** (tiny, read-hot, rarely changes → staleness/duplication barely matter).

**Q11. ⭐ URL shortener (read-heavy, global, immutable mappings, viral skew): where to cache, which pattern, which hard problem disappears, and why is the hot key easy here?**
**Where:** CDN edge (global) + external Redis + in-process for hot links (layered). **Pattern:** cache-aside. **Disappears: invalidation/consistency** — mappings are **immutable**, so a cached value can **never go stale**; nothing to invalidate, no TTL tradeoff, no cache-aside race. **Hot key easy:** an immutable value can be **replicated everywhere with zero consistency cost** (no sync, no short TTL, no refresh stampede) — unlike a live-changing scoreboard key.

**Q12. CDN holds a wrong article after a correction; the browser also cached it. Fix each, and the TTL principle.** ⭐
**CDN:** issue an active **purge / invalidation** on the URL → edge refetches immediately (vs. passively waiting for TTL). **Browser:** you **can't purge clients** → use **fingerprinted/versioned filenames** (`app.<hash>.js`) so a content change changes the URL → forced fresh fetch (lets you cache assets immutably). **TTL principle: set TTL = how much staleness the data can tolerate** (infinite for fingerprinted assets; near-zero for a live price).

**Q13. In 60 seconds: the checklist for adding a cache + the single most important justification.**
Checklist: **(1) justify** (quantify; rule out index/query/replica), **(2) what**, **(3) where** (client/CDN/in-process/external), **(4) pattern** (cache-aside/write-through/write-back), **(5) freshness** (TTL + invalidate), **(6) failure modes** (stampede/hot-key/load-bearing) with mitigations. Justification: *"I only add a cache after I've quantified the workload justifies it — and I match the strategy to how much staleness the data can tolerate."*
