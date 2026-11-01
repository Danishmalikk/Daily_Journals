# Testing Notes — Unit Testing with JUnit 5 (@Test, @BeforeEach, @AfterEach, @ParameterizedTest, @Nested)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Unit Test?

*Definition:* A **unit test** checks **one small piece of code** (a method or class) **in isolation** — no real DB, no network, no Spring context. It should run in milliseconds and give the same result every time.

Real life: Before assembling a car, each part (brake pad, spark plug) is tested on its own bench. If the brake pad fails on the bench, you know exactly which part is broken.

```
        /\        Few   → E2E / UI tests        (slow, brittle)
       /  \
      /----\      Some  → Integration tests     (real DB via Testcontainers)
     /      \
    /--------\    Many  → Unit tests            (fast, isolated)  ← this file
```

Good unit tests are **FIRST**:

| Letter | Meaning | Why |
|--------|---------|-----|
| **F** | Fast | Runs on every save / every CI push |
| **I** | Independent | No test depends on another's order or state |
| **R** | Repeatable | Same result on laptop, CI, any timezone |
| **S** | Self-validating | Pass/fail via assertions, no reading logs |
| **T** | Timely | Written with (or before) the code |

Memory trick: **"FIRST"** — good unit tests come FIRST in the pyramid.

**Interview one-liner:** A unit test verifies one class or method in isolation, with dependencies faked, and should be Fast, Independent, Repeatable, Self-validating and Timely.

---

## 2. JUnit 5 architecture & the basic @Test

*Definition:* **JUnit 5** is the standard Java test framework. It is made of 3 modules: **Platform** (launches tests), **Jupiter** (the new API — `@Test`, assertions) and **Vintage** (runs old JUnit 4 tests).

```
JUnit 5 = Platform + Jupiter + Vintage
          (engine     (what you    (legacy
           launcher)   write)       JUnit 4)
```

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PriceCalculatorTest {                       // no need for public in JUnit 5

    @Test
    void appliesTenPercentDiscount() {
        PriceCalculator calc = new PriceCalculator();
        assertEquals(900, calc.applyDiscount(1000, 10));
    }
}
```

| JUnit 4 | JUnit 5 (Jupiter) |
|---------|-------------------|
| `org.junit.Test` | `org.junit.jupiter.api.Test` |
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| `@BeforeClass` / `@AfterClass` | `@BeforeAll` / `@AfterAll` |
| `@Ignore` | `@Disabled` |
| `@RunWith(...)` | `@ExtendWith(...)` |
| `@Test(expected = X.class)` | `assertThrows(X.class, ...)` |
| Classes/methods must be `public` | Package-private is fine |

> ⚠️ Mixing `org.junit.Test` (JUnit 4) with Jupiter assertions is a classic bug — the test silently doesn't run under the Jupiter engine. Check your imports.

**Interview one-liner:** JUnit 5 = Platform + Jupiter + Vintage; you write tests with Jupiter's `@Test`, and `@ExtendWith` replaces JUnit 4's `@RunWith`.

---

## 3. Lifecycle — @BeforeEach, @AfterEach, @BeforeAll, @AfterAll

*Definition:* **Lifecycle annotations** run setup/cleanup code around tests. `@BeforeEach`/`@AfterEach` run around **every** test; `@BeforeAll`/`@AfterAll` run **once** for the whole class.

Real life: A restaurant cleans every table after each customer (`@AfterEach`) but opens the kitchen only once in the morning (`@BeforeAll`).

```
@BeforeAll          (once, static)
  ├─ @BeforeEach → @Test #1 → @AfterEach
  ├─ @BeforeEach → @Test #2 → @AfterEach
  └─ @BeforeEach → @Test #3 → @AfterEach
@AfterAll           (once, static)
```

```java
class CartServiceTest {

    static ExpensiveClient client;
    CartService cart;

    @BeforeAll
    static void initOnce() { client = new ExpensiveClient(); }   // must be static

    @BeforeEach
    void setUp() { cart = new CartService(); }                    // fresh object per test

    @AfterEach
    void tearDown() { cart.clear(); }

    @AfterAll
    static void closeOnce() { client.close(); }

