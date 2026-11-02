# Testing Notes — Integration Testing with Testcontainers (Postgres, Redis, Kafka, @ServiceConnection)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is an integration test?

*Definition:* An **integration test** checks that **several real pieces work together** — your code + a real database, cache, message broker or HTTP layer — instead of faking them.

Real life: Unit tests check each car part on a bench. An integration test puts the engine **in the car** and drives it around the block.

```
Unit test:         OrderService ──► mock(Repo)                  (fake, ms)
Integration test:  OrderService ──► OrderRepository ──► real Postgres  (real, seconds)
```

| | Unit test | Integration test |
|---|-----------|------------------|
| Dependencies | Mocked | Real (DB, Redis, Kafka) |
| Speed | Milliseconds | Seconds |
| Catches | Logic bugs | SQL errors, mapping, config, migrations, serialization |
| Runs | Every save | Every PR in CI |
| Tool | JUnit + Mockito | **Testcontainers** + `@SpringBootTest` / `@DataJpaTest` |

> Mocks only prove your code calls the repository. They can't prove your native Postgres query, `jsonb` column or Flyway migration actually works.

**Interview one-liner:** Integration tests verify your code against real infrastructure — they catch SQL, mapping, config and serialization bugs that mocks can never see.

---

## 2. Why not H2? (the in-memory DB trap)

*Definition:* **H2** is an in-memory Java database often used in tests. It's fast but it is **not Postgres or MySQL** — different SQL dialect, types and behaviour.

| Feature | H2 (compat mode) | Real Postgres (Testcontainers) |
|---------|------------------|--------------------------------|
| Startup | Instant | ~2-5s (first pull slower) |
| `jsonb`, arrays, `ON CONFLICT` | Partial / missing | ✅ Real |
| Window functions, CTEs, extensions | Partial | ✅ Real |
| Locking / isolation behaviour | Different | ✅ Same as prod |
| Flyway/Liquibase migrations | May fail or need hacks | ✅ Same scripts as prod |
| Confidence | "Works on H2" | "Works on Postgres 16" |

```sql
-- Postgres upsert: works in prod, but H2 doesn't support ON CONFLICT → test fails or gets rewritten
INSERT INTO stock(sku, qty) VALUES ('SKU-1', 5)
ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;
```

> ⚠️ The classic trap: tests green on H2, production breaks on a Postgres-specific query. "Green on H2" ≠ "works in prod".

**Easy rule of thumb:** Test against **the same database engine and major version** you run in production.

**Interview one-liner:** H2 is fast but behaves differently from Postgres/MySQL, so tests can pass on H2 and fail in prod — Testcontainers runs the real engine instead.

---

## 3. What is Testcontainers?

*Definition:* **Testcontainers** is a library that starts **real, throwaway Docker containers** (Postgres, Redis, Kafka, LocalStack, ...) from your test code, gives you their connection details, and destroys them after the tests.

Real life: A disposable test kitchen — spun up fresh for each chef's exam, thrown away after, no leftovers.

```
 JUnit test starts
       │
       ▼
 Testcontainers ──► Docker daemon ──► postgres:16-alpine  (random host port, e.g. 54012)
       │                           ──► redis:7-alpine
       │                           ──► apache/kafka
       ▼
 Spring gets JDBC URL / host:port ──► tests run against REAL services
       │
       ▼
 Tests done ──► Ryuk sidecar removes all containers (even if JVM crashes)
```

Key facts:
- Needs Docker (Docker Desktop, Colima, Rancher, or Testcontainers Cloud) on the dev machine and CI runner.
- Containers get **random host ports** → no clashes with your local `docker-compose` Postgres on 5432.
- Ready-made modules: `postgresql`, `mysql`, `kafka`, `localstack`, `mongodb`, `elasticsearch`, `rabbitmq`; anything else via `GenericContainer`.
- Exists for Node too — `testcontainers` npm package works with Jest in NestJS.

**Interview one-liner:** Testcontainers spins up real disposable Docker containers for your tests on random ports and cleans them up automatically via the Ryuk reaper.

---

## 4. @Testcontainers + @Container + @DynamicPropertySource (classic way)

*Definition:* **`@Testcontainers`** is a JUnit 5 extension that manages the lifecycle of fields annotated **`@Container`**. **`@DynamicPropertySource`** injects the container's runtime URL/credentials into Spring properties.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository repo;

    @Test
    void savesAndFindsOrder() {
        Order saved = repo.save(new Order("user-7", 1500));
        assertThat(repo.findById(saved.getId())).isPresent();
    }
}
```

| Field type | Lifecycle |
|------------|-----------|
| `static @Container` | Started **once per test class**, shared by all its tests |
| instance `@Container` | Started/stopped **for every test method** (slow!) |

> ⚠️ Always make containers `static` unless you truly need a fresh one per test. With `@DataJpaTest`, also add `@AutoConfigureTestDatabase(replace = Replace.NONE)` — otherwise Spring swaps your container for an embedded DB.

**Interview one-liner:** `@Testcontainers` manages `@Container` fields (static = once per class) and `@DynamicPropertySource` feeds the container's random URL into Spring properties.

---

## 5. @ServiceConnection (Spring Boot 3.1+) — less boilerplate

*Definition:* **`@ServiceConnection`** tells Spring Boot to read the connection details straight from the container — no `@DynamicPropertySource` needed for supported types.

```java
@SpringBootTest
@Testcontainers
class CheckoutIT {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container @ServiceConnection(name = "redis")       // GenericContainer needs a name hint
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @Container @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer("apache/kafka:3.8.0"); // org.testcontainers.kafka

