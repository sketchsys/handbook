# 01 — Fundamentals

Before calling a system "well designed" you have to agree on what "well designed" means. This chapter builds that vocabulary: the quality attributes every serious system is judged on, the arithmetic of back-of-envelope capacity estimates, the mental models that keep you from designing the wrong thing, and the operational language teams use to talk about reliability day to day.

Nothing here is tool-specific. These are the foundations the rest of the handbook rests on.

## What this chapter covers

1. **What does "good" mean in system design?** The three-axis framework (reliability, scalability, maintainability) and the tensions between them.
2. **Reliability — deep dive.** Fault vs failure, the availability math, redundancy patterns, MTBF / MTTR.
3. **Scalability — deep dive.** Load parameters, vertical vs horizontal, stateless design, percentile thinking, Amdahl's and the Universal Scalability Law.
4. **Maintainability — deep dive.** Operability, simplicity (essential vs accidental complexity), evolvability.
5. **Back-of-envelope estimation.** Latency numbers, throughput numbers, powers of two; QPS, storage, and bandwidth math.
6. **Capacity planning as a ritual.** DAU → peak QPS, read/write ratios, hot path vs cold path.
7. **Mental models.** Little's Law, queuing theory intuition, the four bottlenecks (CPU / memory / disk / network).
8. **SLI, SLO, SLA, and error budgets.** The operational language of reliability targets.
9. **Engineering principles.** Boring technology, premature optimization, simplicity as a feature, YAGNI vs design-for-change.
10. **Interview-ready summary.**

Sections are added to this document as they are worked through. Some sections that work as standalone reference material (the numbers in §5–7, the SLI/SLO/SLA card, the engineering principles) will move into their own supplementary files in this folder when they land.

---

## 1. What does "good" mean in system design?

### The question

A system can be fast and constantly broken. It can be unbreakable and impossible to change. It can be easy to modify and collapse under load. It can scale forever while costing more to run than the company earns.

"Good" is not a single dimension. There are multiple quality axes and they pull against each other. Before judging a system — or deciding how much to invest in a change — you have to name the axis you are judging on and the bar you are trying to meet.

### Three axes — Kleppmann's framing

Martin Kleppmann's *Designing Data-Intensive Applications* (2017) collapses the quality space into three axes. This book is the de facto common vocabulary in system design; this handbook uses the same frame.

- **Reliability** — *"doing the right thing, even when things go wrong."* The system tolerates expected and unexpected faults, keeps performance acceptable under expected load, and rejects unauthorized use. Keyword: *fault tolerance*. Full treatment in §2.
- **Scalability** — *"coping with growth."* As load increases (more users, more data, more expensive queries), the system still holds acceptable performance. Critical point: **"scalable" alone is a meaningless claim** — you have to name the axis you are scaling along. Full treatment in §3.
- **Maintainability** — *"making life easier for the engineers who come after you."* Most of a system's total cost is paid not during the first release but over years of bug fixing, feature work, onboarding, and 3 a.m. on-call pages. Three sub-axes:
  - **Operability** — easy to run
  - **Simplicity** — easy to understand
  - **Evolvability** — easy to change

  Full treatment in §4.

### "Why only these three?" — common objections

| Not listed | Why |
|---|---|
| **Performance** | Folded into scalability — "acceptable response time under load." |
| **Cost** | Treated as a **constraint**, not a quality. Every axis is asked "at what price?" |
| **Security** | Kleppmann bundles it under reliability ("correct behaviour even against unauthorized use"). In practice most teams treat it as its own axis; we give it a dedicated chapter (12) for exactly that reason. See also follow-up 2 below. |
| **Observability** | Inside maintainability (operability). Given its own chapter (11) because the tooling discipline is specific. |

### The tension between the three

These axes cannot all be maximized simultaneously. Pushing one up typically pulls another down.

| If you want more… | You usually pay in… |
|---|---|
| Reliability (more "9"s) | **Cost** (each extra 9 ≈ 10×) and often **simplicity** (redundancy, failover, replication complexity). |
| Scalability (horizontal) | **Simplicity** (distributed-systems problems) and **consistency options** (some strong-consistency patterns no longer fit). |
| Maintainability (simpler code) | Sometimes **performance** — the "clever" optimization is usually the unmaintainable one. |
| Performance (lower latency) | **Maintainability** (caches, custom data structures, premature optimization) and sometimes **reliability** (aggressive timeouts). |

The key insight: **"good" is never absolute.** It is "good enough on each axis, for *these* requirements, at *this* point in time, for *these* users."

This is exactly why the clarification playbook (Scope → Scale → Performance → Data → Specials → Failure → Design) comes before any design work. Without the requirements, you cannot decide which axis to sacrifice for which. See [How to approach a system design question](../00-overview/how-to-approach-a-design-question.md).

### Same problem, different priorities — a concrete example

**Design a "send a message" feature.** Two contexts, opposite designs.

- **Twitter-scale public posting.** Reliability must be high (no lost posts), scalability must be very high (hundreds of millions of DAU), maintainability is moderate (large team, known domain). Solution: accept the write into an async queue, fan out asynchronously, accept **eventual consistency** of timelines.
- **A 5,000-user internal chat tool.** Reliability is moderate (brief downtime is tolerable), scalability is low (user base will not grow 1000×), maintainability is the dominant concern (two-person team). Solution: Postgres with a synchronous insert. Reaching for Kafka here is textbook over-engineering.

Same feature. The requirements moved the "good enough" bar on each axis, and that produced completely different architectures.

---

