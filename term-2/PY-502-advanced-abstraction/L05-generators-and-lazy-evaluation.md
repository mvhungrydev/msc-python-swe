# PY-502 · Lesson 05 — Generators and Lazy Evaluation

**Estimated study time:** 3.5 hours
**Prerequisites:** PY-501 L06, L07, L09

---

## 1. Orientation

```python
rows = (parse(line) for line in open("10gb.csv"))
valid = (r for r in rows if r.amount > 0)
total = sum(r.amount for r in valid)
```

Constant memory over a 10 GB file, three lines, no framework. That is the case for
laziness, and it is a strong one.

Now the same code with a subtle defect:

```python
rows = (parse(line) for line in open("10gb.csv"))
n = sum(1 for _ in rows)
total = sum(r.amount for r in rows)     # 0 — rows is exhausted
```

And another:

```python
def read(path):
    with open(path) as f:
        yield from f            # file closes when the generator is exhausted OR collected
                                # — and if the caller abandons it, when? (see §2.4)
```

Laziness moves *when* things happen, and everything that depends on timing — resource
lifetime, exception location, memory retention — moves with it. This lesson is about
getting that right.

## 2. Theory

### 2.1 What a generator is

A function containing `yield` compiles to a code object with the `CO_GENERATOR` flag;
calling it does not execute the body but constructs a **generator object** holding a frame.
`next()` resumes the frame at the saved instruction pointer; `yield` suspends it, saving the
instruction pointer and the value stack.

Consequences you must be able to derive:

- **Nothing runs until the first `next()`.** Argument validation written at the top of a
  generator function does not run at call time — it runs at first iteration, possibly far
  away, possibly never. This is a real bug pattern; the fix is a non-generator wrapper:

```python
def read_rows(path: str) -> Iterator[Row]:
    p = Path(path)                     # validation runs at call time
    if not p.is_file():
        raise FileNotFoundError(path)
    return _read_rows(p)               # returns the generator

def _read_rows(p: Path) -> Iterator[Row]: ...
```

- **The frame stays alive while suspended**, holding every local. A suspended generator in a
  long-lived structure retains its whole working set (PY-501 L09 §2.3).
- **A generator is an iterator, not an iterable-with-restart.** One pass. `iter(gen) is gen`.

### 2.2 The generator protocol, all four operations

```python
gen.__next__()          # resume; StopIteration when the body returns
gen.send(value)         # resume, making `yield` evaluate to value
gen.throw(exc)          # resume by raising exc at the yield point
gen.close()             # throw GeneratorExit at the yield point
```

`send` is what makes a generator a *coroutine* in the original sense — a two-way channel.
`x = yield y` sends `y` out and receives the next `send` argument as `x`. Note that a fresh
generator must be advanced to the first `yield` before `send` can deliver anything
(`gen.send(None)` or `next(gen)`); this is the "priming" step, and forgetting it is the
standard error.

`throw` and `close` are how cleanup works, and they are the subject of §2.4.

`return value` inside a generator sets `StopIteration.value`. That is invisible to a `for`
loop and is the mechanism `yield from` uses (L06).

### 2.3 Laziness: what it buys and what it costs

**Buys:**

- **Constant memory** over arbitrarily large sources.
- **Time to first result.** A pipeline over a stream produces its first output after one
  element, not after the whole input.
- **Infinite sequences.** `itertools.count`, a poll loop, a stream.
- **Short-circuiting.** `any(is_bad(r) for r in rows)` stops at the first bad row and never
  reads the rest.
- **Composition without intermediates.** Five chained generators allocate no intermediate
  lists.

**Costs, and these are underrated:**

- **Exceptions surface at consumption**, in the consumer's stack frame, far from the code
  that created the generator. A traceback from a five-stage pipeline is genuinely harder to
  read than one from five list operations.
- **Resource lifetime becomes unclear.** §2.4.
- **One pass.** Any algorithm needing two passes must materialize or re-create.
- **No `len`, no indexing, no reversal.** Every consumer must be written for streaming.
- **Debugging is harder.** You cannot print the intermediate collection; you must `tee` or
  materialize.
