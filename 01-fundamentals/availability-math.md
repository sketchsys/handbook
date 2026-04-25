# Availability math

The arithmetic for predicting and reasoning about service availability. Three things:

- The availability formula relating MTBF and MTTR to A.
- Composition rules — how a system's availability is built from its components (series and parallel).
- The "9's" downtime table — what an availability target costs in real-world downtime.

This page is a reference card. The conceptual treatment of fault, failure, and the metrics behind these formulas lives in [§2 of the chapter](./README.md#2-reliability--deep-dive).

## MTBF, MTTR, and availability

**MTBF — Mean Time Between Failures.** The average time a component runs between failures. A *fleet* metric — for a single component it has no "now" interpretation. Vendors quote disk MTBF in the 1–2.5M-hour range; this means *"in a fleet of 100,000 disks, expect ~580 failures per year,"* not *"this disk will live 100 years."*

Two close cousins:

- **MTTF** — Mean Time To Failure. Used for non-repairable components (disks, SSDs, fans). Once it dies, you replace it.
- **MTBF** — used for repairable components (servers, services). They die and come back.
- In practice the terms blur; vendor docs use MTBF even for disks. Know the distinction; don't fight over it.

**MTTR — Mean Time To Recovery.** The average time from failure start to service restoration. Four phases summed:

```
detect    →  monitoring/alerting notices the failure
respond   →  the right person/system starts acting
repair    →  the actual fix — failover, rollback, restart
verify    →  confirming the system is back
```

Each phase is a distinct discipline. *Detect* is observability (chapter 11), *respond* is on-call + runbooks, *repair* is architecture + automation, *verify* is health checks + smoke tests. To shrink MTTR you must first measure which phase dominates.

A note on the "R": MTTR is variously expanded as **R**ecovery, **R**epair, **R**estore, or **R**esolve. They differ subtly (resolution may include root-cause work; recovery may be just service restoration). Pick a definition and stay consistent.

**Availability.**

```
            MTBF
A  =  ─────────────────
        MTBF + MTTR
```

Availability is the fraction of time the system is up. Three illustrative rows:

| MTBF | MTTR | Availability |
|---|---|---|
| 1,000 h | 10 h | 99.0% |
| 1,000 h | 1 h | 99.9% |
| 10,000 h | 1 h | 99.99% |

Compare row 1 → row 2: the same 10× availability gain comes from cutting MTTR 10×, with MTBF unchanged. The strategic point: **you can attack reliability from either side, and MTTR is usually the cheaper side.** Modern SRE practice — the **DORA** metrics (MTTR + Change Failure Rate + Deploy Frequency + Lead Time) — reflects this. MTBF is conspicuously absent; the modern bet is on fast recovery rather than rare faults.

## The 9's — availability vs allowed downtime

Each additional nine cuts allowed downtime by 10×.

| Availability | Allowed downtime / year | Allowed downtime / day |
|---|---|---|
| 99% | 3.65 days | ~14.4 min |
| 99.9% | 8.76 hours | ~1.44 min |
| 99.99% | 52.6 min | ~8.6 sec |
| 99.999% | 5.26 min | ~0.86 sec |
| 99.9999% | 31.5 sec | ~86 ms |

Google's SRE book formalizes the rule of thumb: *"An extra nine of availability costs roughly 10× more to achieve."* The five stacking forces behind that multiplier are unpacked in [§1 follow-up 1](./README.md#1-why-does-each-9-of-availability-cost-10-more).

## Series composition — the dependency chain

If a request must traverse N components in series, all must be up:

```
A_total  =  A_1  ×  A_2  ×  …  ×  A_n
```

Worked example for a typical web request:

| Component | Availability |
|---|---|
| API gateway | 99.99% |
| Auth service | 99.95% |
| Application service | 99.99% |
| Database | 99.99% |
| Cache | 99.9% |

Total = 0.9999 × 0.9995 × 0.9999 × 0.9999 × 0.999 = **0.9982 ≈ 99.82%**

**The dependency chain ceiling.** The total cannot exceed the weakest link. The cache at 99.9% caps the system at 99.9% no matter what you do upstream. To raise the ceiling:

1. **Strengthen the weakest link** — usually expensive, often vendor-dependent.
2. **Remove the dependency from the critical path** — convert it from hard to soft. (The cache becomes optional with a database fallback.)
3. **Parallelize the dependency** — multiple cache instances behind one logical interface.

### Soft vs hard dependencies

Not every series dependency is equal:

- **Hard** — if it fails, the request fails. Auth, payment authorization, primary database write.
- **Soft** — if it fails, the request still completes, possibly with degraded quality. Cache (miss → DB), recommendation service (down → empty list), search ranking (down → chronological fallback).

A soft dependency in a failed state is treated as A = 1 in the chain — it doesn't tax the product. Mature architecture converts hard dependencies into soft ones wherever possible.

Concrete examples:

- Twitter timelines serve a stale ranking when the ranking service is down.
- Stripe accepts payments and pushes fraud-detection to an async pipeline if it slows down.
- DNS resolvers serve stale records past TTL during upstream outages (RFC 8767's *serve-stale*).

## Parallel composition — redundancy

If you have N replicas and need only one to be available, all must fail simultaneously to fail the system. Under the assumption of **independent** failures with per-replica failure probability *p* = 1 − A:

```
A_total  =  1  −  (1 − A)^N
```

For A = 99% (*p* = 0.01):

| Replicas | A_total |
|---|---|
| 1 | 99% |
| 2 | 99.99% |
| 3 | 99.9999% |
| 4 | 99.999999% |

Each replica multiplies failure probability by *p*. This is why redundancy is so powerful — *when independence holds*. When it doesn't, the formula degrades toward 1 − ρ × (1 − A), where ρ is the correlation coefficient. The correlation sources that break independence (shared rack, AZ, vendor batch, firmware, control plane) are catalogued in [§2 — hardware faults](./README.md#hardware-faults--random-and-mostly-independent).

## Why each 9 costs 10× — the formal picture

Solving 1 − (1 − A)² = 0.9999 for per-replica A gives A = 99%. So **two replicas at 99% each yield 99.99% combined.** No replica had to be made more reliable; we just doubled the count.

The 10× cost intuition does not come from making one component 10× better. It comes from:

- Each tier of availability requires a structurally different architecture (single → active-passive → active-active → multi-AZ → multi-region).
- The detection and failover budget shrinks faster than human operators can act, so automation becomes mandatory — and automation has its own failure modes.
- The independence assumption breaks at higher tiers: replicas share a rack switch, an AZ, a vendor batch, a control plane.
- Each tier unlocks failure modes the previous tier did not have (split-brain, cross-region replication lag, anycast convergence).
- Your availability is capped by the product of your dependencies' availabilities — chasing more 9's eventually means upgrading every dependency in the path.

## Practical analysis — steps

When evaluating a real architecture:

1. Trace the request through the system; list **hard** dependencies on the critical path.
2. Note each dependency's measured or claimed availability.
3. Multiply the hard chain → ceiling.
4. Combine parallel groups bottom-up with `1 − (1 − A)^N`.
5. Mark correlation points (shared rack, vendor, control plane). The formula weakens here.
6. Compare against measured uptime. If they disagree, you missed a dependency, undercounted correlation, or treated a soft dependency as hard.

A common outcome: an architecture that "feels" fine reveals a 99.5% ceiling. The arithmetic is unsentimental.