## Follow-up questions

The three-axis framework raises three questions that recur often enough to deserve answers up front.

### 1. Why does each "9" of availability cost ~10× more?

Availability targets translate to allowed downtime per year:

| Availability | Allowed downtime / year | Allowed downtime / day |
|---|---|---|
| 99% | 3.65 days | ~14.4 min |
| 99.9% | 8.76 hours | ~1.44 min |
| 99.99% | 52.6 min | ~8.6 sec |
| 99.999% | 5.26 min | ~0.86 sec |
| 99.9999% | 31.5 sec | ~86 ms |

Each row cuts allowed downtime by 10×. Google's SRE book formalizes this as a rule of thumb: *"An extra nine of availability costs roughly 10× more to achieve."*

The 10× multiplier is not one reason but five stacking ones:

1. **Redundancy levels escalate.** Single node → active-passive → active-active → multi-AZ → multi-region → multi-cloud. Each tier multiplies infrastructure cost.
2. **Detection and recovery time drops below human reaction.** At 99.99% (~8 sec/day budget) a human cannot intervene; the system needs fully automated health checks, failover, and traffic steering under 30 seconds.
3. **New failure modes appear at each tier.** Zone failures, cross-region replication lag, split-brain, DNS/Anycast convergence, cross-provider API drift — each unlocked by the previous tier's mitigation.
4. **Testing and operational overhead explode.** Chaos engineering, DR drills, game days, 24/7 follow-the-sun on-call. Engineering hours scale with reliability targets.
5. **Dependency ceiling.** Your availability is capped by the product of your dependencies' availabilities. A service that depends on three 99.99% downstream services cannot exceed 99.97%. Chasing more 9's means controlling — or upgrading — every dependency in the path.

Full treatment of fault tolerance, MTBF / MTTR, and redundancy patterns is in §2.

### 2. Is security a subset of reliability, or its own axis?

Both positions are defensible.

**For Kleppmann's bundling.** His definition — *"the system does the right thing, even when things go wrong"* — covers hardware faults, software bugs, human error, **and** malicious actors. An unauthorized access is one kind of "thing going wrong"; the system's job is to reject it. Clean and self-consistent.

**For treating it as its own axis (stronger in practice).**

- **Different failure model.** Reliability faults are usually random and distributed; security faults are adversarial and targeted. Probability theory does not model an attacker with a strategy.
- **Different mitigation toolkit.** Reliability leans on redundancy, retries, failover, chaos engineering. Security leans on threat modeling (STRIDE), defense in depth, least privilege, pen testing, secure SDLC. The daily tools of a reliability engineer and a security engineer barely overlap.
- **Different measurement.** Reliability is measured in MTBF / MTTR / availability. Security is measured in mean time to detect, time to contain, attack surface, mean days to patch.
- **Different organisation.** Most companies split SRE / Platform and SecOps / AppSec into separate teams with separate budgets.
- **Different regulatory regime.** Reliability has no compliance framework (beyond contractual SLAs). Security has SOC 2, ISO 27001, PCI-DSS, HIPAA, GDPR — each with its own audit trail.

Industry conventions reflect the split. AWS Well-Architected has six pillars (Operational Excellence, **Security**, Reliability, Performance, Cost, Sustainability); ISO 25010 treats reliability and security as separate quality characteristics.

**This handbook's stance.** Kleppmann's three stay as the conceptual anchor because they are pedagogically clean. Security gets its own chapter (12) because its toolkit is distinct. Saying *"security is a subset of reliability"* is definitionally defensible and operationally misleading — designing for security with reliability thinking (probability, redundancy) means you have never modeled an attacker.

### 3. Is there a fourth axis? Cost, and the pragmatic five-axis frame

Candidates, briefly evaluated:

- **Cost** — strongest candidate. Kleppmann treats it as a constraint; AWS Well-Architected promotes it to a pillar. In cloud-native systems cost behaves like a trade-off axis because every other axis spends money (more reliability = more infra, more scalability = more nodes). Treating cost as a separate axis stops "reliability at any price" thinking.
- **Security** — discussed above. Functionally an axis; this handbook acknowledges it as one in practice while keeping Kleppmann's three as the pedagogical anchor.
- **Observability** — sub-axis of maintainability (operability), but critical enough that major frameworks give it separate standing. Handbook: dedicated chapter (11), not a separate top-level axis.
- **Portability / interoperability** — a specialized form of evolvability. Not independent enough to justify its own axis in most systems.
- **Developer experience (DX)** — real for APIs, SDKs, developer platforms. Too specific to be a general axis.
- **Sustainability / energy** — emerging (AWS added it as a pillar in 2021). Too immature to standardize on yet.
- **Compliance** — cross-cuts security and reliability; regulated-sector specific.

**Pragmatic five-axis frame** — what serious teams implicitly use:

1. **Reliability**
2. **Scalability**
3. **Maintainability**
4. **Security**
5. **Cost**

Kleppmann's three + security + cost ≈ AWS Well-Architected's five core pillars (excluding sustainability). This handbook teaches the three-axis pedagogy, but when you are actually designing something treat the five as the real list.

---

## Next

- §2 — Reliability deep dive
- §3 — Scalability deep dive
- §4 — Maintainability deep dive
- §5 — Back-of-envelope estimation
- §6 — Capacity planning ritual
- §7 — Mental models
- §8 — SLI / SLO / SLA and error budgets
- §9 — Engineering principles
- §10 — Interview-ready summary

Sections are added as they are studied. Some will graduate into standalone supplementary files in this folder.