- **Per-element overhead.** Each `next()` is a frame resume. For CPU-bound work over small
  elements, a list comprehension is measurably faster than a generator chain (PY-602 L01
  measures it). Laziness is a memory optimization, not a speed one.

The decision rule: **lazy when the data is large, unbounded, or expensive to produce and
you may not need all of it; eager when it is small, needed in full, or needed twice.**

### 2.4 Cleanup: the hard part

```python
def read(path):
    with open(path) as f:
        for line in f:
            yield line.strip()
```

When does the file close?

- **If fully consumed:** the `for` ends, the `with` exits, the generator returns,
  `StopIteration`. Clean.
- **If the consumer stops early and the generator is collected:** CPython's refcount drops
  to zero, `gen.close()` is called, which throws `GeneratorExit` at the suspended `yield`,
  which propagates out of the `with`, which closes the file. Also clean — *on CPython, at
  an unpredictable moment*.
- **If the generator is retained** (in a list, a closure, a traceback): never, until it is
  collected.
- **If the interpreter exits with the generator alive:** best-effort; may not run.
- **On PyPy or any non-refcounting implementation:** at the next GC, which may be much
  later.

So the "the `with` inside the generator handles it" belief is *conditionally* true and the
conditions are not ones you control. Consequences:

1. **Do not open resources inside generators that may be partially consumed**, unless the
   caller is disciplined. Prefer taking the already-open resource as a parameter and letting
   the caller own its lifetime:

```python
def read(f: TextIO) -> Iterator[str]:      # caller owns the file
    for line in f:
        yield line.strip()

with open(path) as f:
    for row in read(f):
        ...
```

2. **`contextlib.closing`** or an explicit `with` around the generator itself when it does
   own a resource:

```python
with contextlib.closing(read(path)) as rows:
    for r in rows:
        if done: break                 # close() is called on exit
```

3. **Handle `GeneratorExit` correctly if you catch it at all.** You may catch it to do
   cleanup, but you must not `yield` again (raises `RuntimeError`) and you should re-raise or
   return. Catching it with a bare `except:` and continuing is a way to make a generator
   uncloseable.

```python
def gen():
    try:
        while True:
            yield compute()
    except GeneratorExit:
        cleanup()
        raise                      # or just `return`
    finally:
        always()
```

### 2.5 `itertools` as a vocabulary

Learning `itertools` well is disproportionately valuable, because it turns loops into named
operations. The ones worth memorizing:

| | |
|---|---|
| `islice(it, start, stop, step)` | slicing, lazily; the only way to skip/limit an iterator |
| `chain(*its)` / `chain.from_iterable` | concatenation; `from_iterable` is lazy in the outer |
| `groupby(it, key)` | **requires sorted input** — the single most common misuse |
| `tee(it, n)` | n independent iterators — buffers everything between the slowest and fastest consumer |
| `zip_longest`, `zip(..., strict=True)` | `strict=True` (3.10+) catches length mismatch, which is a real bug class |
| `accumulate` | running totals, running max, any fold |
| `pairwise` | adjacent pairs (3.10+); replaces a whole family of index gymnastics |
| `batched` | fixed-size chunks (3.12+) |
| `takewhile` / `dropwhile` | prefix/suffix by predicate |
| `product`, `permutations`, `combinations` | combinatorics, lazily |
| `count`, `cycle`, `repeat` | infinite generators |
| `filterfalse`, `compress`, `starmap` | the remaining gaps |

Two traps:

- **`groupby` needs sorted input** and yields *shared* sub-iterators that are invalidated
  when you advance to the next group. `[(k, list(g)) for k, g in groupby(...)]` is safe;
  storing `g` is not.
- **`tee` is not free.** If one branch runs far ahead, `tee` buffers the difference. Teeing
  a 10 GB stream and consuming one branch fully before the other materializes the whole
  thing. This defeats the point.