    @Test
    void emptyCartHasZeroTotal() { assertEquals(0, cart.total()); }
}
```

| Annotation | Runs | Must be static? | Typical use |
|------------|------|-----------------|-------------|
| `@BeforeEach` | Before every test | No | Create fresh object under test |
| `@AfterEach` | After every test | No | Reset state, delete temp files |
| `@BeforeAll` | Once before all | **Yes** (unless `PER_CLASS`) | Start expensive resource |
| `@AfterAll` | Once after all | **Yes** (unless `PER_CLASS`) | Close resource |

> ⚠️ JUnit 5 creates a **new instance of the test class for every test method** (default `Lifecycle.PER_METHOD`). That's why instance fields are fresh per test and why `@BeforeAll` must be `static`. With `@TestInstance(Lifecycle.PER_CLASS)` it can be non-static.

**Easy rule of thumb:** Put the object under test in `@BeforeEach`. Use `@BeforeAll` only for genuinely expensive, read-only things.

**Interview one-liner:** `@BeforeEach/@AfterEach` wrap every test; `@BeforeAll/@AfterAll` run once and must be static because JUnit creates a new test instance per method by default.

---

## 4. The AAA pattern & assertions

*Definition:* **AAA (Arrange–Act–Assert)** is the standard shape of a test: set up data, call the method once, verify the result. (BDD calls it **Given–When–Then**.)

```java
@Test
void placingOrderReducesStock() {
    // Arrange (Given)
    Inventory inv = new Inventory(Map.of("SKU-1", 5));

    // Act (When)
    inv.reserve("SKU-1", 2);

    // Assert (Then)
    assertEquals(3, inv.available("SKU-1"));
}
```

Core assertions:

| Assertion | Checks |
|-----------|--------|
| `assertEquals(expected, actual)` | Equality (expected **first**!) |
| `assertTrue / assertFalse` | Boolean condition |
| `assertNull / assertNotNull` | Null-ness |
| `assertSame` | Same reference (`==`) |
| `assertIterableEquals` | Collections element-by-element |
| `assertAll(...)` | Runs **all** checks, reports every failure |
| `assertTimeout(Duration, ...)` | Finishes within time |

```java
// ❌ stops at first failure — you fix one, rerun, find the next
assertEquals("PAID", order.getStatus());
assertEquals(1500, order.getAmount());

// ✅ reports all failures in one run
assertAll("order",
    () -> assertEquals("PAID", order.getStatus()),
    () -> assertEquals(1500, order.getAmount()));
```

> Many teams use **AssertJ** (`assertThat(total).isEqualTo(900)`) — fluent, better messages, ships with `spring-boot-starter-test`. In NestJS/Node the equivalent is Jest's `expect(total).toBe(900)`.

**Interview one-liner:** Structure every test as Arrange–Act–Assert, one behaviour per test, and use `assertAll` when you want every failed check reported at once.

---

## 5. Testing exceptions — assertThrows

*Definition:* **`assertThrows`** asserts that a piece of code throws a specific exception type, and **returns** the exception so you can check its message.

```java
@Test
void rejectsNegativeAmount() {
    PaymentService svc = new PaymentService();

    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> svc.charge(-100)                    // lambda = the code that should throw
    );

    assertEquals("Amount must be positive", ex.getMessage());
}
```

```java
// ❌ old try/catch style — passes silently if nothing is thrown and you forget fail()
try { svc.charge(-100); } catch (IllegalArgumentException e) { /* ok */ }

// ✅ assertThrows fails loudly if nothing (or the wrong type) is thrown
```

> ⚠️ `assertThrows` also passes for **subclasses** of the expected type. Use `assertThrowsExactly` (JUnit 5.8+) when the exact type matters.

**Interview one-liner:** Use `assertThrows(Type.class, () -> call())` — it fails if nothing is thrown and returns the exception so you can assert on its message.

---

## 6. @ParameterizedTest — one test, many inputs

*Definition:* **`@ParameterizedTest`** runs the same test method multiple times with different inputs supplied by a **source** annotation. Needs `junit-jupiter-params` (included in `spring-boot-starter-test`).

Real life: Same exam paper, many students — you don't write a new paper for each one.

```java
@ParameterizedTest(name = "{0} kg to pincode {1} → ₹{2}")
@CsvSource({
    "0.5, 110001, 40",
    "1.0, 110001, 60",
    "2.5, 560001, 120"
})
void shippingCost(double weightKg, String pincode, int expected) {
    assertEquals(expected, shippingCalc.cost(weightKg, pincode));   // e.g. ShipRocket rate rules
}
```

| Source | Supplies |
|--------|----------|
| `@ValueSource(ints = {1, 2, 3})` | One simple value per run |
| `@CsvSource({...})` | Multiple columns inline |
| `@CsvFileSource(resources = "/rates.csv")` | Rows from a CSV file |
| `@EnumSource(OrderStatus.class)` | Every enum constant |
| `@MethodSource("orders")` | Objects from a static factory method |
| `@NullAndEmptySource` | `null` and `""` (edge cases) |

**Easy rule of thumb:** If you're copy-pasting a test and changing only the numbers → make it parameterized.

**Interview one-liner:** `@ParameterizedTest` with sources like `@CsvSource` or `@MethodSource` runs one test body over many inputs — great for boundary and edge cases.

---

## 7. Readability — @DisplayName, @Nested, @Disabled, @Tag

*Definition:* These annotations organise tests so the report reads like a **specification**.

```java
@DisplayName("Order cancellation")
class OrderCancellationTest {

