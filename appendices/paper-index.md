# Appendix — Paper Index

Every primary source cited in the programme, organised so you can find one by topic and know why it
matters. Where a paper is freely available from the author or a conference, it usually is — search
the title.

**How to use this.** Not as a reading list to work through. When a lesson cites a paper, read that
one then. This index exists for the moment when you need to find something again, or want to know
what else is worth reading in an area.

**The twelve-paper core** is marked ★. If you read nothing else from this index, read those. They
are the ones whose ideas recur in every other paper here.

---

## 1. Foundations of distributed systems

★ **Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (CACM 1978).**
Happens-before, logical clocks, and the insight that ordering is causal rather than temporal.
Everything in DS-701 L02 descends from it. *(DS-701 L02)*

★ **Fischer, Lynch & Paterson, "Impossibility of Distributed Consensus with One Faulty Process"
(JACM 1985).** FLP. Short, and the result that bounds what any protocol can promise. *(DS-701 L01,
L05)*

★ **Waldo, Wyant, Wollrath & Kendall, "A Note on Distributed Computing" (Sun, 1994).** Why local and
remote calls are not the same thing and why pretending otherwise fails. Ages extremely well.
*(DS-701 L01)*

**Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available,
Partition-Tolerant Web Services" (SIGACT News 2002).** The formal CAP result. *(DS-701 L04)*

**Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed" (Computer 2012).** The author's own
correction of the folklore. *(DS-701 L04)*

**Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (Computer 2012).**
PACELC. *(DS-701 L04)*

★ **Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects"
(TOPLAS 1990).** The definition, and the composability result. *(DS-701 L04, FM-751 L07)*

**Chandra & Toueg, "Unreliable Failure Detectors for Reliable Distributed Systems" (JACM 1996).**
Failure detectors as a first-class abstraction; the ◇S result. *(DS-701 L09)*

**Skeen & Stonebraker, "A Formal Model of Crash Recovery in a Distributed System" (TSE 1983).** The
blocking result for atomic commit. *(DS-701 L08)*

**Alpern & Schneider, "Defining Liveness" (IPL 1985)** and "Recognizing Safety and Liveness"
(Distributed Computing 1987). The decomposition theorem. *(FM-751 L01, L03)*

## 2. Consensus and replication

★ **Lamport, "Paxos Made Simple" (SIGACT News 2001).** *(DS-701 L05)*

★ **Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm" (USENIX ATC 2014).**
Raft, and an explicit argument that understandability is a design goal. Read with the TLA+
specification. *(DS-701 L05, FM-751 L06)*

**Lamport, "The Part-Time Parliament" (TOPLAS 1998).** The original Paxos paper. Famous, and best
read after "Paxos Made Simple".

**Gray & Lamport, "Consensus on Transaction Commit" (TODS 2006).** 2PC's relationship to consensus,
and Paxos Commit. *(DS-701 L08)*

**van Renesse & Altinbuken, "Paxos Made Moderately Complex" (ACM CSUR 2015).** The implementation
detail the original omits.

**DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007).** Quorums,
vector clocks, hinted handoff, and the trade-offs stated honestly. *(DS-701 L03)*

★ **Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012).** TrueTime,
external consistency, and 2PC done correctly over Paxos groups. *(DS-701 L02, L08)*

**Kulkarni et al., "Logical Physical Clocks" (OPODIS 2014).** Hybrid logical clocks. *(DS-701 L02)*

**Das, Gupta & Motivala, "SWIM: Scalable Weakly-consistent Infection-style Process Group Membership
Protocol" (DSN 2002).** *(DS-701 L06)*

**Karger et al., "Consistent Hashing and Random Trees" (STOC 1997).** *(DS-701 L06)*

**Hayashibara et al., "The φ Accrual Failure Detector" (SRDS 2004).** *(DS-701 L09)*

## 3. Coordination-free systems

★ **Shapiro, Preguiça, Baquero & Zawirski, "Conflict-Free Replicated Data Types" (SSS 2011).** The
catalogue and the lattice argument. *(DS-701 L07)*

**Hellerstein & Alvaro, "Keeping CALM: When Distributed Consistency is Easy" (CACM 2020).**
Monotonicity as the condition for coordination freedom. *(DS-701 L07)*

**Bailis et al., "Coordination Avoidance in Database Systems" (VLDB 2014).** Invariant confluence and
the escrow analysis. *(DS-701 L07)*

**Kleppmann & Beresford, "A Conflict-Free Replicated JSON Datatype" (TPDS 2017).** *(DS-701 L07)*

