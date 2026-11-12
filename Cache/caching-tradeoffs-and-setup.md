# Caching: Trade-offs and How to Set It Up

---

# Part 1: Trade-offs of Caching

## The big trade-off: what you gain vs what it costs

| You gain | You pay |
|---|---|
| Much lower latency on repeated reads | Risk of stale or inconsistent data |
| Less load on the database and downstream APIs | An extra system to run, monitor, and secure |
| Higher throughput and lower cost per request | Memory cost, and extra complexity in code |
| Resilience to traffic spikes (absorbs reads) | New failure modes: stampede, avalanche, cold start |

**Interview one-liner:** *"A cache trades freshness and simplicity for speed and capacity, so I decide per data type how much staleness is acceptable."*

## Trade-offs by decision

| Decision | Upside | Downside |
|---|---|---|
| **Cache-aside** | Simple, only caches what is requested, survives cache failure | First read after miss is slow, app owns the logic, stale window |
| **Read-through** | Simpler app code | Cache layer is tightly coupled to the data source |
| **Write-through** | Cache always fresh | Slower writes, caches data that may never be read |
| **Write-behind** | Very fast writes, batches DB load | Data loss risk if cache dies before flush, harder to reason about |
| **Write-around** | Doesn't pollute cache with write-once data | First read after write always misses |
| **Short TTL** | Fresher data | More misses, more DB load |
| **Long TTL** | High hit ratio, cheap | Staler data |
| **TTL jitter** | Avoids synchronized expiry | Slightly less predictable freshness |
| **Delete on write** | Removes stale data quickly | Write path must know every key to delete; small race window |
| **Update cache on write** | Next read is a hit | Race conditions can leave old values |
| **LRU** | Good general default | One big scan can evict hot keys |
| **LFU** | Keeps consistently popular keys | Needs counters, slow to forget formerly hot keys |
| **Local (in-process) cache** | Fastest, no network | Copies differ per instance, harder to invalidate, uses app memory |
| **Distributed cache (Redis)** | Shared, consistent across instances | Network hop, extra system, can be a bottleneck or SPOF |
| **CDN / edge cache** | Closest to users, offloads origin | Hard to invalidate quickly, not for private data |
| **Multi-level cache** | Very high hit rate, resilience | Harder invalidation, more places for staleness |
| **Caching nulls** | Stops penetration | Briefly hides newly created records (keep TTL short) |
| **Bloom filter** | Cheap guard against nonexistent keys | False positives, extra structure to maintain |
| **Lock / single-flight on miss** | Prevents stampede | Waiting requests add latency; lock needs a timeout |
| **Serve stale while revalidating** | Smooth latency, no stampede | Users briefly see old data |
| **Persistence (RDB/AOF) on Redis** | Warm restart | Extra I/O, more complexity; pure caches often skip it |
| **Replication / Cluster** | Availability and scale | Replication lag, operational complexity |

## Stale data trade-off by data type

| Data | Staleness tolerated? | Approach |
|---|---|---|
| Blog post, static content | Hours | Long TTL, CDN |
| Product description | Minutes | TTL + delete on update |
| Product price | Seconds | Short TTL + invalidate; re-check at checkout |
| Inventory at checkout, account balance | None | Don't cache, read the source of truth |

## How to say it in an interview

> "Caching is a trade between freshness and speed. I'd use cache-aside with Redis, a TTL sized to the business tolerance plus jitter, and delete-on-write for quick invalidation. I'd protect against stampede with single-flight and against penetration by caching nulls. For anything that must be exact, like price at checkout, I'd read from the database. And the system must still work if the cache is down."

---

# Part 2: How to Set Up Caching

For learning and interview demos, start with Option 1 (Redis in Docker). For a Java backend, Option 2 shows both annotation-based and manual cache-aside. Option 3 covers a local cache, and Option 4 covers AWS ElastiCache for production.

---

