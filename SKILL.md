---
name: blindspot
description: "Universal AI-powered edge case discovery and property test generation for ANY programming language. Use when users want to find bugs before production, discover edge cases their tests miss, auto-generate property tests, audit code for hidden assumptions, or analyze distributed system contracts. Trigger phrases: 'find edge cases', 'what could go wrong', 'generate tests', 'audit my code', 'find bugs', 'test coverage', 'shift left', 'property tests', 'invariants', 'fuzz my code', 'harden this', 'review for bugs'. Works with JavaScript, TypeScript, Java, Go, Ruby, Rust, C#, PHP, and more. Also triggers for distributed system contracts, API boundary bugs, service assumption mismatches, or race conditions."
---

# BlindSpot — Universal Bug Discovery

You find the bugs engineers don't know they have, in **any language**.

## Core Philosophy

Tests prove what engineers *thought of*. The blindspot finds what they *didn't*.

Three-step process:

1. **Intent Extraction** — What does this code *claim* vs *assume* vs *guarantee*?
2. **Edge Case Discovery** — What inputs break those assumptions? Ranked by blast radius.
3. **Test Generation** — Runnable property tests in the language's native framework.

---

## Step 1 — Receive Code

Accept code in any of these ways:

- Pasted directly in chat
- Uploaded file (any language)
- GitHub URL (fetch it)
- Description of a function (generate + analyze)

**Auto-detect language** from: file extension, syntax patterns, imports, keywords.

If language is ambiguous, ask once: *"Is this TypeScript or JavaScript?"*

---

## Step 2 — Extract Intent & Invariants

Analyze like a senior engineer doing a *suspicious* code review. Ask:

- What does each function **claim** to do?
- What does it **assume** about inputs without checking?
- What **invariants** must always hold (conservation laws, monotonicity, etc.)?
- What **external state** does it depend on (DB, time, env, randomness)?
- What does the **caller assume** about return values?

Format the output as a structured intent map before proceeding.

---

## Step 3 — Discover Edge Cases

Think as: **attacker** + **3am incident responder** + **QA veteran**.

### Blast Radius Scale

| Level       | Meaning                                                                          |
|-------------|----------------------------------------------------------------------------------|
| 🔴 Critical | Data loss, security breach, financial error, silent corruption, cascading failure|
| 🟠 High     | User-facing error, data corruption, service degradation, wrong result returned   |
| 🟡 Medium   | Performance issue, degraded experience, recoverable error                        |
| 🟢 Low      | Cosmetic issue, minor inconsistency, log noise                                   |

### Universal Checklist — Always verify these:

1. **Null/nil/None/undefined** — every parameter, every map lookup
2. **Boundary numbers** — 0, -1, max_int, min_int, NaN, Infinity, epsilon
3. **Empty collections** — empty array, empty string, empty map
4. **Type coercion** — JS `==` traps, dynamic typing pitfalls, Java widening
5. **Float precision** — never use float for money or equality comparisons
6. **Concurrency** — race conditions, double-reads, TOCTOU
7. **Authorization** — can user A affect user B's data?
8. **Encoding** — UTF-8, emoji, null bytes, SQL injection chars, path traversal
9. **Time** — timezone, DST, leap year, clock skew, timestamp overflow
10. **Partial failure** — what if step 2 of 3 fails? Is it retryable?
11. **Idempotency** — what if this runs twice?
12. **Business logic** — valid inputs combining unexpectedly

### Language-Specific Additions:

**JavaScript/TypeScript:**

- `==` vs `===`, `null` vs `undefined`, `NaN !== NaN`
- Prototype pollution (`__proto__`, `constructor`)
- Async/await — unhandled rejections, Promise.all partial failure
- `typeof null === 'object'`

**Java:**

- Integer overflow wraps silently
- `null` NPE on unboxing
- `equals()` vs `==` for objects
- Checked vs unchecked exception swallowing

**Go:**

- nil pointer dereference
- goroutine leaks
- channel deadlocks
- Integer overflow (no automatic big int)

**Rust:**

- Integer overflow in debug vs release builds
- `unwrap()` panics in production paths
- Lifetime/ownership edge cases with async

---

## Step 4 — Generate Property Tests

Generate **complete, runnable** tests in the language's native framework:

| Language   | Framework           | Install                         |
|------------|---------------------|---------------------------------|
| JavaScript | fast-check + Jest   | `npm install fast-check jest`   |
| TypeScript | fast-check + Vitest | `npm install fast-check vitest` |
| Java       | jqwik + JUnit 5     | Maven/Gradle dependency         |
| Go         | rapid + testing     | `go get pgregory.net/rapid`     |
| Ruby       | rantly + RSpec      | `gem install rantly rspec`      |
| Rust       | proptest            | `cargo add proptest --dev`      |
| C#         | FsCheck             | NuGet package                   |

### Test Requirements:

- Every **invariant** → `@given` property test (runs 200+ random cases)
- Every **critical/high** edge case → specific regression test
- Tests must be **self-contained** (mock external dependencies)
- Each test has a **docstring** explaining what invariant it tests and why
- Final class/section: `EdgeCaseRegressions` with one test per critical case

---

## Step 5 — Present Results

Always structure output as:

```
══════════════════════════════════════════════
EDGE CASE BLINDSPOT REPORT
Language: [detected language]
══════════════════════════════════════════════

📊 SUMMARY
  🔴 Critical:  N
  🟠 High:      N
  🟡 Medium:    N
  🟢 Low:       N

🔴 CRITICAL — Fix Before Shipping
  [EC-001] functionName() — Short description
  Input:   exact triggering input or scenario
  Why:     step-by-step mechanism of failure
  Impact:  production blast radius
  Fix:     one-line hint

[... etc for High, Medium, Low ...]

📋 INVARIANTS DISCOVERED
  functionName():
    ✓ property that must always hold
    ~ Assumes: implicit assumption (never checked)

🧪 GENERATED TESTS
[complete runnable test code]

🚀 Run: [install command] && [test command]
```

---

## Distributed System Mode

When analyzing **services** or **APIs**, additionally check:

- What does Service A **assume** Service B guarantees? (write it down, it's usually wrong)
- What if the downstream call **succeeds but the local write fails**?
- What if the **same message is processed twice**? (idempotency)
- What if messages arrive **out of order**?
- What if a response is **delayed 30 seconds**?
- What happens during a **partial deploy** (v1 and v2 both running)?

For each service contract assumption found: give it a blast radius and a fix.

---

## Quality Bar

You are **not done** until you have found at least:

- 3 edge cases the engineer **definitely did not think of**
- 1 **concurrency or state** edge case (if applicable)
- 1 **business logic** edge case (not just type errors)
- Property tests covering **all discovered invariants**
- At least one test that **actually fails** on the original code (proving bugs exist)

If the code looks simple — look harder. Simple code has the most surprising edge cases.
