# Scalability — laws of parallel speedup

Every act of horizontal scaling runs into the same arithmetic: the speedup you can buy by adding machines is bounded, and beyond a point, it is *negative*. Two laws describe these bounds. **Amdahl's Law** (1967) puts a ceiling on how much parallelism can compress a fixed workload; **Universal Scalability Law** (Gunther, 1993) extends Amdahl with a coordination penalty that bends the curve down past an optimum cluster size. Between them, they explain why shared-nothing wins, why eventual consistency exists, and why no production cluster runs on the strategy of *"just add more nodes."*

This document is the reference companion to §3.6 of [01-fundamentals](./README.md). It carries the formulas and the practical interpretations; the chapter itself stays at three-takeaways density.

## Amdahl's Law — the ceiling on parallel speedup

Gene Amdahl's 1967 paper observed that any program has some fraction of work that cannot be parallelized — initialization, serialization at a critical section, a single coordinator's decision, a final aggregation step. No matter how many processors are available, that fraction runs on one processor and bounds the achievable speedup.

### The formula

Let `s` be the serial fraction of the work and `p = 1 − s` the parallel fraction. With `N` parallel workers:

```
                 1
Speedup(N) = ─────────────
              s + p / N
```

In the limit:

```
                  1
max Speedup = ─────
                s
```

The maximum is independent of `N`. Adding workers cannot push past `1/s`.

### Numerical intuition

| Serial fraction `s` | N=10 | N=100 | N=1000 | N=∞ |
|---|---|---|---|---|
| 50% | 1.8× | 2.0× | 2.0× | 2× |
| 10% | 5.3× | 9.2× | 9.9× | 10× |
| 5% | 6.9× | 16.8× | 19.6× | 20× |
| 1% | 9.2× | 50.2× | 91.0× | 100× |
| 0.1% | 9.9× | 90.9× | 500.3× | 1000× |

A workload that is 95% parallelizable still tops out at 20× — *the remaining 5% serial fraction caps the system*. A 99% parallelizable workload reaches 91× at 1,000 nodes; the additional 999 nodes recover only ~9× of speedup over a perfectly used 100. The marginal value of an added node falls off a cliff long before the asymptote.

### The shape of the curve

```
Speedup
       │              ┌─── asymptote = 1/s
   1/s │           ___/
       │       ___/
       │     _/
       │    /
       │  _/
       │ /
     1 └──────────────────────────► N
       1   10   100   1000
```

The curve looks linear at small N, then saturates as the serial fraction starts dominating. The exact knee is where `p/N` becomes comparable to `s`.

### Engineering interpretation

The practical lesson is not *"how do we estimate `s`?"* but *"where does the serial fraction live, and can we kill it?"* Typical sources of serial work in distributed systems:

- **A single primary handling all writes.** Read replicas parallelize reads; the primary serializes writes. Write-heavy systems must shard the primary itself, because no amount of replicas removes the serial bottleneck.
- **Distributed locks held during a critical section.** Whatever a held lock protects runs serially across the system.
- **A singleton coordinator** — leader, scheduler, sequencer. Capacity is bounded by the coordinator's throughput.
- **Process initialization and configuration loading** — small in absolute terms, but rolling restarts pay it N times.
- **The reduce step in MapReduce-style aggregations.** Map is parallel; reduce is often less so, sometimes single-node.

The engineering exercise is to identify the longest-running serial operation in the request path. That operation is the scaling ceiling. Optimizing the parallel part is secondary; the parallel part already divides by N.

## Gustafson's Law — when the workload grows with the cluster

John Gustafson (1988) argued that Amdahl's framing is too narrow. In Amdahl's setup the workload is fixed; in practice, larger clusters run *larger* workloads. A search engine with ten million documents may scale to a billion when the cluster scales 100×. The user-perceived metric is not *"how fast did the same job get?"* but *"did per-user latency stay flat as both users and machines grew?"*

This distinction is named:

