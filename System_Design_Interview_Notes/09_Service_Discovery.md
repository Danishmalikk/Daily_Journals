# System Design Notes — Service Discovery (Eureka, Consul, Client-side vs Server-side, Health Checks, K8s DNS)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem — "where is the payment service right now?"

*Definition:* **Service discovery** is how one service finds the **current network location** (IP + port) of another service's healthy instances, automatically, without hard-coding addresses.

Real life: Calling a friend by **name** in your contacts instead of memorizing their number — if they change SIMs, the contact updates; you still just tap "Danish".

```
Monolith days:                       Microservices + containers + autoscaling:
PAYMENT_URL=http://10.0.1.15:3000    payment-service instances
   (one box, never moves)              10.0.3.7:3000   ← started 2 min ago
                                       10.0.4.21:3000  ← autoscaled
                                       10.0.1.15:3000  ← dying 💀
                                    IPs change on every deploy/scale/crash
```

Why hard-coding fails:
- Docker/ECS/K8s assign **new IPs** on every restart.
- **Autoscaling** adds and removes instances all the time.
- Dead instances must stop receiving traffic **within seconds**.

**Interview one-liner:** Service discovery lets services find each other by logical name instead of fixed IPs, because in containerized, autoscaled systems instance addresses change constantly.

---

## 2. Service Registry — the phone book

*Definition:* A **service registry** is a database of available service instances: `name → [ip:port, health, metadata]`. Instances **register** on startup, send **heartbeats** (or get health-checked), and **deregister** on shutdown.

```
           register / heartbeat
 order-svc  ──────────────────►  ┌────────────────────────────┐
 payment-svc ─────────────────►  │      Service Registry      │
 shipping-svc ────────────────►  │ payment-service:           │
                                 │   10.0.3.7:3000  UP        │
                    lookup       │   10.0.4.21:3000 UP        │
 order-svc  ◄──────────────────  │ shipping-service:          │
                                 │   10.0.5.9:8080  UP        │
                                 └────────────────────────────┘
```

| Registration style | How | Example |
|--------------------|-----|---------|
| **Self-registration** | The service registers itself (library in the app) | Spring Boot + Eureka client |
| **Third-party registration** | A separate agent/platform registers it | Consul agent, Kubernetes, AWS ECS + Cloud Map, Registrator |

> Third-party registration keeps app code clean (no discovery library), which is why platforms like Kubernetes do it for you.

**Interview one-liner:** A service registry maps service names to live instances; instances register via self-registration (library) or third-party registration (platform/agent) and are kept fresh by heartbeats or health checks.

---

## 3. Client-side discovery

*Definition:* In **client-side discovery**, the **calling service** queries the registry, gets the list of instances, and **picks one itself** using a client-side load balancer.

```
 order-service
 ┌───────────────────────────────┐     1. "payment-service?"   ┌──────────┐
 │ app code                      │ ──────────────────────────► │  Eureka  │
 │ + Spring Cloud LoadBalancer   │ ◄────────────────────────── │ registry │
 │   (cached list, round robin)  │     2. [10.0.3.7, 10.0.4.21] └──────────┘
 └──────────────┬────────────────┘
                │ 3. call chosen instance directly
                ▼
          10.0.4.21:3000
```

Spring Boot + Eureka:

```java
@SpringBootApplication
@EnableEurekaServer                         // the registry app itself
public class DiscoveryServer { ... }
```

```yaml
# order-service application.yml (spring-cloud-starter-netflix-eureka-client)
spring:
  application:
    name: order-service                     # the name others look up
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
  instance:
    prefer-ip-address: true
```

```java
@Bean
@LoadBalanced                               // resolves logical names via registry
RestClient.Builder restClientBuilder() { return RestClient.builder(); }

// "payment-service" is a NAME, not a host — LoadBalancer swaps in a real ip:port
restClientBuilder.build().post().uri("http://payment-service/payments").body(req).retrieve();
```

> ⚠️ **Ribbon** (old Netflix client LB) is in maintenance — **Spring Cloud LoadBalancer** is its replacement. `@FeignClient(name = "payment-service")` uses it too.

**Pros:** no extra network hop, smart per-client LB (zone-aware, weighted). **Cons:** discovery logic in every service and every language (Java has it; your NestJS service needs its own client).

**Interview one-liner:** In client-side discovery the caller fetches instances from the registry (e.g. Eureka) and load-balances itself (Spring Cloud LoadBalancer) — no extra hop, but every client needs discovery logic.

---

## 4. Server-side discovery