### 2.6 Pipelines

The compositional style:

```python
def read(f):        return (line.rstrip("\n") for line in f)
def parse(lines):   return (json.loads(l) for l in lines if l)
def valid(recs):    return (r for r in recs if r.get("id"))
def enrich(recs, lookup):
    for r in recs:
        r["region"] = lookup.get(r["country"], "unknown")
        yield r

with open(path) as f:
    for record in enrich(valid(parse(read(f))), regions):
        sink(record)
```

Each stage is independently testable with a list input and a list output — which is the
main engineering advantage, more than the memory.

Design notes:

- **Nesting reads inside-out.** For more than three stages, build the composition explicitly
  or use a `reduce` over a list of stages; readability degrades fast.
- **A stage should not know its source.** Take an iterator, return an iterator. No stage
  opens files or connects to anything.
- **Errors need a policy.** One bad record in ten million should not kill the pipeline —
  but silently skipping it is worse. The standard shape is a stage that yields
  `Ok(record) | Err(record, exc)` and a later stage that routes, with a counter and a
  threshold. Decide this explicitly; a pipeline with no error policy has one by accident.
- **Backpressure does not exist here.** A generator pipeline is pull-based and
  single-threaded, so the consumer sets the pace naturally. The moment you introduce
  threads or async between stages, backpressure becomes a real design question
  (PY-601 L09).

### 2.7 Memory: what laziness does not fix

A lazy pipeline still materializes at any stage that must:

- `sorted()`, `set()`, `list()`, `max(..., key=)` over the whole input.
- `groupby` without sorted input requires sorting first.
- A join keyed on one side requires holding that side.
- `tee` with divergent consumers (§2.5).
- Any aggregation with unbounded cardinality — `Counter` over user ids is O(users).

The design move for the last one is *bounded* approximate structures: HyperLogLog for
distinct counts, count-min sketch for frequencies, reservoir sampling for samples. Those are
DI-721 L07 material; here it is enough to notice which stage of your pipeline is the one
that decides its memory profile. **In every pipeline there is exactly one such stage, and
you should be able to name it.**

## 3. Construction: a log-processing pipeline

Build one, incrementally, from a naive eager version.

**Version 1 — eager.**

```python
def process(path):
    lines = open(path).read().splitlines()      # whole file in memory
    records = [parse(l) for l in lines]
    valid = [r for r in records if r.ok]
    return Counter(r.endpoint for r in valid)
```

Defects: three full copies in memory; the file is never closed; a parse error kills
everything; no way to see progress.

**Version 2 — lazy stages.** Convert each list comprehension to a generator, take the file
handle as a parameter, and measure peak memory (`tracemalloc`) on a 1 GB input. Report the
number. It should drop by roughly the file size times three.

**Version 3 — error policy.** Introduce

```python
@dataclass(frozen=True)
class Bad:
    line: str
    error: Exception

def parse_stage(lines: Iterable[str]) -> Iterator[Record | Bad]:
    for line in lines:
        try:
            yield parse(line)
        except ParseError as e:
            yield Bad(line, e)
```

and a routing stage that counts, samples the first N bad lines for a report, and aborts if
the bad fraction exceeds a threshold. Note that the threshold check requires *state across
elements* — which is why this is a generator function with a loop, not a generator
expression.

**Version 4 — the aggregation.** `Counter(r.endpoint for r in good)` is bounded by the
number of distinct endpoints, which is fine. Now change the requirement to "distinct users
per endpoint" and watch the memory profile change. Implement it exactly (a dict of sets) and
approximately (HyperLogLog), and report the memory and accuracy of each. Name the stage that
decides the memory profile.

**Version 5 — observability and cleanup.** Add: a progress counter that does not consume the
iterator (a passthrough stage), correct behaviour when the consumer breaks early
(verify with `contextlib.closing` and a `finally` that logs), and a test that the file is
closed in all four termination cases of §2.4.

## 4. Failure modes

