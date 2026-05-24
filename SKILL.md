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

## Foundational Rule — Read Source Before Claiming Values

**This rule applies to EVERY step below, not just Step 5's verification
checklist.** It exists because the failure mode it addresses
(confabulating specific values about the code) shows up in Step 1's intent
map, Step 2's invariant list, Step 3's edge-case descriptions, Step 4's
generated tests, AND Step 5's audit — not just at the very end.

**The rule:** any time you state a specific value about the code under
analysis — thresholds, defaults, constants, env-var names, line numbers,
function signatures, SQL clauses, formula terms — open the file and Read
the relevant lines in this session FIRST, then cite `path:line` in
whatever artifact you produce (intent map, finding, generated test).

Do not answer from:
- Memory of the file from earlier in the session (long sessions drift)
- Training data ("functions named like this usually have signature X")
- Inference ("a sensible developer would have set this constant to Y")
- Repeated patterns in other codebases ("most retry configs use 0.5
  as the threshold")

### Why this is a skill-wide rule, not a Step 5 check

Step 5a's `Quantitative claims are measured, not estimated` catches
confabulated numbers IN FINDINGS. But the skill produces values in
multiple places before findings exist:

- **Step 2 intent map** says "this function assumes input length ≤ 200" —
  but the code says 500. The downstream finding is built on a wrong
  premise.
- **Step 3 edge-case description** says "fails when budget exceeds 60s" —
  but the budget cap is actually 30s. The trigger described in the
  finding doesn't exist.
- **Step 4 generated test** asserts `assert_eq!(result, 0.25)` against a
  default that's actually 0.30. The test fails on correct code; the user
  spends 20 minutes debugging the test before realizing the SKILL was
  wrong.

All three are caught by reading the source before writing the artifact,
not after.

### What counts as a "value" that needs a source

- Numeric thresholds, defaults, caps, timeouts, retry counts, batch sizes
- Window lengths in days/weeks/seconds, intervals
- Env var names, config keys, table names, column names
- SQL `WHERE`/`ORDER BY`/`LIMIT` clauses described in prose
- Function signatures, parameter names, return types
- "X happens when Y" claims about runtime behavior of the code (NOT
  about general project process — those don't need file citations)
- Test counts ("19 tests pass") — verify by running, not by recall
- Formulas — verify the EXACT numerator and denominator terms, not just
  the shape

### Minimum bar per claim

For each value you're about to put in any artifact (intent map, finding,
test, invariant description, report summary):

1. **Have you Read the source file in this session?** If no — Read it
   now, even if you "remember" the value from earlier work or from a
   sibling file.
2. **Cite the path and line.** Format: `module/file.rs:42-58`. The user
   can verify in one click; you can re-find it for the next claim.
3. **If the value depends on a config parameter the caller passes**,
   trace to the caller and cite that line too. Don't say "the window is
   `days`" without showing what value `days` actually gets.

### When estimation is OK

Order-of-magnitude or "roughly N" claims about external systems (LLM
latency, network round trips, browser timeouts, database query costs in
generic terms) — these are fine without file citations because they're
not from the code under analysis. Be explicit when you estimate:
"roughly ~200ms" not "exactly 200ms."

### Concrete failure that motivated this rule

When asked to explain a velocity headline ratio in a codebase the skill
had been analyzing for hours, the skill confidently cited grade
thresholds (`>= 10 = A`, `>= 5 = B`, etc.) and a "rolling 4 week" window.
Reading the actual file showed the thresholds were `>= 3 = A`, `>= 2 = B`,
`>= 1 = C`, `>= 0.5 = D` and the window was 90 days (driven by the
caller's parameter). About half the specifics were wrong. Every number
came from memory + inference, not from re-reading the file.

The skill had Read the file in an earlier turn — but the values had
drifted in memory across the session, and the dashboard caller's
parameter had never been read at all. The fix is: read again, cite
again, even when "I'm pretty sure I remember."

### Correction protocol when a previous claim was unverified

Same rule as Step 5a withdrawals: open the file, read the actual value,
post a correction with the cited line. Don't double down. Don't hedge
with "I think it might be." Read, cite, correct. A correction citing
`path:line` rebuilds trust faster than three sentences of hedging.

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

## Step 5 — Verify Every Finding Before Writing It Down

This step exists because a previous run of this skill shipped findings that
didn't survive a 60-second self-audit. The skill is graded on **truth**, not
**volume**. Inflating a report with shaky claims poisons the user's trust in
the real findings.

Verification runs in **three sub-passes** — truth checklist (5a), adversarial
falsification (5b), confidence calibration (5c). All three must complete
before Step 6 (presenting the report). Any finding that fails 5a or 5b is
withdrawn; every surviving finding carries a confidence label from 5c.

### Step 5a — Truth checklist (per finding)

Before listing any finding, run it through this checklist. **Withdraw the
finding silently if any answer is "no" or "I'm not sure."** Don't write it
down and then explain why you're withdrawing it — that leaves doubt in the
artifact and burns the user's attention.

1. **Code-grounded.** Have you Read the relevant lines in the actual source
   file in this session? (Not "I think I remember" — actually opened the
   file. Cite `path:line` in the report.)
2. **Mechanism is concrete.** Can you write the exact input → exact wrong
   behavior in one sentence, without hedging? If you wrote "could", "may",
   "might possibly" — the finding isn't ready.
3. **Quantitative claims are measured, not estimated.** If you said "0.1ms
   per call" or "200ns per lookup" or "50 KB per second" — did you measure,
   or did you confabulate a plausible number to justify the finding? **Never
   put a number in the report you didn't measure or cite a source for.**
   Either run the measurement, find a citation, or remove the number.
4. **Severity matches the mechanism.** "Critical" means production breaks for
   real users, not "could theoretically happen in a contrived scenario." If
   the only way to hit it requires the LLM to hallucinate AND the operator
   to ignore three warnings, it's not Critical.
5. **Not a theoretical class.** The skill is "edge case discovery", not
   "computer science seminar." A finding that says "in principle, X could
   happen" without a concrete trigger is a seminar finding. Withdraw.
6. **Distinct from another finding.** If two findings describe the same
   mechanism from different angles, collapse them. Two cross-referencing
   findings inflate the count without informing.

### What to do when a finding fails the checklist

Withdraw it. Silently. Do **not**:
- Write the finding then add "actually, withdrawn" — the user reads both,
  cost is doubled
- Downgrade Critical → High → Medium → Low to keep it in (the bar is
  truth-or-out, not severity)
- Add caveats like "could be theoretical, but worth noting" — that's the
  seminar finding pattern

What to do **instead**:
- If you found something but couldn't verify it, file a one-line "needs
  verification" note in a separate `🔬 UNVERIFIED HUNCHES` section. The
  user can grep them later. They are NOT counted in the summary numbers.
- Be willing to ship a report with **0 findings in a severity bucket**.
  "Critical: 0" is the correct answer when the code genuinely has no
  critical bugs. Don't promote a Medium to Critical to fill the slot.

### Self-audit before pressing send (still 5a)

After drafting the report but BEFORE moving to 5b, do this pass:

> "Read my own report cold. For every finding, ask: would I bet $100 of my
> own money this is real and the severity is right? If no, withdraw or
> downgrade. For every number I wrote, ask: did I measure this or guess?
> If guess, remove the number."

A report with 3 audited findings is more valuable than 12 mixed ones. The
user has finite attention; spend it on truth.

### Step 5b — Adversarial falsification

This step exists because the first pass is the same LLM optimizing for
"find bugs"; it has a systemic bias toward false positives. The fix is
explicitly running a SECOND pass that optimizes for "defend why these are
NOT bugs." Findings that survive both passes ship; findings that don't are
withdrawn.

For every finding that passed 5a, run this prompt **on yourself** before
keeping it:

> "Switch roles. I am the engineer who wrote this code and I think the
> blindspot report is wrong about finding [EC-N]. Argue the strongest case
> for why this is NOT a bug. What input or precondition is in the report
> impossible? What guard already exists upstream that the report missed?
> What's the user's actual workflow that makes this scenario unreachable?
> Be willing to call the original finding wrong if the defense is stronger
> than the accusation."

Three possible outcomes:

1. **Defense wins.** The finding is wrong. Withdraw silently — same rules as
   5a. Don't write "I considered this but withdrew" — write nothing.
2. **Defense partially wins.** The mechanism is real but the severity was
   inflated, or the trigger is narrower than originally described. Downgrade
   severity, tighten the trigger description, keep the finding.
3. **Defense loses.** The original finding stands. Carry it to 5c with the
   defense's strongest counterargument noted in a one-line `Defense
   considered:` field, so the user can see you actually tried to falsify it.

A finding that hasn't survived 5b is not a finding. Don't ship a report that
skipped this step — it's the single highest-leverage filter for false
positives. Skipping it gets you back to the pre-skill error rate.

#### What to look for as the defender

These are the patterns that most often "defeat" a falsely-flagged finding:

- **Upstream guard.** The function the report flagged is only ever called
  after a validator that rules out the bad input. Look for the actual
  call sites with Grep, not from memory.
- **Type-system invariant.** The "what if it's null" finding is bogus when
  the type is `Foo` not `Option<Foo>`. Check the actual type.
- **Already-handled in a different layer.** The "race condition" finding is
  bogus when there's a serializing queue between the two writers. Read the
  caller, not just the function.
- **Wrong language semantics.** "Integer overflow" is bogus in Python (big
  ints) but real in Rust (debug panics, release wraps). Match the finding
  to the language's actual behavior.
- **Wrong runtime semantics.** "Mutable default argument" is real in Python
  function defs but NOT in dataclass field types (`field(default_factory=...)`
  exists for exactly this reason). Match the finding to the actual idiom in
  use.

### Step 5c — Confidence calibration

Every finding that survived 5a + 5b ships with a **confidence label** the
user can triage on. Without this, the user has to treat every finding
identically, which is wrong — some you proved with a runnable test, some you
reasoned from code, some you're 70% sure of.

Three confidence levels, with HARD requirements per level:

| Confidence    | Requirement                                                                |
|---------------|----------------------------------------------------------------------------|
| **Proven**    | You generated AND ran a test that demonstrates the bug (test FAILS on the   |
|               | unfixed code; would PASS after the proposed fix). Include the test in the  |
|               | report. This is the only "I am 100% sure" label.                            |
| **High**      | You Read the load-bearing lines with cited `path:line`, traced the data    |
|               | flow, and the mechanism is concrete. No execution, but no plausible        |
|               | falsifier survived 5b either.                                              |
| **Medium**    | The mechanism is plausible from the code you Read, but you couldn't       |
|               | trace every preconditon (e.g. caller behavior wasn't in scope). 5b raised  |
|               | a defense you couldn't fully refute.                                       |

If you can't honestly assign at least **Medium** confidence, the finding
goes in the `🔬 UNVERIFIED HUNCHES` section instead of the main report. It
is not counted in the summary numbers.

**Severity and confidence are independent axes.** A Critical bug at Medium
confidence is still worth listing — the user might choose to investigate
even at 70% sure. But the LABEL is not optional. Reports without confidence
labels per finding are incomplete.

The format in Step 6 carries the confidence on the same line as the finding
ID so it can't be omitted by accident.

---

## Step 6 — Present Results

Always structure output as:

```
══════════════════════════════════════════════
EDGE CASE BLINDSPOT REPORT
Language: [detected language]
══════════════════════════════════════════════

📊 SUMMARY
  🔴 Critical:  N    (Proven: a / High: b / Medium: c)
  🟠 High:      N    (Proven: a / High: b / Medium: c)
  🟡 Medium:    N    (Proven: a / High: b / Medium: c)
  🟢 Low:       N    (Proven: a / High: b / Medium: c)

  Pass 5a withdrawals: M findings dropped (truth checklist)
  Pass 5b withdrawals: K findings dropped (adversarial falsification)

🔴 CRITICAL — Fix Before Shipping
  [EC-001] [Proven|High|Medium] functionName() — Short description
  Code:    path/to/file.rs:42-58  (cite the lines you actually Read)
  Input:   exact triggering input or scenario
  Why:     step-by-step mechanism of failure
  Impact:  production blast radius
  Fix:     one-line hint
  Defense considered:  one line summarizing 5b's strongest counter (omit
                       only when Proven — for those the test IS the
                       defense and it failed)
  Test:    name of the test that proves it (Proven-only; omit otherwise)

[... etc for High, Medium, Low ...]

Every finding line MUST carry both severity emoji AND confidence label
[Proven|High|Medium] in brackets immediately after the ID. A finding
without a confidence label is malformed and goes back to Step 5c.

📋 INVARIANTS DISCOVERED
  functionName():
    ✓ property that must always hold
    ~ Assumes: implicit assumption (never checked)

🧪 GENERATED TESTS
[complete runnable test code]

🚀 Run: [install command] && [test command]

🔬 UNVERIFIED HUNCHES (optional)
  Things I noticed but couldn't ground in the code in this session.
  Not counted in the summary numbers above. Treat as "worth a 5-minute
  look", not findings.
  - hunch 1
  - hunch 2
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

**Truth over volume.** A report with 0 findings on solid code is the correct
output. A report with 6 inflated findings is worse than useless — it teaches
the user to ignore future reports.

**Effort over count.** You are not done until you have:

- **Actually Read** (with the file-reading tool, not from memory) every file
  you cite in a finding. Cite `path:line` so the user can verify in one click.
- **Run** the universal checklist against the code, not just listed it. For
  every item, write down "checked, no issue found" or "checked, finding X
  filed" — even if only in your scratchpad. Skipping items quietly = blindspot.
- **Generated runnable tests** for every Critical or High finding. Tests that
  don't compile or that pass against the buggy code are worse than no tests.
- **Tried to falsify your own findings.** For each one, ask "what would make
  this NOT a bug?" If the falsifier is plausible, withdraw or downgrade.

**Things that are NOT a quality signal:**

- Finding count. "3 minimum" is a former rule, removed because it caused
  inflation. Sometimes the right count is 0; sometimes it's 12.
- Severity distribution. Don't promote a Medium to Critical to make the
  report "look serious." Severity reflects the mechanism's blast radius, not
  the operator's blood pressure.
- Volume of generated tests. One test that exercises the load-bearing
  invariant beats ten tests that exercise type-system trivialities.

**If the code looks simple, look harder — but also consider that it may
genuinely be simple.** Sometimes a function with three branches really does
have three branches. Don't invent a fourth.
