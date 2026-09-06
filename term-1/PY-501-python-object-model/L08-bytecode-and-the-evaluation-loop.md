# PY-501 · Lesson 08 — Bytecode and the Evaluation Loop

**Estimated study time:** 4 hours
**Prerequisites:** L01–L07
**Version warning:** bytecode changes between minor releases. Every disassembly here is
illustrative; **run it on your interpreter** and expect differences. Explaining the
differences is part of the work.

---

## 1. Orientation

Documentation runs out. Sooner or later you will have a question the docs do not answer:
does this expression evaluate its operands left to right? Does this `finally` run before or
after the return value is computed? Does the comprehension allocate? Is this attribute
access being cached?

Bytecode answers all of these definitively, because it *is* the semantics as implemented.
Learning to read it converts a class of unanswerable questions into a two-line experiment.

This lesson teaches enough disassembly to be self-sufficient, and enough of the evaluation
loop's architecture to understand why CPython performs the way it does — which is the
foundation for PY-602.

## 2. Theory

### 2.1 The pipeline

```
source ──tokenize──► tokens ──parse──► AST ──symtable──► symbol tables
                                        │
                                    compile
                                        ▼
                                   code object  ──► eval loop ──► result
```

Each stage is exposed:

```python
import ast, dis, symtable
src = "x = [i*2 for i in range(3)]"
print(ast.dump(ast.parse(src), indent=2))
code = compile(src, "<s>", "exec")
dis.dis(code)
```

A **code object** (`types.CodeType`) is immutable and holds: the bytecode string, the
constants, the names, the variable-name tables (L07 §2.2), the line-number table, the
stack-size requirement, and flags. It is *not* a function: a function object is a code
object plus `__globals__`, `__defaults__`, `__closure__`, and a name.

Nested functions, comprehensions (pre-3.12), lambdas, and class bodies each compile to
their own code object, stored in the parent's `co_consts`.

### 2.2 The stack machine

CPython's evaluation is a stack machine. Almost every instruction pops operands and pushes
a result.

```python
>>> dis.dis("a + b * c")
  0  LOAD_NAME   a
  2  LOAD_NAME   b
  4  LOAD_NAME   c
  6  BINARY_OP   5 (*)
  8  BINARY_OP   0 (+)
```

Read it as reverse Polish. Two things are immediately visible: operands are evaluated
strictly **left to right**, and precedence is resolved at compile time (nothing at runtime
knows about precedence).

The instruction families you need:

| Family | Examples | Notes |
|---|---|---|
| Load/store locals | `LOAD_FAST`, `STORE_FAST` | array index; no dict lookup |
| Load/store globals | `LOAD_GLOBAL`, `STORE_GLOBAL` | dict lookup in globals then builtins |
| Load/store cells | `LOAD_DEREF`, `STORE_DEREF` | closures (L07 §2.4) |
| Attributes | `LOAD_ATTR`, `STORE_ATTR` | invokes L03's algorithm |
| Subscripts | `BINARY_SUBSCR`, `STORE_SUBSCR` | `__getitem__`/`__setitem__` |
| Calls | `CALL`, `CALL_KW`, `PUSH_NULL` | calling convention changed in 3.11+ |
| Control flow | `POP_JUMP_IF_FALSE`, `JUMP_BACKWARD` | |
| Iteration | `GET_ITER`, `FOR_ITER`, `END_FOR` | |
| Exceptions | `PUSH_EXC_INFO`, `CHECK_EXC_MATCH`, `RERAISE` | zero-cost model, §2.5 |
| Building | `BUILD_LIST`, `LIST_APPEND`, `BUILD_MAP` | |

`dis.dis` with `adaptive=True` (3.11+) or `dis.dis(f, show_caches=True)` reveals the
inline caches, which is how you observe specialization (§2.6).

### 2.3 Reading a real function

```python
def f(xs):
    total = 0
    for x in xs:
        total += x
    return total

dis.dis(f)
```

Roughly (3.12):