*Definition:* In **server-side discovery**, the caller sends the request to a **fixed router/load balancer**, which queries the registry and forwards to a healthy instance. The client knows nothing about instances.

```
 order-service ──► http://payment-service ──► ┌────────────────────┐ ──► 10.0.3.7
   (dumb client)       (one stable name)      │ Load balancer /    │ ──► 10.0.4.21
                                              │ router             │
                                              │ (asks registry)    │
                                              └────────────────────┘
       AWS ALB + target groups │ K8s Service (kube-proxy) │ Nginx/Envoy + Consul
```

| | Client-side | Server-side |
|---|-------------|-------------|
| Who picks the instance | Calling service | Load balancer / router |
| Extra hop | No | Yes (LB) |
| Language-agnostic | No (library per language) | **Yes** |
| LB is a bottleneck/SPOF | No | Must be HA (managed LBs are) |
| Examples | Eureka + Spring Cloud LoadBalancer, Consul + client | AWS ALB/NLB, K8s Service, Nginx, Envoy/Istio |

Memory trick: **Client-side = "I'll pick myself"**, **Server-side = "Let the receptionist route me."**

**Interview one-liner:** Server-side discovery puts a load balancer in front that does the lookup and routing — language-agnostic and simple for clients, at the cost of an extra hop that must be highly available.

---

## 5. Health checks & heartbeats — keeping the registry honest

*Definition:* A **health check** verifies an instance can actually serve traffic; unhealthy instances are removed from the registry/LB so nobody routes to them.

| Mechanism | How | Example |
|-----------|-----|---------|
| **Heartbeat (push)** | Instance says "I'm alive" every N seconds | Eureka: renew every **30s**, evicted after **90s** without renewal |
| **Active check (pull)** | Registry/agent calls the instance | Consul HTTP/TCP/gRPC checks, ALB target health, K8s probes |
| **TTL check** | App must update a TTL before it expires | Consul TTL check |

```json
// Consul service definition with an HTTP health check (NestJS payment-service)
{
  "service": {
    "name": "payment-service",
    "port": 3000,
    "tags": ["v2"],
    "check": { "http": "http://localhost:3000/health", "interval": "10s", "timeout": "2s" }
  }
}
```

Liveness vs readiness (what a `/health` should mean):
- **Liveness** = process isn't stuck → restart it if failing.
- **Readiness** = can serve now (DB pool up, warmed cache) → remove from traffic if failing, **don't** restart.
- In NestJS use `@nestjs/terminus`; in Spring Boot, Actuator `/actuator/health/liveness` and `/readiness`.

> ⚠️ **Eureka self-preservation:** if too many heartbeats are lost at once (renewals drop below ~85% of expected), Eureka assumes a **network problem** and stops evicting instances — you'll see the red "EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP" banner. That's by design (AP choice), so clients still need **retries + circuit breakers**.

> ⚠️ Don't make liveness depend on the DB: a DB blip would restart every pod at once. Put dependencies in **readiness**.

**Interview one-liner:** Registries stay accurate through heartbeats (Eureka: 30s renew, 90s evict) or active health checks (Consul, ALB, K8s probes); readiness removes an instance from traffic, liveness restarts it.

---

## 6. Eureka vs Consul vs others

| Feature | **Eureka** (Netflix) | **Consul** (HashiCorp) | **etcd / ZooKeeper** | **K8s built-in** |
|---------|----------------------|------------------------|----------------------|------------------|
| CAP leaning | **AP** (availability, peer replication) | **CP** (Raft consensus) | CP | CP (etcd underneath) |
| Health checks | Client heartbeats | Rich agent checks (HTTP/TCP/gRPC/TTL/script) | Session/lease based | Liveness/readiness probes |
| Interface | REST + Java client | HTTP API + **DNS** (port 8600) | Client libs | DNS + Endpoints API |
| Extras | — | KV config store, multi-datacenter, service mesh (Consul Connect) | Coordination, leader election | Everything (Services, Ingress) |
| Best fit | Spring Cloud on VMs/EC2 | Polyglot, VMs + containers, multi-DC | Infra building block | Anything already on Kubernetes |

> AP vs CP in one line: during a network partition, Eureka keeps serving a possibly **stale** list (better than none for discovery); Consul refuses writes without a quorum to stay **consistent**. (See the CAP Theorem note.)

Other real options: **AWS Cloud Map** (ECS service discovery), **Nacos** (Alibaba), **Istio/Linkerd** (mesh handles discovery + mTLS + retries).

