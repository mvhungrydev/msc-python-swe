# SE-511 · Lesson 08 — Static Analysis Beyond Types

**Estimated study time:** 3.5 hours
**Prerequisites:** L06, L07, PY-501 L08

---

## 1. Orientation

Type checkers answer one family of question. There is another family they cannot touch:

- Is this SQL string built by concatenation with user input?
- Does every code path that acquires this lock release it?
- Did someone add a new API endpoint without the `@authenticated` decorator?
- Is this `datetime` naive when our convention says all datetimes are UTC-aware?
- Does this module import from a layer it is not allowed to depend on?

Each is a project-specific invariant, mechanically checkable, and invisible to every
off-the-shelf tool. **Writing your own checks is a normal engineering activity**, and the
tooling is a few hours of work. Most teams never do it, and consequently rediscover the
same defect in code review a hundred times.

## 2. Theory

### 2.1 The static analysis landscape

Ordered by what they can prove:

| Technique | Sees | Cost | Example |
|---|---|---|---|
| **Lexical / regex** | Text | trivial | banned words, TODO format |
| **AST pattern matching** | Syntax structure | low | `except: pass`, `assert` in prod |
| **Control-flow analysis** | Paths through a function | medium | unreachable code, missing return |
| **Data-flow analysis** | How values move | medium-high | uninitialized use, taint tracking |
| **Type checking** | Types | high (needs annotations) | mypy, pyright |
| **Abstract interpretation** | Over-approximation of all runs | high | value ranges, nullability |
| **Symbolic execution** | Path constraints | very high | input reaching a crash |
| **Model checking / proof** | All behaviours of a model | highest | FM-751 |

Every technique in this table trades **precision** against **cost** and against **false
positives**. The single most important practical fact about static analysis:

> **A check with a 10% false-positive rate will be disabled within a month.**

Developers do not tolerate noise from a gate. This means your first design question for any
new rule is not "what can I detect?" but "what can I detect with near-zero false
positives?" A narrow, boring rule that is always right beats a clever rule that is right
80% of the time — the clever rule has negative value because it also erodes trust in the
other rules.

### 2.2 Ruff and the ecosystem

`ruff` has effectively consolidated Python linting: it reimplements most of `flake8`,
`isort`, `pyupgrade`, `pydocstyle`, `bandit`, `pylint` (partially), and more, fast enough to
run on save.

A defensible starting configuration and *why* each family is on:

```toml
[tool.ruff.lint]
select = [
  "E", "W",    # pycodestyle — mostly formatting; keep, it is free
  "F",         # pyflakes — unused imports/variables, undefined names. Non-negotiable.
  "I",         # isort — import order; removes a whole class of review comment
  "N",         # pep8-naming
  "UP",        # pyupgrade — modernizes syntax as you raise the floor version
  "B",         # bugbear — real bug patterns (mutable defaults, B008, B012 return-in-finally)
  "A",         # builtins shadowing — `list = ...` is a real bug source
  "C4",        # comprehension simplifications
  "DTZ",       # naive datetime usage — catches a genuine class of production defect
  "S",         # bandit security rules
  "SIM",       # simplifications
  "TID",       # tidy imports — banned-api and relative-import control
  "PTH",       # prefer pathlib
  "RUF",       # ruff-specific
]
ignore = ["E501"]        # line length: leave to the formatter
```

`B` (flake8-bugbear) and `DTZ` earn their place more than the rest: bugbear's `B006`
(mutable default), `B008` (function call in default), and `B012` (break/return in
`finally`) are exactly PY-501's failure modes, caught mechanically.

The **formatter** (`ruff format`, or `black`) is a separate concern and should be
unconditional and unarguable. The value is not the style; it is the elimination of style
from code review entirely.

### 2.3 Writing your own AST rule

Python's `ast` module makes this genuinely easy. The pattern:

```python
import ast
from dataclasses import dataclass

@dataclass(frozen=True)
class Finding:
    line: int
    col: int
    code: str
    message: str

class NaiveDatetimeChecker(ast.NodeVisitor):
    """Flag datetime.now()/utcnow()/fromtimestamp() without tz."""

    def __init__(self) -> None:
        self.findings: list[Finding] = []

    def visit_Call(self, node: ast.Call) -> None:
        f = node.func
        if isinstance(f, ast.Attribute) and f.attr in {"now", "fromtimestamp"}:
            has_tz = any(k.arg == "tz" for k in node.keywords) or len(node.args) > (
                0 if f.attr == "now" else 1
            )
            if not has_tz:
                self.findings.append(Finding(
                    node.lineno, node.col_offset, "TZ001",
                    f"datetime.{f.attr}() without tzinfo; use datetime.now(UTC)",
                ))
        if isinstance(f, ast.Attribute) and f.attr == "utcnow":
            self.findings.append(Finding(
                node.lineno, node.col_offset, "TZ002",
                "datetime.utcnow() is deprecated and returns a naive datetime",
            ))
        self.generic_visit(node)
```

