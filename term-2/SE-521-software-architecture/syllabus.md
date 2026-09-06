# SE-521 — Software Architecture & Design

**Term:** 2 · **Credits:** 15 · **Nominal hours:** 125
**Prerequisites:** SE-511; PY-501 helpful
**Co-requisite:** PY-502

---

## Driving question

> What distinguishes a structure that can absorb change from one that cannot?

Architecture is not diagrams. It is the set of decisions that are expensive to reverse, and
the discipline of architecture is deciding which those are, making them deliberately, and
writing down why.

The failure this course exists to prevent is the system that was designed correctly for the
requirements of three years ago and now costs six weeks to add a field to. Nothing about it
is wrong; nothing about it is changeable. That outcome is not caused by bad decisions so
much as by *undocumented* ones — nobody knows which constraints are load-bearing, so nobody
dares move anything.

## Learning outcomes

On completion you will be able to:

1. **Apply** Parnas's information-hiding criterion to decompose a system, and explain why it
   beats decomposition by processing step.
2. **Measure** coupling and cohesion in real code, and connect both to observed change
   cost.
3. **Use** dependency inversion correctly — and identify where applying it makes a system
   worse.
4. **Deploy** design patterns as a vocabulary while articulating the standard critiques.
5. **Design** a ports-and-adapters system and defend the boundary placement.
6. **Model** a domain: ubiquitous language, entities and value objects, aggregates,
   bounded contexts, and context mapping.
7. **Reason** about transactional boundaries and what consistency the design actually
   promises.
8. **Define** architectural fitness functions and automate them.
9. **Choose** a service decomposition (or decline to) with a stated cost model.
10. **Work** on legacy systems: characterization, seams, strangler fig, and branch by
    abstraction.
11. **Write** architectural decision records that are worth reading in three years.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Modularity and Information Hiding | 3.5 |
| L02 | Coupling, Cohesion, and the Cost of Change | 3.5 |
| L03 | Dependency Inversion, Composition, and Their Limits | 3.5 |
| L04 | Design Patterns as Vocabulary — and the Critique | 4 |
| L05 | Ports, Adapters, and the Dependency Rule | 4 |
| L06 | Domain-Driven Design: Language, Model, Boundaries | 4 |
| L07 | Aggregates, Transactions, and Consistency Boundaries | 4 |
| L08 | Evolutionary Architecture and Fitness Functions | 3.5 |
| L09 | Service Boundaries and the Distribution Decision | 4 |
| L10 | Legacy Systems, Seams, and Architectural Decision Records | 3.5 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a decomposition, argued and measured | 20% |
| Problem set 2 (L05–L07): a domain model with explicit boundaries | 20% |
| Problem set 3 (L08–L10): fitness functions, a legacy strategy, and an ADR log | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 2 build artifact

A non-trivial domain model — a real problem, modelled properly — with: a documented
architecture; an ADR log with at least eight decisions including rejected alternatives;
automated fitness functions enforcing the layering; a test suite whose shape follows the
architecture (SE-511 L05); and a written argument for the boundaries you chose.

Not a CRUD app. Choose a domain with genuine rules: scheduling, pricing, entitlements,
compliance, logistics, or something from your own work with the details changed.

## Required reading

- Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM, 1972).
- Ousterhout, *A Philosophy of Software Design*, 2nd ed. — all of it; it is short.
- Evans, *Domain-Driven Design*, Parts I–II.
- Richards & Ford, *Fundamentals of Software Architecture*, chs. 1–8, 19–21.
- Lampson, "Hints for Computer System Design" (SOSP 1983).
- Brooks, "No Silver Bullet" (1986).

## Recommended

- Fowler, *Patterns of Enterprise Application Architecture*.
- Gamma et al., *Design Patterns* — read with the critiques alongside.
- Feathers, *Working Effectively with Legacy Code*.
- Ford, Parsons & Kua, *Building Evolutionary Architectures*.
- Percival & Gregory, *Architecture Patterns with Python* — the closest thing to a
  Python-native treatment of DDD and ports/adapters.
- Skelton & Pais, *Team Topologies*.