    @Test
    void checkoutFlow() { /* real DB + real cache + real broker */ }
}
```

| Approach | Boot version | Boilerplate | Works for |
|----------|--------------|-------------|-----------|
| `@DynamicPropertySource` | Any (2.2.6+) | Manual property mapping | Anything |
| `@ServiceConnection` | **3.1+** | Zero | Postgres, MySQL, Redis, Kafka, Mongo, RabbitMQ, ... |

### Reuse across test classes
```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {
    @Bean @ServiceConnection
    PostgreSQLContainer<?> postgres() { return new PostgreSQLContainer<>("postgres:16-alpine"); }
}
// then: @SpringBootTest @Import(TestcontainersConfig.class)
```
Bonus: the same config powers `SpringApplication.from(App::main).with(TestcontainersConfig.class)` → run the app locally with containers instead of a hand-written compose file.

**Interview one-liner:** In Spring Boot 3.1+, `@ServiceConnection` auto-wires connection details from a container, replacing manual `@DynamicPropertySource` mapping.

---

## 6. Real e-commerce example — Kafka event after order

```
POST /orders ─► OrderService ─► Postgres (orders table)
                         └────► Kafka topic "order.placed" ─► ShippingConsumer ─► ShipRocket client (mocked)
```

```java
@SpringBootTest
@Import(TestcontainersConfig.class)            // Postgres + Kafka with @ServiceConnection
class OrderPlacedFlowIT {

    @Autowired OrderService orderService;
    @MockitoBean ShipRocketClient shipRocket;  // external SaaS → still mocked (@MockBean before Boot 3.4)

    @Test
    void orderEventTriggersShipment() {
        orderService.place(new OrderRequest("user-7", 1500));

        // async consumer → poll instead of Thread.sleep
        await().atMost(Duration.ofSeconds(10))
               .untilAsserted(() -> verify(shipRocket).createShipment(any()));
    }
}
```

> Use **Awaitility** (`await().untilAsserted(...)`) for async flows (Kafka consumers, `@Async`, schedulers). ❌ `Thread.sleep(5000)` is both slow and flaky.

**Easy rule of thumb:** Containerise what **you own and run** (DB, Redis, Kafka). Mock or stub (WireMock) what **a third party runs** (Razorpay, ShipRocket).

**Interview one-liner:** Real infra in containers, third-party APIs mocked or WireMocked, and Awaitility for async assertions — that's a solid integration test.

---

## 7. Speed, CI & trade-offs (what interviewers probe)

| Concern | Technique |
|---------|-----------|
| Slow startup per class | Singleton container pattern (static start in base class) or shared `@TestConfiguration` |
| Spring context restarts | Keep test config identical across classes so the **context cache** is reused |
| Local dev reruns | `testcontainers.reuse.enable=true` + `.withReuse(true)` (not for CI) |
| Data leaking between tests | `@Transactional` rollback, `@Sql` cleanup, or truncate tables in `@BeforeEach` |
| Separate from unit tests | Name `*IT.java`, run via **Failsafe** (`mvn verify`) vs Surefire (`mvn test`) |
| CI | GitHub Actions `ubuntu-latest` has Docker; or Testcontainers Cloud |

```java
abstract class BaseIT {                               // singleton container pattern
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
    static { POSTGRES.start(); }                      // once per JVM; Ryuk cleans up
}
```

| Pros | Cons |
|------|------|
| Prod-like confidence (same engine/version) | Needs Docker everywhere |
| No shared test DB, no "who broke staging" | Slower than unit tests (seconds) |
| Tests own their infra — clone & run | Image pulls on cold CI runners |

> ⚠️ Pin image tags (`postgres:16-alpine`, not `latest`) so tests match prod and don't break on an upstream release.

**Interview one-liner:** Testcontainers trades a few seconds per suite for prod-like confidence; keep it fast with static/singleton containers and context-cache reuse, and run ITs separately via Failsafe.

---

## Quick Revision Cheat Sheet

- **Integration test**: code + real DB/cache/broker together; catches SQL, mapping, config bugs.
- **H2 trap**: different dialect/behaviour → green tests, broken prod; test on the prod engine.
- **Testcontainers**: real throwaway Docker containers, random ports, Ryuk cleanup.
- **@Testcontainers + @Container**: static = once per class, instance = per test (slow).
- **@DynamicPropertySource**: map container URL/user/password into Spring properties.
- **@ServiceConnection (Boot 3.1+)**: auto connection details, no property mapping.
- **@DataJpaTest + container**: add `@AutoConfigureTestDatabase(replace = NONE)`.
- **Own infra → container; third-party API → mock/WireMock**; async → Awaitility.
- **Speed**: singleton containers, shared config for context caching, `*IT` via Failsafe.
- **Pin image tags** to prod versions; needs Docker in CI.
