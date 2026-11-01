# Testing Notes — Mockito (mock, spy, when/thenReturn, verify, ArgumentCaptor, @InjectMocks)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why mocking? (test doubles)

*Definition:* **Mocking** means replacing a real dependency (DB repository, payment gateway, Kafka producer) with a **fake object you control**, so you can unit-test one class in isolation. **Mockito** is the most popular Java mocking library.

Real life: A flight simulator. Pilots practise engine failure without crashing a real plane — the simulator behaves exactly how the instructor programs it.

```
          REAL                                   IN TEST
OrderService ──► PaymentGateway (Razorpay)   OrderService ──► mock(PaymentGateway)
             ──► OrderRepository (Postgres)               ──► mock(OrderRepository)
             ──► EventPublisher (Kafka)                   ──► mock(EventPublisher)
```

| Test double | What it is |
|-------------|------------|
| **Dummy** | Passed but never used (fills a parameter) |
| **Stub** | Returns canned answers (`when(...).thenReturn(...)`) |
| **Mock** | Stub + records calls so you can `verify` interactions |
| **Spy** | Wraps a **real** object; real methods run unless stubbed |
| **Fake** | Working lightweight impl (in-memory repo, H2) |

Memory trick: **Stub = State** (what it returns), **Mock = Message** (what was called).

**Interview one-liner:** Mocking replaces real dependencies with controllable fakes so a unit test checks only one class; Mockito gives you stubs, mocks and spies.

---

## 2. Creating mocks — mock(), @Mock, @InjectMocks

*Definition:* **`@Mock`** creates a mock field; **`@InjectMocks`** creates the real class under test and injects those mocks into it (constructor → setter → field). `@ExtendWith(MockitoExtension.class)` wires it all.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepo;          // fake
    @Mock PaymentGateway paymentGateway;      // fake

    @InjectMocks OrderService orderService;   // REAL object, mocks injected

    @Test
    void placesOrder() {
        when(paymentGateway.charge(1500)).thenReturn(new PaymentResult("pay_123", true));

        Order o = orderService.place(new OrderRequest(1500));

        assertEquals("pay_123", o.getPaymentId());
    }
}
```

Default return values of an unstubbed mock:

| Return type | Default |
|-------------|---------|
| `int`, `long`, ... | `0` |
| `boolean` | `false` |
| Object | `null` |
| `Optional` / `Stream` | `Optional.empty()` / empty stream |
| Collections | empty collection |

> ⚠️ `@InjectMocks` fails **silently** — if it can't inject, the field just stays `null`, and you get an NPE in the test. Prefer constructor injection in your code; or skip `@InjectMocks` and write `new OrderService(orderRepo, paymentGateway)` explicitly.

**Interview one-liner:** `@Mock` makes fakes, `@InjectMocks` builds the real class under test with those fakes injected, and `MockitoExtension` initialises both.

---

## 3. Stubbing — when/thenReturn, thenThrow, thenAnswer, doReturn

*Definition:* **Stubbing** tells a mock what to return (or throw) when a method is called with given arguments.

```java
when(repo.findById(1L)).thenReturn(Optional.of(order));          // return value
when(gateway.charge(anyLong())).thenThrow(new GatewayTimeout());  // throw
when(repo.save(any(Order.class))).thenAnswer(inv -> {             // compute from args
    Order o = inv.getArgument(0);
    o.setId(42L);
    return o;
});
when(rates.get()).thenReturn(40, 60, 80);                          // consecutive calls

doThrow(new IllegalStateException()).when(publisher).publish(any()); // void method
```

| Style | Syntax | Use when |
|-------|--------|----------|
| **when/thenReturn** | `when(m.call()).thenReturn(x)` | Normal case — type-safe, readable |
| **doReturn/when** | `doReturn(x).when(m).call()` | **Spies** (avoids calling real method), **void** methods (`doThrow`, `doNothing`) |

### Argument matchers
```java
when(repo.findByEmail(eq("d@app.com"))).thenReturn(user);
when(gateway.charge(anyLong())).thenReturn(ok);
when(svc.ship(anyString(), doubleThat(w -> w > 0.5))).thenReturn(label); // primitive → doubleThat, not argThat

// ❌ mixing raw value and matcher → InvalidUseOfMatchersException
when(svc.ship("110001", anyDouble())).thenReturn(label);
// ✅ all matchers
when(svc.ship(eq("110001"), anyDouble())).thenReturn(label);
```

> ⚠️ If one argument uses a matcher, **all** arguments must use matchers. Wrap literals with `eq(...)`.

**Interview one-liner:** Stub with `when(...).thenReturn/thenThrow/thenAnswer`; use `doReturn/doThrow` for void methods and spies; and if you use one matcher, use matchers for every argument.

---

## 4. verify — checking interactions

*Definition:* **`verify`** asserts that a mock method **was called** (how many times, with what arguments). Use it for side effects that have no return value — sending emails, publishing Kafka events, calling ShipRocket.

```java
@Test
void publishesOrderPlacedEvent() {
    orderService.place(new OrderRequest(1500));

    verify(publisher).publish(any(OrderPlacedEvent.class));      // exactly once (default)
    verify(emailService, never()).sendFailureMail(any());
    verify(gateway, times(1)).charge(1500L);
    verify(repo, atLeastOnce()).save(any());
    verifyNoMoreInteractions(gateway);                            // nothing else on gateway
}
```

```java
// Order of calls matters? (charge BEFORE ship)
InOrder inOrder = inOrder(gateway, shipRocketClient);
inOrder.verify(gateway).charge(1500L);
inOrder.verify(shipRocketClient).createShipment(any());
```

| Mode | Meaning |
|------|---------|
| `times(n)` | Exactly n (default is `times(1)`) |
| `never()` | Zero calls |
| `atLeast(n)` / `atMost(n)` | Bounds |
| `only()` | The only method called on that mock |

> ⚠️ Don't `verify` what you already stubbed **and** asserted on — if `findById` returned the order and the result is correct, it was obviously called. Over-verifying ties tests to implementation and breaks on every refactor.

**Easy rule of thumb:** **Queries** (return data) → assert the result. **Commands** (side effects) → `verify`.

**Interview one-liner:** `verify` checks that a side-effect call happened with the right arguments and count; assert return values for queries, verify interactions for commands.

---

## 5. ArgumentCaptor — inspect what was passed

*Definition:* An **`ArgumentCaptor`** captures the actual argument passed to a mock so you can assert on its fields — useful when the object is built **inside** the method under test.

```java
@Captor ArgumentCaptor<OrderPlacedEvent> eventCaptor;

