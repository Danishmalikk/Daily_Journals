# Testing Notes — @MockBean vs @Mock (Spring context vs pure unit, @MockitoBean in Boot 3.4+)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The core difference in one picture

*Definition:* **`@Mock`** (Mockito) creates a plain fake object with **no Spring involved**. **`@MockBean`** (Spring Boot) creates a Mockito mock and **puts it into the Spring ApplicationContext**, replacing (or adding) a bean of that type.

Real life: `@Mock` is a cardboard cut-out you hold up in your own rehearsal room. `@MockBean` is a body double placed **on the actual film set** — every actor (bean) on the set now talks to the double.

```
@Mock (pure unit)                         @MockBean (Spring context)
───────────────────                       ──────────────────────────────────
JUnit + MockitoExtension                  JUnit + SpringExtension
                                          ┌──────── ApplicationContext ────────┐
 new OrderService(mockRepo)               │ OrderController ─► OrderService    │
        │                                 │                       │            │
        ▼                                 │                       ▼            │
   mockRepo  (just a Java object)         │   PaymentGateway = Mockito MOCK ◄──┼── replaced bean
                                          └────────────────────────────────────┘
 ⚡ milliseconds                            🐢 seconds (context startup)
```

**Interview one-liner:** `@Mock` is a plain Mockito fake for pure unit tests; `@MockBean` registers a Mockito mock as a bean inside the Spring context, replacing the real one.

---

## 2. @Mock — pure unit test (no Spring)

*Definition:* **`@Mock`** is initialised by **`MockitoExtension`**; you wire it into the class under test yourself (or via `@InjectMocks`).

```java
@ExtendWith(MockitoExtension.class)
class RefundServiceTest {

    @Mock PaymentGateway gateway;          // Razorpay wrapper, faked
    @Mock OrderRepository orderRepo;

    @InjectMocks RefundService refundService;

    @Test
    void refundsPaidOrder() {
        when(orderRepo.findById(7L)).thenReturn(Optional.of(paidOrder(7L, 1500)));

        refundService.refund(7L);

        verify(gateway).refund("pay_123", 1500);
    }
}
```

- No `@SpringBootTest`, no `@Autowired`, no application context → **runs in milliseconds**.
- Works because the service uses **constructor injection** — you can build it with fakes.

> ✅ This should be the **default** for testing services and business logic. Most of your tests should look like this.

**Interview one-liner:** Use `@Mock` with `MockitoExtension` to test a class in total isolation — no Spring context, fastest feedback.

---

## 3. @MockBean — mock inside the Spring context

*Definition:* **`@MockBean`** is used in Spring tests (`@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`...). It replaces the bean of that type in the context, so **every bean that depends on it gets the mock**. It's reset after each test by default.

```java
@WebMvcTest(OrderController.class)                 // loads only web layer
class OrderControllerTest {

    @Autowired MockMvc mockMvc;

    @MockBean OrderService orderService;           // controller needs it; not loaded in slice

    @Test
    void returnsOrder() throws Exception {
        when(orderService.find(7L)).thenReturn(new OrderDto(7L, "PAID", 1500));

        mockMvc.perform(get("/orders/7"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("PAID"));
    }
}
```

When you **need** it:
- **Slice tests** — `@WebMvcTest` doesn't load services, so the controller's dependencies must be mocked beans.
- **Full-context tests** — real wiring, but fake the external edge: Razorpay, ShipRocket, SES email, a Kafka producer.
- Testing **Spring behaviour itself** — security filters, validation, `@ControllerAdvice`, JSON serialisation, `@Transactional`, AOP.

> Use `@SpyBean` (now `@MockitoSpyBean`) to wrap the **real** bean and only stub/verify part of it.

**Interview one-liner:** `@MockBean` replaces a bean in the Spring context with a mock — needed in slice tests and full-context tests where Spring wiring, MVC, security or JSON must be real.

---

## 4. Side-by-side comparison (the money table)

| Feature | `@Mock` | `@MockBean` / `@MockitoBean` |
|---------|---------|------------------------------|
| From | Mockito (`org.mockito`) | Spring Boot / Spring Framework test |
| Needs Spring context | ❌ No | ✅ Yes |
| Initialised by | `MockitoExtension` / `openMocks` | Spring TestContext framework |
| Injected via | `@InjectMocks` or constructor | Autowiring into other beans |
| Replaces a real bean | No (nothing to replace) | Yes, by type (or name) |
| Speed | ⚡ ms | 🐢 seconds (context start) |
| Reset between tests | New mock per test instance | Reset after each test (`MockReset.AFTER`) |
| Typical test | Service / domain unit test | `@WebMvcTest`, `@SpringBootTest` |
| Tests Spring config? | No | Yes (MVC, security, validation, AOP) |

Memory trick: **"Mock = Mockito alone; MockBean = Mock + Bean in the box (context)."**

**Easy rule of thumb:** Testing **your logic** → `@Mock`. Testing **how Spring wires/serves your logic** → `@MockBean`.

**Interview one-liner:** `@Mock` = Mockito-only, no context, fastest; `@MockBean` = mock registered in the Spring context, slower but lets you test real Spring wiring.

---

## 5. The hidden cost — context caching

*Definition:* Spring's **TestContext framework caches** application contexts and reuses them across test classes **if their configuration is identical**. `@MockBean` changes the configuration — a **different set of mock beans = a different cache key = a brand new context**.