    @Nested
    @DisplayName("when order is PAID")
    class WhenPaid {
        @Test @DisplayName("triggers a Razorpay refund")
        void refunds() { /* ... */ }
    }

    @Nested
    @DisplayName("when order is SHIPPED")
    class WhenShipped {
        @Test @DisplayName("is rejected")
        void rejected() { /* ... */ }
    }

    @Test @Disabled("flaky — JIRA-421")
    void legacyCase() { }

    @Test @Tag("slow")
    void bigReport() { }               // run with: mvn test -Dgroups=slow
}
```

Report output:
```
Order cancellation
 ├─ when order is PAID     ✔ triggers a Razorpay refund
 └─ when order is SHIPPED  ✔ is rejected
```

| Annotation | Purpose |
|------------|---------|
| `@DisplayName` | Human-readable name in reports |
| `@Nested` | Group tests by scenario (inner non-static class) |
| `@Disabled` | Skip a test (always give a reason) |
| `@Tag` | Filter tests in Maven/Gradle (e.g. "slow", "integration") |
| `@RepeatedTest(5)` | Run N times (hunting flakiness) |
| `@Timeout(2)` | Fail if slower than 2s |

> `@Nested` classes must be **non-static inner classes** — they can use the outer class's `@BeforeEach`.

**Interview one-liner:** `@DisplayName` and `@Nested` turn the test report into readable living documentation; `@Tag` lets CI split fast and slow tests.

---

## 8. What interviewers probe (and good answers)

| Question | Strong answer |
|----------|---------------|
| What makes a test a *unit* test? | No I/O (DB, network, file, clock); dependencies mocked; runs in ms |
| Why does `@BeforeAll` need `static`? | New test instance per method; use `@TestInstance(PER_CLASS)` to avoid it |
| How do you test time-based logic? | Inject a `java.time.Clock` and use `Clock.fixed(...)` in tests |
| How do you test private methods? | You don't — test through the public API; if it's hard, extract a class |
| One assert per test? | One **behaviour** per test; multiple asserts on that behaviour are fine |
| What causes flaky tests? | Shared static state, real time, randomness, test order, real network |

```java
// ❌ untestable — hidden dependency on real time
boolean isExpired() { return LocalDate.now().isAfter(expiry); }

// ✅ inject a Clock
boolean isExpired(Clock clock) { return LocalDate.now(clock).isAfter(expiry); }
// test: Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC)
```

> ⚠️ A test that hits a real Postgres or calls the real Razorpay sandbox is an **integration** test, not a unit test — tag it and run it separately.

**Interview one-liner:** A good unit test is isolated and deterministic — inject anything non-deterministic (clock, random, IDs) and test behaviour through the public API.

---

## Quick Revision Cheat Sheet

- **Unit test**: one class in isolation, no I/O, milliseconds; follow **FIRST**.
- **JUnit 5** = Platform + Jupiter + Vintage; `@ExtendWith` replaces `@RunWith`.
- **Lifecycle**: `@BeforeEach/@AfterEach` per test; `@BeforeAll/@AfterAll` once, **static** by default.
- **New instance per test method** (PER_METHOD) → fields are always fresh.
- **AAA / Given-When-Then**: arrange, act once, assert; `assertEquals(expected, actual)`.
- **assertAll** reports every failure; **AssertJ** for fluent assertions.
- **assertThrows** returns the exception; `assertThrowsExactly` for exact type.
- **@ParameterizedTest** + `@CsvSource/@MethodSource/@EnumSource` = one test, many inputs.
- **@DisplayName / @Nested / @Tag / @Disabled** = readable, groupable, filterable tests.
- **Determinism**: inject `Clock`, avoid shared static state; don't test private methods directly.
