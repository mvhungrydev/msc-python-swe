# Program Reading List

Books are listed as **Core** (buy it, work through it) or **Reference** (own it, consult
it). Papers are listed as **Required** (read in full, write your four-line note) or
**Recommended**.

Nothing here is reproduced in the lessons — the lessons cite and you read. Building the
habit of going to the primary source is itself a program outcome.

---

## Books by course

### PY-501 / PY-502 — Python semantics and abstraction

- **Core** — Ramalho, *Fluent Python*, 2nd ed. (O'Reilly, 2022). The single best book on
  the Python object model. Chapters 1, 11–16, 22–24 map almost directly onto these courses.
- **Core** — Shaw, *CPython Internals* (Real Python, 2021). Read alongside PY-501 L07–L09.
- **Reference** — Beazley & Jones, *Python Cookbook*, 3rd ed. Dated in places; still the
  best worked catalogue of metaprogramming techniques.
- **Reference** — The CPython source itself: `Objects/typeobject.c`, `Python/ceval.c`.
  PY-501 L09 walks you into it.

**PEPs (required reading, all short):** 8, 20, 484, 544, 557, 585, 604, 612, 634–636, 673,
695, 3119.

### SE-511 — Software construction

- **Core** — Winters, Manshreck & Wright, *Software Engineering at Google* (O'Reilly, 2020).
  Chapters on testing, dependency management, and large-scale change.
- **Core** — Freeman & Pryce, *Growing Object-Oriented Software, Guided by Tests*
  (Addison-Wesley, 2009). The book that explains *why* mocks exist, and their proper scope.
- **Reference** — Meszaros, *xUnit Test Patterns* (2007). The taxonomy of test doubles.
- **Reference** — Beck, *Test-Driven Development: By Example* (2002). Short. Read it in an
  evening for the rhythm, not the dogma.
- **Reference** — Humble & Farley, *Continuous Delivery* (2010).

**Required papers:**
- Claessen & Hughes, "QuickCheck: A Lightweight Tool for Random Testing of Haskell
  Programs" (ICFP 2000) — the origin of property-based testing.
- Dijkstra, "Notes on Structured Programming" (1970), section on testing — the source of
  "testing shows the presence, not the absence, of bugs".

### SE-521 — Architecture and design

- **Core** — Ousterhout, *A Philosophy of Software Design*, 2nd ed. (2021). Short, opinionated,
  and the best available treatment of interface depth.
- **Core** — Evans, *Domain-Driven Design* (2003). Read Parts I–II carefully, III–IV
  selectively.
- **Core** — Richards & Ford, *Fundamentals of Software Architecture* (O'Reilly, 2020).
- **Reference** — Fowler, *Patterns of Enterprise Application Architecture* (2002).
- **Reference** — Gamma, Helm, Johnson & Vlissides, *Design Patterns* (1994). Read it as
  a historical vocabulary document, and read the critiques alongside.
- **Reference** — Feathers, *Working Effectively with Legacy Code* (2004).
- **Reference** — Ford, Parsons & Kua, *Building Evolutionary Architectures* (2017).
- **Reference** — Nygard, *Release It!*, 2nd ed. (2018).

**Required papers:**
- Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM, 1972).
  The foundational paper of the discipline. Five pages.
- Parnas, "Designing Software for Ease of Extension and Contraction" (1979).
- Brooks, "No Silver Bullet — Essence and Accident in Software Engineering" (1986).
- Lampson, "Hints for Computer System Design" (SOSP 1983).
- Liskov & Wing, "A Behavioral Notion of Subtyping" (TOPLAS 1994).

### PY-601 — Concurrency

- **Core** — Herlihy & Shavit, *The Art of Multiprocessor Programming*, 2nd ed. (2020).
  Hard, and worth it. Chapters 1–3, 7, 9–10.
- **Reference** — Arpaci-Dusseau & Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*
  (free online). The concurrency section is the clearest introduction that exists.

**Required papers:**
- Hoare, "Communicating Sequential Processes" (CACM 1978).
- Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects"
  (TOPLAS 1990).
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess
  Programs" (1979) — sequential consistency, two pages.

**PEPs:** 492, 525, 530, 654, 3156, 684, 703, 734.

### PY-602 — Performance

- **Core** — Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*, 3rd ed.
  Chapters 5–6 (optimization, the memory hierarchy) are the ones that matter here.
- **Core** — Gregg, *Systems Performance*, 2nd ed. (2020). Methodology first: USE method,
  workload characterization.
- **Reference** — Gregg, *BPF Performance Tools* (2019).
- **Reference** — Drepper, "What Every Programmer Should Know About Memory" (2007). Long,
  free, still the reference on cache behaviour.

**Required papers:**
- Knuth, "Structured Programming with go to Statements" (1974) — the *actual* context of
  "premature optimization is the root of all evil". Read the surrounding paragraph.
- Amdahl, "Validity of the Single Processor Approach…" (1967); Gustafson,
  "Reevaluating Amdahl's Law" (1988). Two pages each; read them together.
- Dean & Barroso, "The Tail at Scale" (CACM 2013).

**PEPs:** 659 (specializing adaptive interpreter), 744 (JIT).

### CS-621 — Algorithms and complexity

- **Core** — Kleinberg & Tardos, *Algorithm Design* (2005). The best book for *learning to
  design* algorithms rather than look them up.
- **Reference** — Cormen, Leiserson, Rivest & Stein, *Introduction to Algorithms*, 4th ed.
  The encyclopedia. Use it as one.
- **Core** — Sipser, *Introduction to the Theory of Computation*, 3rd ed. Chapters 3–5, 7.
- **Reference** — Arora & Barak, *Computational Complexity: A Modern Approach* (2009).
  Beyond scope, but the place to go next.
- **Reference** — Mitzenmacher & Upfal, *Probability and Computing*, 2nd ed. — for the
  randomized algorithms lessons.

**Required papers:**
- Turing, "On Computable Numbers, with an Application to the Entscheidungsproblem" (1936).
  Read at least §1–§8.
- Cook, "The Complexity of Theorem-Proving Procedures" (1971).
- Karp, "Reducibility Among Combinatorial Problems" (1972).

### CS-641 — Programming languages and type systems

- **Core** — Pierce, *Types and Programming Languages* (MIT Press, 2002). Chapters 3, 5,
  8–11, 15, 22–23. This is the spine of the course.
- **Core** — Nystrom, *Crafting Interpreters* (2021, free online). Build the thing.
- **Reference** — Harper, *Practical Foundations for Programming Languages*, 2nd ed.
- **Reference** — Abelson & Sussman, *Structure and Interpretation of Computer Programs*
  (free online). Chapters 3–4.
- **Reference** — Cooper & Torczon, *Engineering a Compiler*, 3rd ed.

**Required papers:**
- Milner, "A Theory of Type Polymorphism in Programming" (1978).
- Wadler, "Theorems for Free!" (FPCA 1989).
- Cardelli & Wegner, "On Understanding Types, Data Abstraction, and Polymorphism"
  (Computing Surveys, 1985).
- Reynolds, "Types, Abstraction and Parametric Polymorphism" (1983).

### DS-701 — Distributed systems

- **Core** — van Steen & Tanenbaum, *Distributed Systems*, 4th ed. (free PDF from the
  authors).
- **Core** — Kleppmann, *Designing Data-Intensive Applications* (O'Reilly, 2017).
  Chapters 5, 8, 9 for this course; 3, 7, 10–12 for DI-721.
- **Reference** — Cachin, Guerraoui & Rodrigues, *Introduction to Reliable and Secure
  Distributed Programming*, 2nd ed.

**Required papers:**
- Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (CACM 1978).
- Fischer, Lynch & Paterson, "Impossibility of Distributed Consensus with One Faulty
  Process" (JACM 1985).
- Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available,
  Partition-Tolerant Web Services" (SIGACT News, 2002).
- Lamport, "Paxos Made Simple" (2001).
- Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm" (USENIX ATC
  2014) — Raft.
- DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007).
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012).
- Waldo, Wyant, Wollrath & Kendall, "A Note on Distributed Computing" (1994).
- Shapiro et al., "Conflict-Free Replicated Data Types" (SSS 2011).