```
OrderControllerTest    @MockBean OrderService                 → Context A (start 6s)
PaymentControllerTest  @MockBean OrderService                 → Context A reused ✅ (0s)
ShippingControllerTest @MockBean OrderService, ShipRocketClient → Context B (start 6s) ❌
RefundIT               @MockBean PaymentGateway               → Context C (start 6s) ❌

40 test classes × unique mock combos → minutes of extra CI time
```

```java
// ✅ Fix: one shared base class with the SAME mocks for all integration tests
@SpringBootTest
abstract class BaseIT {
    @MockitoBean PaymentGateway paymentGateway;     // Razorpay
    @MockitoBean ShipRocketClient shipRocketClient;
}

class CheckoutIT extends BaseIT { /* reuses the cached context */ }
class RefundIT   extends BaseIT { /* same key → same context */ }
```

Also creates new contexts: different `@ActiveProfiles`, `@TestPropertySource`, `@Import`, `@DirtiesContext`.

> ⚠️ Symptom in CI: logs show the Spring Boot banner **dozens of times** and the suite takes 10+ minutes. Count your unique `@MockBean` combinations.

**Interview one-liner:** Each unique set of `@MockBean`s creates a new cache key, forcing a fresh Spring context — consolidate mocks in a shared base class to keep the context cache hot.

---

## 6. @MockitoBean — the new name (Spring Framework 6.2 / Boot 3.4+)

*Definition:* In **Spring Boot 3.4**, `@MockBean` and `@SpyBean` were **deprecated** (for removal) in favour of **`@MockitoBean`** and **`@MockitoSpyBean`**, which live in **Spring Framework's** own test module (`org.springframework.test.context.bean.override.mockito`).

```java
// ❌ Spring Boot ≤ 3.3 style (deprecated in 3.4)
import org.springframework.boot.test.mock.mockito.MockBean;
@MockBean PaymentGateway gateway;

// ✅ Spring Boot 3.4+ / Spring Framework 6.2+
import org.springframework.test.context.bean.override.mockito.MockitoBean;
@MockitoBean PaymentGateway gateway;
@MockitoSpyBean OrderService orderService;
```

| | `@MockBean` (old) | `@MockitoBean` (new) |
|---|-------------------|----------------------|
| Module | Spring Boot test | Spring Framework test (core) |
| Status | Deprecated in Boot 3.4, removed in Boot 4 | Current API |
| Works without Boot | No | Yes (plain Spring tests too) |
| On `@Configuration` classes | Supported | **Not** supported — declare on test class fields (or, in 6.2.2+, at the type level via `types`) |
| Mechanism | Boot's `MockitoPostProcessor` | Generic **Bean Override** framework (`@TestBean` is a sibling) |

> ⚠️ Migration gotcha: if you declared `@MockBean` fields inside a shared `@TestConfiguration`, they won't move 1:1 — move them to a base test class instead.

**Interview one-liner:** Since Spring Boot 3.4, use `@MockitoBean`/`@MockitoSpyBean` from Spring Framework instead of the deprecated `@MockBean`/`@SpyBean` — same idea, built on the new Bean Override mechanism.

---

## 7. Decision guide & common mistakes (what interviewers probe)

```
What am I testing?
 ├─ Pure business logic (pricing, refunds, state machine) ──► @Mock + MockitoExtension
 ├─ Controller: routes, validation, JSON, status codes ─────► @WebMvcTest + @MockitoBean services
 ├─ Repository / SQL ───────────────────────────────────────► @DataJpaTest + Testcontainers (no mocks)
 └─ Whole flow with real wiring ────────────────────────────► @SpringBootTest + @MockitoBean only at external edges
```

| Mistake | Why it hurts | Fix |
|---------|--------------|-----|
| `@SpringBootTest` + `@MockBean` to test one service | Seconds of startup for a ms test | Plain `@Mock` unit test |
| `@Mock` field in a `@SpringBootTest` | Spring never sees it → real bean is used, mock ignored | Use `@MockitoBean` |
| `@MockBean` in a pure Mockito test | No context → stays `null` | Use `@Mock` |
| Mocking repositories in `@SpringBootTest` | Hides real SQL bugs | Use Testcontainers |
| Unique mock set per class | Context cache misses | Shared base class |

NestJS parallel: `Test.createTestingModule({...}).overrideProvider(PaymentGateway).useValue(mockGateway)` ≈ `@MockitoBean`; constructing the service with `new` and `jest.fn()` deps ≈ `@Mock`.

**Interview one-liner:** Default to `@Mock` for logic; reach for `@MockitoBean` only when Spring itself is part of what you're testing, and mock only the external edges.

---

## Quick Revision Cheat Sheet

- **@Mock**: Mockito-only fake, no Spring, `MockitoExtension` + `@InjectMocks`, milliseconds.
- **@MockBean**: Mockito mock registered **as a bean** in the context, replaces the real one by type.
- **Use @Mock** for services/domain logic; **@MockBean** for slice tests and real-wiring tests.
- **@SpyBean / @MockitoSpyBean**: wrap the real bean, stub/verify only parts.
- **Context caching**: unique mock-bean sets = new context each time → slow CI; share a base class.
- **@MockitoBean / @MockitoSpyBean** (Spring Framework 6.2, Boot 3.4+) replace deprecated `@MockBean/@SpyBean`.
- **Gotcha**: `@Mock` inside `@SpringBootTest` is ignored by Spring; `@MockBean` without context is null.
- **Mock only external edges** (Razorpay, ShipRocket) in integration tests; real DB via Testcontainers.
