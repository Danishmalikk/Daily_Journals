# Testing Notes — TDD Approach (Red-Green-Refactor, worked example, when not to)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is TDD?

*Definition:* **Test-Driven Development (TDD)** is a workflow where you write a **failing test first**, then write the **minimum code** to make it pass, then **clean up** the code — in tiny repeating cycles.

Real life: A tailor takes your measurements (the test) **before** cutting cloth (the code). Cutting first and measuring later wastes cloth.

| | Test-after (traditional) | Test-first (TDD) |
|---|--------------------------|------------------|
| Order | Code → tests (maybe) | Test → code → refactor |
| Tests drive | Nothing — they check | **Design** — they shape the API |
| Coverage | Whatever time allows | High by default |
| Feedback | Late | Every few minutes |
| Risk | Tests fitted to existing bugs | Tests describe intended behaviour |

> TDD is primarily a **design technique**, not a testing technique. The tests are a (very useful) by-product.

**Interview one-liner:** TDD means writing a failing test first, making it pass with minimal code, then refactoring — it drives design and gives a safety net as a by-product.

---

## 2. The Red–Green–Refactor cycle

*Definition:* **Red-Green-Refactor** is the 3-step loop at the heart of TDD.

```
        ┌────────────────────────────────────────┐
        ▼                                        │
   🔴 RED ───────────► 🟢 GREEN ───────────► 🔵 REFACTOR
   Write a small        Write the SIMPLEST     Clean up code + tests
   failing test         code to pass           (all tests stay green)
   (compile fail        (hard-coding is OK     Remove duplication,
    counts as red)       at first!)             rename, extract
```

| Step | Goal | Rule |
|------|------|------|
| 🔴 **Red** | Define the next behaviour | Watch it fail — proves the test can fail |
| 🟢 **Green** | Make it work | Minimum code, no extra features |
| 🔵 **Refactor** | Make it right | No new behaviour; tests must stay green |

Memory trick: **"Make it fail, make it work, make it right."**

> ⚠️ Skipping **Red** is the most common mistake. A test you never saw fail might be testing nothing (wrong assertion, test not even running).

**Interview one-liner:** Red — write a failing test; Green — the simplest code to pass; Refactor — clean up while staying green; repeat in cycles of minutes.

---

## 3. Uncle Bob's 3 Laws of TDD

1. Don't write production code unless it makes a **failing test pass**.
2. Don't write more of a test than is **sufficient to fail** (not compiling = failing).
3. Don't write more production code than is **sufficient to pass** the one failing test.

Kent Beck's strategies for going green:

| Strategy | What you do | When |
|----------|-------------|------|
| **Fake it** | Return a constant (`return 40;`) | Unsure how to start |
| **Triangulate** | Add a 2nd test that forces generalisation | Fake breaks with new example |
| **Obvious implementation** | Just write the real code | Solution is clear |

**Interview one-liner:** The 3 laws keep the cycle tiny — no production code without a failing test, and only enough test or code to move one step.

---

## 4. Worked example — shipping fee calculator

Requirement (e-commerce, ShipRocket-style rules): **₹40 up to 0.5 kg, +₹20 per extra 0.5 kg (or part of it), free shipping for orders ≥ ₹999.**

### Cycle 1 — 🔴 Red
```java
class ShippingFeeTest {
    ShippingFee fee = new ShippingFee();

    @Test
    void baseFeeUpToHalfKg() {
        assertEquals(40, fee.calculate(0.5, 500));   // ShippingFee doesn't exist → RED
    }
}
```
### Cycle 1 — 🟢 Green (fake it)
```java
class ShippingFee {
    int calculate(double weightKg, int orderValue) { return 40; }
}
```

### Cycle 2 — 🔴 Red (triangulate)
```java
@Test
void addsTwentyPerExtraHalfKg() {
    assertEquals(60, fee.calculate(1.0, 500));
    assertEquals(80, fee.calculate(1.2, 500));       // part of a slab counts
}
```
### Cycle 2 — 🟢 Green (real logic)
```java
int calculate(double weightKg, int orderValue) {
    int slabs = (int) Math.ceil(weightKg / 0.5);     // 0.5→1, 1.0→2, 1.2→3
    return 40 + (slabs - 1) * 20;
}
```

### Cycle 3 — 🔴 Red → 🟢 Green
```java
@Test
void freeShippingAtOrAbove999() {
    assertEquals(0, fee.calculate(3.0, 999));
}
```
```java
int calculate(double weightKg, int orderValue) {
    if (orderValue >= 999) return 0;
    int slabs = (int) Math.ceil(weightKg / 0.5);
    return 40 + (slabs - 1) * 20;
}
```