## Option 1: Redis in Docker (local)

**1. Start Redis with a memory limit and an eviction policy:**

```bash
docker run -d --name redis -p 6379:6379 redis:7 \
  redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
```

**2. Try the basics in `redis-cli`:**

```bash
docker exec -it redis redis-cli

SET product:42 "{\"name\":\"Mug\",\"price\":12}" EX 60
GET product:42
TTL product:42
DEL product:42
SET lock:42 1 NX EX 10      # only sets if it doesn't exist
INCR page:views
```

**3. Watch your hit ratio:**

```bash
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
# hit ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

**4. See what is happening live:** `redis-cli MONITOR` (development only, it is heavy).

**Docker Compose version:**

```yaml
services:
  redis:
    image: redis:7
    command: ["redis-server", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
    ports: ["6379:6379"]
```

---

## Option 2: Spring Boot + Redis (Java)

**Dependencies (Maven):**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

**application.yml:**

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
```

### 2a. Annotation-based caching

**Config with per-cache TTL:**

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory cf) {
        RedisCacheConfiguration base = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(cf)
            .cacheDefaults(base)
            .withCacheConfiguration("products", base.entryTtl(Duration.ofMinutes(30)))
            .withCacheConfiguration("prices",   base.entryTtl(Duration.ofSeconds(30)))
            .build();
    }
}
```

**Using it:**

```java
@Service
public class ProductService {

    private final ProductRepository repo;
    public ProductService(ProductRepository repo) { this.repo = repo; }

    @Cacheable(cacheNames = "products", key = "#id")        // read: cache-aside
    public Product getProduct(long id) {
        return repo.findById(id).orElseThrow();
    }

    @CachePut(cacheNames = "products", key = "#p.id")       // write-through style
    public Product update(Product p) {
        return repo.save(p);
    }

    @CacheEvict(cacheNames = "products", key = "#id")       // delete on write / delete
    public void delete(long id) {
        repo.deleteById(id);
    }
}
```

Typical choice: `@Cacheable` for reads and `@CacheEvict` on updates (delete-on-write). Note that calls within the same class bypass the proxy, so annotations won't fire on self-invocation.

### 2b. Manual cache-aside (shows you understand the pattern)

```java
@Service
public class ProductCache {

    private final StringRedisTemplate redis;
    private final ProductRepository repo;
    private final ObjectMapper mapper;

    public ProductCache(StringRedisTemplate redis, ProductRepository repo, ObjectMapper mapper) {
        this.redis = redis; this.repo = repo; this.mapper = mapper;
    }

    public Product get(long id) throws JsonProcessingException {
        String key = "product:" + id;

        String cached = redis.opsForValue().get(key);          // 1. check cache
        if (cached != null) {                                  // hit
            return "NULL".equals(cached) ? null : mapper.readValue(cached, Product.class);
        }

        Product p = repo.findById(id).orElse(null);            // 2. miss: read DB
        if (p == null) {                                       // 3a. cache the "not found" briefly
            redis.opsForValue().set(key, "NULL", Duration.ofSeconds(30));
            return null;
        }
        long ttl = 600 + ThreadLocalRandom.current().nextInt(60);   // 3b. TTL + jitter
        redis.opsForValue().set(key, mapper.writeValueAsString(p), Duration.ofSeconds(ttl));
        return p;                                              // 4. return
    }

    public void update(Product p) {
        repo.save(p);                                          // write DB first
        redis.delete("product:" + p.getId());                  // then invalidate
    }
}
```

### 2c. Stampede protection (simple lock)

```java
String lockKey = "lock:product:" + id;
Boolean got = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
if (Boolean.TRUE.equals(got)) {
    try {
        // rebuild from DB and set the cache
    } finally {
        redis.delete(lockKey);
    }
} else {
    // someone else is rebuilding: wait briefly and re-read the cache, or serve stale
}
```

For production, consider a library (Redisson locks) or in-process single-flight so one instance doesn't send many rebuilds.

---

## Option 3: Local in-process cache (Caffeine)

Great as a small first level in front of Redis, for tiny and hot data.

```xml
<dependency>
  <groupId>com.github.ben-manes.caffeine</groupId>
  <artifactId>caffeine</artifactId>
