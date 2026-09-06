# SE-521 · Lesson 04 — Design Patterns as Vocabulary — and the Critique

**Estimated study time:** 4 hours
**Prerequisites:** L01–L03; PY-502 L01, L04

---

## 1. Orientation

Peter Norvig's 1996 observation is the right frame for this entire lesson: **sixteen of the
twenty-three Gang of Four patterns are "invisible or simpler" in a dynamic language with
first-class functions.** Strategy is a function. Command is a function. Factory is a
function. Iterator is a language feature. Template Method is a function taking a function.

That does not make the patterns worthless. It relocates their value: **they are a
vocabulary, not a construction kit.** "Let's put an adapter here" communicates a design in
four words to anyone who shares the vocabulary. Implementing an `AdapterFactory` with an
abstract `IAdapter` because the book has a class diagram is cargo cult.

This lesson teaches the vocabulary, the Pythonic form of each pattern, and the critiques —
which are serious and which you should be able to state.

## 2. Theory

### 2.1 What a pattern is

Alexander's original (architectural) definition: *a solution to a problem in a context*. All
three parts matter, and the *context* is the part that gets dropped. A pattern applied
outside its context is not a pattern; it is a structure.

The GoF book's real contribution was not the twenty-three solutions but the **format**:
intent, motivation, applicability, structure, consequences. The consequences section is the
one to read; it is where the costs are, and it is the part nobody quotes.

### 2.2 The patterns that are functions in Python

| Pattern | GoF form | Python form |
|---|---|---|
| **Strategy** | Interface + concrete strategy classes | pass a function |
| **Command** | Interface with `execute()` | a function, or `functools.partial` |
| **Template Method** | Abstract base with hooks | a function taking functions |
| **Factory Method** | Subclass overrides a creation method | a function, or a dict of constructors |
| **Abstract Factory** | Family of factory classes | a module, or a dataclass of callables |
| **Iterator** | Interface with `next()` | generators, `__iter__` |
| **Observer** | `Subject`/`Observer` classes | a list of callbacks, or an event bus |
| **State** | Class per state | an enum + `match`, or a dict-of-transitions |
| **Visitor** | Double dispatch via `accept`/`visit` | `functools.singledispatch`, or `match` |
| **Singleton** | Private constructor, static instance | a module-level object — or don't |

The rule: **if the interface has one method, it is a function.** A `Strategy` interface with
`execute(x)` and three implementing classes is three functions plus 40 lines of ceremony.

```python
# GoF
class PricingStrategy(ABC):
    @abstractmethod
    def price(self, order: Order) -> Money: ...
class StandardPricing(PricingStrategy): ...
class BulkPricing(PricingStrategy): ...

# Python
Pricing = Callable[[Order], Money]
def standard(order: Order) -> Money: ...
def bulk(order: Order) -> Money: ...
```

Use a class when the strategy has **state**, **multiple related operations**, or needs a
**name in the domain** ("a `TieredPricingPolicy` is a thing our business talks about").
Those are real reasons; "the book says so" is not.

### 2.3 The patterns that remain useful in Python

**Adapter.** Convert one interface to another. Genuinely useful, and it is what an *adapter*
means in ports-and-adapters (L05). The Python form is a small class or a function; the
concept, not the class diagram, is what matters.

