# Load Balancing: Trade-offs and How to Set It Up

---

# Part 1: Trade-offs of Load Balancing

## The big trade-off: what you gain vs what it costs

| You gain | You pay |
|---|---|
| Capacity (spread load across servers) | Extra component to run, monitor, and pay for |
| Availability (skip dead servers) | The LB can itself become a single point of failure |
| Zero-downtime deploys and scaling | Added complexity in health checks, draining, and config |
| One stable entry point | One extra network hop, so slightly more latency |

**Interview one-liner:** *"A load balancer buys scalability and fault tolerance, and the price is an extra hop, extra complexity, and a new component that must itself be made highly available."*

## Trade-offs by concept

| Decision | Upside | Downside |
|---|---|---|
| **L4 vs L7** | L4: very fast and cheap. L7: smart routing, TLS termination, auth | L4 can't route by content. L7 uses more CPU and adds latency |
| **Round robin** | Simple, no state | Ignores load, so slow or uneven requests pile up |
| **Least connections** | Adapts to real load | Must track connection counts, which is harder to scale |
| **Power of two choices** | Near-optimal with little coordination | Approximate, not perfect |
| **Sticky sessions** | Easy way to keep in-memory session state | Uneven load, hot spots, session lost if the server dies |
| **Stateless servers + Redis** | Even balancing, easy scaling | Extra dependency and a network hop |
| **Aggressive health checks** | Fast failure detection | Flapping, extra load |
| **Slow health checks** | Stable, cheap | Users hit dead servers longer |
| **Deep health checks** | Catch real problems | One shared dependency failing can remove every server |
| **TLS termination at the LB** | Offloads CPU, central certificates | Traffic is plaintext inside (re-encrypt if compliance needs it) |
| **Consistent hashing** | Only about 1/N of keys move when servers change | Can be imbalanced, so it needs virtual nodes |
| **DNS-based LB** | Cheap and global | Slow failover because of TTL and caching |
| **Client-side LB** | No extra hop, no central bottleneck | Logic lives in every client and language |
| **Single LB layer** | Simple | Bottleneck and single point of failure |
| **Layered LBs (DNS, L4, L7)** | Scales to huge traffic | More moving parts to operate |

## Failure-related trade-offs

- **Retries** hide failures but can cause retry storms. Use exponential backoff with jitter and only retry idempotent requests.
- **Fail-open** (route to all servers when all checks fail) keeps you serving but may send traffic to broken servers.
- **Slow start** protects new servers but delays reaching full capacity.

## How to say it in an interview

> "Every choice here trades simplicity against intelligence or resilience. L4 is fast but blind, and L7 is smart but costlier. Sticky sessions are easy but fragile, and stateless servers are robust but need a session store. Aggressive health checks detect failures fast but risk flapping. I'd pick based on the requirement, usually L7 with least connections, stateless servers, shallow health checks, and redundant LBs across AZs."

---

# Part 2: How to Set Up Load Balancing

The setup depends on where you run things. For learning and interview demos, Option 1 is the fastest. For production on AWS, use Option 2. For Java microservices calling each other, use Option 3.

---

## Option 1: Nginx (local, VM, or Docker)

**1. Run two or more copies of your app** (for example on ports 8081 and 8082).

**2. Configure Nginx** (`/etc/nginx/nginx.conf` or a file in `conf.d/`):

```nginx
upstream backend {
    least_conn;                                   # algorithm (default is round robin)
    server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
    # server 127.0.0.1:8083 weight=2;             # weighted example
    # server 127.0.0.1:8084 backup;               # only used if others are down
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # failover: retry the next server on errors/timeouts
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_connect_timeout 2s;
        proxy_read_timeout 30s;
    }
}
```

**3. Test and reload:**

```bash
sudo nginx -t && sudo nginx -s reload
```

**4. Verify:** call `http://localhost/` repeatedly and confirm responses come from different instances. Stop one instance and confirm traffic keeps working.

**Notes:**
- `max_fails` and `fail_timeout` give you **passive** health checks. Active health checks (polling `/health`) need Nginx Plus, or use HAProxy or a cloud LB.
- For sticky sessions use `ip_hash;` instead of `least_conn;`, though stateless servers are preferred.

### Docker Compose version

```yaml
services:
  app1:
    image: my-app
    environment: [SERVER_ID=1]
  app2:
    image: my-app
    environment: [SERVER_ID=2]
  nginx:
    image: nginx:stable
    ports: ["80:80"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf:ro"]
    depends_on: [app1, app2]
```

In `nginx.conf`, use the service names (`server app1:8080; server app2:8080;`).

---

## Option 2: AWS Application Load Balancer (ALB)

1. **Launch instances** (EC2, or ECS/EKS tasks) in **at least two Availability Zones**, running your app on a port such as 8080.
2. **Create a target group:** type Instances (or IP), protocol HTTP, port 8080. Set the **health check path** (e.g. `/actuator/health`), interval, and healthy/unhealthy thresholds. Register your instances.
3. **Create the ALB:** internet-facing, select subnets in 2 or more AZs, and attach a security group allowing 80/443 from the internet.
4. **Add a listener:** HTTPS 443 with an ACM certificate (TLS terminates at the ALB), and optionally redirect HTTP 80 to 443. Forward to your target group. Add path rules (`/api/orders*` to an orders target group) if you have multiple services.
5. **Lock down the instances:** the instance security group should only allow traffic from the ALB's security group.
6. **Tune:** set deregistration delay (connection draining), enable cross-zone balancing, and choose the algorithm (round robin or least outstanding requests).
7. **Add auto scaling:** attach an Auto Scaling Group to the target group, scaling on CPU or request count per target.
8. **Point DNS at it:** a Route 53 alias record to the ALB.

Use an **NLB** instead if you need L4, static IPs, or non-HTTP protocols.

---

## Option 3: Client-side load balancing in Java (Spring Cloud LoadBalancer)

Useful when one service calls another and you want balancing without a separate LB hop.

**Dependency:**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**Load-balanced client:**
```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    public RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }
}

@Service
public class OrderClient {
    private final RestClient client;
    public OrderClient(RestClient.Builder builder) {
        this.client = builder.baseUrl("http://order-service").build(); // logical name
    }
    public String getOrder(String id) {
        return client.get().uri("/orders/{id}", id).retrieve().body(String.class);
    }
}
```

**Instances come from service discovery** (Eureka, Consul, Kubernetes), or you can list them statically:
```yaml
spring:
  cloud:
    discovery:
      client:
        simple:
          instances:
            order-service:
              - uri: http://localhost:8081
              - uri: http://localhost:8082
```

The default algorithm is round robin. Pair it with **Resilience4j** (timeouts, retries, circuit breaker) so failures don't cascade.

---

## Setup checklist (any option)

- [ ] Servers are **stateless** (sessions in Redis or JWT).
- [ ] **Health check** endpoint is shallow and fast.
- [ ] **Two or more** instances across **two or more AZs**.
- [ ] **Timeouts and retries** with backoff and jitter, and retries only for idempotent calls.
- [ ] **Connection draining** enabled for deploys.
- [ ] The **LB itself is redundant** (managed LB or active-passive pair).
- [ ] **TLS** terminates at the LB, with a certificate set up.
- [ ] **Test failure:** kill an instance during load and confirm users see no errors.

---

## Quick test commands

```bash
# See which server answers (each app returns its SERVER_ID)
for i in $(seq 1 10); do curl -s http://localhost/; echo; done

# Simulate a server failure, then re-run the loop above
docker stop <app1-container>

# Simple load test
ab -n 1000 -c 50 http://localhost/
```
