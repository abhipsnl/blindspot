# BlindSpot

**AI-powered edge case discovery and property test generation.**

Finds the bugs your test suite doesn’t know to look for — in any language.

Proven on real production code: found confirmed bugs in `psf/requests` (52k★),
`encode/httpx` (13k★), and `pallets/flask` (68k★).

-----

## Install in Claude (2 minutes)

### Step 1 — Enable Code Execution

Go to **[Settings → Capabilities](https://claude.ai/settings/capabilities)**
and turn on **Code execution and file creation**.

### Step 2 — Open Skills

Go to **[Customize → Skills](https://claude.ai/customize/skills)**.

### Step 3 — Upload BlindSpot

Click **”+”** → **”+ Create skill”** → **“Upload a skill”**.
Upload the `blindspot.skill` file (it’s a ZIP — rename to `.zip` if your browser requires it).

### Step 4 — Enable It

Toggle BlindSpot **on** in your Skills list.

That’s it. No API key needed in Claude. BlindSpot is now active.

-----

## Use It

Just paste your code and ask naturally:

```
"Find edge cases in this function"
"What could go wrong here?"
"Audit this for bugs"
"Generate property tests for this"
```

Claude will automatically run all three steps:

1. Extract intent and implicit assumptions
1. Discover edge cases ranked by blast radius (🔴 Critical → 🟢 Low)
1. Generate runnable property tests in your language’s native framework

### Supported Languages

Python, JavaScript, TypeScript, Java, Go, Ruby, Rust, C#, PHP, Kotlin, Swift — auto-detected from file extension or syntax.

-----

## Run the CLI Locally (Optional)

If you want to run BlindSpot from your terminal on your own codebase:

```bash
# Install
pip install anthropic hypothesis pytest
export ANTHROPIC_API_KEY=sk-ant-your-key-here

# Run on any file
python scripts/blindspot.py payment.py
python scripts/blindspot.py userService.ts
python scripts/blindspot.py AuthService.java
python scripts/blindspot.py handler.go

# Auto-run tests after generation (Python)
python scripts/blindspot.py payment.py --run-tests
```

**Outputs:**

- `payment_oracle_tests.py` — runnable property tests
- `payment_oracle_report.txt` — full report with blast-radius rankings

**Run the tests:**

```bash
pip install hypothesis pytest
pytest payment_oracle_tests.py -v
```

Red tests = real bugs. Fix the code, rerun until green.

-----

## Real-World Proof

BlindSpot audited three of the most trusted Python libraries and found confirmed bugs:

|Repo           |Bug                                                 |Severity  |
|---------------|----------------------------------------------------|----------|
|`psf/requests` |Non-ASCII passwords crash with `UnicodeEncodeError` |🔴 Critical|
|`psf/requests` |Digest auth `KeyError` on malformed server challenge|🔴 Critical|
|`encode/httpx` |`unquote("")` raises `IndexError`                   |🔴 Critical|
|`pallets/flask`|`SESSION_COOKIE_PATH=""` silently overridden        |🟠 High    |

Audit tests are in `examples/real-world-audits/`. Run them yourself:

```bash
pip install requests httpx flask hypothesis pytest
pytest examples/real-world-audits/ -v
```

-----

## What BlindSpot Does That Tests Don’t

Your unit tests prove what you **thought of**.
BlindSpot finds what you **didn’t** — the implicit assumptions, boundary conditions,
and interaction effects that only appear in production.

It’s not a smarter test runner. It’s a different kind of thinking applied to your code.

-----

## Files in This Package

```
blindspot/
├── SKILL.md                          ← Claude skill instructions
├── README.md                         ← This file
├── scripts/
│   ├── blindspot.py                  ← CLI tool (run locally)
│   └── requirements.txt              ← pip install -r requirements.txt
├── references/
│   └── cli-guide.md                  ← Full CLI options
└── examples/
    ├── python/payment_service.py     ← Buggy example (Python)
    ├── javascript/cart_service.js    ← Buggy example (JavaScript)
    ├── java/InventoryService.java    ← Buggy example (Java)
    └── real-world-audits/            ← Tests proving real bugs in famous repos
        ├── test_requests_blindspot.py
        ├── test_httpx_blindspot.py
        └── test_flask_blindspot.py
```