**Decorator (structural).** Wrap an object, preserving its interface, adding behaviour. Note
this is *not* the same as a Python `@decorator` — but Python decorators are how you often
implement it for functions. For objects, it is a wrapper class holding the wrapped object
(PY-502 L02 §2.4's descriptor composition is an instance).

**Facade.** A simple interface over a complex subsystem. This *is* the deep module of
L01 §2.4, and it is the most valuable pattern in the book.

**Repository / Unit of Work / Data Mapper.** Not GoF (they are Fowler's), and they are the
ones you will actually use. L07 treats Unit of Work properly.

**Null Object.** An implementation that does nothing, to remove `if x is not None` from
callers. This is L01 §2.6's "define errors out of existence" as a pattern. `logging`'s
`NullHandler` and `contextlib.nullcontext` are standard-library instances.

**Composite.** Treat individual objects and compositions uniformly. Natural for trees,
filters, permission rules, UI. Still genuinely useful because the *uniformity* is the point.

**Builder.** For objects with many optional parameters. In Python, keyword arguments with
defaults handle most cases, so Builder earns its place only when construction is
multi-stage, validated incrementally, or fluent by design (PY-502 L09).

**Proxy.** Lazy loading, access control, remoting. Note PY-501 L03 §2.6: a Python proxy
cannot transparently forward special methods via `__getattr__`, so it must declare them.

**Chain of Responsibility.** Middleware. Every web framework is this pattern, and it is a
good example of a pattern that survives because the *composition* is its value.

### 2.4 The critiques, stated properly

You should be able to make each of these, because each is partly right.

**1. Patterns are a symptom of missing language features** (Norvig; also Graham).
Sixteen of twenty-three collapse given first-class functions and dynamic dispatch. The
inference — that a pattern's popularity indicates a language deficiency — is a genuine
insight, and it explains why Python codebases that lean on GoF structures often feel like
Java in disguise. **Where it goes too far:** vocabulary has value independent of
implementation cost, and some patterns (Facade, Composite, Adapter) are about *design
intent*, not about working around a missing feature.

**2. Patterns encourage speculative flexibility.** The catalogue's framing — "make this
aspect vary" — invites you to make things vary that never will. Every varied axis is
indirection paid for a possibility. This is PY-502 L10's argument and it is correct.

**3. Patterns become a substitute for thinking.** "Which pattern should I use?" is the wrong
question; "what varies and what is stable?" is the right one, and the pattern (if any) falls
out of the answer.

**4. Pattern names as architecture.** A codebase of `AbstractRequestHandlerFactoryBean`
names its structures after patterns rather than after the domain. Names should come from the
domain's language (L06's ubiquitous language), not from the catalogue. `PricingPolicy` is a
better name than `PricingStrategy` even when it *is* the Strategy pattern.

**5. Singleton is not a pattern, it is a global.** It defeats testing, hides dependencies,
creates initialization-order problems, and is not thread-safe by default. In Python a module
is already a singleton; if you need one instance, make one instance in the composition root
and pass it. Even the GoF authors have since said Singleton should be dropped.

**6. Patterns cost more in a dynamic language.** Type checkers cannot follow much of the
double-dispatch and factory machinery, so the tooling cost (PY-502 L10 §2.1) is higher than
in a statically typed language where the IDE compensates.

The balanced position, and the one to hold: **patterns are excellent as a shared vocabulary
for describing designs, mediocre as a source of designs, and harmful as a target.**

### 2.5 Reading the consequences section

Take Observer as a worked example of what the catalogue actually offers when read properly.

*Intent:* one-to-many dependency; when one object changes state, dependents are notified.

*Consequences the book lists, and which matter in production:*

- Subject and observers are loosely coupled — good.
- **Broadcast is unpredictable**: you cannot easily see what happens when you fire an event.
  This is the cost that bites. A `save()` that publishes `OrderSaved` may trigger an email,
  a cache invalidation, and an audit write, none visible at the call site.
- **Unexpected update cascades.** An observer that modifies the subject re-fires.
- **Dangling references / memory leaks.** Observers must deregister; a bound method held in
  a subscriber list keeps the object alive forever (PY-501 L09 §2.3). `weakref.WeakMethod`
  is the fix and nobody remembers it.
- **Ordering is unspecified.** If two observers must run in order, you have hidden
  connascence of execution order (L02 §2.4) with nothing enforcing it.
- **Error handling is undefined.** If observer 2 of 5 raises, do 3–5 run? Is the subject's
  operation rolled back? The pattern is silent, and every implementation answers differently.

That list is why "just publish an event" is a bigger decision than it looks, and it is
exactly what a competent reviewer should raise. The pattern gave you the design; the
consequences section is what makes you competent at using it.

### 2.6 Patterns you will actually reach for in Python

Ranked by real frequency in well-written Python systems:

1. **Facade / deep module** — L01.
2. **Adapter** — every external integration.
3. **Strategy as a function** — everywhere, unnamed.
4. **Chain of Responsibility / middleware** — every request path.
5. **Repository + Unit of Work** — every data-backed application (L07).
6. **Null Object** — removes conditionals.
7. **Composite** — trees, rules, filters.
8. **Decorator (object)** — caching, retry, instrumentation wrappers.
9. **Observer / event publishing** — with §2.5's consequences understood.
10. **Builder** — only when construction is genuinely staged.

Notably absent: Abstract Factory, Bridge, Flyweight, Interpreter, Mediator, Memento,
Prototype, Singleton. They are not wrong; they are rarely the best answer in Python, and
reaching for them is usually a sign of translating a Java design.

## 3. Construction: refactoring a conditional, four ways

Take a real `if/elif` chain that dispatches on a type or mode — every codebase has one.

```python
def export(data: Report, fmt: str) -> bytes:
    if fmt == "csv":
        ...30 lines...
    elif fmt == "xlsx":
        ...40 lines...
    elif fmt == "pdf":
        ...50 lines...
    else:
        raise ValueError(fmt)
```

**Version 1 — a dict of functions.**

```python
Exporter = Callable[[Report], bytes]
EXPORTERS: dict[str, Exporter] = {"csv": to_csv, "xlsx": to_xlsx, "pdf": to_pdf}

def export(data: Report, fmt: str) -> bytes:
    try:
        return EXPORTERS[fmt](data)
    except KeyError:
        raise UnknownFormat(fmt, supported=sorted(EXPORTERS)) from None
```

Smallest change. Greppable. Adding a format is one line plus one function. Type-checks.
**This is Strategy, and it is 6 lines.**

**Version 2 — Protocol + classes.** When exporters need configuration and multiple
operations:

```python
class Exporter(Protocol):
    extension: str
    media_type: str
    def export(self, report: Report) -> bytes: ...
    def supports(self, report: Report) -> bool: ...
```

Now the extra methods justify the class. Note the discriminator: *more than one operation*.

**Version 3 — `singledispatch` on the report type.** If dispatch is on the *data* rather
than a mode string:

```python
@singledispatch
def export(report: Report) -> bytes: ...
@export.register
def _(report: FinancialReport) -> bytes: ...
```

Right when you add operations often and types rarely (PY-502 L01 §2.6).

**Version 4 — keep the conditional.** For three branches of five lines each, in one place,
that change together, the `if/elif` is clearer than any of the above. Say so.

**The deliverable is the decision criteria**, not the code:

| Signal | Choose |
|---|---|
| One operation, no state | dict of functions |
| Multiple related operations or config | Protocol + classes |
| Dispatch on data type; operations added often | `singledispatch` |
| Few branches, short, co-located, co-changing | leave it |
| Branches added by third parties | registry with explicit registration |
| Branch selection needs the whole request context | consider a rules engine or table |

Then apply the criteria to five more conditionals in your codebase and report how many
should be left alone. The honest number is usually most of them.

## 4. Failure modes

- **Implementing a class hierarchy where a function would do.**
- **Naming classes after patterns instead of the domain.**
- **Singleton.**
- **Speculative variation.** An abstract factory for one product family.
- **Applying a pattern outside its context.** §2.1.
- **Ignoring the consequences section.** §2.5.
- **Observer without deregistration** — a memory leak, and a bound method holds `self`.
- **Observer with ordering requirements** and no mechanism enforcing them.
- **Observer with undefined error semantics.**
- **A Visitor in Python** where `singledispatch` or `match` is clearer.
- **Translating a Java design wholesale.** The tell is `IFoo`/`FooImpl` naming and a factory
  per class.
- **Refusing patterns entirely.** The vocabulary is genuinely useful; rejecting it means
  describing designs in paragraphs.

## 5. Exercises

### Warm-up (25 min)

**W1.** Take three GoF patterns from §2.2 and write their Python form in under ten lines
each. Compare with the book's class diagrams.

**W2.** Find a class in your codebase named after a pattern. Rename it after the domain.
Report whether anything is now less clear.

**W3.** Find an Observer/event-publisher implementation. Determine its answers to §2.5's
four undefined questions (ordering, errors, cascades, deregistration). Report which are
undocumented.

### Core (2 h)

**C1 — Four ways.** Complete §3 for a real conditional, then apply the criteria table to
five more. Deliverable: the four implementations, the criteria table, the five verdicts, and
a 500-word note on how many you left alone and why that is the right answer.

**C2 — Read the consequences.** Take five patterns you use. For each, read GoF's
consequences section and write down every consequence that applies to your usage and that
you had not consciously accepted. Report the list. This exercise routinely finds two or
three real risks per pattern.

**C3 — De-Java a module.** Find Python code written in a Java idiom (interface per class,
factory per interface, getters and setters, `IFoo`/`FooImpl`). Rewrite it Pythonically.
Report: lines before/after, files before/after, what the type checker knows in each, and
whether anything was genuinely lost.

**C4 — Event semantics.** Take an event-publishing mechanism and *specify* it: ordering
guarantee, error semantics (does a failing handler abort the others? the publisher?),
reentrancy, deregistration and lifetime, and whether delivery is synchronous. Write the
tests. Report how many of your answers were previously undefined and how many callers were
depending on the undefined behaviour.

### Challenge

**X1.** Read Norvig's "Design Patterns in Dynamic Languages" (1996) slides, the GoF preface
and Introduction, and one substantial critique. Write 1,500 words on the claim that patterns
are a symptom of missing language features: state it at its strongest, give three patterns
where it clearly holds and two where it clearly does not, and derive a position on what
patterns are *for* in a language like Python.

**X2.** Take the GoF catalogue and produce a table: for each of the twenty-three patterns —
its Python form, whether it is a language feature, a function, a small class, or genuinely a
structure; how often it appears in three large open-source Python projects (measure, do not
guess); and whether its appearances look justified. The measurement is the deliverable.

## 6. Self-check

1. Give Alexander's three-part definition and say which part is usually dropped.
2. Which section of a GoF pattern is the most valuable, and why?
3. Give the rule that decides between a function and a class for Strategy.
4. Name five patterns that remain genuinely useful in Python and say why.
5. State Norvig's critique at its strongest and its limit.
6. Give four consequences of Observer that are undefined by default.
7. Why is Singleton not a pattern?
8. Give the criteria table for refactoring a dispatch conditional.

## 7. Primary sources

- Gamma, Helm, Johnson & Vlissides, *Design Patterns* (1994) — the Introduction and the
  Consequences sections. Do not read it as a construction kit.
- Norvig, "Design Patterns in Dynamic Languages" (1996 slides).
- Alexander, *A Pattern Language* / *The Timeless Way of Building* — the origin, and the
  source of the "context" requirement.
- Fowler, *Patterns of Enterprise Application Architecture* — the patterns you will actually
  use.
- Hettinger, "Beyond PEP 8" and "Python's Class Development Toolkit" — the Pythonic forms.

---

**Previous:** [L03](L03-dependency-inversion.md) · **Next:**
[L05 — Ports, Adapters, and the Dependency Rule](L05-ports-and-adapters.md)