**Kleppmann, Wiggins, van Hardenberg & McGranaghan, "Local-First Software" (Onward! 2019).**
*(DS-701 L07)*

**Bailis et al., "Highly Available Transactions: Virtues and Limitations" (VLDB 2014).** Which
isolation levels survive without coordination. *(DI-721 L04)*

**Garcia-Molina & Salem, "Sagas" (SIGMOD 1987).** Short, and still the clearest statement.
*(DS-701 L08)*

**Helland, "Life beyond Distributed Transactions: An Apostate's Opinion" (CIDR 2007).** *(DS-701 L08)*

## 4. Reliability and operations

★ **Dean & Barroso, "The Tail at Scale" (CACM 2013).** Fan-out arithmetic, hedged and tied requests.
*(DS-701 L09, ML-741 L05)*

**Bronson, Aghayev, Charapko & Zhu, "Metastable Failures in Distributed Systems" (HotOS 2021).**
Short, and it gives you the vocabulary for most large outages. *(DS-701 L09)*

**Hamilton, "On Designing and Deploying Internet-Scale Services" (LISA 2007).** Old, mostly still
right. *(CA-731 L08)*

**Sigelman et al., "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Google 2010).**
*(DS-701 L10)*

**Basiri et al., "Chaos Engineering" (IEEE Software 2016)** and the *Principles of Chaos Engineering*
statement. *(DS-701 L10)*

**Verma et al., "Large-scale cluster management at Google with Borg" (EuroSys 2015)** and **Burns,
Grant, Oppenheimer, Brewer & Wilkes, "Borg, Omega, and Kubernetes" (ACM Queue 2016).** *(CA-731 L06)*

**Agache et al., "Firecracker: Lightweight Virtualization for Serverless Applications" (NSDI 2020).**
*(CA-731 L02)*

**Zhou et al., "FoundationDB: A Distributed Unbundled Transactional Key Value Store" (SIGMOD 2021).**
Read the deterministic simulation testing section. *(DS-701 L10, FM-751)*

## 5. Storage engines and databases

★ **Hellerstein, Stonebraker & Hamilton, "Architecture of a Database System" (FnTDB 2007).** The map
of what is inside a database. *(DI-721 L01)*

★ **O'Neil, Cheng, Gawlick & O'Neil, "The Log-Structured Merge-Tree" (Acta Informatica 1996).**
*(DI-721 L02)*

**Athanassoulis et al., "Designing Access Methods: The RUM Conjecture" (EDBT 2016).** Short, and the
best organising frame for storage design. *(DI-721 L02)*

**Lehman & Yao, "Efficient Locking for Concurrent Operations on B-Trees" (TODS 1981).** B-link trees.
*(DI-721 L02)*

**Graefe, "Modern B-Tree Techniques" (FnTDB 2011)** and **"Query Evaluation Techniques for Large
Databases" (ACM CSUR 1993).** Both comprehensive and both still the reference. *(DI-721 L02, L07)*

**Dayan, Athanassoulis & Idreos, "Monkey: Optimal Navigable Key-Value Store" (SIGMOD 2017).**
*(DI-721 L02)*

**Dong et al., "Optimizing Space Amplification in RocksDB" (CIDR 2017).** Production numbers.
*(DI-721 L02)*

**Bender et al., "An Introduction to Bε-trees and Write-Optimization" (;login: 2015).** *(DI-721 L02)*

**Mohan et al., "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking"
(TODS 1992).** *(DI-721 L01)*

**Aggarwal & Vitter, "The Input/Output Complexity of Sorting and Related Problems" (CACM 1988).** The
external memory model. *(DI-721 L01)*

**Selinger et al., "Access Path Selection in a Relational Database Management System" (SIGMOD 1979).**
The System R optimiser; the shape of every optimiser since. *(DI-721 L01, L03)*

**Leis et al., "How Good Are Query Optimizers, Really?" (VLDB 2015).** The empirical answer.
*(DI-721 L01, L03)*

**Gray & Putzolu, "The 5 Minute Rule for Trading Memory for Disc Accesses" (1987)** and its periodic
restatements. *(DI-721 L01)*

## 6. Transactions and isolation

★ **Berenson, Bernstein, Gray, Melton, O'Neil & O'Neil, "A Critique of ANSI SQL Isolation Levels"
(SIGMOD 1995).** Read before trusting any isolation documentation. *(DI-721 L04)*

**Adya, Liskov & O'Neil, "Generalized Isolation Level Definitions" (ICDE 2000).** The
implementation-independent definitions. *(DI-721 L04)*