Run it over a tree:

```python
def check(path: Path) -> list[Finding]:
    tree = ast.parse(path.read_text(), filename=str(path))
    v = NaiveDatetimeChecker()
    v.visit(tree)
    return v.findings
```

Thirty lines, and it eliminates a defect class permanently. Note what it does *not* do:
resolve whether `f.value` is actually the `datetime` class (it could be any object with a
`now` method). That imprecision is a **false-positive source**, and handling it properly
requires name resolution — which is exercise C2 and the point at which you learn why real
linters are hard.

### 2.4 Architectural rules

The highest-value custom checks are usually about **structure**, not statements:

**Layer/import rules.** "The domain layer may not import from the infrastructure layer."
This is the single most valuable automated check in a layered codebase, because layering
erodes one import at a time and no review catches it.

```python
LAYERS = {"domain": 0, "application": 1, "adapters": 2, "entrypoints": 3}

def check_imports(module: str, imports: list[str]) -> list[str]:
    mine = layer_of(module)
    return [imp for imp in imports if layer_of(imp) < mine and not allowed(mine, imp)]
```

Wait — that is backwards, and deliberately so: think about it. Lower layers must not import
higher ones. Getting the direction right requires stating the dependency rule precisely,
which is exercise C3 and is exactly the thinking SE-521 L05 formalizes. Tools:
`import-linter` does this off the shelf with a declarative contract file, and is worth
adopting rather than writing.

**Decorator requirements.** "Every function in `api/routes/` must have exactly one of
`@public` or `@authenticated`." Twenty lines of AST, and it closes an entire class of
security defect.

**Banned APIs.** `ruff`'s `TID251` (`banned-api`) does this declaratively:

```toml
[tool.ruff.lint.flake8-tidy-imports.banned-api]
"datetime.datetime.utcnow".msg = "Use datetime.now(UTC)"
"requests".msg = "Use the shared httpx client in mypkg.http"
"pickle".msg = "Unsafe for untrusted data; use msgspec or json"
```

**Naming conventions with semantics.** "Any function named `*_async` must be `async def`."
"Any Protocol class must end in `Protocol` or `Port`."

### 2.5 Security-relevant analysis

`bandit` (via ruff's `S` rules) catches the standard list: `subprocess` with `shell=True`,
`yaml.load` without `SafeLoader`, hardcoded passwords, weak hashes, `eval`/`exec`,
`assert` in production code, insecure temp files, `pickle` on untrusted data,
`random` for security purposes.

Its limitation is that it is **pattern-based, not taint-tracking**. It cannot tell whether
the string passed to `subprocess` is user-controlled. True taint analysis (does data flow
from an untrusted *source* to a dangerous *sink* without passing a *sanitizer*?) requires
inter-procedural data flow, which is what CodeQL and Semgrep Pro do and what `bandit` does
not.

`semgrep` sits usefully in between: pattern matching with metavariables, easier to write
than AST visitors, with some flow sensitivity:

```yaml
rules:
  - id: raw-sql-fstring
    pattern: $CURSOR.execute(f"...")
    message: f-string in SQL execute — use parameters
    severity: ERROR
    languages: [python]
```

For most teams the right stack is: `ruff` (fast, always on) + a handful of custom AST rules
(project invariants) + `semgrep` for anything with metavariables + dependency scanning
(L09). Full taint analysis is worth it above a certain risk level and not below it.

### 2.6 Making checks land

A rule nobody follows is worse than no rule. What actually works:

- **Autofix where possible.** `ruff --fix` and formatters change behaviour because they
  cost nothing to comply with.
- **Fail the build, not the review.** A rule enforced by a human is enforced
  inconsistently and generates resentment. A rule enforced by CI is just a fact.
- **Introduce with a ratchet.** New violations fail; existing ones are baselined
  (`ruff` per-file ignores generated once, `# noqa` with codes, or a baseline file). Then
  the number only goes down.
- **Every rule needs a message that says what to do instead.** "TZ001: naive datetime" is
  bad. "TZ001: use `datetime.now(UTC)`; this codebase requires aware datetimes (see
  ADR-014)" is good.
- **Every rule needs an owner and a rationale.** Rules with no rationale get deleted the
  first time they are inconvenient, and they should be.

## 3. Construction: a project-invariant checker

Build a small linter for your own codebase's rules.

**Step 1 — collect the rules from review history.** Go through 50 recent pull request
comments. Which ones are the *same comment again*? Those are your rules. This is a real
exercise and it produces a surprisingly short, surprisingly actionable list.

**Step 2 — for each, decide the cheapest technique that works.**