```
  RESUME                   0
  LOAD_CONST               1 (0)
  STORE_FAST               1 (total)
  LOAD_FAST                0 (xs)
  GET_ITER
  FOR_ITER                 to <exit>
  STORE_FAST               2 (x)
  LOAD_FAST                1 (total)
  LOAD_FAST                2 (x)
  BINARY_OP               13 (+=)
  STORE_FAST               1 (total)
  JUMP_BACKWARD            to FOR_ITER
<exit>
  END_FOR
  LOAD_FAST                1 (total)
  RETURN_VALUE
```

Now use it to answer real questions:

- **Is `total += x` one operation?** No — `BINARY_OP` then `STORE_FAST`. The unconditional
  store of L01 §2.5 is visible.
- **Does the loop re-look-up `xs` each iteration?** No — `GET_ITER` once.
- **How much would `sum(xs)` save?** One `LOAD_GLOBAL` plus a C-level loop instead of
  ~6 bytecodes per element. Now you can predict the ~3–5× speedup rather than measuring
  blind (though you must still measure — PY-602 L01).

### 2.4 Compile-time work

The compiler does more than translate. Observing what it does statically explains a lot of
performance folklore.

**Constant folding and constant tuples:**

```python
dis.dis("x = 2 * 3 + 1")        # LOAD_CONST 7
dis.dis("x in (1, 2, 3)")       # LOAD_CONST (1, 2, 3) — the tuple is a constant
dis.dis("x in [1, 2, 3]")       # BUILD_LIST at runtime... or, on newer versions, a frozenset for `in`
```

This is the mechanism behind the advice "use a tuple/frozenset literal for membership
tests": for a `set` literal in an `in` test the compiler emits a `frozenset` constant, so
there is no per-call construction.

**Docstrings, `__doc__`, and dead code.** `if False:` blocks are eliminated. Assertions
disappear entirely under `-O` — which is why `assert` must never be used for validation of
untrusted input, only for internal invariants.

**Comprehension vs loop.** Disassemble both forms of building a list. Pre-3.12 the
comprehension is a separate code object and a function call (`MAKE_FUNCTION` + `CALL`);
3.12+ inlines it (PEP 709). Which version you are on changes the answer to "is a
comprehension faster than a loop", and being able to *demonstrate* that rather than assert
it is the graduate-level move.

### 2.5 Zero-cost exceptions

Before 3.11, entering a `try` block pushed a block onto a stack — a small cost paid on
every entry, whether or not an exception occurred. Since 3.11, CPython uses an **exception
table**: a side table mapping instruction ranges to handler offsets. Entering `try` costs
*nothing*; the cost is paid only when an exception is actually raised, at which point the
interpreter looks up the current instruction in the table.

Consequences for how you write code:

- `try/except` around a hot loop is now free when nothing raises. The old advice to avoid
  `try` in hot paths is obsolete.
- EAFP ("easier to ask forgiveness than permission") is now cheaper relative to LBYL
  ("look before you leap") *for the success path*, and more expensive when the exception
  actually fires. So `try: d[k] except KeyError:` beats `if k in d:` when hits dominate,
  and loses when misses dominate. This is measurable and worth measuring (PY-602 L01).

View the table with `dis.dis(f, show_caches=True)`, which prints `ExceptionTable` entries.

### 2.6 The specializing adaptive interpreter

PEP 659 (3.11+) introduced *quickening*: after a code object has been executed a few times,
CPython rewrites certain instructions in place into specialized forms based on the types
actually observed.

`LOAD_ATTR` on an instance whose class has not changed becomes
`LOAD_ATTR_INSTANCE_VALUE`, which reads a known offset instead of running L03's full
algorithm. `BINARY_OP` on two ints becomes `BINARY_OP_ADD_INT`. Each specialized form
carries a *guard*; when the guard fails the instruction **deoptimizes** back to the generic
form.

Practical implications, and these are the ones that matter:

- **Type stability is a performance property.** A function called with `int` a million times
  is faster than one alternating `int` and `float`, not because of interpretation overhead
  but because specialization holds. A "polymorphic" hot loop is a real cost.
