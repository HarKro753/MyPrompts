---
name: test-coverage
description: Test coverage forms, tools, and strategy for Python and TypeScript projects. Use when writing tests, setting up CI coverage gates, adding mutation testing, or deciding how much coverage is enough.
---

# Test Coverage

A complete guide to test coverage: what the forms mean, which tools implement them, and how to think about targets without gaming metrics.

## Core Principle (Martin Fowler)

> Coverage is a tool for finding untested code — not a measure of test quality.

High numbers are easy to reach with low-quality tests. The real signals:
1. Bugs rarely escape into production
2. You can change code without fear

## Coverage Forms — Hierarchy of Confidence

### 1. Statement / Line Coverage
Every executable line runs at least once. The baseline — necessary but not sufficient.

```bash
# Python
coverage run -m pytest
coverage report --show-missing

# JS/TS (Vitest)
vitest run --coverage

# JS/TS (Jest)
jest --coverage
```

### 2. Branch Coverage
Every decision point (if/else, try/except, ternary, `&&`/`||`) must have ALL exits exercised.

```bash
# Python — adds --branch flag
coverage run --branch -m pytest
coverage report --branch --show-missing
coverage html  # visual report: yellow = partial branch
```

```ini
# pyproject.toml
[tool.coverage.run]
branch = true
source = ["src", "backend", "services"]

[tool.coverage.report]
show_missing = true
fail_under = 80  # CI fails below this threshold
```

Branch coverage catches the missed "else path" that statement coverage masks:
```python
def fn(x):
    if x:      # statement coverage: call fn(1) → "covered"
        y = 10 # branch coverage: also need fn(0) → False branch flagged
    return y
```

### 3. Mutation Testing — The Strongest Form
Injects small code mutations (flip `>` to `>=`, delete `not`, change `+` to `-`). If tests still pass = mutant "survived" = your tests don't catch that bug class.

**Mutation score** = killed / total. High line coverage + low mutation score = tests that run code but assert nothing.

```bash
# Python: mutmut
pip install mutmut
mutmut run              # slow; run in CI or overnight
mutmut results          # list survived mutants
mutmut show 42          # what does mutant 42 look like?
mutmut apply 42         # apply to disk for manual inspection

# JS/TS: Stryker
npx stryker run
```

**Where to prioritise mutation testing:**
- Payment / escrow logic
- Authentication and permission checks
- Data transformation pipelines
- Any function whose bugs have high business cost

### 4. Property-Based Testing (Hypothesis / fast-check)
Instead of fixed examples, define invariants and let the framework generate hundreds of random inputs. Finds the minimal failing case via shrinking.

```python
# Python: Hypothesis
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)

@given(st.text())
def test_json_roundtrip(s):
    assert json.loads(json.dumps(s)) == s
```

```ts
// TypeScript: fast-check
import fc from 'fast-check';

test('sort idempotent', () => {
  fc.assert(fc.property(fc.array(fc.integer()), (xs) => {
    expect([...xs].sort()).toEqual([...[...xs].sort()].sort());
  }));
});
```

**Best for:** parsers, serialisers, math functions, encode/decode roundtrips, any function with a provable invariant.

### 5. Contract Testing (Pact)
Consumer writes a contract (requests + expected responses). Provider verifies it can satisfy every contract. Prevents integration failures without full E2E tests.

```bash
npm install @pact-foundation/pact
```

**Best for:** microservices, mobile ↔ backend APIs.

## Python Full Setup

```bash
pip install pytest coverage mutmut hypothesis pytest-cov
```

```ini
# pyproject.toml
[tool.coverage.run]
branch = true
source = ["src"]

[tool.coverage.report]
show_missing = true
fail_under = 80

[tool.pytest.ini_options]
addopts = "--cov=src --cov-report=term-missing --cov-report=html"
```

```bash
# One-liner: run tests + branch coverage
pytest --cov=src --cov-branch --cov-report=term-missing

# Mutation testing (run separately — slow)
mutmut run && mutmut results
```

## TypeScript Full Setup

```bash
# Vitest
npm install -D vitest @vitest/coverage-v8

# vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      branches: 80,
      lines: 80,
      thresholds: { branches: 80, lines: 80 }
    }
  }
});
```

## Coverage Targets

| % | Meaning |
|---|---|
| < 50% | Serious gap |
| 50–70% | Partial — happy paths covered, edges not |
| 70–85% | Reasonable for most business logic |
| 85–95% | Healthy range (Fowler) |
| 100% | Suspicious — optimising for metric, not quality |

## What to Cover First

Priority order for new codebases:
1. **Happy path + error paths** for every public API endpoint
2. **Branch coverage** on all business logic (payment, auth, permissions)
3. **Mutation testing** on highest-risk modules
4. **Property tests** on any data transformation or parsing logic
5. **Contract tests** on inter-service APIs

