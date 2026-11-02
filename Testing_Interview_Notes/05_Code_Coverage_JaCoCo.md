# Testing Notes — Code Coverage with JaCoCo (line/branch coverage, thresholds, excludes)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is code coverage?

*Definition:* **Code coverage** is a metric that tells you **what percentage of your code was executed** while the tests ran. It shows which code is **untested** — not whether the tested code is correct.

Real life: A security guard's patrol log shows which rooms he walked through. It doesn't prove he actually checked the locks in each room.

```
  src/main/java  ──(tests run with JaCoCo agent)──►  jacoco.exec  ──►  HTML/XML report
                                                     (raw hits)       🟢 covered
                                                                      🟡 partly (some branches)
                                                                      🔴 never executed
```

> Coverage answers **"what did my tests NOT touch?"** — that's its real value.

**Interview one-liner:** Code coverage measures how much code the tests executed; it's great at revealing untested code but says nothing about whether assertions are meaningful.

---

## 2. Coverage types — line vs branch (and the rest)

*Definition:* JaCoCo reports several **counters**. The two that matter most are **line** (was this line run?) and **branch** (was every path of each `if`/`switch` taken?).

```java
int shippingFee(int orderValue) {
    if (orderValue >= 999) return 0;      // branch: true / false
    return 40;
}

@Test void freeShipping() { assertEquals(0, shippingFee(1200)); }
// Line coverage   : "return 40" never ran → 1/2 lines = 50%
// Branch coverage : only TRUE path taken → 1/2 = 50%
```

```java
String label(boolean cod) {
    return cod ? "COD" : "PREPAID";       // ONE line, TWO branches
}
@Test void cod() { assertEquals("COD", label(true)); }
// Line coverage   : 100% (the single line ran)
// Branch coverage : 50%  (PREPAID path never tested) ← branch catches it
```

| Counter | Counts | Notes |
|---------|--------|-------|
| **Instructions (C0)** | JVM bytecode instructions | Most granular; JaCoCo's default headline |
| **Lines** | Source lines with ≥1 executed instruction | Easy to read; can hide ternaries |
| **Branches (C1)** | `if`/`switch`/ternary/`&&` outcomes | **Stronger** — catches missed paths |
| **Cyclomatic complexity** | Min. paths needed to cover a method | Hints how many tests a method needs |
| **Methods / Classes** | Executed at least once | Coarse sanity check |

Memory trick: **"Lines lie, branches bite."** — line coverage can look 100% while half the logic is untested.

**Easy rule of thumb:** Gate CI on **branch** coverage (plus line), not line alone.

**Interview one-liner:** Line coverage checks lines were executed; branch coverage checks every decision outcome was taken — branch is stricter and the better quality gate.

---

## 3. How JaCoCo works

*Definition:* **JaCoCo** (Java Code Coverage) attaches a **Java agent** to the test JVM that instruments bytecode **on the fly** as classes load, records which probes were hit, and writes them to `jacoco.exec`. A report goal turns that into HTML/XML/CSV.

```
mvn verify
  │
  ├─ jacoco:prepare-agent   → sets argLine = -javaagent:jacocoagent.jar=destfile=target/jacoco.exec
  ├─ surefire:test          → tests run, probes record hits
  ├─ jacoco:report          → target/site/jacoco/index.html (+ jacoco.xml for Sonar)
  └─ jacoco:check           → FAIL build if below thresholds
```

- No source changes; works with any test framework (JUnit, TestNG).
- `jacoco.xml` is what **SonarQube / SonarCloud, Codecov, Coveralls** consume.
- Node/NestJS equivalent: **Istanbul (nyc)** via `jest --coverage` with `coverageThreshold` in `jest.config`.

> ⚠️ If you set your own `<argLine>` in Surefire, you **overwrite** JaCoCo's agent and get **0% coverage**. Use `<argLine>@{argLine} -Xmx1g</argLine>` to append.

**Interview one-liner:** JaCoCo uses a Java agent to instrument bytecode at class-load time, records executed probes into `jacoco.exec`, and generates HTML/XML reports from it.

---

## 4. Maven setup with thresholds

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>verify</phase>
      <goals><goal>report</goal></goals>
    </execution>
    <execution>
      <id>check</id>
      <phase>verify</phase>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
              <limit>
                <counter>BRANCH</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.70</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

- `element`: `BUNDLE` (whole module), `PACKAGE`, `CLASS`, `METHOD`.
- `value`: `COVEREDRATIO`, `MISSEDCOUNT`, `MISSEDRATIO`, ...
- Run `mvn verify` → open `target/site/jacoco/index.html`.
- Integration tests (Failsafe): add `prepare-agent-integration` + `report-integration`, or `merge` the exec files.

**Interview one-liner:** In Maven, `prepare-agent` attaches JaCoCo, `report` builds the HTML/XML, and `check` with `rules/limits` fails the build below your coverage thresholds.

---

## 5. Gradle setup

