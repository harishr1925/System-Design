# System Design: Load Balancing - Complete Study Guide

> Goal: build a reusable mental toolkit, not finish the most content.
> For every concept, be able to answer:
> 1. What problem does it solve?
> 2. How does it work?
> 3. How does it scale?
> 4. What are the trade-offs?
> 5. Where would you use it in a real system?

**Interview focus:** explain how a load balancer distributes traffic and what happens when one server fails.

**Free resource:** Cloudflare - "What is Load Balancing?"

---

## Table of Contents

1. [Why load balancing is needed](#1-why-load-balancing-is-needed)
2. [L4 vs L7](#2-l4-vs-l7)
3. [Load balancing algorithms](#3-load-balancing-algorithms)
   - Round Robin
   - Least Connections
   - Others (weighted, IP hash, consistent hashing, power of two choices)
4. [Health checks](#4-health-checks)
5. [Failover](#5-failover)
6. [Horizontal scaling and statelessness](#6-horizontal-scaling-and-statelessness)
7. [Session persistence (sticky sessions)](#7-session-persistence-sticky-sessions)
8. [Connection draining](#8-connection-draining)
9. [Load balancer bottlenecks](#9-load-balancer-bottlenecks)
10. [Types of load balancing by location](#10-types-of-load-balancing-by-location)
11. [Trade-offs summary](#11-trade-offs-summary)
12. [Real-world usage and AWS mapping](#12-real-world-usage-and-aws-mapping)
13. [Interview deep dive: "What happens when a server fails?"](#13-interview-deep-dive-what-happens-when-a-server-fails)
14. [Interview Q&A bank](#14-interview-qa-bank)
15. [Self-test](#15-self-test)
16. [How to structure your answer in an interview](#16-how-to-structure-your-answer-in-an-interview)
17. [30-second summary](#17-30-second-summary)

---

## 1. Why load balancing is needed

### What problem does it solve?

A single server has limits on CPU, memory, connections, and bandwidth, and it is a single point of failure (SPOF). A load balancer (LB) sits in front of a group of servers and solves four problems:

| Problem | How the LB helps |
|---|---|
| **Capacity** | Spreads traffic so no one server is overwhelmed |
| **Availability** | Stops sending traffic to dead servers, so users don't see failures |
| **Latency** | Routes to the least-busy or closest server |
| **Flexibility** | Add, remove, or deploy servers without clients noticing (one stable entry point) |

### Vertical vs horizontal scaling

- **Vertical scaling:** a bigger machine. It hits a hard ceiling, gets expensive, and still leaves a single point of failure.
- **Horizontal scaling:** more machines. This needs something to split traffic among them, which is the load balancer.

### How it works at a high level

```
Client -> [DNS] -> Load Balancer -> Server 1
                                 -> Server 2
                                 -> Server 3
```

On every request the LB:
1. **Receives** the request on a single virtual IP or DNS name.
2. **Picks** a healthy backend using an algorithm.
3. **Forwards** it, and returns the response (or lets the server respond directly).

**Interview one-liner:** *"A load balancer decouples clients from the number and health of backend servers, giving horizontal scalability and fault tolerance."*

---

## 2. L4 vs L7

| | **L4 (Transport)** | **L7 (Application)** |
|---|---|---|
| Decides using | IP, port, TCP/UDP | HTTP headers, URL, cookies, host, body |
| Speed | Very fast, low overhead | Slower (parses requests) |
| Routing | By IP/port only | By path (`/api`, `/images`), host, header, cookie |
| Extras | Little | TLS termination, auth, rate limiting, A/B routing, caching |
| AWS | **NLB** | **ALB** |
| Use for | Raw throughput, non-HTTP (gaming, DB, MQTT), static IPs | Microservices, web apps |

**Rule of thumb:** use L7 when routing depends on request content (`/api/orders` vs `/api/cart`). Use L4 for raw speed, non-HTTP protocols, or static IPs. Large systems often use both: L4 at the edge, L7 behind it.

**Trade-off:** L7 costs CPU and latency but gives smart routing. L4 is cheap and fast but cannot route by content.

---

## 3. Load balancing algorithms

| Algorithm | How it works | Best for | Weakness |
|---|---|---|---|
| **Round Robin** | Rotate through servers in order | Identical servers, similar requests | Ignores server load |
| **Weighted Round Robin** | More requests to stronger servers | Mixed hardware | Static weights |
| **Least Connections** | Send to server with fewest active connections | Long-lived or uneven requests (WebSockets) | Needs connection tracking |
| **Least Response Time** | Pick the fastest-responding server | Latency-sensitive apps | More monitoring overhead |
| **IP Hash** | `hash(client IP) % N` | Simple stickiness | Remapping when N changes |
| **Consistent Hashing** | Servers and keys on a hash ring; key goes to next server clockwise | Caches, sharded stores | More complex; needs virtual nodes |
| **Power of Two Choices** | Pick 2 random servers, send to the less loaded | Large-scale systems | Slightly approximate |
| **Random** | Pick randomly | Simple, stateless | Uneven in the short term |

### 3.1 Round Robin

**How:** requests go 1 -> 2 -> 3 -> 1 -> 2 -> 3 ...

**Good when:** servers are identical and requests cost about the same.

**Breaks when:** requests vary in cost or servers differ in power, because it ignores current load. A server stuck on slow requests still gets its turn.

**Variant:** weighted round robin gives a server with 2x capacity 2x the turns.

### 3.2 Least Connections

**How:** send each new request to the server with the fewest active connections.

**Good when:** requests have uneven duration (file uploads, WebSockets, slow queries).

**Example:** Server A has 2 long-running requests, Server B has 10 short ones.
- Round robin keeps feeding A and B equally.
- Least connections favors A, since it has fewer in-flight connections.

**Trade-off:** the LB must track connection counts. For huge fleets, **power of two choices** (pick 2 random servers, take the less loaded) gets nearly the same result with far less coordination.

### 3.3 Consistent hashing (key insight)

Plain `hash(key) % N` reshuffles almost every key when a server is added or removed. Consistent hashing moves only about `1/N` of the keys. This matters for caches, where a reshuffle means a cache-miss storm. Use **virtual nodes** to keep the distribution balanced.

---

## 4. Health checks

| Type | How it works |
|---|---|
| **Active** | LB calls `/health` every few seconds. After N consecutive failures the server is marked unhealthy; after M successes it returns to rotation |
| **Passive** | LB watches real traffic (5xx errors, timeouts) and ejects misbehaving servers |

**Shallow vs deep checks**
- *Shallow* ("process is up"): cheap, but can miss a broken dependency.
- *Deep* (verifies DB connectivity): more accurate, but if a shared dependency fails, every server fails its check at once and the LB has zero targets.
- **Guidance:** keep health checks mostly shallow.

**Tuning**
- Too aggressive: flapping and extra load.
- Too slow: users get routed to dead servers.

**Fail-open behavior:** if all targets are unhealthy, some LBs route to all of them rather than none, which is another reason shallow checks matter.

---

## 5. Failover

Failover happens at several levels.

### 5.1 Backend server fails
The LB detects the failure (active or passive checks) and stops routing to that server. See [section 13](#13-interview-deep-dive-what-happens-when-a-server-fails) for the full timeline.

### 5.2 Load balancer fails
- **Active-passive:** two LBs share a virtual IP. If the active one dies, the standby takes over via heartbeat (e.g. keepalived).
- **Active-active:** multiple LB nodes serve traffic at once across AZs, with DNS or anycast spreading load. Managed cloud LBs (ALB/NLB) do this for you.

### 5.3 Regional failure
DNS-level failover (e.g. Route 53 health checks) redirects traffic to another region. It is slower because of DNS TTL and caching.

---

## 6. Horizontal scaling and statelessness

For an LB to work well, servers should be **stateless**:
- Keep sessions in Redis, a DB, or a signed token (JWT), not in server memory.
- Then any server can handle any request, and you can add or remove instances freely.

**Auto scaling flow**
1. Metrics (CPU, request count) trigger the Auto Scaling Group.
2. New instances register with the LB.
3. They pass health checks and receive traffic.
4. On scale-in, **connection draining** lets in-flight requests finish before removal.

**Other scaling tools**
- **Cross-zone balancing:** spreads load evenly across AZs even if instance counts differ.
- **Anycast:** the same IP is advertised from many locations; the network routes users to the nearest.
- **Rate limiting and backpressure:** protect backends during spikes.

---

## 7. Session persistence (sticky sessions)

If a server holds user session state in memory, the same user must keep returning to it.

| Option | Pros | Cons |
|---|---|---|
| **Sticky sessions** (cookie or IP hash) | Simple | Uneven load, hot spots, session lost if server dies |
| **Stateless servers + external session store** (Redis, JWT) | Even balancing, easy scaling | Extra dependency and network hop |

**Preferred:** stateless servers with an external session store.

---

## 8. Connection draining

When removing or deploying a server, the LB stops sending *new* requests but lets in-flight requests finish (AWS calls this *deregistration delay*). This is essential for zero-downtime deploys and safe scale-in.

---

## 9. Load balancer bottlenecks

The LB can become the problem it was meant to solve.

| Bottleneck | Mitigation |
|---|---|
| Single LB = SPOF | Redundant LBs across AZs |
| CPU limits (TLS handshakes, L7 parsing) | Offload TLS, scale LB nodes out, put L4 in front of L7 |
| Bandwidth / connection limits | Multiple LB nodes, DNS or anycast spreading |
| Extra network hop | Client-side LB or service mesh for internal calls |
| Shared state (sticky tables, connection tracking) | Prefer stateless algorithms |

### Multi-tier load balancing (how big systems do it)

```
Users
  | (1) DNS / Global LB (Route 53, geo/latency routing)
Region
  | (2) L4 LB (NLB): fast, handles millions of connections
  | (3) L7 LB (ALB / Envoy / Nginx): path-based routing
Service A / B / C instances
  | (4) Client-side or service-mesh LB for internal service-to-service calls
```

---

## 10. Types of load balancing by location

| Type | Description | Downside |
|---|---|---|
| **DNS-based / global (GSLB)** | Returns different IPs by geography, latency, or health | Slow failover due to DNS TTL and caching |
| **Hardware / software LB** | F5, Nginx, HAProxy, Envoy | Operational overhead (if self-managed) |
| **Client-side** | Client holds the server list and picks one (gRPC, Netflix Ribbon, Spring Cloud LoadBalancer) | Logic in every client; harder to update |
| **Service mesh (sidecar)** | Envoy proxies next to each service handle LB, retries, mTLS | Added complexity and resource overhead |

---

## 11. Trade-offs summary

| Decision | Upside | Downside |
|---|---|---|
| **L4 vs L7** | L4: speed. L7: smart routing | L7 costs CPU/latency; L4 can't route by content |
| **Sticky sessions** | Simple state handling | Hot spots, lost sessions on failure, hard to scale |
| **Stateless + external session store** | Even balancing, easy scaling | Extra dependency (Redis) and a network hop |
| **Deep health checks** | Catch real failures | Cascading or false removal |
| **Aggressive health checks** | Fast failure detection | Flapping, extra load |
| **Round robin vs least connections** | RR: simple, no state. LC: adapts to load | LC needs tracking; RR can overload slow servers |
| **TLS termination at LB** | Offloads CPU, central cert management | Plaintext inside the network (re-encrypt if compliance requires) |
| **Single LB layer** | Simple | SPOF and bottleneck |
| **Client-side LB** | No extra hop, no central bottleneck | Logic in every client/language |
| **DNS LB** | Cheap, global | Slow failover (TTL) |
| **Consistent hashing** | Minimal reshuffling | Possible imbalance; needs virtual nodes |

### Common failure modes
- **Thundering herd / retry storms:** when a server fails, retries from all clients overload survivors. Fix: exponential backoff with jitter, circuit breakers.
- **Cascading failure:** a few slow servers cause queue buildup everywhere. Fix: timeouts, load shedding, bulkheads.
- **Cold start:** a new server is hammered immediately. Fix: *slow start* (ramp traffic gradually).

---

## 12. Real-world usage and AWS mapping

### Example: e-commerce platform (Amazon-style)

```
User -> Route 53 (latency-based routing to nearest region)
     -> CloudFront (CDN for static content)
     -> ALB (path routing)
           /api/products -> Product Service (ASG)
           /api/cart     -> Cart Service (ASG)
           /api/orders   -> Order Service (ASG)
     -> Internal NLB/ALB or service mesh between services
     -> DB read replicas (reads load-balanced across replicas)
```

### Where load balancing appears in system design problems

| Area | How |
|---|---|
| **Web/API tier** | LB in front of stateless app servers |
| **Microservices** | Internal LBs or service mesh between services |
| **Databases** | Read traffic balanced across replicas (writes go to primary) |
| **Caches** | Consistent hashing across cache nodes (Memcached, Redis cluster) |
| **Queues and consumers** | Partition assignment is a form of load balancing |
| **WebSockets / chat / gaming** | L4 plus least connections (long-lived connections) |
| **Video streaming** | CDN plus geo-DNS routing |
| **URL shortener, feed, ride-hailing** | LB is the first box after the client |

### AWS services

| Service | Role |
|---|---|
| **Route 53** | DNS-based global routing (latency, geo, weighted, failover) |
| **ALB** | L7: HTTP/HTTPS, path/host routing, WebSockets, gRPC |
| **NLB** | L4: millions of requests/sec, ultra-low latency, static IPs |
| **GWLB** | Routes traffic through firewalls and inspection appliances |
| **Global Accelerator** | Anycast IPs, routes users over the AWS backbone |
| **Auto Scaling Group** | Adds/removes targets behind the LB |

---

## 13. Interview deep dive: "What happens when a server fails?"

Walk through it as a timeline:

1. **Normal:** the LB sends requests across servers A, B, C using least connections.
2. **Server B crashes** (or hangs).
3. **Detection**
   - *Passive:* in-flight requests to B time out or error; the LB notices.
   - *Active:* B fails N consecutive health checks (e.g. 3 checks at 5s intervals = up to ~15s).
4. **Ejection:** B is marked unhealthy; new requests go only to A and C.
5. **Impact on users**
   - Requests in flight to B may fail. The LB or client **retries** on another server, which is only safe if the request is **idempotent**.
   - Sessions stored on B are lost, unless sessions are external (Redis).
6. **Survivor protection:** A and C now take about 50% more load. Guard with timeouts, circuit breakers, retry backoff with jitter, and load shedding. Without these, a failure can cascade.
7. **Recovery:** auto scaling launches a replacement. Once it passes health checks, the LB adds it with **slow start** so it isn't hit with a full share instantly.
8. **B comes back:** it must pass M consecutive successful checks before rejoining.

**Strong closing line:** *"The goal is that a single server failure is invisible to users, apart from a small window of retried requests, thanks to redundancy, fast detection, stateless servers, and safe retries."*

---

## 14. Interview Q&A bank

### Fundamentals

**Q1. What is a load balancer and why do we need one?**
A: It distributes incoming requests across multiple backend servers. We need it for capacity (no single server overwhelmed), availability (avoid dead servers), and flexibility (add/remove servers transparently). It enables horizontal scaling and removes the server as a single point of failure.

**Q2. Why can't vertical scaling replace load balancing?**
A: Vertical scaling has a hard hardware ceiling, grows expensive nonlinearly, and still leaves one machine as a SPOF. Horizontal scaling gives near-linear capacity growth and fault tolerance, but needs an LB to spread traffic.

**Q3. What is the difference between L4 and L7 load balancing?**
A: L4 routes on IP and port (TCP/UDP) without looking at content: fast, cheap, protocol-agnostic. L7 inspects HTTP data (path, host, headers, cookies) and can do content-based routing, TLS termination, auth, and rate limiting, at higher CPU and latency cost.

**Q4. When would you choose L4 over L7?**
A: For maximum throughput and minimal latency, non-HTTP protocols (gaming, MQTT, databases), static IP requirements, or as a fast front tier ahead of L7 LBs.

**Q5. Where does TLS terminate?**
A: Usually at the LB, to offload CPU and centralize certificate management. Re-encrypt to backends if compliance requires end-to-end encryption.

### Algorithms

**Q6. Explain round robin and when it fails.**
A: It cycles through servers in order. It works for identical servers with similar request costs. It fails when requests vary in cost or servers differ in capacity, since it ignores current load.

**Q7. Give a case where round robin fails but least connections works.**
A: A mix of long uploads and short API calls. Round robin may stack several long uploads on one server. Least connections sees that server has many active connections and routes new requests elsewhere.

**Q8. What is weighted round robin?**
A: Round robin where stronger servers get proportionally more requests (e.g. weight 2 gets twice the turns). Weights are static, so it doesn't adapt to live load.

**Q9. Round robin vs consistent hashing?**
A: Round robin suits stateless, interchangeable servers. Consistent hashing suits stateful or cached data where the same key should hit the same node, and it minimizes remapping when nodes change.

**Q10. Why is `hash(key) % N` a problem, and how does consistent hashing fix it?**
A: When N changes, nearly all keys remap, causing a cache-miss storm. Consistent hashing places servers and keys on a ring so only about `1/N` of keys move. Virtual nodes smooth out imbalance.

**Q11. What is power of two choices?**
A: Pick two random servers and send the request to the less loaded one. It nearly matches least connections while needing far less global state, so it scales well in large fleets.

### Health checks and failure

**Q12. How do health checks work?**
A: Active checks poll an endpoint (e.g. `/health`) at intervals; N failures mark a server unhealthy and M successes restore it. Passive checks observe real traffic errors and timeouts and eject bad servers.

**Q13. Why can a deep health check cause an outage?**
A: If `/health` checks a shared dependency like the DB and that dependency goes down, every server fails its check, and the LB removes them all, turning a partial degradation into a full outage. Keep checks shallow, and handle dependency failures with degradation or circuit breakers.

**Q14. What if all health checks fail?**
A: Some LBs fail open, routing to all targets rather than none. That is why checks should be shallow and why you should alert on fleet-wide failures.

**Q15. How do you handle a server that is slow but not dead?**
A: Least-response-time or least-connections routing, passive health checks, outlier ejection, request timeouts, and circuit breakers.

**Q16. What if the load balancer itself fails?**
A: Run it redundantly: active-passive with a virtual IP, or active-active across AZs with DNS/anycast. Managed cloud LBs handle this automatically.

**Q17. What happens to in-flight requests when a server dies?**
A: They fail or time out. The LB or client can retry on another server, but only safely if the operation is idempotent (or protected with idempotency keys).

### Scaling and state

**Q18. What are sticky sessions and why are they discouraged?**
A: They pin a client to one server (via cookie or IP hash) so in-memory session state works. Downsides: uneven load, hot spots, lost sessions when that server dies, and harder scaling. Prefer stateless servers with sessions in Redis or JWT.

**Q19. How do you do zero-downtime deployments?**
A: Connection draining, rolling or blue-green deployments, and health checks that gate traffic to new instances.

**Q20. How do you handle a traffic spike?**
A: Auto scaling, LB pre-warming, rate limiting, load shedding, queue-based buffering, and a CDN for static content.

**Q21. How does auto scaling work with an LB?**
A: Metrics trigger the ASG, new instances register with the LB, pass health checks, and start receiving traffic. On scale-in, connection draining lets in-flight requests finish.

**Q22. How would you load balance WebSockets?**
A: Use L4, or an L7 LB with WebSocket support, with least connections since connections are long-lived. Plan for rebalancing and reconnection storms during deploys (stagger reconnects with jitter).

### Architecture and scale

**Q23. How do you stop the LB from being a bottleneck or SPOF?**
A: Multiple LB nodes across AZs, L4 in front of L7, TLS offload and scale-out, DNS or anycast to spread load, and client-side LB or service mesh for internal traffic.

**Q24. What is DNS-based load balancing and its drawback?**
A: DNS returns different IPs by geography, latency, or health. It is cheap and global, but failover is slow because of TTL and resolver caching.

**Q25. What is client-side load balancing?**
A: The client holds the server list (from service discovery) and picks a server itself (e.g. gRPC, Spring Cloud LoadBalancer). It removes a hop and a central bottleneck, but puts logic in every client.

**Q26. What is a service mesh and how does it relate to LB?**
A: Sidecar proxies (e.g. Envoy) next to each service handle load balancing, retries, timeouts, and mTLS for service-to-service traffic, without changing application code.

**Q27. What is slow start?**
A: A gradual ramp of traffic to a newly added or recovered server so it can warm up (caches, JIT, connection pools) instead of being hit with a full share immediately.

**Q28. How does load balancing apply to databases?**
A: Reads are balanced across replicas while writes go to the primary. Beware replication lag for read-your-writes consistency.

**Q29. Design question: how would you design a load balancer?**
A: Clarify requirements (L4 or L7, throughput, latency, availability). Core components: listener/frontend, routing and algorithm module, backend pool with health checker, connection management, config/control plane, and metrics. Scale with multiple stateless LB nodes behind anycast/ECMP or DNS, share backend state via a control plane, and add TLS termination, draining, slow start, and observability. Discuss trade-offs (L4 vs L7, algorithm choice, health-check design).

---

## 15. Self-test

Answer out loud without notes.

1. Why can't vertical scaling replace load balancing?
2. When would you choose L4 over L7?
3. Give a case where round robin fails but least connections works.
4. Why can a deep health check cause an outage?
5. Walk through what happens when 1 of 3 servers crashes mid-request.
6. How do you prevent the LB from being a SPOF or bottleneck?
7. Why does `hash % N` cause problems when scaling, and what fixes it?
8. Why are sticky sessions discouraged, and what is the alternative?
9. What is connection draining and when does it matter?
10. How would you route `/api/orders` and `/api/cart` to different services?

---

## 16. How to structure your answer in an interview

1. **State the need:** "We'll run multiple stateless app servers behind a load balancer for scale and availability."
2. **Choose the layer:** "L7 ALB since we need path-based routing; L4 if raw throughput or non-HTTP."
3. **Choose the algorithm:** "Least connections, since request durations vary."
4. **Address state:** "Servers are stateless, with sessions in Redis, so no sticky sessions."
5. **Address failure:** "Health checks, multi-AZ, and connection draining for deploys."
6. **Address LB resilience:** "Managed LB across AZs, so it is not a SPOF."
7. **Mention protection:** "Timeouts, retries with backoff and jitter, circuit breakers, and load shedding."

---

## 17. 30-second summary

> **Problem:** one server can't handle the load or survive failure.
> **How:** an LB (L4/L7) distributes requests across healthy servers using an algorithm, with health checks.
> **Scale:** stateless servers, multi-AZ, multi-tier LBs, DNS/geo routing, auto scaling.
> **Trade-offs:** L4 speed vs L7 intelligence, stickiness vs even load, health-check sensitivity, LB as SPOF.
> **Use:** the front door of every scalable system, plus internal services, DB replicas, and caches.