@Test
void eventContainsCorrectData() {
    orderService.place(new OrderRequest(1500, "user-7"));

    verify(publisher).publish(eventCaptor.capture());
    OrderPlacedEvent evt = eventCaptor.getValue();

    assertEquals("user-7", evt.userId());
    assertEquals(1500, evt.amount());
}
```

- Called many times? `captor.getAllValues()` returns a list.
- Non-annotation form: `ArgumentCaptor.forClass(OrderPlacedEvent.class)`.

> Capture in `verify`, not in `when(...)`. Stubbing with `capture()` works but makes tests confusing — for simple checks prefer `argThat(evt -> evt.amount() == 1500)`.

**Interview one-liner:** `ArgumentCaptor` grabs the real argument a mock received so you can assert on objects created inside the method under test.

---

## 6. Spy — partial mock on a real object

*Definition:* A **spy** wraps a **real** object: real methods run by default, but you can stub selected ones and verify calls.

```java
List<String> real = new ArrayList<>();
List<String> spy = spy(real);

spy.add("a");                       // real add() runs
assertEquals(1, spy.size());        // real size() → 1

// ❌ calls the REAL get(5) while stubbing → IndexOutOfBoundsException
when(spy.get(5)).thenReturn("x");
// ✅ doReturn does not invoke the real method
doReturn("x").when(spy).get(5);
```

| | **mock** | **spy** |
|---|---------|---------|
| Base | Fake shell of the class | Real instance |
| Unstubbed call | Returns default (0/null/empty) | Runs **real** code |
| Stub syntax | `when/thenReturn` is fine | Prefer `doReturn/when` |
| Use for | Dependencies | Legacy code you can't refactor yet |

> ⚠️ Needing a spy on your own class is often a **design smell** — the class does too much. Extract the part you want to fake into its own dependency.

**Interview one-liner:** A mock is a fully fake object; a spy wraps a real one and runs real methods unless stubbed — stub spies with `doReturn` to avoid triggering real code.

---

## 7. Strict stubs & common pitfalls

*Definition:* **Strict stubs** (default with `MockitoExtension`) make Mockito fail the test on **unnecessary stubbing** — a stub that the code never used — throwing `UnnecessaryStubbingException`.

```java
// Throws UnnecessaryStubbingException if place() never calls findById
when(repo.findById(1L)).thenReturn(Optional.of(order));

// If it's genuinely needed only in some paths:
lenient().when(repo.findById(1L)).thenReturn(Optional.of(order));
```

| Pitfall | Fix |
|---------|-----|
| Mocking `final` / `static` methods | Mockito 5 uses the **inline mock maker** by default → finals work; statics via `mockStatic(...)` in try-with-resources |
| Mocking value objects / DTOs / `List` | Just create real ones |
| Mocking the class under test | Never — test the real thing |
| Mocking types you don't own (Razorpay SDK) | Wrap in your own `PaymentGateway` interface, mock that |
| `@InjectMocks` field is null | Use constructor injection / build manually |
| Over-verifying | Verify only meaningful side effects |

```java
try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
    mocked.when(UUID::randomUUID).thenReturn(fixedUuid);
    // ... test code
}   // static mock removed here — always close it
```

> NestJS/Jest equivalent: `jest.fn()`, `mockResolvedValue`, `toHaveBeenCalledWith` — same ideas, different syntax.

**Easy rule of thumb:** **"Don't mock what you don't own"** — mock your own interfaces, use real value objects, and integration-test the adapter.

**Interview one-liner:** Strict stubs flag unused stubbing; avoid mocking value objects, the class under test, or third-party SDKs directly — wrap them in your own interface instead.

---

## Quick Revision Cheat Sheet

- **Mocking**: replace real deps with controllable fakes; Stub = State, Mock = Message.
- **@Mock + @InjectMocks + MockitoExtension**: fakes injected into the real class under test.
- **Unstubbed mock** returns 0 / false / null / empty collection or Optional.
- **Stubbing**: `when().thenReturn/thenThrow/thenAnswer`; `doReturn/doThrow` for voids & spies.
- **Matchers**: one matcher → all args matchers (`eq(...)` for literals).
- **verify**: `times/never/atLeast`, `InOrder`; verify commands, assert queries.
- **ArgumentCaptor**: capture args inside `verify`, then assert on fields.
- **Spy**: real object, real methods unless stubbed; use `doReturn`; often a design smell.
- **Strict stubs**: unused stub → `UnnecessaryStubbingException`; `lenient()` to relax.
- **Pitfalls**: don't mock what you don't own, DTOs, or the SUT; `mockStatic` in try-with-resources.