**Fekete, Liarokapis, O'Neil, O'Neil & Shasha, "Making Snapshot Isolation Serializable" (TODS 2005).**
The dangerous-structure theorem. *(DI-721 L05)*

**Cahill, Röhm & Fekete, "Serializable Isolation for Snapshot Databases" (SIGMOD 2008).** SSI.
*(DI-721 L05)*

**Ports & Grittner, "Serializable Snapshot Isolation in PostgreSQL" (VLDB 2012).** Candid about the
trade-offs. *(DI-721 L05)*

**Wu et al., "An Empirical Evaluation of In-Memory Multi-Version Concurrency Control" (VLDB 2017).**
*(DI-721 L05)*

## 7. Analytical and streaming systems

**Abadi, Boncz & Harizopoulos, "The Design and Implementation of Modern Column-Oriented Database
Systems" (FnTDB 2013).** The definitive survey. *(DI-721 L06)*

**Stonebraker et al., "C-Store: A Column-oriented DBMS" (VLDB 2005)** and Lamb et al., "The Vertica
Analytic Database" (VLDB 2012). *(DI-721 L06)*

**Abadi, Madden & Ferreira, "Integrating Compression and Execution in Column-Oriented Database
Systems" (SIGMOD 2006).** Operating on compressed data. *(DI-721 L06)*

**Boncz, Zukowski & Nes, "MonetDB/X100: Hyper-Pipelining Query Execution" (CIDR 2005).** Vectorised
execution. *(DI-721 L06)*

**Ailamaki et al., "Weaving Relations for Cache Performance" (VLDB 2001).** PAX. *(DI-721 L06)*

**Melnik et al., "Dremel: Interactive Analysis of Web-Scale Datasets" (VLDB 2010).** *(DI-721 L06)*

**Armbrust et al., "Delta Lake" (VLDB 2020)** and "Lakehouse" (CIDR 2021). *(DI-721 L06)*

★ **Akidau et al., "The Dataflow Model" (VLDB 2015).** The four questions; the paper that settled
event-time thinking. *(DI-721 L08)*

**Dean & Ghemawat, "MapReduce" (OSDI 2004)** and **Zaharia et al., "Resilient Distributed Datasets"
(NSDI 2012).** *(DI-721 L07)*

**Carbone et al., "Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015)** and
**Chandy & Lamport, "Distributed Snapshots" (TOCS 1985).** *(DI-721 L08)*

**Sax et al., "Streams and Tables: Two Sides of the Same Coin" (BIRTE 2018).** *(DI-721 L09)*

**Stonebraker & Çetintemel, "'One Size Fits All': An Idea Whose Time Has Come and Gone" (ICDE 2005).**
*(DI-721 L01)*

## 8. Programming languages and types

**Milner, "A Theory of Type Polymorphism in Programming" (JCSS 1978)** and Damas & Milner,
"Principal Type-Schemes for Functional Programs" (POPL 1982). Hindley–Milner. *(CS-641 L04)*

**Wright & Felleisen, "A Syntactic Approach to Type Soundness" (Information and Computation 1994).**
Progress and preservation. *(CS-641 L03)*

**Siek & Taha, "Gradual Typing for Functional Languages" (Scheme Workshop 2006).** *(CS-641 L08,
SE-511 L06)*

**Wadler, "Monads for Functional Programming" (1995)** and "The Essence of Functional Programming"
(POPL 1992). *(CS-641 L07)*

**Wadler, "Theorems for Free!" (FPCA 1989).** Parametricity. *(CS-641 L05)*

**Cardelli & Wegner, "On Understanding Types, Data Abstraction, and Polymorphism" (ACM CSUR 1985).**
*(CS-641 L05)*

**Reynolds, "Types, Abstraction and Parametric Polymorphism" (1983).** *(CS-641 L05)*

**Wadler, "The Expression Problem" (1998, email).** Two paragraphs, and it names a real tension.
*(CS-641 L06)*

## 9. Algorithms and complexity

**Cook, "The Complexity of Theorem-Proving Procedures" (STOC 1971).** NP-completeness. *(CS-621 L08)*

**Karp, "Reducibility Among Combinatorial Problems" (1972).** The twenty-one problems. *(CS-621 L08)*

**Turing, "On Computable Numbers…" (1936).** *(CS-621 L10)*

**Rice, "Classes of Recursively Enumerable Sets and Their Decision Problems" (1953).** Why no tool
can decide any non-trivial semantic property. *(CS-621 L10, SE-511 L08)*