</dependency>
```

```java
Cache<Long, Product> local = Caffeine.newBuilder()
    .maximumSize(10_000)                    // bounded memory, evicts (W-TinyLFU)
    .expireAfterWrite(Duration.ofSeconds(30))
    .recordStats()                          // hit/miss metrics
    .build();

Product p = local.get(id, k -> loadFromRedisOrDb(k));   // atomic load, avoids local stampede
```

Keep the TTL short, since each instance has its own copy and they can briefly disagree.

---

## Option 4: AWS ElastiCache (production Redis)

1. **Create a cache:** ElastiCache for Redis (or Valkey) with a replication group, **Multi-AZ** and automatic failover enabled. Use cluster mode if you need to shard for capacity.
2. **Place it in private subnets** of your VPC (never public).
3. **Security group:** allow port 6379 only from your application's security group.
4. **Enable encryption** in transit (TLS) and at rest, and AUTH or IAM auth.
5. **Set a parameter group:** `maxmemory-policy allkeys-lru`, and set memory alarms.
6. **Connect your app** to the primary (or configuration) endpoint, with timeouts and a retry policy.
7. **Monitor in CloudWatch:** cache hit rate, evictions, CPU, memory, replication lag, connections.
8. **Load test and failover test** (reboot the primary and confirm the app degrades gracefully).

For a CDN layer, put **CloudFront** in front of your origin and set `Cache-Control` headers.

---

## HTTP / CDN caching basics

```http
Cache-Control: public, max-age=3600          # cache for 1 hour (browser + CDN)
Cache-Control: private, max-age=60           # only the user's browser
Cache-Control: no-store                      # never cache (sensitive data)
ETag: "v7"                                   # validator: client sends If-None-Match
```

On the next request, the server can answer `304 Not Modified` if the ETag still matches, saving bandwidth.

---

## Setup checklist (any option)

- [ ] Decided **what to cache** (read-heavy, expensive, tolerates staleness).
- [ ] Chose the **pattern** (cache-aside by default).
- [ ] Set a **TTL** per data type, **with jitter**.
- [ ] **Delete the key on write**, with TTL as a safety net.
- [ ] Set **maxmemory** and an **eviction policy** (`allkeys-lru` or `allkeys-lfu`).
- [ ] Protect against **stampede** (lock or single-flight) and **penetration** (null caching or Bloom filter).
- [ ] The app **works without the cache** (fallback to the DB, timeouts, circuit breaker).
- [ ] **Keys are namespaced** and versioned (`product:42`, `v2:product:42`).
- [ ] **Redis is private**, authenticated, and encrypted.
- [ ] **Monitor** hit ratio, evictions, latency, and memory.
- [ ] **Test**: flush the cache under load and confirm the DB survives.

---

## Quick test commands

```bash
# 1. Cold cache: first call is slow (miss), second is fast (hit)
redis-cli FLUSHALL
time curl -s localhost:8080/products/42 > /dev/null
time curl -s localhost:8080/products/42 > /dev/null

# 2. Inspect the key and its TTL
redis-cli GET product:42
redis-cli TTL product:42

# 3. Invalidate after an update, then confirm the next read refills it
curl -X PUT localhost:8080/products/42 -d '{"price":15}' -H 'Content-Type: application/json'
redis-cli EXISTS product:42        # 0 = deleted

# 4. Hit ratio
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# 5. Load test (watch DB load stay low)
ab -n 5000 -c 50 http://localhost:8080/products/42
```

> `FLUSHALL` wipes everything, so use it only on a local or test instance.