| Rule | Technique |
|---|---|
| No `print()` in library code | AST: `Call` with `Name id='print'` — but exclude `__main__` blocks |
| All routes have an auth decorator | AST: `FunctionDef` in a path, check `decorator_list` |
| Domain must not import infrastructure | import graph — use `import-linter` |
| No naive datetimes | AST, §2.3 |
| No `Decimal(float)` | AST: `Call` to `Decimal` with a `Constant` float arg |
| Public functions have docstrings | `ast.get_docstring` on non-underscore `FunctionDef` |

**Step 3 — implement three of them** as `ast.NodeVisitor`s in a single module with a
common `Finding` type and a CLI that walks the tree, reports, and exits non-zero.

**Step 4 — measure the false-positive rate.** Run on the whole codebase. Manually review
every finding. Compute the rate. **If it is above ~2%, the rule is not ready.** Narrow it
until it is, even if that means missing cases — a precise rule that catches 60% of
instances is worth more than a noisy rule that catches 95%.

**Step 5 — write tests for the linter.** A linter is code. Test it with a fixture directory
of files that should and should not trigger each rule, including the tricky negatives you
discovered in step 4.

**Step 6 — wire it into CI with a baseline** and a documented rationale per rule.

## 4. Failure modes

- **Noisy rules.** §2.1. The whole system loses credibility.
- **Rules without rationale.** Deleted at the first inconvenience, correctly.
- **Big-bang enablement.** 4,000 findings, everyone adds a global ignore.
- **`# noqa` without a code.** Suppresses everything on the line, forever. Enable
  `RUF100` (unused noqa) and require codes.
- **Style rules debated in review.** A formatter ends this; nothing else does.
- **Trusting `bandit` as security assurance.** Pattern matching without taint tracking.
- **Writing an AST rule where `import-linter` or `semgrep` already does it.** Check first.
- **Checking the AST when you need the symbol table.** Name resolution is not free; §2.3.
- **Linting generated code.** Exclude it; the findings are not actionable.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write an AST visitor that finds every bare `except:` and every `except Exception:
pass`. Run it on a real project.

**W2.** Use `ast.dump` on five constructs and identify the node types you would match for a
rule about each.

**W3.** Configure `ruff` with the `select` list from §2.2 on a real project. Report the
findings by rule family and how many are real.

### Core (2 h)

**C1 — Your project's linter.** Complete §3, steps 1–6, with three rules. Deliverable: the
linter, its tests, the false-positive analysis with the actual rate, the CI wiring, and the
rule rationale document.

**C2 — Name resolution.** Improve the naive-datetime checker of §2.3 so that it only fires
when the receiver actually resolves to `datetime.datetime` — handling `import datetime`,
`from datetime import datetime`, `import datetime as dt`, and aliasing. Report what you
still cannot resolve and why (the honest answer involves dynamic assignment and is a good
illustration of why sound static analysis in Python is hard).

**C3 — Layer enforcement.** Define the layers of a real codebase, write the dependency
contract with `import-linter`, and run it. Report every violation. For each, decide: is it
a genuine architecture violation, or is the layer definition wrong? The second outcome is
common and is itself a finding.

**C4 — Rule from review history.** Do step 1 for real: read 50 PR comments, extract the
repeated ones, and report the list. Implement the most frequent one. Then estimate the
review-hours per year it saves.

### Challenge

**X1.** Implement a small **taint analysis** for one source/sink pair: `flask.request.*` as
source, `cursor.execute` as sink, with `sqlalchemy.text` parameters as a sanitizer.
Intra-procedural only. Then explain precisely what inter-procedural analysis would require
and why it is exponentially harder. Compare your results with `semgrep` on the same code.

**X2.** Write an `ast`-based tool that computes, per function, a *cognitive complexity*
score (Campbell's metric, not cyclomatic) and correlates it with the file's defect history
from `git log`. Report whether the correlation holds in your codebase. Publish the negative
result if that is what you find — it is more interesting than the positive one.

## 6. Self-check

1. Rank the static analysis techniques by power and cost, with an example of each.
2. Why does a 10% false-positive rate kill a rule?
3. Give three project-specific invariants that no off-the-shelf tool checks.
4. What is the difference between pattern-based security scanning and taint analysis?
5. Why is the AST insufficient for the naive-datetime rule, and what is needed?
6. What is a ratchet and why does it work where a target does not?
7. What makes a good lint message?
8. Why is the formatter's value not the style it produces?

## 7. Primary sources

- `ast` module documentation and the Green Tree Snakes guide.
- `ruff` rule documentation — read the `B` (bugbear) rules in full; each is a real bug
  pattern with an explanation.
- Campbell, "Cognitive Complexity: A New Way of Measuring Understandability" (SonarSource,
  2018) — read it critically.
- Bessey et al., "A Few Billion Lines of Code Later: Using Static Analysis to Find Bugs in
  the Real World" (CACM 2010). The best paper written on why static analysis adoption
  fails, and it is mostly not technical.

---

**Previous:** [L07](L07-generics-variance-protocols.md) · **Next:**
[L09 — Packaging, Dependencies, and Reproducibility](L09-packaging-and-reproducibility.md)