### DI-721 — Data-intensive systems

- **Core** — Kleppmann, *Designing Data-Intensive Applications*.
- **Core** — Petrov, *Database Internals* (O'Reilly, 2019).
- **Core** — Akidau, Chernyak & Lax, *Streaming Systems* (O'Reilly, 2018).
- **Reference** — Hellerstein & Stonebraker (eds.), *Readings in Database Systems*
  ("the Red Book", free online).

**Required papers:**
- O'Neil, Cheng, Gawlick & O'Neil, "The Log-Structured Merge-Tree" (1996).
- Berenson et al., "A Critique of ANSI SQL Isolation Levels" (SIGMOD 1995).
- Dean & Ghemawat, "MapReduce: Simplified Data Processing on Large Clusters" (OSDI 2004).
- Zaharia et al., "Resilient Distributed Datasets" (NSDI 2012).
- Akidau et al., "The Dataflow Model" (VLDB 2015).
- Chang et al., "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006).
- Stonebraker et al., "C-Store: A Column-oriented DBMS" (VLDB 2005).

### CA-731 — Cloud and platform architecture

- **Core** — Beyer, Jones, Petoff & Murphy (eds.), *Site Reliability Engineering* (free
  online) and *The Site Reliability Workbook*.
- **Core** — Skelton & Pais, *Team Topologies* (2019). Platform work is sociotechnical;
  this is the treatment that takes that seriously.
- **Reference** — Burns, *Designing Distributed Systems* (2018).
- **Reference** — Hausenblas & Dobies, *Programming Kubernetes*, or Ibryam & Huß,
  *Kubernetes Patterns*.
- **Reference** — Forsgren, Humble & Kim, *Accelerate* (2018) — for the measurement claims
  and their methodology, which you should read critically.

**Required papers/documents:**
- Verbitski et al., "Amazon Aurora: Design Considerations for High Throughput
  Cloud-Native Relational Databases" (SIGMOD 2017).
- Burns et al., "Borg, Omega, and Kubernetes" (ACM Queue 2016).
- Hunt, Konar, Junqueira & Reed, "ZooKeeper: Wait-free Coordination for Internet-scale
  Systems" (USENIX ATC 2010).
- Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015).

### ML-741 — ML systems engineering

- **Core** — Huyen, *Designing Machine Learning Systems* (O'Reilly, 2022).
- **Reference** — Lakshmanan, Robinson & Munn, *Machine Learning Design Patterns* (2020).
- **Reference** — Ameisen, *Building Machine Learning Powered Applications* (2020).

**Required papers:**
- Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015).
- Breck et al., "The ML Test Score" (IEEE Big Data 2017).
- Amershi et al., "Software Engineering for Machine Learning: A Case Study" (ICSE 2019).
- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"
  (NeurIPS 2020).

### FM-751 — Formal methods

- **Core** — Lamport, *Specifying Systems* (2002, free from the author).
- **Core** — Wayne, *Practical TLA+* (Apress, 2018).
- **Reference** — Nipkow & Klein, *Concrete Semantics* (free online) — if you want to go
  further into proof assistants.
- **Reference** — Bradley & Manna, *The Calculus of Computation* — for the SMT lessons.

**Required papers:**
- Hoare, "An Axiomatic Basis for Computer Programming" (CACM 1969).
- Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015).
- Lamport, "The Temporal Logic of Actions" (TOPLAS 1994) — read §1–3.

---

## The twelve-paper core

If you read nothing else, read these, in this order, over the six terms:

1. Parnas (1972) — modularity
2. Brooks (1986) — essence vs accident
3. Hoare (1969) — axiomatic basis
4. Lamport (1978) — time and clocks
5. Milner (1978) — polymorphism
6. Lampson (1983) — hints for system design
7. FLP (1985) — impossibility of consensus
8. Cardelli & Wegner (1985) — types and abstraction
9. Wadler (1989) — theorems for free
10. Herlihy & Wing (1990) — linearizability
11. Waldo et al. (1994) — a note on distributed computing
12. Dean & Barroso (2013) — the tail at scale

Together they are about 200 pages and they contain most of what the field actually knows.