- **Class mutation invalidates caches.** Monkey-patching a class at runtime invalidates the
  type version tag and deoptimizes every specialized attribute access on it.
- **Warm-up exists.** Microbenchmarks must run enough iterations for specialization to
  kick in, or they measure the unspecialized interpreter. This is the single most common
  microbenchmark error on modern CPython.

3.13 added a copy-and-patch JIT (PEP 744) behind a build flag; its effect on typical code
at time of writing is modest, and it is off by default in standard builds. Check
`sys._is_gil_enabled()` and your build's configure flags rather than assuming.

### 2.7 Where the time goes

A rough model of interpreter cost per operation, useful for reasoning before measuring:

- `LOAD_FAST` — array index. Essentially free.
- `LOAD_GLOBAL` — dict lookup ×2 (globals, builtins), with an inline cache since 3.11. This
  is why the "bind globals to locals in a hot loop" trick used to matter and matters much
  less now — but *still* matters when the global is a module attribute chain.
- `LOAD_ATTR` — L03's algorithm, cached when the shape is stable.
- `CALL` — frame creation. Since 3.11, Python-to-Python calls are *inlined* into the same
  C stack frame, making calls substantially cheaper than they were.
- `BINARY_OP` on objects — a dispatch through the type's slots, plus allocation of the
  result. Allocation is usually the dominant cost for small-object arithmetic.

The recurring theme: **allocation and dispatch dominate; instruction count is secondary.**
This is why NumPy is fast (one dispatch for a million elements) and why `__slots__` helps
(fewer allocations per instance).

## 3. Construction: a bytecode-level analyzer

Build a tool that answers structural questions about functions.

**Version 1 — count opcodes.**

```python
import dis
from collections import Counter

def opcount(fn) -> Counter[str]:
    return Counter(i.opname for i in dis.get_instructions(fn))
```

Immediately useful for comparing two implementations without running them.

**Version 2 — find global lookups in loops.** A classic, real optimization target:

```python
def globals_in_loops(fn) -> list[tuple[int, str]]:
    instrs = list(dis.get_instructions(fn))
    # loop bodies: everything between a FOR_ITER/backward jump target and the jump
    back_edges = [i for i in instrs if i.opname == "JUMP_BACKWARD"]
    ranges = [(i.argval, i.offset) for i in back_edges]     # (target, source)
    out = []
    for i in instrs:
        if i.opname in ("LOAD_GLOBAL",) and any(lo <= i.offset <= hi for lo, hi in ranges):
            out.append((i.offset, i.argval))
    return out
```

Run it on your own code. Then — critically — *measure* whether hoisting the globals it
finds actually helps on your interpreter version. Since 3.11's inline caches, often it does
not, and discovering that yourself is worth more than being told.

**Version 3 — a semantics oracle.** Write `explain(expr: str) -> str` that disassembles an
expression and reports: evaluation order of subexpressions, which names are local vs
global, whether anything was constant-folded, and how many allocations (`BUILD_*`,
`CALL` to constructors) occur. Use it to settle at least five questions you previously
guessed at.

**Version 4 — specialization observer.** Run a function 10,000 times, then disassemble
with `adaptive=True` and report which instructions specialized. Then run it with mixed
argument types and show the deoptimization. This is the exercise that makes §2.6 real.

## 4. Failure modes

- **Optimizing from bytecode counts alone.** Fewer instructions ≠ faster; a single
  `CALL` into C can beat twenty `LOAD_FAST`s.
- **Benchmarking without warm-up.** §2.6. Measures the wrong interpreter.
- **Assuming bytecode is stable.** It is explicitly not part of the language spec. Never
  ship code that parses bytecode without a version guard, and expect to rewrite it every
  release.
- **Using `assert` for validation.** Removed under `-O`.
- **Concluding from CPython about "Python".** PyPy's semantics are the same; its
  performance model is entirely different. Any claim of the form "Python is slow at X"
  should be qualified by implementation.
