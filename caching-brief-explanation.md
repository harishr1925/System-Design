# System Design: Caching - Brief Explanation

> Goal: build a reusable mental toolkit. For every concept, be able to answer:
> 1. What problem does it solve?
> 2. How does it work?
> 3. How does it scale?
> 4. What are the trade-offs?
> 5. Where would you use it in a real system?

**Why it matters:** caching reduces repeated work and can dramatically improve latency. The interesting interview part is rarely "use Redis". It is deciding **what** to cache, **where**, **for how long**, and **how to deal with stale data**.

**Interview focus:** explain where you would place a cache, what happens on a cache miss, and how you handle stale data.

**Free resource:** Cloudflare - "Get Started with Cache"

---

## Table of Contents

1. [Why caching improves performance](#1-why-caching-improves-performance)
2. [Cache hit vs cache miss](#2-cache-hit-vs-cache-miss)
3. [Cache-aside pattern (and other patterns)](#3-cache-aside-pattern-and-other-patterns)
4. [TTL](#4-ttl-time-to-live)
5. [Eviction policies](#5-eviction-policies)
6. [Cache invalidation](#6-cache-invalidation)
7. [Redis basics](#7-redis-basics)
8. [When caching can cause problems](#8-when-caching-can-cause-problems)
9. [Where to place a cache](#9-where-to-place-a-cache)
10. [Interview deep dive: miss flow and stale data](#10-interview-deep-dive)
11. [Interview Q&A bank](#11-interview-qa-bank)
12. [Self-test](#12-self-test)
13. [30-second summary](#13-30-second-summary)

---

## 1. Why caching improves performance

**Problem it solves:** the same expensive work (DB queries, API calls, computation, rendering) is repeated for every request, even though the answer rarely changes. That costs latency, database load, and money.

**How it works:** keep a copy of the result in a faster, closer store (usually memory). Later requests read the copy instead of redoing the work.

| Source | Typical latency |
|---|---|
| In-process memory | nanoseconds to microseconds |
| Redis / Memcached (network) | about 0.5 to 2 ms |
| Database query | about 5 to 100+ ms |
| Cross-region or third-party API | 100 ms to seconds |

**Why it works:** access patterns are skewed. A small share of data (popular products, hot profiles, trending posts) gets most of the reads (the 80/20 idea, called temporal and spatial locality).

**Best candidates for caching:**
- Read-heavy data (reads far outnumber writes).
- Expensive to compute or fetch.
- Tolerates being slightly stale.
- Requested repeatedly by many users.

**Poor candidates:** write-heavy data, data that must always be exactly current (account balance, inventory at checkout), and rarely repeated, unique requests.

**How it scales:** a cache absorbs most reads, so the database sees only misses. A 90% hit ratio means the DB sees about 10x less read traffic. Scale the cache itself by sharding keys across nodes (consistent hashing or Redis Cluster) and adding replicas.

**Trade-offs:** extra component to run, risk of stale data, added complexity (invalidation), memory cost.

**Where used:** product pages, user sessions, feeds, config, rate limiting counters, API responses, rendered pages, static assets (CDN).

---

## 2. Cache hit vs cache miss

| | **Cache hit** | **Cache miss** |
|---|---|---|
| Meaning | Requested key is in the cache | Key is absent (never cached, expired, or evicted) |
| Path | Cache returns data directly | Go to the source (DB), then usually store in cache |
| Latency | Fast (about 1 ms) | Slow (cache check + DB + cache write) |
| DB load | None | Full |

**Hit ratio** = hits / (hits + misses). This is the main health metric of a cache. A cache with a 50% hit ratio may barely help. 95%+ is typical for a well-chosen workload.

**Why a miss happens:**
- **Cold start:** the cache is empty after a deploy or restart.
- **Expiry:** the TTL ran out.
- **Eviction:** memory was full and the key was removed.
- **Invalidation:** the key was deleted on purpose after a write.
- **Never requested before.**

**Average latency formula:** `hit_ratio x cache_time + (1 - hit_ratio) x (cache_time + db_time)`.
Example: 95% hit, 1 ms cache, 50 ms DB gives about 0.95 x 1 + 0.05 x 51 = about 3.5 ms, versus 50 ms without a cache.

---

## 3. Cache-aside pattern (and other patterns)

### Cache-aside (lazy loading), the default answer

**Read path:**
1. App checks the cache.
2. **Hit:** return it.
3. **Miss:** app reads from the DB.
4. App writes the result to the cache (with a TTL).
5. App returns the result.

**Write path:** update the DB, then **delete** the cache key (the next read refills it).

```
read(key):
    v = cache.get(key)
    if v != null: return v            # hit
    v = db.query(key)                 # miss
    cache.set(key, v, ttl)
    return v

write(key, value):
    db.update(key, value)
    cache.delete(key)                 # invalidate, don't update
```

**Why delete instead of update on write?** Updating the cache racing with another writer can leave an older value in the cache. Deleting is simpler and safer.

**Pros:** simple, only caches what is actually requested, the cache can fail without breaking the system (falls back to the DB).
**Cons:** first request after a miss is slow, stale window exists, the app code owns the caching logic.

### Other patterns (know the names)

| Pattern | How | Pros | Cons |
|---|---|---|---|
| **Read-through** | Cache itself loads from the DB on a miss | App code simpler | Cache layer must know the DB |
| **Write-through** | Write to cache and DB together, synchronously | Cache always fresh | Slower writes, caches data that may never be read |
| **Write-behind (write-back)** | Write to cache, flush to the DB asynchronously | Very fast writes | Risk of data loss if the cache dies before flush |
| **Write-around** | Write to the DB only, skip the cache | Avoids polluting cache with write-once data | First read after write is a miss |

---

## 4. TTL (Time To Live)

**Problem it solves:** without expiry, stale entries live forever and memory fills up. TTL puts an automatic upper bound on staleness and on memory use.

**How it works:** each key stores an expiry time. After that, it is treated as missing (Redis: `SET key value EX 60`, `TTL key`).

**Choosing a TTL:**

| Data type | Typical TTL |
|---|---|
| Static assets, rarely changing config | Hours to days |
| Product details, profiles | Minutes to an hour |
| Feeds, search results, counters | Seconds to minutes |
| Highly dynamic data | A few seconds, or don't cache |

**Rule of thumb:** TTL = the amount of staleness the business can tolerate.

**TTL jitter:** add a random offset (for example TTL = 600 + random(0..60) seconds) so many keys don't expire at the same instant, which prevents a spike of misses (cache avalanche).

**Trade-offs:** short TTL means fresher data but more misses and DB load. Long TTL means faster and cheaper, but staler.

---

## 5. Eviction policies

**Problem it solves:** memory is limited. When the cache is full, something must be removed to make room for new entries.

| Policy | Evicts | Good for | Weakness |
|---|---|---|---|
| **LRU** (least recently used) | The key untouched the longest | General workloads, the usual default choice | A one-time scan can push out hot keys |
| **LFU** (least frequently used) | The key with the fewest accesses | Stable popularity patterns | Old popular keys can linger, needs counters |
| **FIFO** | The oldest inserted | Simple cases | Ignores usage entirely |
| **Random** | A random key | Cheap, surprisingly decent | Unpredictable |
| **TTL-based** | Keys closest to expiry | Mixed workloads with TTLs | Doesn't consider usage |

**Redis:** controlled by `maxmemory` and `maxmemory-policy`. Common cache choices are `allkeys-lru` or `allkeys-lfu`. `volatile-*` policies only evict keys that have a TTL. The default policy is `noeviction`, which rejects writes when full, so set it explicitly for a cache.

**LRU implementation question:** hash map + doubly linked list gives O(1) get and put.

---

## 6. Cache invalidation

**Problem it solves:** when the source data changes, the cached copy becomes **stale**. Keeping them consistent is the hardest part of caching.

| Strategy | How | Staleness | Cost |
|---|---|---|---|
| **TTL expiry** | Let it expire on its own | Up to the TTL | Lowest complexity |
| **Delete on write** | Update the DB, delete the key | Tiny window | Needs the write path to know the keys |
| **Write-through** | Update cache and DB together | Almost none | Slower writes |
| **Event-driven** | Publish change events (Kafka, DB CDC) and invalidate subscribers | Small | More infrastructure |
| **Versioned keys** | `product:42:v7`, bump the version on change | None for new reads | Old versions linger until TTL |

**Practical approach:** combine them. Use delete-on-write for freshness **and** a TTL as a safety net in case an invalidation is missed.

**Race condition to know:** a reader misses and reads the old value from the DB, a writer updates the DB and deletes the key, then the reader writes the old value into the cache. Result: stale data until the TTL. Mitigations: short TTLs, delayed double delete, or versioning.

**Multi-level caches:** invalidation must reach every layer (CDN, local caches, Redis), so prefer short TTLs on outer layers.

---

## 7. Redis basics

**What it is:** an in-memory key-value data store, commonly used as a cache, session store, rate limiter, queue, and leaderboard.

**Key facts:**
- Data lives in **RAM**, so reads and writes are sub-millisecond.
- Commands execute on a **single thread**, so each command is atomic (no locks needed for `INCR` and similar).
- Rich data types, not just strings.

| Type | Use |
|---|---|
| String | Cached JSON, counters, locks |
| Hash | Object fields (`HSET user:1 name Ana`) |
| List | Queues, recent items |
| Set | Unique members, tags |
| Sorted set | Leaderboards, time-ordered data |

**Essential commands:**
```
SET product:42 "{...}" EX 600     # set with a 10 minute TTL
GET product:42
DEL product:42
EXPIRE product:42 300
TTL product:42
INCR page:views
SET lock:42 1 NX EX 10            # simple lock: only if not exists
```

**Scaling and availability:**
- **Replication:** a primary with replicas for read scaling and failover.
- **Sentinel:** automatic failover for a primary/replica setup.
- **Cluster:** shards keys across nodes using 16384 hash slots for capacity and throughput.
- **Persistence (optional):** RDB snapshots and AOF logs. Pure caches often skip persistence.

**Redis vs Memcached:** Redis has richer data types, persistence, and replication. Memcached is simpler, and multithreaded for plain key-value caching.

---

## 8. When caching can cause problems

| Problem | What happens | Fixes |
|---|---|---|
| **Stale data** | Users see old values after a change | TTL, delete on write, event-driven invalidation |
| **Cache stampede (thundering herd)** | A hot key expires and many requests hit the DB at once | Lock or single-flight (one request rebuilds), refresh before expiry, serve stale while revalidating |
| **Cache avalanche** | Many keys expire together, or the cache goes down | TTL jitter, multi-AZ replicas, rate limiting the DB |
| **Cache penetration** | Requests for keys that don't exist always miss and hit the DB | Cache nulls with a short TTL, Bloom filter, input validation |
| **Hot key** | One key gets a huge share of traffic and overloads a node | Local in-process cache, replicate the key, split into shards |
| **Cold start** | Empty cache after restart sends a flood to the DB | Pre-warm, gradual traffic ramp |
| **Inconsistency across layers or nodes** | Different copies disagree | Short TTLs, versioned keys, central invalidation |
| **Memory pressure** | Eviction churn lowers hit ratio | Right-size memory, tune policy, cache smaller values |
| **Cache as a SPOF** | App collapses when the cache is down | Fall back to the DB with limits, replicas, circuit breaker |

**Core rule:** the system must stay correct, and survive, **without** the cache. The cache is an optimization, not the source of truth.

---

## 9. Where to place a cache

```
Browser cache
   -> CDN (static + cacheable responses, close to users)
      -> API gateway / reverse proxy cache
         -> Application in-process cache (Caffeine, Guava): fastest, per instance
            -> Distributed cache (Redis / Memcached): shared by all instances
               -> Database buffer pool / query cache
                  -> Database (source of truth)
```

| Layer | Best for | Limitation |
|---|---|---|
| **Browser / CDN** | Static files, public pages | Hard to invalidate, no per-user data (unless varied) |
| **In-process (local)** | Tiny, hot, rarely changing data | Each instance has its own copy, so they can differ |
| **Distributed (Redis)** | Shared data across instances (sessions, product data) | Network hop, extra system to run |
| **DB-level** | Query results, index pages | Not controlled by the app |

**Common design:** two levels. A small local cache (seconds of TTL) in front of Redis, in front of the DB.

---

## 10. Interview deep dive

### "Where would you place a cache?"
- Close to the consumer of the data, and as high as correctness allows.
- Static and public content: CDN. Shared, frequently read data: Redis. Tiny hot data: in-process.
- Not in front of write-heavy or must-be-exact data.

### "What happens on a cache miss?" (cache-aside)
1. App calls `cache.get(key)` and gets null.
2. App queries the DB.
3. App writes the value into the cache with a TTL (plus jitter).
4. App returns the value to the caller.
5. Under concurrency, only **one** request should rebuild the key (lock or single-flight), so others don't all hit the DB.
6. If the key doesn't exist in the DB, cache a short-lived null marker to avoid repeated misses.

### "How do you handle stale data?"
1. Decide how much staleness is acceptable per data type.
2. Set a TTL matching that tolerance.
3. Invalidate (delete) the key on writes so most staleness disappears immediately.
4. For critical data (price at checkout, stock), bypass the cache or read from the DB.
5. Consider versioned keys or event-driven invalidation for multi-layer caches.
6. Accept that it is a trade-off between freshness, latency, and load, and say so.

---

## 11. Interview Q&A bank

**Q1. Why use a cache?**
A: To cut latency and DB load by serving repeated reads from fast memory, so the system handles more traffic with fewer resources.

**Q2. What is a cache hit ratio and why does it matter?**
A: Hits divided by total lookups. It decides whether the cache is worth it: a high ratio means most requests skip the slow path.

**Q3. Explain cache-aside.**
A: The app checks the cache first. On a miss it reads the DB, writes the result to the cache with a TTL, and returns it. On writes it updates the DB and deletes the cache key.

**Q4. Cache-aside vs read-through?**
A: In cache-aside the application loads data on a miss. In read-through the cache layer does the loading itself, so the app only talks to the cache.

**Q5. Write-through vs write-behind?**
A: Write-through writes cache and DB synchronously, so data is consistent but writes are slower. Write-behind writes to the cache and persists later, which is faster but risks losing data if the cache fails first.

**Q6. Why delete the key on write instead of updating it?**
A: Updating risks races where an older write lands last and leaves a wrong value. Deleting makes the next read reload the current value.

**Q7. What is TTL and how do you pick it?**
A: A key's lifetime. Choose it from the staleness the business tolerates, then add jitter to avoid synchronized expiry.

**Q8. LRU vs LFU?**
A: LRU evicts what was used least recently and adapts quickly to changing access. LFU evicts what is used least often and favors consistently popular keys, but needs counters and can keep formerly hot keys too long.

**Q9. How would you implement an LRU cache?**
A: A hash map for O(1) lookup plus a doubly linked list ordered by recency. On access, move the node to the head. On insert when full, remove the tail.

**Q10. What is a cache stampede and how do you prevent it?**
A: A hot key expires and many requests rebuild it simultaneously, overloading the DB. Prevent with a lock or single-flight, TTL jitter, refreshing before expiry, or serving stale data while one request refreshes.

**Q11. Cache avalanche vs stampede vs penetration?**
A: Stampede: one hot key expires. Avalanche: many keys expire together or the cache goes down. Penetration: requests for nonexistent keys bypass the cache every time.

**Q12. How do you stop cache penetration?**
A: Cache null results briefly, use a Bloom filter of valid keys, and validate input.

**Q13. How do you handle a hot key?**
A: Add an in-process cache in front, replicate the key across several names (`key#1..N`), or shard the load.

**Q14. How do you keep cache and DB consistent?**
A: You can't make it perfectly consistent cheaply. Use delete-on-write plus TTL, or event-driven invalidation. For must-be-correct data, skip the cache.

**Q15. What if Redis goes down?**
A: The app should degrade gracefully by falling back to the DB, protected by rate limiting and a circuit breaker. Reduce risk with replicas, Sentinel or Cluster, and multi-AZ.

**Q16. How does Redis scale?**
A: Replicas for reads, Cluster sharding across 16384 hash slots for capacity and write throughput, consistent hashing in the client for other systems.

**Q17. Redis vs Memcached?**
A: Redis supports rich data types, persistence, and replication. Memcached is a simpler, multithreaded key-value cache.

**Q18. What should you not cache?**
A: Highly volatile or write-heavy data, data that must be exactly current (balances, final inventory), and rarely repeated unique requests.

**Q19. Local cache vs distributed cache?**
A: Local is fastest but each instance has its own copy and can differ. Distributed is shared and consistent across instances but costs a network hop.

**Q20. How do you decide what to cache?**
A: Measure. Look at read/write ratio, query cost, request repetition, and tolerance for staleness. Cache the expensive, popular, slowly changing data first.

**Q21. How would you cache a product page at Amazon scale?**
A: CDN for images and static assets, Redis for product details and prices with short TTL and delete on update, local cache for tiny hot config, and bypass cache for cart total and inventory at checkout.

---

## 12. Self-test

Answer out loud without notes.

1. Why does caching improve performance, and which data is a good candidate?
2. Define hit, miss, and hit ratio. Compute average latency for a 90% hit ratio.
3. Walk through cache-aside read and write paths.
4. Why delete instead of update on write?
5. How do you choose a TTL and why add jitter?
6. LRU vs LFU: when would you choose each?
7. Explain stampede, avalanche, and penetration, with one fix each.
8. Where would you place caches for a product detail page?
9. How do you handle stale data for price versus for a blog post?
10. What happens to your system if the cache disappears?

---

## 13. 30-second summary

> **Problem:** repeating slow work (DB queries, API calls) wastes latency and capacity.
> **How:** keep hot results in fast memory. Check the cache first (cache-aside), fall back to the DB on a miss, store with a TTL.
> **Scale:** high hit ratio, sharded or replicated Redis, multi-level caches, eviction (LRU/LFU) to bound memory.
> **Trade-offs:** freshness vs speed, extra complexity, stampede and avalanche risks, invalidation is hard.
> **Use:** CDN for static content, Redis for shared hot data, local cache for tiny hot data, never as the source of truth.