```kotlin
// build.gradle.kts
plugins { java; jacoco }

jacoco { toolVersion = "0.8.12" }

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)                 // report after every test run
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports { xml.required.set(true); html.required.set(true) }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule { limit { counter = "BRANCH"; minimum = "0.70".toBigDecimal() } }
    }
}

tasks.check { dependsOn(tasks.jacocoTestCoverageVerification) }
```

| | Maven | Gradle |
|---|-------|--------|
| Attach agent | `prepare-agent` goal | Automatic with `jacoco` plugin |
| Report | `jacoco:report` | `jacocoTestReport` |
| Enforce | `jacoco:check` | `jacocoTestCoverageVerification` |
| Output | `target/site/jacoco/` | `build/reports/jacoco/` |

**Interview one-liner:** Gradle's `jacoco` plugin auto-attaches the agent; `jacocoTestReport` makes reports and `jacocoTestCoverageVerification` enforces thresholds when wired into `check`.

---

## 6. Excludes — don't measure what isn't logic

*Definition:* **Excludes** remove generated or trivial code from the report so the percentage reflects real business logic.

```xml
<configuration>
  <excludes>
    <exclude>**/*Application.class</exclude>        <!-- Spring Boot main -->
    <exclude>**/config/**</exclude>                 <!-- @Configuration wiring -->
    <exclude>**/dto/**</exclude>                    <!-- plain data carriers -->
    <exclude>**/generated/**</exclude>              <!-- MapStruct, OpenAPI, protobuf -->
  </excludes>
</configuration>
```

- Excludes match **class files** (`.class`), not `.java` sources.
- **Lombok:** add `lombok.addLombokGeneratedAnnotation = true` in `lombok.config` → JaCoCo auto-ignores generated getters/`equals`/`hashCode` (it filters anything annotated `@Generated`).
- JaCoCo also auto-filters compiler noise: Kotlin/record boilerplate, enum `values()`, try-with-resources synthetic branches.

> ⚠️ Excludes can be abused to **game** the number. Excluding `OrderService` because it's "hard to test" hides exactly the risk coverage should expose. Review exclude lists in PRs.

**Easy rule of thumb:** Exclude **generated & wiring** code; never exclude **business logic**.

**Interview one-liner:** Exclude generated code, DTOs, configs and the main class (and mark Lombok code `@Generated`) so coverage reflects real logic — but never exclude logic just to hit a target.

---

## 7. Coverage ≠ quality (what interviewers really probe)

```java
@Test
void placeOrder() {
    orderService.place(new OrderRequest(1500));   // runs 100% of place()
}                                                  // ...but asserts NOTHING → still "covered"
```

| Myth | Reality |
|------|---------|
| "100% coverage = no bugs" | Code ran; nothing proves the result was checked |
| "Higher is always better" | Chasing 100% → tests for getters, brittle mocks, wasted time |
| "Coverage measures test quality" | It measures **execution**; **mutation testing** measures quality |
| "Untested code is fine if coverage is 80%" | The missing 20% may be the payment/refund path |

**Mutation testing (PIT / pitest):** it mutates your code (`>=` → `>`, `return 0` → `return 1`) and reruns tests. If tests still pass, the mutant **survived** → your tests are weak.

```
Code:   if (orderValue >= 999)      Mutant: if (orderValue > 999)
Tests still green?  → mutant SURVIVED → no test checks the exact boundary 999
```

Practical policy (what I'd say in an interview):
- **70-85%** line, **~70%** branch as a team gate; higher for payments/pricing modules.
- Gate **new code** coverage in SonarQube (e.g. ≥ 80% on the PR diff) instead of forcing legacy to 100%.
- Use the red lines in the HTML report to find **untested risk**, then write meaningful tests.

> Goodhart's law: *"When a measure becomes a target, it ceases to be a good measure."*

**Interview one-liner:** Coverage shows what code ran, not whether it's verified — use it to find gaps, gate new code around 80%, and use mutation testing (PIT) to judge test quality.

---

## Quick Revision Cheat Sheet

- **Coverage** = % of code executed by tests; finds **untested** code, not correctness.
- **Line** vs **Branch**: branch checks every `if`/ternary outcome — "lines lie, branches bite".
- **JaCoCo**: Java agent instruments bytecode on the fly → `jacoco.exec` → HTML/XML.
- **Surefire argLine gotcha**: use `@{argLine}` or coverage becomes 0%.
- **Maven**: `prepare-agent` → `report` → `check` (rules/limits, `COVEREDRATIO`).
- **Gradle**: `jacoco` plugin, `jacocoTestReport`, `jacocoTestCoverageVerification` in `check`.
- **Excludes**: main class, config, DTOs, generated code; Lombok via `@Generated`; never logic.
- **Coverage ≠ quality**: assertion-free tests still count; **PIT mutation testing** measures strength.
- **Policy**: ~80% gate on new code (SonarQube), higher for payment/pricing paths.