- **Micro-optimizing before measuring the algorithm.** CS-621's constant factors argument:
  an O(n²) loop with a beautifully optimized body is still O(n²).

## 5. Exercises

### Warm-up (30 min)

**W1.** Disassemble `a and b or c` and explain the jump structure. Then explain why
`a and b or c` is not equivalent to a conditional expression when `b` is falsy.

**W2.** Disassemble `x = a[i] = b` (chained assignment) and state the order of the stores.
Predict first.

**W3.** Disassemble a function with a `try/finally` containing a `return` in both. Explain
the observed behaviour (L01 §... and L10) from the bytecode alone.

### Core (2.5 h)

**C1 — Evaluation order catalogue** *(Problem Set 3.)* Determine, from bytecode, the exact
evaluation order for: `f(a(), b(), c=d())`, `{a(): b(), c(): d()}`, `x[a()] = b()`,
`a() if b() else c()`, `[a() for _ in b()]`, `a() + b() * c()`, and an f-string with three
interpolations. Verify each with a side-effecting function that records call order.
Produce a reference table. Note which of these the Language Reference actually guarantees
versus which are CPython implementation detail — this distinction is the assessed part.

**C2 — EAFP vs LBYL, measured.** Build a benchmark comparing `try/except KeyError` against
`if k in d` across hit rates from 0% to 100%, on 3.10 (pre-zero-cost) if you can get it and
on 3.12+. Plot the crossover. Explain the shape of the curve from §2.5. State the rule you
would put in a style guide.

**C3 — Specialization.** Complete version 4 of §3. Then write a function that is
deliberately *megamorphic* (called with ten different types) and one that is monomorphic,
with identical work. Measure both. Report the ratio and explain it. Then relate this to a
design guideline about `Union` types in hot paths.

**C4 — A bytecode-level linter.** Write a check that flags: attribute chains of depth ≥3
inside loops, `LOAD_GLOBAL` of a name that is never reassigned (hoistable), and
list-building in a loop where a comprehension would compile better. Run it on a real
project. For each finding, measure whether the fix actually helps. Report your true-positive
rate — the interesting result is likely to be low, and explaining *why* is the deliverable.

### Challenge

**X1.** Write a tiny bytecode interpreter in Python for a subset of the instruction set
(loads, stores, binary ops, jumps, calls, `FOR_ITER`, `RETURN_VALUE`) that can execute a
real, simple function's code object correctly. Compare your output to CPython's on 20
functions. This is the single best way to internalize the stack machine and it feeds
directly into CS-641 L07.

**X2.** Read `Python/ceval.c`'s main loop and `Python/bytecodes.c` (the DSL from which the
loop is generated in 3.12+). Produce a two-page written account of: how the dispatch works
(computed gotos), how inline caches are stored, and how specialization and deoptimization
are implemented. Identify three things you did not understand and what you would read next.

## 6. Self-check

1. What is in a code object that is not in a function object, and vice versa?
2. What does the bytecode for `a + b * c` prove about evaluation order?
3. What is quickening, and what invalidates a specialized instruction?
4. Explain zero-cost exceptions and one style consequence.
5. Why must a microbenchmark warm up on CPython 3.11+?
6. Why is `assert` unsafe for input validation?
7. Name three things the compiler does before the eval loop ever runs.
8. Why does allocation usually dominate instruction count?

## 7. Primary sources

- `dis` module documentation — read the full opcode list once.
- PEP 659 — Specializing Adaptive Interpreter. The design section is excellent.
- PEP 669 (low-impact monitoring), PEP 744 (JIT), PEP 709 (inlined comprehensions).
- Shaw, *CPython Internals*, chapters on the compiler and the evaluation loop.
- Brandt Bucher & Mark Shannon's talks on the 3.11/3.12 interpreter work.

---

**Previous:** [L07](L07-scopes-frames-and-closures.md) · **Next:**
[L09 — Memory: Reference Counting, Cycles, and Finalization](L09-memory-refcounting-gc-finalization.md)