- **Strong scaling** (Amdahl's view) — fixed workload, more nodes; the goal is to finish the same work faster. Bounded by `1/s`.
- **Weak scaling** (Gustafson's view) — workload and node count grow together; the goal is to hold per-unit time constant. Achievable in workloads where work is naturally divisible (web traffic, MapReduce batches, embarrassingly parallel scientific computing).

Most user-facing systems live in the weak-scaling regime. As traffic grows, capacity is added at the same time, and the design goal is that each user's latency does not degrade. Strong scaling matters in HPC and in batch jobs with hard deadlines on a fixed dataset.

The two laws are not contradictions; they answer different questions. Amdahl bounds *fixed-workload speedup*; Gustafson points out that fixed-workload is not the most common framing.

## Universal Scalability Law — coordination as the second drag

Amdahl assumes that the parallel portion divides perfectly: with N workers, parallel work runs in `1/N` of the time. The empirical reality is worse: at large N the parallel work itself slows down because the workers spend time *coordinating* with each other.

Neil Gunther's Universal Scalability Law (1993) formalizes this with two coefficients:

```
                          N
C(N) = ────────────────────────────────────────
         1 + α (N − 1) + β N (N − 1)
```

- `α` (contention) — the cost of competing for a shared resource. A generalization of Amdahl's serial fraction. With `α > 0` and `β = 0`, USL behaves like Amdahl.
- `β` (coherence) — the cost of keeping nodes consistent with each other. Communication between every pair of nodes scales with `N²`; this term captures that quadratic cost.

The shape of the curve is the part that matters:

```
Throughput
         │       ┌── peak (optimum N*)
         │      / \
         │     /   \___
         │    /        \____
         │   /              \____  ← negative scaling
         │  /
         │ /
         │/
         └───────────────────────► N
              N*
```

Past the optimum `N*`, **adding nodes decreases throughput.** The system is spending more on coordination than the new nodes contribute. This is not a theoretical curiosity; production systems hit the inflection regularly, and capacity tests that fit USL parameters to real measurements consistently find `N*` smaller than the design target.

### What `β` looks like in real systems

The coherence term is concrete:

- **Distributed cache invalidation.** A write must invalidate cache entries on N nodes. Pub/sub fanout grows with N; replica latency grows with N.
- **Service mesh and gossip-based health membership.** Each node tracks every other node's state — *N²* messages.
- **Synchronous write replication.** A write commits only after all replicas acknowledge. Adding replicas *increases* write latency; reliability and write throughput pull in opposite directions.
- **Distributed consensus (Raft, Paxos).** Reaching quorum requires *N/2 + 1* acknowledgments. The cost per decision rises with N. This is why Raft and Paxos clusters are typically 3, 5, or 7 nodes — beyond that the coordination cost erases the benefit.
- **Cross-shard transactions.** A transaction touching K shards requires K-way coordination (two-phase commit or equivalent); cost rises faster than linearly in K.
- **Lock contention.** Many writers competing for the same row, partition, or queue head. The contention is `α`, not `β` — but the symptom is identical: nodes waiting instead of working.

### Why USL changes architectural decisions

If `β > 0`, the question is no longer *"how big can we scale?"* but *"where is the optimum, and how do we extend it?"* Three architectural patterns are direct responses to USL:

- **Shared-nothing** removes most of the `β` term by eliminating shared state. Nodes have nothing to coordinate about; coherence cost approaches zero. This is the deep reason why every web-scale system (Cassandra, DynamoDB, S3, Kafka) is shared-nothing.
- **Eventual consistency** lets nodes proceed without waiting for global agreement on every write. Coordination becomes asynchronous; `β` falls. The CAP and PACELC trade-offs in chapter 6 are the formal version of this same physics.
- **Partitioning (sharding)** is the most powerful single technique, because it cuts coherence cost categorically. State split into K independent shards has its `β` term effectively divided by K — a single global database with `β = 0.001` becomes K databases with `β = 0.001` each, *per shard*. Cross-shard work is minimized to keep that property.

USL also explains why teams rediscover the same patterns: every distributed system that needs to grow eventually pushes against the coherence ceiling, and the only escape is to remove the thing that requires coherence.

## Practical implications

The two laws condense into three rules of thumb:

1. **Find the serial fraction first.** Profile, trace, measure. Whatever shows up as serial — a primary, a coordinator, a lock — is the scaling ceiling. Optimizing parallel work past this point yields nothing.
2. **Treat coordination as the enemy of scalability.** Synchronous replication, global locks, two-phase commit, gossip across the fleet — each is a `β` contribution. Eliminate where possible; relax where elimination is impossible (eventual consistency, async replication).
3. **Partition aggressively.** A single global resource scales worse than K independent partitions even at the same total capacity, because the coherence cost shrinks per partition.

The general result, applied to almost every chapter of this handbook:

- *Why eventual consistency exists* — `β` for strong consistency is too high to scale across regions and large clusters (chapter 6).
- *Why microservices push for stateless, asynchronous communication* — to drive `β` toward zero (chapter 4).
- *Why CDNs work the way they do* — edge nodes are nearly independent, with almost no coherence requirement; that is the entire architectural reason caching at the edge scales (chapter 5).
- *Why consensus clusters stay small* — Raft and Paxos pay `β` directly, so 3–7 nodes is the practical ceiling (chapter 6).
- *Why "deploy more nodes" is rarely the answer to a scaling problem* — past `N*`, more nodes hurt; the answer is to reduce coordination, not add capacity (chapter 9).

## Summary

- **Amdahl's Law** — fixed workload, parallel speedup capped at `1/s`. Identifies the *serial fraction* as the scaling ceiling.
- **Gustafson's Law** — when workload grows with the cluster, the relevant question is per-unit latency, not total speedup. *Weak scaling* matters more than *strong scaling* in user-facing systems.
- **Universal Scalability Law** — adds coordination cost to Amdahl. Past optimum N\*, adding nodes makes the system slower. The architectural answer is *shared-nothing + eventual consistency + partitioning.*

The single sentence to keep:

> *Linear scaling does not exist. Beyond some N, more nodes are worse. Find the serial fraction, eliminate coordination, partition.*
