# CS-641 · Lesson 07 — Effects, Evaluation Strategies, and Monads

**Estimated study time:** 4.5 hours
**Prerequisites:** L02, L04, L06

---

## 1. Orientation

"A monad is a monoid in the category of endofunctors" is a true sentence that has taught
nobody anything. Here is the useful version:

> **A monad is a design pattern for sequencing computations that carry extra structure —
> failure, state, nondeterminism, I/O, asynchrony — where each step depends on the result of
> the previous one.**

You already use several. `Optional` chaining, `async`/`await`, list comprehensions, and
exception handling are all instances of the same shape, and noticing that is worth more than
any amount of category theory.

The wider subject is **effects**: what it means for a computation to *do something* rather
than merely produce a value, and how a type system can track that. This is where L02's
observation — that adding mutable references changed every rule — gets its proper treatment.

## 2. Theory

### 2.1 Purity and what it buys

A **pure** function: its result depends only on its arguments, and it has no observable effect
beyond returning.

What purity buys, concretely:

- **Referential transparency.** `f(x)` can be replaced by its value anywhere. Which means:
  memoization is always safe, common subexpressions can be eliminated, and reordering is safe.
- **Local reasoning.** You can understand `f` from `f` alone.
- **Trivial testing.** No setup, no doubles, no order dependence (SE-511 L05 §2.4's small
  tier).
- **Free parallelism.** No shared state, so no synchronization (PY-601 L04 §2.7's design
  hierarchy).
- **Equational reasoning.** You can substitute equals for equals — which is what makes the
  free theorems of L04 §2.7 and formal proofs possible at all.

What is impure: mutation, I/O, exceptions, randomness, the clock, and non-determinism.

**The purity distinction is a spectrum in practice**, and the engineering move is not "be
pure" but *push effects to the edges*: a large pure core, a thin impure shell (SE-521 L03
§2.6). Every technique in this lesson is machinery for making that split explicit.

### 2.2 Evaluation strategies, revisited

L01 §2.5 introduced them; here is what they do to effects.

**Call-by-value.** Arguments evaluated once, before the call. Effects happen in a
**predictable order**, which is why every language with pervasive effects chose it.

**Call-by-name.** Argument substituted unevaluated, re-evaluated at each use. Effects happen
*at each use*, possibly zero times, possibly many.

**Call-by-need (lazy).** Evaluated at most once, on first use. Effects happen *once, at an
unpredictable time*.

That last point is why **Haskell is pure**: with lazy evaluation, uncontrolled side effects
would be impossible to reason about, so effects had to be moved into the type system. Laziness
did not merely permit purity — it forced it. That is a genuinely instructive case of a
language's features determining each other.

Laziness buys: infinite structures, avoided work, and a kind of modularity Hughes argues for
in "Why Functional Programming Matters" (you can separate generation from selection —
`take 10 (sort xs)` need not sort everything). It costs: space leaks that are hard to
diagnose, unpredictable timing, and much harder reasoning about performance.

Python is call-by-value with explicit laziness available (generators, PY-502 L05).

### 2.3 The monad shape

Three things:

1. A type constructor `M[A]` — a computation producing an A, with some structure.
2. `unit : A → M[A]` — wrap a plain value. (`return`, `pure`, `just`.)
3. `bind : M[A] → (A → M[B]) → M[B]` — run a computation, feed its result to the next.
   (`>>=`, `flatMap`, `then`, `and_then`.)

Plus three laws, which are what make it a *monad* rather than merely two functions:

```
left identity:   bind(unit(a), f)      = f(a)
right identity:  bind(m, unit)         = m
associativity:   bind(bind(m,f), g)    = bind(m, λx. bind(f(x), g))
```

The laws say: wrapping and immediately unwrapping does nothing, and **the grouping of
sequenced steps does not matter**. The third is the important one — it is what lets you
refactor `a; (b; c)` into `(a; b); c`, which is the thing you do constantly without thinking.

**The instances you already know:**

| Monad | `M[A]` | What it sequences |
|---|---|---|
| Maybe/Option | `A \| None` | computations that may be absent |
| Result/Either | `Ok[A] \| Err[E]` | computations that may fail with a reason |
| List | `list[A]` | non-deterministic computations (many results) |
| State | `S → (A, S)` | computations threading state |
| Reader | `E → A` | computations reading an environment |
| Writer | `(A, Log)` | computations accumulating output |
| IO | a description of an effect | effectful computations |
| Future/Promise | eventual `A` | asynchronous computations |
| Parser | `str → (A, str) \| Fail` | consuming input |