### 🔵 Refactor — name the magic numbers, tests stay green
```java
class ShippingFee {
    private static final int BASE_FEE = 40, PER_SLAB = 20, FREE_ABOVE = 999;
    private static final double SLAB_KG = 0.5;

    int calculate(double weightKg, int orderValue) {
        if (orderValue >= FREE_ABOVE) return 0;
        int extraSlabs = Math.max(0, (int) Math.ceil(weightKg / SLAB_KG) - 1);
        return BASE_FEE + extraSlabs * PER_SLAB;
    }
}
```

> Notice: the **edge cases** (exact 999, partial slab 1.2 kg) were forced out by tests *before* the code existed. Next cycles would add: `weight <= 0` → `assertThrows`, then convert to a `@ParameterizedTest`.

**Interview one-liner:** In TDD you grow the code one failing test at a time — fake it, triangulate with a second example, then refactor once tests are green.

---

## 5. Benefits (and the honest costs)

| Benefit | Why |
|---------|-----|
| Better design | Hard-to-test code shows up immediately → pushes DI, small classes |
| Safety net for refactoring | Change internals fearlessly |
| Living documentation | Tests show exactly how the API is meant to be used |
| Fewer regressions | Every bug fix starts with a failing test that reproduces it |
| Less debugging | Failures are minutes old, so the cause is obvious |

| Cost | Reality |
|------|---------|
| Slower at first | ~15-35% more initial dev time reported in studies; pays back in maintenance |
| Learning curve | Takes weeks of practice to feel natural |
| Test maintenance | Tests coupled to implementation break on refactors |
| False confidence | 100% green ≠ correct if requirements were misunderstood |

> The best habit even if you don't do full TDD: **"Bug → failing test first → fix."** This alone kills most regressions.

**Interview one-liner:** TDD costs some speed upfront but buys better design, fearless refactoring and fewer regressions — at minimum, always reproduce a bug with a failing test before fixing it.

---

## 6. When to use TDD — and when not to

| ✅ Great fit | ❌ Poor fit |
|-------------|------------|
| Business rules: pricing, discounts, tax, shipping | Throwaway spikes / prototypes |
| Algorithms, parsers, validators | Exploratory UI / CSS tweaks |
| Bug fixes (reproduce first) | Learning a new library/API |
| Payment state machines (CREATED → PAID → REFUNDED) | Thin glue code / config wiring |
| Code with clear inputs/outputs | Requirements totally unknown |

**Spike then TDD:** explore the Razorpay SDK in a throwaway spike, **delete it**, then TDD the real `PaymentService` behind your own `PaymentGateway` interface.

**Easy rule of thumb:** The more **logic and risk**, the more TDD pays off. The more **exploratory**, the less.

**Interview one-liner:** TDD shines for business logic, algorithms and bug fixes; for spikes and exploratory work, explore first, then throw it away and TDD the real thing.

---

## 7. Styles & related ideas (what interviewers probe)

| Term | Meaning |
|------|---------|
| **Inside-out (Chicago/Classic)** | Start from domain logic; use real objects; verify state |
| **Outside-in (London/Mockist)** | Start from the API/controller; mock collaborators; verify interactions |
| **BDD** | TDD in business language — Given/When/Then (Cucumber, JBehave) |
| **ATDD** | Acceptance test written first with product owner, then TDD inside it |
| **Double loop TDD** | Outer failing acceptance test + inner unit-test cycles |

```
Outer loop (acceptance):  🔴 POST /orders returns 201 ...................... 🟢
Inner loops (unit):          🔴🟢🔵  🔴🟢🔵  🔴🟢🔵   (service, pricing, repo)
```

Common interview questions:
- *"Does TDD guarantee bug-free code?"* → No; it guarantees code does what the **tests** say. Wrong tests = wrong code.
- *"How do you TDD with a database?"* → Unit-TDD the logic with a mocked/fake repository; cover SQL with Testcontainers integration tests.
- *"TDD vs high coverage?"* → TDD naturally gives high coverage, but coverage doesn't imply TDD or quality.

**Interview one-liner:** Classic TDD verifies state with real objects, London-style mocks collaborators outside-in; BDD and ATDD apply the same test-first idea at the business/acceptance level.

---

## Quick Revision Cheat Sheet

- **TDD**: test first → minimal code → refactor; a design technique with tests as by-product.
- **Red-Green-Refactor**: make it fail, make it work, make it right — always see Red.
- **3 Laws**: no prod code without failing test; only enough test to fail; only enough code to pass.
- **Fake it → Triangulate → Obvious implementation** = ways to get Green.
- **Worked example**: edge cases (exact threshold, partial slab) emerge from tests first.
- **Benefits**: better design, safe refactoring, docs, fewer regressions; **cost**: slower start.
- **Bug fix rule**: reproduce with a failing test before fixing.
- **Use for** business rules/algorithms/bug fixes; **skip for** spikes/exploratory/glue code.
- **Chicago** (state, real objects) vs **London** (mocks, outside-in); **BDD/ATDD** = business-level test-first.