## CI Integration

```yaml
# GitHub Actions example
- name: Test with coverage
  run: |
    coverage run --branch -m pytest
    coverage report --fail-under=80

- name: Mutation testing (weekly)
  if: github.event_name == 'schedule'
  run: mutmut run && mutmut results
```

## Formal Testing Theory (University / Exam Level)

### Equivalence Class Testing (Äquivalenzklassen)

Group all possible inputs into **equivalence classes** — partitions where the program behaves identically for every value in the class. Write one test case per class.

**Test case notation:** `(input) → (expected output)`

Example: function `isAdult(age: int) → bool`
| Class | Condition | Representative | Expected |
|---|---|---|---|
| EC1 — negative | age < 0 | -1 | error / false |
| EC2 — minor | 0 ≤ age < 18 | 10 | false |
| EC3 — adult | age ≥ 18 | 25 | true |

Three equivalence classes → minimum 3 test cases. **Equivalence class coverage** = satisfied when every class has at least one test case.

**Boundary value analysis** extends this: also test the boundary values (0, 17, 18) since off-by-one errors cluster there.

### Coverage Criteria (Überdeckungskriterien)

Four criteria used in formal exam questions:

| Criterion | German | Satisfied when |
|---|---|---|
| **Equivalence class coverage** | Äquivalenzklassenüberdeckung | Every equivalence class has ≥1 test case |
| **Function coverage** | Funktionsüberdeckung | Every function is called by ≥1 test |
| **Input coverage** | Eingabeüberdeckung | Every possible input appears — usually impossible for unbounded types (all strings) |
| **Output coverage** | Ausgabeüberdeckung | Every possible output is produced (e.g. both `true` and `false` for a boolean function) |

Exam tip: **input coverage** is almost never fully satisfiable for string/integer inputs. **Output coverage** for booleans = easy: ensure both true and false are tested.

### Control Flow Graph + C-Coverage (Strukturorientiertes Testen)

**Control Flow Graph (Kontrollflussgraph):** directed graph where:
- **Nodes** = individual statements / basic blocks
- **Edges** = possible transitions (sequential, branches, loops)

From this graph, three structural coverage criteria:

| Criterion | German | Requirement |
|---|---|---|
| **C₀** | Knotenüberdeckung | Every **node** must be reached (= statement coverage) |
| **C₁** | Zweigüberdeckung | Every **edge/branch** must be traversed (= branch coverage) |
| **C₂** | Pfadüberdeckung | Every **path** from entry to exit must be executed |

**Hierarchy:** C₂ ⊇ C₁ ⊇ C₀ — C₂ implies C₁ implies C₀. But C₂ is exponentially expensive (loops = infinite paths in theory).

**Practical mapping to tools:**
- C₀ → `coverage run` (statement)
- C₁ → `coverage run --branch`
- C₂ → approximated by mutation testing + property-based testing

### Cyclomatic Complexity (Zyklomatische Komplexität)

McCabe's metric — measures the number of linearly independent paths through a function.

```
V(G) = #Edges − #Nodes + 2
```

Alternative formula (easier to compute by hand):
```
V(G) = #Decision points + 1
```
(count every `if`, `while`, `for`, `case`, `&&`, `||` — add 1)

| V(G) | Meaning |
|---|---|
| 1–4 | Simple, low risk |
| 5–10 | Moderate complexity |
| 11–20 | High complexity, hard to test fully |
| > 20 | Very high — refactor strongly advised |

**Why it matters for testing:** V(G) = minimum number of test cases needed for C₁ branch coverage. A function with V(G) = 7 needs at least 7 test cases to cover all branches.

```python
# Example: V(G) calculation
def classify(x, y):          # +0 (entry)
    if x > 0:                # +1
        if y > 0:            # +1
            return "Q1"
        else:
            return "Q2"
    elif x < 0:              # +1
        return "Q3/Q4"
    else:
        return "Y-axis"
# V(G) = 3 decision points + 1 = 4
# Minimum 4 test cases for full branch coverage
```

**Practical rule:** if V(G) > 10, split the function. High cyclomatic complexity = hard to test, hard to understand, high defect probability.

## Key References

- [coverage.py docs](https://coverage.readthedocs.io/en/latest/branch.html) — branch coverage measurement
- [mutmut](https://mutmut.readthedocs.io/en/latest/) — Python mutation testing
- [Hypothesis](https://hypothesis.readthedocs.io/) — property-based testing
- [Stryker](https://stryker-mutator.io/) — JS/TS mutation testing
- [Martin Fowler on Test Coverage](https://martinfowler.com/bliki/TestCoverage.html)
- [Pact](https://docs.pact.io/) — consumer-driven contract testing