In Python:

```python
# Maybe, by hand
def bind(m: T | None, f: Callable[[T], U | None]) -> U | None:
    return None if m is None else f(m)

user   = find_user(uid)
addr   = bind(user, lambda u: u.address)
city   = bind(addr, lambda a: a.city)

# which is exactly what optional chaining is sugar for, and what this reads as:
city = find_user(uid)?.address?.city        # (not Python syntax, but the idea)
```

`await` is `bind` for the Future monad. A list comprehension with multiple `for` clauses is
`bind` for the list monad. `try/except` is close to `bind` for Either. **You have been writing
monadic code all along; the abstraction just names the shape.**

### 2.4 Why the abstraction earns its keep

If you already write these by hand, what does naming the pattern buy?

- **Generic combinators.** `sequence : list[M[A]] → M[list[A]]`, `traverse`, `map_m`. Written
  once, work for every monad. `asyncio.gather` is `sequence` for Future; `all(...)` over
  Optionals is close to it for Maybe.
- **The laws license refactoring.** Associativity is what lets you extract a middle chunk of a
  chain into a helper without changing behaviour. You rely on this daily and would notice
  immediately if it failed.
- **Do-notation / async syntax.** `async`/`await` is do-notation specialized to one monad. A
  language with general do-notation gets that syntax for *every* monad, which is why Haskell's
  Maybe chains read like imperative code.
- **Effect tracking in types.** `IO String` is visibly different from `String`. You cannot
  accidentally perform I/O in a pure function, because the type says so — this is the strong
  version of "push effects to the edges", enforced.

And the honest limitations:

- **Monads do not compose.** `Maybe[Future[A]]` is not automatically a monad. Monad
  transformers exist and are notoriously awkward; algebraic effects (§2.7) are the modern
  answer.
- **Everything becomes monadic.** Once one function returns `M[A]`, its callers must too. This
  is "colouring" (§2.6) and it is the real cost.
- **Python lacks the syntax.** Without do-notation, monadic code in Python is a chain of
  lambdas, which is worse than the imperative version. This is why the pattern is rare in
  Python outside `Result` types.

### 2.5 The Result monad, practically

The one that genuinely earns its place in Python.

```python
@dataclass(frozen=True)
class Ok[T]:  value: T
@dataclass(frozen=True)
class Err[E]: error: E

Result = Ok[T] | Err[E]

def and_then[T, U, E](r: Result[T, E], f: Callable[[T], Result[U, E]]) -> Result[U, E]:
    match r:
        case Ok(v):  return f(v)
        case Err(_): return r
```

Compared with exceptions:

| | Exceptions | Result |
|---|---|---|
| Visible in the signature | no | **yes** |
| Enforced handling | no | yes, via exhaustive matching |
| Cost on the happy path | ~zero (PY-501 L08 §2.5) | an allocation and a branch |
| Ergonomics in Python | excellent | poor without do-notation |
| Composes with stdlib | yes | no |
| Distant errors | propagate automatically | must be threaded explicitly |