**Flajolet et al., "HyperLogLog" (AofA 2007)** and Cormode & Muthukrishnan, "An Improved Data Stream
Summary: The Count-Min Sketch" (2005). *(CS-621 L07)*

**Tarjan, "Amortized Computational Complexity" (SIAM J. Alg. Disc. Meth. 1985).** The potential
method. *(CS-621 L01)*

## 10. Software engineering

★ **Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM 1972).** Four
pages; information hiding; still the most important paper in software design. *(SE-521 L01)*

**Parnas, "On the Design and Development of Program Families" (TSE 1976).** *(SE-521 L01)*

**Brooks, "No Silver Bullet" (Computer 1987).** Essential versus accidental complexity.

**Lehman, "Programs, Life Cycles, and Laws of Software Evolution" (Proc. IEEE 1980).**
*(SE-521 L08)*

**Fowler, "Who Needs an Architect?" (IEEE Software 2003).** *(CA-731 L10)*

**Nygard, "Documenting Architecture Decisions" (2011).** ADRs. *(SE-521 L10)*

**DeMillo, Lipton & Sayward, "Hints on Test Data Selection" (Computer 1978).** Mutation testing's
origin. *(SE-511 L03)*

**Claessen & Hughes, "QuickCheck" (ICFP 2000).** *(SE-511 L04, FM-751 L08)*

**Beizer, *Software Testing Techniques*** and Meszaros, *xUnit Test Patterns* — the test double
taxonomy. *(SE-511 L02)*

## 11. Formal methods

★ **Hoare, "An Axiomatic Basis for Computer Programming" (CACM 1969).** Six pages; it founded the
field. *(FM-751 L02)*

**Dijkstra, *A Discipline of Programming* (1976).** Weakest preconditions. *(FM-751 L02)*

**Lamport, "The Temporal Logic of Actions" (TOPLAS 1994)** and *Specifying Systems* (2002).
*(FM-751 L03, L05)*

★ **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015).** The industrial
case, honestly reported. *(FM-751, throughout)*

**Abadi & Lamport, "The Existence of Refinement Mappings" (TCS 1991).** History and prophecy
variables. *(FM-751 L07)*

**Clarke, Emerson & Sifakis, Turing Award lecture, "Model Checking" (CACM 2009).** *(FM-751 L04)*

**Clarke, Grumberg, Jha, Lu & Veith, "Counterexample-Guided Abstraction Refinement" (CAV 2000).**
*(FM-751 L04)*

**Zave, "Using Lightweight Modeling to Understand Chord" (SIGCOMM CCR 2012).** A published protocol
shown incorrect. *(FM-751 L01, L06)*

**de Gouw et al., "OpenJDK's `java.utils.Collection.sort()` Is Broken" (CAV 2015).** *(FM-751 L02)*

**Klein et al., "seL4: Formal Verification of an OS Kernel" (SOSP 2009)** and **Leroy, "Formal
Verification of a Realistic Compiler" (CACM 2009).** The extreme end, with honest costs.
*(FM-751 L02)*

**Hawblitzel et al., "IronFleet: Proving Practical Distributed Systems Correct" (SOSP 2015).**
*(FM-751 L07)*

**de Moura & Bjørner, "Z3: An Efficient SMT Solver" (TACAS 2008).** *(FM-751 L09)*

**Backes et al., "Semantic-based Automated Reasoning for AWS Access Policies using SMT" (2018).**
Zelkova; formal methods deployed invisibly at scale. *(FM-751 L09)*

**Cadar, Dunbar & Engler, "KLEE" (OSDI 2008).** *(FM-751 L09)*

**Desai et al., "P: Safe Asynchronous Event-Driven Programming" (PLDI 2013).** *(FM-751 L04)*

## 12. Machine learning systems

★ **Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015).** Eight
pages; the spine of ML-741. *(ML-741, throughout)*

**Breck, Cai, Nielsen, Salib & Sculley, "The ML Test Score" (IEEE Big Data 2017).** *(ML-741 L01)*

★ **Bottou et al., "Counterfactual Reasoning and Learning Systems" (JMLR 2013).** Feedback loops and
off-policy evaluation. Hard, and the most valuable paper in ML-741. *(ML-741 L08)*

**Polyzotis, Zinkevich, Roy, Breck & Whang, "Data Validation for Machine Learning" (MLSys 2019).**
*(ML-741 L02)*

**Kaufman, Rosset & Perlich, "Leakage in Data Mining" (KDD 2011).** *(ML-741 L02)*

**Bergstra & Bengio, "Random Search for Hyper-Parameter Optimization" (JMLR 2012).** *(ML-741 L03)*