- **Validation inside a generator body.** Runs at first `next()`, not at call. §2.1.
- **Consuming an iterator twice.** Silent empty second pass.
- **Opening a resource inside a partially-consumed generator.** §2.4.
- **`groupby` on unsorted input.** Produces adjacent-only groups, silently wrong.
- **Storing `groupby`'s sub-iterator.** Invalidated on advance.
- **`tee` with divergent consumers.** Materializes.
- **Retaining a suspended generator.** Retains its frame and all locals.
- **`return` in a generator expecting the value to reach the `for` loop.** It does not; it
  becomes `StopIteration.value`.
- **`StopIteration` raised inside a generator body.** Converted to `RuntimeError` since
  PEP 479 — which is correct, and worth understanding rather than working around.
- **Assuming laziness is faster.** It is a memory technique.
- **A pipeline with no error policy.**
- **`list()` in the middle of a pipeline "just to debug"** left in the code.

## 5. Exercises

### Warm-up (25 min)

**W1.** Show that a generator's body does not run until first `next()`. Then write the
eager-validation wrapper of §2.1 and prove the difference.

**W2.** Write a generator that opens a file, consume it partially, delete the reference, and
show when the file closes. Then retain it in a list and show that it does not.

**W3.** Demonstrate the `groupby`-on-unsorted-input bug with data where it is not obvious.

### Core (2.5 h)

**C1 — The pipeline.** Complete §3, all five versions. Deliverable: the code, peak-memory
measurements for each version on a ≥1 GB input, the error-policy design with its threshold
rationale, the exact-vs-approximate cardinality comparison, and the four cleanup tests.

**C2 — Stage testability.** Take a real loop-based transformation from your code
(30+ lines, several concerns). Decompose it into generator stages. Write a test per stage
with list input and list output. Report: lines of code before/after, test count before/after,
and one bug you found during the decomposition.

**C3 — `itertools` fluency.** Take fifteen loops from real code and rewrite each using
`itertools`. For each, state whether the rewrite is clearer. Report the proportion where it
is *not* — that number is the honest answer to "should I use itertools more", and it is
often around half.

**C4 — Lazy vs eager, measured.** For a fixed transformation, benchmark: list
comprehension chain, generator chain, and a single fused loop, over input sizes from 10³ to
10⁸. Plot time and peak memory. Identify the crossover. Explain the result using PY-501
L08's cost model, and state the rule you would put in a style guide.

### Challenge

**X1.** Implement a `Pipeline` class that composes stages with `|`, supports named stages,
per-stage metrics, an error policy, and correct cleanup of all stages when the consumer
breaks early (this last is the hard part — you need `ExitStack`, L07). Then compare with an
existing library and say what you learned.

**X2.** Implement reservoir sampling, HyperLogLog, and a count-min sketch as pipeline
stages. For each: state the space bound and the error bound, verify the error bound
empirically over 1,000 trials, and report whether the theoretical bound held. (Return to the
concentration inequalities behind these after CS-621 L08.)

## 6. Self-check

1. What happens when you *call* a generator function? What runs?
2. Where do you put argument validation in a generator, and why?
3. Give the four generator protocol operations and what each does.
4. Enumerate the four cases for when a `with` inside a generator exits.
5. Name five costs of laziness.
6. Why does `groupby` require sorted input, and what is the failure mode?
7. When does `tee` defeat the purpose of a lazy pipeline?
8. In any pipeline, what is "the stage that decides the memory profile"?

## 7. Primary sources

- Language Reference §6.2.9 (yield expressions) and the `itertools` docs including the
  recipes section.
- PEP 255 (simple generators), PEP 342 (coroutines via generators), PEP 479
  (`StopIteration` handling).
- Beazley, "Generator Tricks for Systems Programmers" (dabeaz.com) — still the best
  practical treatment of generator pipelines.
- Ramalho, *Fluent Python* 2e, ch. 17.

---

**Previous:** [L04](L04-decorators.md) · **Next:**
[L06 — Coroutines, `yield from`, and Delegation](L06-coroutines-and-delegation.md)