**When Result is worth it in Python:** a library boundary where callers must handle failures;
a parser or validator returning many errors; a pipeline where errors are data to be routed
rather than propagated (PY-502 L05 §2.6's error policy). **When it is not:** ordinary
application code, where exceptions are idiomatic, integrate with the standard library, and are
cheaper.

Rust chose Result and made it ergonomic with `?`; Go chose explicit error returns without the
monadic structure; Python chose exceptions. Each choice is coherent, and mixing them in one
codebase is the failure mode.

### 2.6 The colouring problem

Bob Nystrom's "What Color is Your Function?" names a real cost:

Once a function is `async`, every caller must be `async`, all the way up. The same is true of
`Result`-returning functions, of `IO`-typed functions in Haskell, and of anything that changes
the *shape* of the return type. You end up with two disjoint worlds and awkward bridges
between them (`asyncio.run`, `unwrap`, `unsafePerformIO`).

The responses:

- **Accept it**, and keep the effectful part small.
- **Provide bridges** — `asyncio.run`, `to_thread`, `.unwrap()`.
- **Sans-I/O design** (PY-502 L06 §2.6): write the *logic* colourless — a pure state machine —
  and provide thin sync and async shells. **This is the best available answer in Python**, and
  it is why `h11` is structured as it is.
- **Effect systems / algebraic effects** — the research answer (§2.7).
- **Green threads** — Go's goroutines and Java's virtual threads make all functions the same
  colour by making blocking cheap. This is a genuine solution and it is why those languages do
  not have this problem.

The engineering guidance: **notice when you are about to colour a function, and ask whether
the effect can be pushed outward instead.** Most of the time it can, and the sans-I/O split is
the concrete technique.

### 2.7 Effect systems and algebraic effects

The research direction, worth knowing because it is arriving.

**Effect types.** Annotate a function with the effects it may perform:
`f : Int -> Int ! {IO, Exn}`. The type system tracks and enforces them. Koka, Eff, and
Frank do this; Java's checked exceptions are a crude, unloved instance.

**Algebraic effects and handlers.** Effects are *operations* whose meaning is supplied by a
handler, dynamically:

```
effect Ask : String
effect Log : String -> Unit

handle (body) with
  | Ask       k -> k("value from config")
  | Log msg   k -> print(msg); k(())
```

The handler receives a **continuation** `k`, so it decides what happens next: resume,
resume many times (non-determinism), or not at all (exceptions). This subsumes exceptions,
state, generators, async, and backtracking in one mechanism, and — crucially — **effects
compose**, which monad transformers do not.

Python has one instance of this already: **generators are a limited effect handler.** `yield`
is an operation whose handling is decided by the driver; `send` resumes the continuation
(PY-502 L06 §2.2). That is why you could build an event loop from generators (PY-601 L05 X1) —
you were writing an effect handler for an async effect.

OCaml 5 shipped effect handlers in 2022 and used them to implement its concurrency library.
This is likely where mainstream languages are going, and understanding generators as effect
handlers is the bridge.

### 2.8 What this means for Python

Concrete takeaways:

- **Push effects to the edges.** Pure core, thin shell. Testable (SE-511 L05), parallelizable
  (PY-601 L04), and reasonable-about.
- **Type your effects informally.** A function that does I/O should be visibly named and
  positioned; if you cannot tell from the signature, the design is hiding something.
- **`Result` at boundaries, exceptions inside.** A defensible convention if you write it down.
- **Sans-I/O for protocols.** The colouring solution that actually works.
- **Generators are effect handlers.** This reframing makes `contextlib`, `asyncio`, and
  coroutine-based libraries obvious rather than mysterious.
- **Do not import Haskell wholesale.** Monad transformers in Python are worse than the
  imperative code they replace. Take the *ideas* — purity, explicit effects, the pure core —
  and leave the encoding.

## 3. Construction: effects in your language

Continue from L06.

**Stage 1 — the purity audit.** Take your interpreter. Which parts are pure? Which touch the
store, raise, or read the clock? Draw the boundary. Report the ratio of pure to impure code
and whether the impure part is at the edge.

**Stage 2 — Result in your language.** Add `Result[T, E]` as an ADT with `and_then`, `map`,
`or_else`, and `unwrap_or`. Then rewrite your type checker to return `Result[Type, TypeError]`
instead of raising, so it can report *multiple* errors. Report what that cost: how much
threading, how much readability lost, and whether multiple-error reporting was worth it
(L02 §2.7).

**Stage 3 — the State monad, by hand.** Implement `State[S, A] = S → (A, S)` with `unit` and
`bind`. Use it to thread the store through your evaluator *without* an explicit store
parameter on every function. Compare with the explicit-threading version from L02 §2.4 on:
lines changed, readability, and how easy it is to forget to thread it (the monad makes it
impossible, which is the point).

**Stage 4 — verify the laws.** Property-test the three monad laws for Maybe, Result, List, and
your State monad (SE-511 L04 §2.2, pattern 4 — algebraic laws). Then deliberately break
associativity in one implementation and confirm the test catches it.

**Stage 5 — generic combinators.** Implement `sequence`, `traverse`, and `map_m` generically
over any monad (a Protocol with `unit` and `bind`). Verify they work for all four of your
monads. This is the payoff of the abstraction, demonstrated.

**Stage 6 — effect types.** Add a simple effect system to your language: annotate functions
with the effects they may perform (`!{IO}`, `!{State}`, `!{Exn}`), check that a function's body
only performs declared effects, and check that a pure function is called only where purity is
expected. Report what it caught and what it made annoying.

**Stage 7 — generators as effect handlers.** Implement a small effect-handler mechanism using
generators: an operation `yield`s a request, and a handler `send`s back a result. Implement
handlers for: reading configuration (Reader), logging (Writer), non-determinism (resume the
continuation multiple times — this is the hard and interesting one), and exceptions (do not
resume). Report which were natural and which fought the generator protocol.

**Stage 8 — the colouring experiment.** Write one non-trivial component of your interpreter
twice: once colourless (a pure state machine driven by a caller), once with the effect inlined.
Then write both a synchronous and an asynchronous driver for the colourless version. Report the
line counts and what the colourless version cost.

## 4. Failure modes

- **"Monad" used as a mystique rather than a shape.** If you cannot state `unit` and `bind` for
  it, you are not talking about a monad.
- **Importing monad transformers into Python.** Worse than the imperative code.
- **`Result` everywhere in application code.** Exceptions are idiomatic; use them.
- **Mixing `Result` and exceptions without a stated boundary.**
- **Colouring a function unnecessarily.** Push the effect outward.
- **Effects hidden in a "pure-looking" function.** A cache write, a log line, a clock read —
  each one breaks referential transparency, and the second-order cost is that memoization and
  reordering are no longer safe.
- **Laziness with side effects.** Unpredictable timing, and space leaks.
- **Assuming monads compose.** They do not.
- **Not verifying the laws** for a hand-written monad. An implementation that violates
  associativity will produce bugs that look like anything but a monad law violation.
- **Ignoring that generators are effect handlers**, and therefore reimplementing coroutine
  machinery badly.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write `unit` and `bind` for Maybe, List, and Future (as `asyncio` uses it). Identify
the standard-library operation that corresponds to `bind` in each.

**W2.** Verify the three monad laws for Maybe by hand, then as a property test.

**W3.** Take a function in your codebase that is "almost pure" and identify the effect that
spoils it. Push it out and report what changed at the call sites.

### Core (3 h)

**C1 — Result and State.** Complete §3 stages 2–4. Deliverable: `Result` with combinators, the
type checker rewritten to report multiple errors, the State monad threading the store, the law
property tests, and the deliberate-break demonstration.

**C2 — Generic combinators.** Complete §3 stage 5. Deliverable: `sequence`, `traverse`,
`map_m` over a monad Protocol, working for four monads, with tests. Then write 300 words on
what this abstraction buys in Python specifically, and whether you would use it.

**C3 — Effect types.** Complete §3 stage 6. Deliverable: the effect annotations, the checker,
five programs it correctly rejects, and an honest report of the annotation burden.

**C4 — Generators as handlers.** Complete §3 stage 7. Deliverable: the mechanism and four
handlers, with a report on the non-determinism handler (resuming a continuation multiple
times) — Python's generators cannot do this directly, and working out why, and what you would
need instead, is the assessed part.

### Challenge

**X1.** Read Nystrom's "What Color is Your Function?" and Kiselyov & Ishii on effect handlers.
Complete §3 stage 8, then write 1,200 words: state the colouring problem precisely, evaluate
the five responses of §2.6 for Python specifically, and recommend one with the argument you
would use on a team. Include what green threads would change, and why Python is unlikely to
get them.

**X2.** Implement a full algebraic effect system in your language: effect declarations,
operations, handlers with access to the continuation, and resumption (once, many times, or
never). Then implement exceptions, state, generators, and non-determinism *on top of it*, with
no primitive support for any of them. Report which was hardest and what your implementation
cannot do. This is a substantial project and it is the most direct way to understand where
language design is heading.

## 6. Self-check

1. State what purity buys, with five concrete consequences.
2. Why did laziness force Haskell to be pure?
3. Give the three parts of a monad and the three laws, and say which law licenses which
   everyday refactoring.
4. Name six monads you already use and the operation that is `bind` in each.
5. Give three things the abstraction buys and three limitations.
6. Compare `Result` and exceptions on six dimensions, and say when each is right in Python.
7. State the colouring problem and the five responses, naming the best one for Python.
8. Explain why generators are effect handlers, and what Python's cannot do.

## 7. Primary sources

- Wadler, "Monads for Functional Programming" (1995) — the readable introduction, written for
  programmers.
- Moggi, "Notions of Computation and Monads" (1991) — the origin.
- Hughes, "Why Functional Programming Matters" (1990) — the modularity argument for laziness
  and higher-order functions. Short and excellent.
- Nystrom, "What Color is Your Function?" (2015).
- Plotkin & Pretnar, "Handlers of Algebraic Effects" (ESOP 2009); the Koka and OCaml 5
  documentation for the modern practice.
- Peyton Jones, "Tackling the Awkward Squad" (2001) — how Haskell actually does I/O,
  concurrency, and exceptions.

---

**Previous:** [L06](L06-adts-and-expression-problem.md) · **Next:**
[L08 — Type Systems in Practice: Gradual, Dependent, Linear](L08-type-systems-in-practice.md)
