# Advanced Python & Software Engineering — A Self-Directed MSc-Equivalent Program

**Student:** Mike
**Program code:** MPSE
**Version:** 1.0 — September 2026
**Total nominal load:** 6 terms · 13 courses · ~95 lessons · 1 capstone

---

## What this is

This is a complete, university-grade curriculum that takes you from *competent working
Python programmer* to *master's-level software engineer* — someone who can reason about
language semantics from first principles, design systems that survive contact with
production, read and critique research papers, and defend architectural decisions to a
hostile committee.

It is modelled on the structure of a real taught MSc: a foundations tier, a core tier, a
theory tier, specialization tiers, and a capstone. Every course has a syllabus with
learning outcomes, a sequence of lessons, problem sets, and an assessment rubric. Every
lesson is written to be read in one sitting of 60–120 minutes and worked through with a
terminal open.

It assumes you already write Python professionally. It does **not** assume you have
formal CS training, and it does not skip the theory — it teaches the theory the way a
graduate program would, from definitions and proofs, but always landing on something you
can run.

## How the program is organized

```
msc-python-swe/
├── README.md                         ← you are here
├── 00-program/                       ← handbook, placement test, rubrics, reading list
├── term-1/  Foundations of Craft
│   ├── PY-501  The Python Object Model & Execution Semantics
│   └── SE-511  Software Construction: Testing, Types, and Tooling
├── term-2/  Abstraction & Design
│   ├── PY-502  Advanced Abstraction: Protocols, Descriptors, Metaprogramming
│   └── SE-521  Software Architecture & Design
├── term-3/  Systems & Performance
│   ├── PY-601  Concurrency, Parallelism, and Asynchrony
│   └── PY-602  Performance Engineering & the Systems Interface
├── term-4/  Theory Core
│   ├── CS-621  Algorithms, Complexity, and Computability
│   └── CS-641  Programming Languages & Type Systems
├── term-5/  Distributed & Data-Intensive Systems
│   ├── DS-701  Distributed Systems
│   └── DI-721  Data-Intensive Systems
├── term-6/  Advanced Practice
│   ├── CA-731  Cloud & Platform Architecture at Scale
│   ├── ML-741  Machine Learning Systems Engineering
│   └── FM-751  Formal Methods & Verification for Practitioners
├── capstone/                         ← CAP-799 capstone handbook & project catalog
└── appendices/                       ← glossary, notation, paper index, errata log
```

Each course directory contains:

| File | Purpose |
|---|---|
| `syllabus.md` | Outcomes, prerequisites, lesson map, assessment weights, reading |
| `L01…Lnn.md` | The lessons themselves |
| `problem-sets.md` | Graded problem sets with rubrics |
| `exam.md` | A written exam in the style of a real MSc paper |

## Start here

1. Read `00-program/program-handbook.md` — it explains the pedagogy, the time budget, and
   how to actually finish something this large without stalling in month three.
2. Take `00-program/placement-self-assessment.md`. It is honest, it is uncomfortable, and
   it tells you which lessons in Term 1 you may skim rather than study.
3. Set up your working environment with `00-program/lab-setup.md`.
4. Begin `term-1/PY-501-python-object-model/syllabus.md`.

## A note on how the lessons teach

Every lesson builds code **incrementally**: a first version that is wrong or naive, an
observation about why it is wrong, then a revision. You are expected to type the
intermediate versions and watch them fail. Reading a finished implementation teaches you
almost nothing; watching one evolve teaches you the reasoning that produced it.

Code is written for **Python 3.12 or newer**. Where a feature is newer than 3.12 or is
version-sensitive (the free-threaded build, the JIT, `typing` additions), the lesson says
so explicitly and gives the fallback.

## License and provenance

These materials were authored for your personal study. Third-party works are cited, never
reproduced. Where a lesson leans on a specific paper or book, the citation is given so you
can go to the primary source — and you should, at least once per course. Reading primary
literature is itself a graduate skill this program intends to build.