**Interview one-liner:** Eureka is an AP, Java-centric registry with heartbeats; Consul is a CP, polyglot registry with DNS, rich health checks and a KV store; on Kubernetes you usually need neither.

---

## 7. Kubernetes DNS-based discovery

*Definition:* In **Kubernetes**, a **Service** gives a set of pods (selected by labels) one stable name and virtual IP; **CoreDNS** resolves that name, and **kube-proxy** (iptables/IPVS) load-balances to ready pods. It's server-side discovery built in.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: shop
spec:
  selector:
    app: payment          # all pods with this label
  ports:
    - port: 80
      targetPort: 3000
```

```
order pod ──► http://payment-service.shop.svc.cluster.local
                 │ CoreDNS → ClusterIP 10.96.12.34
                 ▼ kube-proxy picks a READY pod
            payment-pod-a  payment-pod-b  (not-ready pods excluded via readiness probe)
```

- Same namespace? Just use `http://payment-service`.
- **Headless Service** (`clusterIP: None`) → DNS returns **pod IPs** directly (StatefulSets, Kafka, client-side LB, gRPC).
- docker-compose has the same idea in miniature: service names resolve on the compose network (`tejas_network` → `http://redis:6379`).

> ⚠️ **gRPC + ClusterIP gotcha:** gRPC keeps one long-lived HTTP/2 connection, so kube-proxy (L4) balances **connections, not requests** → one pod gets all the load. Use a headless service + client-side LB, or a mesh (Linkerd/Istio) for L7 balancing.

**Interview one-liner:** Kubernetes does discovery for you — a Service gives pods a stable DNS name via CoreDNS, kube-proxy routes to ready pods, and headless services expose pod IPs when you need client-side balancing.

---

## 8. When to use which + what interviewers probe

| Situation | Choice |
|-----------|--------|
| Running on Kubernetes | **K8s Services + DNS** (add a mesh for L7/mTLS) |
| AWS ECS / EC2 without K8s | **ALB target groups** or **AWS Cloud Map** |
| Spring Cloud stack on VMs | **Eureka** + Spring Cloud LoadBalancer |
| Polyglot (NestJS + Spring) on VMs, multi-DC | **Consul** (DNS works for any language) |
| 2–3 services in docker-compose | Compose DNS names — no registry needed |

Common interview questions:
1. *"Client-side vs server-side?"* → who picks the instance; trade an extra hop vs library-per-language.
2. *"What if the registry goes down?"* → clients cache the last known list (Eureka refreshes every 30s), run the registry as a cluster (3+ Consul servers, peer Eureka nodes).
3. *"How fast is a dead instance removed?"* → depends on heartbeat/check interval + eviction + client cache refresh — can be 30–90s+ with Eureka defaults, hence retries/circuit breakers.
4. *"Graceful shutdown?"* → deregister first, drain in-flight requests, then stop (K8s `preStop` + `terminationGracePeriodSeconds`; Spring `server.shutdown=graceful`; NestJS `enableShutdownHooks()`).
5. *"Discovery vs API gateway?"* → discovery is for **internal** service-to-service lookup; the gateway is the **external** entry point (and itself uses discovery).

**Easy rule of thumb:** On K8s, use K8s. Off K8s with Spring only, Eureka. Mixed languages or multi-DC, Consul.

**Interview one-liner:** Pick the platform's built-in discovery when you have one (K8s, ECS); use Eureka for Spring-only fleets and Consul for polyglot or multi-DC — and always pair discovery with health checks, client caching and retries.

---

## Quick Revision Cheat Sheet

- **Service discovery**: find instances by logical name, because container IPs change on deploy/scale/crash.
- **Registry**: `name → instances`; self-registration (Eureka client) vs third-party (K8s, Consul agent, ECS).
- **Client-side**: caller queries registry + load-balances (Eureka + Spring Cloud LoadBalancer, `@LoadBalanced`); no extra hop, library per language.
- **Server-side**: LB/router does lookup (ALB, K8s Service, Nginx/Envoy); language-agnostic, extra hop.
- **Health**: heartbeats (Eureka 30s renew / 90s evict, self-preservation) or active checks (Consul, probes); readiness ≠ liveness.
- **Eureka vs Consul**: AP + Java heartbeats vs CP (Raft) + DNS + rich checks + KV.
- **K8s**: Service + CoreDNS (`svc.ns.svc.cluster.local`) + kube-proxy; headless for pod IPs; gRPC needs L7 balancing.
- **Choose**: K8s → built-in; Spring on VMs → Eureka; polyglot/multi-DC → Consul; always add retries + graceful shutdown.