**Goyal et al., "Accurate, Large Minibatch SGD" (2017).** Linear scaling and warmup. *(ML-741 L04)*

**Rajbhandari et al., "ZeRO" (SC 2020).** *(ML-741 L04)*

**Jeon et al., "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads"
(USENIX ATC 2019).** Where the utilisation actually goes. *(ML-741 L04)*

**Crankshaw et al., "Clipper" (NSDI 2017)** and Yu et al., "Orca" (OSDI 2022). Serving and
continuous batching. *(ML-741 L05)*

**Blalock et al., "What is the State of Neural Network Pruning?" (MLSys 2020).** Read before
believing any pruning claim. *(ML-741 L05)*

**Gama et al., "A Survey on Concept Drift Adaptation" (ACM CSUR 2014).** The taxonomy.
*(ML-741 L06)*

**Rabanser, Günnemann & Lipton, "Failing Loudly" (NeurIPS 2019).** Which shift detectors actually
work. *(ML-741 L06)*

**Guo, Pleiss, Sun & Weinberger, "On Calibration of Modern Neural Networks" (ICML 2017).**
*(ML-741 L05, L07)*

**Kohavi et al., "Online Controlled Experiments at Large Scale" (KDD 2013)** and "Seven Rules of
Thumb for Web Site Experimenters" (KDD 2014). *(ML-741 L07)*

**Deng, Xu, Kohavi & Walker, "Improving the Sensitivity of Online Controlled Experiments…"
(WSDM 2013).** CUPED. *(ML-741 L07)*

**Lum & Isaac, "To Predict and Serve?" (Significance 2016)** and Ensign et al., "Runaway Feedback
Loops in Predictive Policing" (FAT* 2018). *(ML-741 L08)*

**Chaney, Stewart & Engelhardt, "How Algorithmic Confounding in Recommendation Systems Increases
Homogeneity and Decreases Utility" (RecSys 2018).** *(ML-741 L08)*

**Lewis et al., "Retrieval-Augmented Generation…" (NeurIPS 2020)**; **Zheng et al., "Judging
LLM-as-a-Judge…" (NeurIPS 2023)**; **Greshake et al., "Not What You've Signed Up For" (AISec 2023).**
*(ML-741 L09)*

**Mitchell et al., "Model Cards" (FAT* 2019)** and Gebru et al., "Datasheets for Datasets"
(CACM 2021). *(ML-741 L10)*

**Sambasivan et al., "'Everyone wants to do the model work, not the data work'" (CHI 2021).**
*(ML-741 L01, L02)*

---

## The twelve-paper core (★)

If you read only twelve papers from this programme, these:

1. Lamport, "Time, Clocks, and the Ordering of Events" (1978)
2. Fischer, Lynch & Paterson, "Impossibility of Distributed Consensus…" (1985)
3. Waldo et al., "A Note on Distributed Computing" (1994)
4. Herlihy & Wing, "Linearizability" (1990)
5. Lamport, "Paxos Made Simple" (2001) — or Ongaro & Ousterhout, "Raft" (2014)
6. Corbett et al., "Spanner" (2012)
7. Shapiro et al., "Conflict-Free Replicated Data Types" (2011)
8. Dean & Barroso, "The Tail at Scale" (2013)
9. Hellerstein, Stonebraker & Hamilton, "Architecture of a Database System" (2007)
10. Berenson et al., "A Critique of ANSI SQL Isolation Levels" (1995)
11. Akidau et al., "The Dataflow Model" (2015)
12. Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (1972)

Plus, for the practice rather than the theory: Hoare (1969), Sculley et al. (2015), and Newcombe et
al. (2015) — three short papers that between them change how you write code, build ML systems, and
design protocols.

## How to read a paper

The three-pass method (Keshav) works and takes practice:

1. **First pass, five minutes.** Title, abstract, introduction, section headings, conclusions,
   references you recognise. Decide whether to continue.
2. **Second pass, an hour.** Read carefully, ignore proofs. Look at figures and tables — are the
   axes labelled, are the baselines fair? Note terms you did not understand and references to
   follow.
3. **Third pass, several hours.** Reconstruct the paper: re-derive the result with the same
   assumptions. This is how you find the unstated assumptions, and it is the only pass that
   produces the ability to critique.

For this programme: first pass on everything cited, second pass on the papers a lesson calls
required, third pass on the twelve-paper core and on anything your capstone depends on.

And the habit that matters most: **read the evaluation section adversarially.** What was the
baseline? On what hardware? With what workload? What is *not* shown? A paper's most interesting
content is often in what its evaluation declines to measure.
