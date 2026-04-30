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

Each additional nine of availability cuts allowed downtime by 10× — full table in [availability-math.md](./availability-math.md#the-9s--availability-vs-allowed-downtime). Google's SRE book formalizes this as a rule of thumb: *"An extra nine of availability costs roughly 10× more to achieve."*

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

## 2. Reliability — deep dive

§1 defined reliability as *"the system does the right thing, even when things go wrong."* That sentence quietly contains the most important distinction in the chapter — *a thing going wrong is not the same as the system failing.* The job of reliability engineering is to put daylight between those two events.

### Fault vs failure

- **Fault** — a component (a disk, a process, a network link, a deployment, an operator's hand on the keyboard) deviates from its specification. Faults are *local*. A disk had a bit flip. A node lost power. A service returned a 500. A junior engineer truncated the wrong table.
- **Failure** — the system as a whole stops providing the service its users expect. Failures are *global*, user-visible.

A fault-tolerant system is one in which **faults happen but failures don't** — or do so rarely enough. The whole engineering project of reliability is converting *"1 fault → 1 failure"* into *"1 fault → 0 failure."*

You cannot prevent faults. Hardware degrades, networks partition, processes get OOM-killed, deployments ship bugs, humans mistype. Integrated over a long enough time and a large enough fleet, the probability of *some* fault is 1. What you can do is design so that no single fault — and ideally no realistic combination of faults — translates into a failure.

> Reliability engineering is not about making faults stop happening. It is about making faults **not matter**.

This reframing matters because the two intuitions lead to opposite designs. *"Make faults not happen"* → expensive single-vendor enterprise hardware, long change-freezes, gatekeeping reviews, a *"don't touch it, it'll break"* culture. *"Make faults not matter"* → commodity hardware, redundancy, fast deploys, automated recovery, chaos engineering. The cloud-native industry won the second argument; Google's SRE book, Amazon's *"everything fails all the time"* doctrine, and Netflix's chaos engineering all rest on it.

#### Same fault, three architectures

Take a single fault: **the database primary loses power.**

- **System A** — single Postgres on a single VM. No replica, no failover script. The database is unreachable, every dependent request errors out. **1 fault → 1 failure.**
- **System B** — Postgres primary + warm standby + manual failover. An alert fires; on-call logs in, promotes the standby, updates DNS or connection strings. ~20 minutes of degraded service. **1 fault → 1 short failure.**
- **System C** — Postgres primary + sync replica + automatic failover (e.g., Patroni). Health checks detect the primary loss in ~5 seconds, the replica is promoted, traffic flows. Users see a brief error spike, then recovery. **1 fault → ~0 user-visible failure.**

Same fault, three architectures. The difference is the engineering invested in making the fault *not matter*.

### The three classes of fault

Designing fault-tolerant systems requires knowing what kind of fault you're tolerating. Kleppmann groups them into three classes; each behaves differently statistically and demands a different toolkit.

#### Hardware faults — random and (mostly) independent

Disk failures, memory bit-flips, NIC failures, power-supply deaths, server crashes. At fleet scale these follow predictable rates (vendors quote MTBF in millions of hours), and they are *approximately* independent — one server crashing does not raise the probability that the next one crashes.

Independence is the friendly property. It is the only reason redundancy works. Two replicas of a 99% available component, if their failures are uncorrelated, give 1 − (0.01)² = 99.99%. The whole *"more 9's = more redundancy"* intuition rests on this assumption.

**The bathtub curve.** Hardware failure rate is not constant in time:

```
failure
  rate │\
       │ \                            /
       │  \                          /
       │   \________________________/
       │  infant      useful life    wear-out
       │ mortality
       └────────────────────────────────────  time
```

- **Infant mortality** — new hardware shows elevated failure in the first weeks (manufacturing defects, bad batches, calibration).
- **Useful life** — flat, low rate. MTBF is meaningful here.
- **Wear-out** — disks, SSDs, fans, PSUs age out; the rate climbs again.

Practical rule: **age diversity is a feature.** A fleet bought and deployed in one batch enters wear-out in one batch — correlated failure. Disciplined operators rotate the fleet incrementally.

**Where independence breaks.** *"Two replicas = 99.99%"* assumes uncorrelated failures. In practice, common-mode causes break that:

| Correlation source | Failure mode | Mitigation |
|---|---|---|
| Same **rack** | Rack switch or PDU dies → whole rack down | Spread replicas across racks |
| Same **PSU/circuit** | Power feed fails → all machines on that feed go | Dual-homed power |
| Same **AZ** | Datacenter-wide event (fire, cooling, fiber cut) | Multi-AZ |
| Same **region** | Region-wide network/power, control-plane bug | Multi-region |
| Same **vendor batch** | Manufacturing defect surfaces in the same week | Vendor and batch diversity |
| Same **firmware** | A triggered bug locks every disk on that firmware | Phased firmware rollout |
| Same **upstream dependency** | All replicas hit a single DNS, auth, or storage | Replicate the dependency or cache |

Engineers tend to be correct about *horizontal* independence and blind to *vertical* correlation. If both replicas hang off the same rack switch, the switch's availability sets the ceiling — replica count is irrelevant.

**Field data — vendor MTBF lies.** Backblaze's annual disk-failure reports put **AFR (Annualized Failure Rate)** around 1–2%, with model-to-model spread of 10×. Vendor-claimed MTBF, back-converted to AFR, is typically 2–3× more optimistic than measured AFR. Google's 2007 *"Failure Trends in a Large Disk Drive Population"* paper found the same: SMART indicators predict failure poorly, temperature correlates more weakly than expected, and field behavior diverges from spec sheets. **Measure your own fleet.**

**Silent corruption.** The nastiest hardware fault is one no one notices: a bit flips and the application returns wrong answers. Without ECC memory, network checksums, and end-to-end storage checksums, the fault has no visible signature — it shows up as an "application bug." Half of ZFS's appeal is that it puts checksums in the storage layer end-to-end.

#### Software faults — deterministic and correlated

A bug triggered by a specific input. A memory leak that builds over days. A misconfigured timeout. A race condition that surfaces only under load.

These are vicious because they are **correlated** — the same bug exists on every replica. Adding a second instance does not double availability; both fail on the same input. The redundancy math from hardware faults breaks here.

**Bohrbug vs Heisenbug.** Two classic shapes:

- **Bohrbug** — Bohr-atom solid, deterministic. Same input → same crash. Caught in tests, or at worst rolled back from production. Painful but tractable.
- **Heisenbug** — vanishes when observed. Race conditions, timing-dependent bugs, memory corruption. Random in production, unreproducible in a debugger. The most expensive enemy of reliability engineering.

Defending against Heisenbugs requires more than testing: deep observability, distributed tracing, deterministic replay, controlled fault injection.

**Gray failure.** The classical model is binary — up or down. The most expensive real-world outages are **gray**:

- Service is up, health check returns 200, but p99 latency is 30 seconds.
- Replica is running but writing at 5% normal speed.
- Process is alive but its queue has overflowed and messages are silently dropped.

Load balancers see a healthy node and route traffic to it; the traffic doesn't actually succeed. The basic GET that the health check sends works; real workloads don't. Microsoft Research's *"Gray Failure: The Achilles' Heel of Cloud-Scale Systems"* (2017) named the pattern. Defenses:

- **Semantic health checks** — simulate real work, not just *"are you alive?"*
- **Outlier detection** — load balancers eject nodes whose p99 latency drifts (Envoy's *Outlier Detection* is the canonical implementation).
- **Client-side load balancing + circuit breaking** — consumers route around slow replicas based on their own observations.

**Slow rollout, fast rollback.** Software-fault correlation can't be defeated by parallelism, but it can be diluted in *time*. Don't let the bug reach every replica simultaneously. Canary deployment, blue-green, feature flags, progressive rollout. The bug ships, but only to 5% of traffic; it is observed; it is rolled back. **An MTTR play, not an MTBF play.**

The alternative — *N-version programming*, where independent teams write independent implementations of the same logic — sounds appealing but rarely works in practice. It doubles cost, halves test coverage, and has shown empirically that independent teams ship correlated bugs more often than expected.

**Cascading failure.** Some software faults appear only under load. The system is fast enough idle; a small slowdown queues requests; the queue grows; servicing slows further; more requests queue. Positive feedback loop, classic cascade.

Typical pattern: a downstream service slows by 100 ms → upstream connection pool fills → requests block → upstream consumers time out → retries fire → load doubles → more slowdown. Thirty minutes later the entire system is down, the original fault long resolved, and the traffic pattern can't heal itself.

The question to ask is not just *"what happens when this fault occurs"* but **"how does load behave when this fault occurs?"** Defenses — backpressure, load shedding, circuit breakers, retry budgets, jittered exponential backoff — all live in chapter 10. Their reason for existing is here.

#### Human errors — the largest single cause of incidents

Bad deploys, misconfigurations, accidental destructive commands in production, miscalculated capacity. Post-incident studies — Google SRE, AWS post-mortems, the *"Why Do Internet Services Fail"* literature — consistently find humans involved in **the majority** of significant outages, often 60–80% by definition. This is not a moral indictment; it is what happens when a complex production system has a normal-fallible-human in the operator role.

**Old model vs new model.** Classical accident analysis assigns blame: *"the operator made a mistake; training was insufficient."* Modern incident analysis (Sidney Dekker, Erik Hollnagel, the resilience-engineering school) rejects this:

> *Humans don't cause failures. Systems fail in the situations where humans make mistakes. The right question is: why was that system in a state where that operator's decision could break it?*

The takeaway for reliability engineering is sharp: **"be more careful" is not a fix.** The fix is the systemic change that makes the same mistake impossible — or at least scoped — next time.

**Blameless postmortems.** Popularized by the Google SRE book. After an incident:

- *Who* made the mistake is **not asked**.
- *Why the system accepted that mistake* is asked.
- Names aren't used; roles are (*"on-call engineer," "deployment operator"*).
- Action items are always **systemic**: runbooks, automation, guardrails. *"Be more careful"* is never an acceptable action item.

This works because blame culture kills information flow. An operator who hides a mistake guarantees the next operator repeats it. Blameless culture surfaces mistakes, and the system improves.

**The two edges of automation.** Removing the human is the headline mitigation. But automation creates two new problems:

1. **The automation paradox** (Lisanne Bainbridge, 1983). Humans hand off the routine; what's left is exactly the edge cases automation can't handle — the hardest ones. With less practice, humans face harder problems. *Automation does not make work easier; it changes work, often making it harder.*
2. **Automation's blast radius is wider than a human's.** A human running the wrong command affects one server. Automation running the wrong command affects the *fleet*. So automation requires guardrails: rate limiting, dry-run modes, *"affect one cell, not all of production"* design.

**Blast-radius limiting — the most important architectural technique.** Bound the impact of any single mistake (human or automated):

- **Cell architecture.** Partition production into isolated *cells*, each carrying 5–10% of traffic. A cell going down means 5% impact. AWS uses this; Slack's *"cellular architecture"* essay is a practical reference.
- **Shuffle sharding.** Assign customers randomly across replicas so no two customers share their full set. One replica's death affects a small fraction of customers, not all of them. This is how AWS Route 53 hits its availability target.
- **Least privilege.** An engineer's deploy permission should not be the same scope as their *drop-table* permission. A mistake in one domain doesn't propagate.
- **Confirmation, dry-run, canary in tooling.** Destructive commands announce intent, refuse to run without explicit approval, and apply changes in small increments before fanning out.

**"Production is hostile."** The cultural posture: every touch on production is a potential incident.

- Manual SSH to production is exceptional (some shops alarm on it).
- All changes are made through code (Infrastructure as Code, GitOps).
- No action is taken without an audit log.
- Tooling investment beats manual labor at scale.

### Measuring reliability — MTBF, MTTR, availability

Reliability needs a numerical language. Three terms:

- **MTBF** — Mean Time Between Failures. Average uptime between failures. A *fleet* metric.
- **MTTR** — Mean Time To Recovery. Average time from failure to restoration; the sum of *detect + respond + repair + verify*.
- **Availability** — A = MTBF / (MTBF + MTTR). The fraction of time the system is up.

The strategic insight: there are two ways to attack availability — make MTBF bigger, or make MTTR smaller — and **MTTR is usually the cheaper side.** Hardening MTBF asymptotes; halving MTTR via observability and automation often pays back many times over. Modern SRE practice — the **DORA** metrics (MTTR + Change Failure Rate + Deploy Frequency + Lead Time) — reflects this. MTBF is conspicuously absent; the modern bet is on fast recovery rather than rare faults.

The full arithmetic — composition rules, the 9's downtime table, why each 9 costs 10× — is in [availability-math.md](./availability-math.md).

### Soft vs hard dependencies

In a series chain, not every dependency is equal. **Hard** dependencies fail the request when they fail. **Soft** dependencies degrade quality but don't break the request — a cache miss, a missing recommendation, a fallback ranking. A soft dependency in a failed state is treated as A = 1 in the chain; it doesn't tax the product.

Mature architectures convert hard dependencies into soft ones wherever possible. Twitter timelines tolerate a down ranker. Stripe accepts payments and pushes fraud-detection async. DNS resolvers serve stale records during upstream outages (RFC 8767's *serve-stale*). Each conversion raises the dependency-chain ceiling.

### Structural redundancy

Once dependencies are minimized and corrected, the remaining lever is **redundancy** — having more than one of each component. The pattern catalog (active-passive, active-active, N+1 / 2N, geographic redundancy, the *"untested redundancy is no redundancy"* discipline) lives in [redundancy-patterns.md](./redundancy-patterns.md).

### What §2 covers, what §2 doesn't

§2 is the *foundation* of reliability — the conceptual lens (fault vs failure), the statistical language (MTBF / MTTR), and the fault taxonomy (hardware / software / human). Real reliability engineering builds five further layers on this foundation, each its own discipline and each its own home in this handbook:

| Layer | What it does | Where in the handbook |
|---|---|---|
| Structural redundancy | Replicas, failover, quorum, multi-AZ/region | Rest of §2 + chapter 6 (consensus) + chapter 9 (deployment & scaling) |
| Reliability patterns | Circuit breaker, bulkhead, retry/backoff, timeout, idempotency, graceful degradation | Chapter 10 |
| Detection & response | Monitoring, alerting, on-call, SLI/SLO, error budgets, incident response | §8 (this chapter, SLI/SLO) + chapter 11 (observability) |
| Recovery | Backup, restore, disaster recovery, RPO/RTO, data integrity | Chapter 7 (storage) |
| Test & exercise | Chaos engineering, fault injection, game days, load testing | Chapter 10 |
| Operational discipline | Blameless postmortems, runbooks, change management, SRE practice | Cross-cutting; mostly chapter 9 + chapter 11 |

§2 teaches the anatomy — bones. The muscles (patterns), nervous system (observability), immune system (chaos), and operational fitness (SRE practice) are taught in dedicated chapters.

### §2 recap

In one sentence: **reliability engineering is the discipline of preventing faults from becoming failures.**

Three ideas in order:

1. **Fault ≠ failure.** Components break; the system stays up. The design goal isn't *"no faults"* — it's *"faults that don't matter."*
2. **Three fault classes, three toolkits.** Hardware (independent → redundancy works), software (correlated → slow rollout + fast rollback), human (systemic → guardrails + automation + blameless culture).
3. **Two metrics, one formula.** MTBF (failure rate), MTTR (recovery speed), A = MTBF / (MTBF + MTTR). MTTR is usually the cheaper side to attack.

Pattern repertoire (full treatment in [redundancy-patterns.md](./redundancy-patterns.md)):

- **Active-passive** for stateful single-writer systems (RDBMS).
- **Active-active** for stateless or partitioned workloads (API, CDN).
- **N+1 / 2N** sized to the failure mode you defend against.
- **Multi-AZ** by default, **multi-region** for critical paths, **multi-cloud** rarely worth it.

The sentence to keep at the top of any architecture review:

> *Hardware will fail. Bugs will ship. Humans will mistype. The only question is whether your system is designed so that any of those turns into the user seeing a broken product.*

---

## 3. Scalability — deep dive

§1 defined scalability as *"the system's ability to cope with growth"* and added the warning that the word *scalable* alone is meaningless. This section unpacks that warning. Scalability is not a property a system either has or doesn't — it is a *response to a specified change in load*, and the response is honest only if you can name the load axis, the target value, the time horizon, and the cost.

The chapter walks the same path the rest of the handbook will refer back to: how to define the load you are scaling for, what *vertical* and *horizontal* actually mean (and why the right answer is usually neither alone), why stateless design is the precondition of horizontal scaling, why averages lie about performance, and the two laws — Amdahl and Gunther's Universal Scalability — that bound how much parallelism can buy you.

### "Scalable" is not a claim on its own

The popular intuition is that *scalable = big, or fast, or able to serve many users.* All three are wrong. A system serving a million users today may not be scalable; a system serving fifty users today may be deeply scalable. Current size says nothing about scalability.

Kleppmann's definition is precise:

> **Scalability — the system's ability to cope with increased load.**

Two words inside the sentence carry the weight: *increased*, and *load*. Scalability is not a static property; it is the system's behavior under a change. And the change has to be specified — *which* load.

That specification matters because the same system can be excellent on one axis and collapse on another. The classical illustration is Twitter's two operations: posting a tweet and reading a timeline. *Fan-out on read* — compute the timeline by joining all followed users at read time — makes posting cheap and reading expensive. *Fan-out on write* — at post time, copy the tweet into every follower's timeline cache — makes posting expensive and reading cheap. Asking *"which is more scalable?"* is the wrong question. The right questions are *which axis is growing*, and *what does the load distribution look like*. Twitter ended up running both: fan-out on write for normal users, fan-out on read for celebrities whose follower counts make write fan-out catastrophic.

The operational rule that comes out of this:

> A system is scalable along **load parameter X**, from **value Y** to **value Z**, over **time horizon W**, at **cost C**.

Five blanks, all mandatory. Drop any of them and the claim becomes marketing.

### Load parameters — the numbers that define your system

A load parameter is a number, or a small set of numbers, that describes how much work the system is doing. It is not necessarily QPS; it has to be the dimension that drives the bottleneck. If increasing the parameter strains the system, it is a load parameter. If it does not, it is just a metric.

Different systems are described by different parameters:

| System | Primary load parameter | Secondary |
|---|---|---|
| HTTP / OLTP API | Requests per second | Concurrent users, p99 budget |
| Realtime chat (Slack, WhatsApp) | Concurrent connections | Message rate, fan-out per message |
| Database (OLTP) | QPS + read/write ratio | Working set size, transaction size |
| Database (OLAP) | Query complexity × scanned data | (QPS is rarely meaningful) |
| Batch / ETL | Volume per window (GB/hour) | Job count |
| Object storage (S3-class) | Total objects + GET/PUT ratio | Object size distribution |
| CDN / edge | Bandwidth (Gbps) + cache hit rate | Origin request rate |
| Search engine | Queries per second × index size | Document update rate |
| Social network (Twitter pattern) | Posts/sec + follower distribution | Timeline read rate |
| Multiplayer game server | Concurrent players per shard | Tick rate × state size |

"QPS" is not a universal currency. Asking an OLAP warehouse for its QPS misses the point — a single query may scan terabytes for ten minutes, and counting them is meaningless. Each system type has its own load language, and capacity planning starts by picking the right one.

#### The mean is the wrong number

The deeper trap is choosing a single average to summarize the load. Real-world load distributions are almost never uniform; they are skewed, often power-law, with a long tail that dominates everything.

Twitter's published numbers from the early 2010s make the point. Posts averaged ~4.6k/sec at steady state and ~12k/sec at peak — modest. Timeline reads averaged ~300k/sec — large but tractable. The dangerous number was elsewhere: the **distribution of follower counts**. The median user has roughly 200 followers, the mean is around 700, p99 is around 10,000, and the maximum is on the order of **100 million**. The ratio between mean and max is roughly five orders of magnitude. A system designed for the *average tweet* — 700 fan-outs per post — falls over the moment a celebrity tweets, because that single post carries the cost of a hundred thousand average posts.

This pattern recurs everywhere under different names — *hot key*, *hot partition*, *celebrity problem*. e-commerce: the average product is viewed twice a minute, the homepage feature is viewed two hundred thousand times. Slack: the average channel has 50 members, the all-hands channel has 50,000. Stripe: the average merchant runs a thousand transactions a month, the largest run ten million a day. YouTube: the average video gets a handful of views, the viral video gets a hundred thousand views per second. In all cases the system designed for the mean breaks under the tail.

The minimum honest description of load:

1. **Mean** — the floor for capacity planning.
2. **Peak** — usually 2–5× mean. Architecture is sized for peak; cost is sized for mean.
3. **p99 (or higher)** — the shape of the tail, the place where hot-key failures live.

If your "transaction count" is one number, suspect it is incomplete. Almost every real load has a *count × size* shape — and in skewed distributions, the few largest items account for most of the cost.

### Vertical vs horizontal — and why diagonal usually wins

Once the load profile is honest, the question of *how to grow* admits two answers. The folk story — *"vertical is dead, everyone scales horizontally now"* — is wrong. The two are complementary; most systems live with both.

**Vertical scaling (scale up)** keeps the architecture and grows the machine. 4 vCPU becomes 64. 16 GB RAM becomes 1 TB. The codebase does not change.

- *Pros.* No distributed-systems problems. ACID, joins, in-process locks all keep working. Latency stays in-process — RAM access is around 100 ns, network round-trip is 500 µs (5,000× the gap). Operationally cheap: one machine to monitor, one to deploy.
- *Cons.* There is a ceiling. Cloud providers' largest instances run into the millions of dollars per year and still represent a single point of failure. Cost above mid-tier is super-linear: a 2× larger instance is often 3–5× more expensive. Resizing usually requires a reboot, which stateful workloads tolerate poorly. NUMA effects on very large machines mean naive software does not get the speedup the spec sheet promises.

**Horizontal scaling (scale out)** adds machines. A single server becomes ten, ten thousand, a hundred thousand. Load is split across them.

- *Pros.* No theoretical ceiling — Google, Meta, AWS run hundreds of thousands of nodes on this principle. Cost stays roughly linear because commodity instances are priced commodity. Failure of one machine is absorbed naturally if the architecture allows it; the same redundancy that buys reliability buys scalability.
- *Cons.* Distributed-systems complexity in full. Sharding stateful workloads is hard. Network turns from a free resource into the dominant latency budget — 0.5 ms within a rack, 1–10 ms across an availability zone, 50–150 ms cross-region. Coordination overhead grows with N (the rest of §3.6 is about exactly this). Operational and observability burdens scale with the number of nodes.

The technical core of horizontal scaling has a name: **shared-nothing**. Each node owns its CPU, memory, and disk; there is no shared state, only message passing. The opposing patterns — shared-disk (Oracle RAC and similar) and shared-memory (NUMA, single-machine) — eventually contend on the shared resource and stop scaling. Every system that earned the *"web-scale"* label — Cassandra, DynamoDB, Spanner, Kafka, Elasticsearch, S3 — is shared-nothing. The real meaning of *horizontal* is not "more boxes" but "no shared state."

In production, almost no one runs pure vertical or pure horizontal. The common shape is **diagonal**: each node is moderately powerful, and there are several of them. A capacity step usually means *upsize the node and add nodes at the same time*, balancing per-node strength against fleet redundancy.

The transition from vertical-first to horizontal has two failure modes. *Premature horizontal* — going distributed on day one to "be ready for scale" — pays the full cost of distribution before there is any benefit. *Late horizontal* — staying on a single node until it tips over — leaves no time to migrate. The realistic pattern is to track utilization against the largest available instance: at ~50% you should be planning the horizontal architecture, at ~80% the migration should already be under way.

| Criterion | Vertical wins | Horizontal wins |
|---|---|---|
| Architectural change required | ✓ none | ✗ significant |
| Latency-sensitive in-process work | ✓ | ✗ |
| Reliability / fault tolerance | ✗ SPOF | ✓ natural redundancy |
| Headroom for growth | ✗ ceiling | ✓ unbounded |
| State management ergonomics | ✓ ACID free | ✗ distributed problem |
| Cost at large scale | ✗ super-linear | ✓ roughly linear |
| Cost at small scale | ✓ | ✗ coordination overhead wasted |
| Speed of development (early stage) | ✓ | ✗ |

### Stateless design — the prerequisite for horizontal scaling

If shared-nothing is the rule of the architecture, **stateless** is the rule of the request-handling tier inside it. The two are versions of the same idea applied at different layers.

A stateless service is one in which any node can handle any request and produce the same result. *Stateless* does not mean *no memory*; it means *no node-local memory that the next request depends on*. The state of the system still exists somewhere — in the database, in a cache, in the client's token — but never inside the request-handling node's process.

There are two kinds of state to think about:

- **Persistent state.** User accounts, orders, messages — already in the database. Not the issue.
- **Session / transient state.** Login session, multi-step form progress, an open WebSocket, a per-user rate-limit counter. This is where the architectural decision sits.

Three places that session state can live:

- **On the client.** JWT, signed cookie, mobile app state. The server validates the token on every request and holds nothing locally. Server is fully stateless. Cost is zero. Drawbacks: token size grows on every request; revocation requires a server-side blacklist (and brings state back); sensitive data cannot ride on the client.
- **In a shared store.** Redis or Memcached, keyed by session ID. The server is still stateless; it dereferences the session at the start of every request. Drawbacks: an extra network hop on every request, and the store itself is a new dependency that needs its own reliability design.
- **Sticky to a node.** The load balancer routes a user's traffic always to the same node, which keeps state in its own process memory. This is an anti-pattern — it preserves the legacy code at the cost of making the system fragile. Load distribution is uneven (a heavy user always hits the same node), node death loses the session, deploys require draining, and adding nodes does not rebalance existing users.

The widely-cited industry doctrine on this is **The Twelve-Factor App** (Heroku, 2011), specifically Factor VI:

> *Execute the app as one or more stateless processes. Sticky sessions are a violation.*

Cloud-native runtimes — Kubernetes, ECS, Cloud Run, Heroku — all assume this factor. Pushing state to backing services is the default; sticky sessions are a workaround for legacy systems that cannot afford the rewrite.

The pragmatic test of statelessness is operational, not architectural: *if you kill any node and bring up an identically-configured replacement, does the system behave the same?* In SRE shorthand the test is **"cattle, not pets"** — nodes that are interchangeable, replaceable, and never named.

#### In-memory cache — the soft violation

The most common compromise is the in-memory cache. The service caches DB query results in its own process to skip a round trip. Strictly speaking the service is no longer stateless; in practice this is acceptable, with conditions:

- Losing the cache must not affect correctness — re-fetching from the DB must still produce the right answer.
- Cache misses on a fresh node must not catastrophically slow the system.
- Hot keys cached in N processes mean N copies of the same data; cache effectiveness is much lower than a shared cache would deliver.
- Invalidation across N nodes is hard, and TTL-only invalidation is the path of least resistance.

For this reason serious systems treat in-memory cache as *L1*, with a shared distributed cache (Redis, Memcached) as *L2*, and the database as the final source of truth. Chapter 5 covers this stack in detail.

#### Where stateless is hard

Some workloads are stateful by their nature, and the right move is to isolate them in their own tier rather than fight to make them stateless.

- **WebSocket / long-lived connections.** The TCP socket physically lives on a node. Either accept stickiness for the connection itself, or split the architecture: a stateless application tier behind a stateful gateway tier whose only job is holding connections (the Slack gateway architecture is a public reference).
- **Real-time game servers.** Tens of thousands of state mutations per second per match make per-tick database writes infeasible. Players are partitioned to match servers; each match server holds its match's state in RAM. Stickiness is the design, not the failure mode.
- **Streaming and transcoding pipelines.** The half-encoded frame buffer must live somewhere; recovery is via checkpointing, not statelessness.
- **The database itself.** The hardest stateful workload, and the topic of chapters 6 (consistency and consensus) and 7 (storage).

Where state must be kept *and* the service must scale horizontally, the technique is **partition + replicate + coordinate**: shard the state across N nodes (horizontal capacity), replicate each shard to K nodes (reliability and read scale), and use a consensus protocol to decide who owns what. All three are heavy enough to deserve their own chapters; they are introduced here because they are how the stateful tier follows the rules of horizontal scaling without giving up its state.

### Percentile thinking — performance is not measured by averages

Latency distributions are skewed to the right. The left side is bounded at zero; the right side is unbounded. The mean sits well above the median, and a single user's experience is not the mean — it is the latency of *that user's specific requests*. Reporting performance as a mean is, in almost every public-facing system, wrong.

Two systems can share a mean of 50 ms and feel completely different. One has p50=50, p95=70, p99=90 — narrow distribution, every user has a similar experience. The other has p50=30, p95=200, p99=1500 — same average, but every hundredth user waits 1.5 seconds and every thousandth waits much longer. Mean does not distinguish these systems; percentiles do.

The vocabulary is:

- **p50 (median)** — the typical experience.
- **p95** — most user-perceived performance lives here. The slowest 5% are still close.
- **p99** — where complaints start. At 1k QPS this is one user per second.
- **p999, p9999** — the deep tail. Critical for high-throughput services.

The general rule: mean is *not* typical; mean always sits above the median; the median answers *"how does my system feel"* and the high percentiles answer *"how does it fail."*

The sectoral conviction about percentiles came from two findings. Amazon (2007) reported that a 100 ms increase in *p99 latency* — with mean unchanged — cost roughly 1% of sales. Google's *Tail at Scale* paper (Dean and Barroso, 2013) showed that a single request fanning out to N internal services inherits the worst of each: if every backend has 1% probability of being slow, the probability that *some* backend is slow on a 100-way fan-out is around 63%. The p99 of one backend becomes the p50 of the user-visible latency. *In a fan-out architecture, tail latency does not stay in the tail.*

Mitigations are about doing extra work to shorten the tail:

- **Hedged requests** — issue the same request to two replicas, take the first answer. ~2× backend cost, dramatically tighter tail.
- **Tied requests** — start the second request, and have the first replica cancel it as soon as it begins responding.
- **Backup-after-delay** — wait N ms; if no response, fire a second request.
- **Outlier eviction** — load balancers (Envoy is a canonical implementation) drop replicas whose p99 has drifted away from the fleet.

There is also a less-discussed statistical fact: *heavy users live in the tail*. The maximum latency a user has experienced over their N requests rises with N. A user with 10 requests experiences roughly the system's p90 as their worst; a user with 1,000 requests experiences roughly the p999. The users at the system's worst percentile are disproportionately the heaviest users — usually the most valuable ones. Caring about p99 is not just statistical hygiene; it is caring about the specific users who matter most.

A widespread hidden bias in latency measurement is **coordinated omission**, named by Gil Tene. Most benchmark loops measure response time only for requests they actually managed to send. When the system stalls, the benchmark stalls with it, and the missed requests are never recorded. The histogram reports one bad sample where there should have been thousands. Tools that handle this correctly — HdrHistogram, wrk2, Tene's own libraries — measure *intended send time* rather than *actual send time*. p99 numbers from older tools (`ab`, early `jmeter`) are routinely 5–10× more optimistic than reality. When reading a percentile number, ask whether the measurement corrected for coordinated omission.

Two further pitfalls in working with percentiles:

- **Percentiles do not add.** *p99(A→B)* is not *p99(A) + p99(B)* — the slow request through A is usually a different request from the slow request through B. To get end-to-end percentiles, trace each individual request and compute the distribution of the totals.
- **Percentiles do not average.** *"Each of 10 nodes has p99 = 100 ms, so the cluster p99 is 100 ms"* is wrong. Percentiles are properties of distributions; you need a central histogram (HdrHistogram, t-digest) to compute the cluster's p99 from the per-node histograms. Modern monitoring stacks do this; older "average across hosts" dashboards do not.

The dialect of operational reliability — SLI, SLO, error budgets (§8) — is written entirely in percentiles for these reasons. *Mean* SLOs are vanishingly rare in modern practice.

### The laws of parallel scaling — Amdahl, Gustafson, USL

Every act of horizontal scaling is bounded by two laws. **Amdahl's Law** (1967) says that the speedup achievable by parallelism is capped by the serial fraction of the work — adding cores cannot speed up code that has to run on one core. **Universal Scalability Law** (Gunther, 1993) extends Amdahl by adding a coordination cost: past a certain N, adding nodes makes the system *slower*, because the cost of keeping them in sync outgrows the work they contribute. The corollary that runs through this entire handbook — *eliminate serial fractions, eliminate coordination, partition* — is a direct consequence.

The full math, the formulas, the shape of the curves, and the implications for shared-nothing architecture, eventual consistency, and consensus protocols are treated as a standalone reference: see [scalability-laws.md](./scalability-laws.md).

The three takeaways to carry into the rest of the handbook:

1. **Linear scaling does not exist.** Some serial fraction always remains; the maximum speedup is `1/s`, regardless of how many machines you add.
2. **Beyond a point, more nodes are *worse*.** Coordination cost rises with N²; the optimum cluster size N\* is usually smaller than the team estimated.
3. **The only general antidote is partition.** Splitting global state into K independent shards effectively divides the coordination cost by K, raising the scaling ceiling by the same factor.

### What §3 covers, what §3 doesn't

§3 is the conceptual layer of scalability — what it means, what to measure, what to scale on, and the laws that bound the answer. The mechanical layers that actually deliver scalability — the protocols, data structures, and operational practices — are spread across the rest of the handbook:

| Layer | Where in the handbook |
|---|---|
| Networking, traffic shaping, load balancers | Chapter 2 |
| Sharding, replication, consistency models | Chapter 3 (data) + chapter 6 (consensus) |
| Communication patterns (sync, async, queues, streams) | Chapter 4 |
| Caching tiers and invalidation | Chapter 5 |
| Storage scalability — partitioning, compaction, hot tiers | Chapter 7 |
| Search-specific scaling — indexes, sharded search | Chapter 8 |
| Capacity planning, autoscaling, deployment topology | §6 of this chapter + chapter 9 |
| Reliability under load — backpressure, load shedding, circuit breakers | Chapter 10 |
| Tail-latency mitigation as an operational practice | Chapter 10 + chapter 11 |

§3 teaches the language. The rest of the handbook teaches the techniques.

### §3 recap

In one sentence: **scalability is a system's response to growth along a specified load axis, bounded by the serial fraction of the work and the cost of coordination.**

The five anchors:

1. **"Scalable" is not a claim** — name the load parameter, the from-value, the to-value, the time horizon, and the cost. Otherwise the word is marketing.
2. **Load is a distribution, not a number** — the mean lies; the tail of the distribution is where systems break (hot keys, celebrities).
3. **Vertical and horizontal both have a place; diagonal usually wins** — the real meaning of horizontal is *shared-nothing*, not *more boxes*.
4. **Stateless is the prerequisite of horizontal scaling** — push state to the client, to a shared store, or to a stateful tier with its own design; sticky sessions are a legacy compromise.
5. **Mean lies about latency too** — performance is a percentile dialect; the tail amplifies under fan-out, and coordinated omission silently understates it.

The sentence to keep at the top of any scalability discussion:

> *No system is scalable in general. A system is scalable along a particular axis, up to a particular target, within a particular budget — and only after the serial fraction and the coordination cost have been measured.*

---

## 4. Maintainability — deep dive

§1 framed maintainability as *"making life easier for the engineers who come after you"* and split it into three sub-axes — operability, simplicity, evolvability. That sentence is easy to read as a soft, cultural concern. It is not. Maintainability sits next to reliability and scalability on the quality-axis table because it is the dominant **cost axis** of any long-lived system, and the daily friction it imposes determines how fast a team can keep doing useful work.

This section walks the three sub-axes in turn — what each means, how it is measured (or its proxies), what its tools are, and how it interacts with the other quality axes.

### Why maintainability is its own axis

Two facts force the framing.

**Most of a system's cost is paid after the first release.** The classic estimate — Boehm's *Software Engineering Economics* (1981), reaffirmed by every subsequent industry survey — is that around **70%** of a software system's lifetime cost is spent after launch: bug fixes, feature additions, on-boarding new engineers, on-call response, dependency upgrades, security patches. The "build it" phase is the small part of the bill. Anything that compounds friction in the post-launch phase compounds the bill.

**Engineering velocity is a function of the codebase, not the engineer.** A senior engineer in a maintainable system ships a feature in a day; the same engineer in an unmaintainable system spends a day understanding the existing code and a week defending against side-effects. The labor cost of every future change is paid in the maintainability that exists at the moment of the change.

This is why modern SRE practice measures maintainability operationally rather than aesthetically. The **DORA metrics** (Forsgren, Humble, Kim — *Accelerate*, 2018) — Deploy Frequency, Lead Time for Changes, Change Failure Rate, MTTR — are, in effect, four numbers that quantify how maintainable a system is in production terms. *"Our codebase is clean"* is unfalsifiable; *"our lead time for a typical change is 30 minutes"* is not.

The headline cultural cliché — *"good ops can compensate for bad software, but good software cannot run reliably with bad ops"* — is a maintainability claim. Either side of the equation, the system's day-to-day practicality dominates its day-one elegance.

### Three sub-axes

Kleppmann's split is canonical: **operability, simplicity, evolvability**.

| Sub-axis | One-line definition | Anchor question |
|---|---|---|
| **Operability** | The system is easy to *run*. | "When the page fires at 3 a.m., what do I have in my hands?" |
| **Simplicity** | The system is easy to *understand*. | "How long until a new engineer ships their first PR?" |
| **Evolvability** | The system is easy to *change*. | "When the requirements move, does the codebase resist?" |

The names map onto three different **time horizons**:

- **Operability** → daily — incidents, deploys, ad-hoc debugging (minutes/hours).
- **Simplicity** → weekly/monthly — new features, bug investigations (days/weeks).
- **Evolvability** → yearly — architectural shifts, schema migrations, framework upgrades (months/years).

A system can be strong on one and weak on the others. Easy to debug at 3 a.m. (operability ✓) but tangled inside (simplicity ✗) — runbooks are good, no one understands the code. Code is short and clear (simplicity ✓) but riddled with implicit assumptions (evolvability ✗) — adding one field breaks twelve files. The three axes have to be tracked separately because the techniques that improve them are mostly distinct.

---

### 4.1 Operability

Operability asks: *"How much does the system help its operators do their job — and how much does it fight them?"*

#### "Good operations vs good software" — the central asymmetry

Kleppmann's hinge sentence in this section:

> *"Good operations can often work around the limitations of bad software, but good software cannot run reliably with bad operations."*

Both halves matter. A senior SRE team can keep a badly designed legacy system afloat for years through runbooks, automation, and human intervention. And a beautifully designed system, deployed without operational discipline — undocumented deploys, no health checks, no on-call rotation — will be unreliable regardless of how clean its code is.

The goal of operability is to **shrink the gap between the information the system *needs* from the operator and the information the system *gives* the operator**. When that gap is wide — when the operator must read source code to understand what is happening — the system is not operable, no matter what its README claims.

#### Operability is a system property, not a team property

A common confusion is to treat operability as the quality of the operations team. It is not. Give the same senior SRE team two systems:

- **System A** — health endpoints, structured logs, automated failover, every alert linked to a runbook, deploys in one command.
- **System B** — log files scattered across hosts, deploys live in a 14-step wiki page, the failover procedure begins with *"the on-call manually changes the DNS record."*

Same team. On A, the on-call makes the right decision half-asleep; on B, they spend twenty minutes reconstructing what happened. The system's operability — not the operator's competence — sets the ceiling.

The practical test: *"Can a new on-call engineer, equipped only with the runbook, respond to this alert without touching the source code?"* If yes, the system is operable.

#### Components of an operable system

Operability decomposes into a small number of concrete attributes. Industry practice converges on a list close to the following:

| Component | What it is | What its absence costs |
|---|---|---|
| **Visibility** | Metrics, structured logs, distributed traces — the operator can *see* what is happening | The operator guesses; MTTR explodes |
| **Predictability** | Behavior is consistent, surprises are rare | Every incident is a new mystery |
| **Good defaults** | Typical use works with zero configuration; deviations require explicit override | When the one person who knew the setup leaves, the system dies |
| **Automation** | Routine work — deploy, restart, scale, failover — is scripted | Manual work invites human error (§2) |
| **Self-healing** | The system recovers from small faults on its own (restarts, retries, automatic failover) | Every small fault becomes a page |
| **Documentation + runbooks** | Every alert has a *"what does this mean / what to do"* page | Tribal knowledge — leaves when the human leaves |
| **Decoupled lifecycle** | Components can be restarted, upgraded, removed without taking the rest of the system down | Every maintenance window is a full outage |

Each missing item from this list lands as friction on the operator's back. Maturity is not about scoring 10/10 on every row — it is about knowing which rows are weak and tracking the ones that matter most for the system's failure modes.

#### "Production is hostile" — the cultural posture

In §2 the human-error section introduced the *"production is hostile"* posture: manual SSH is exceptional, every change is made through code, audit logs are everywhere. That posture is a **consequence** of operability discipline:

- Manual intervention = opportunity for human error → **minimize**.
- Tooling investment beats manual labor at scale → **pay it up front**.
- Every change is auditable → **post-mortems are possible; tribal knowledge is bounded**.

This is why modern operability conversations revolve around *"take the human's hand off production."* But §2's **automation paradox** still applies: when routine work is automated, what remains for the human is the *hard* edge cases. *"I automated everything, operability is done"* is a milestone that does not exist — operability is a continuous practice, not a state.

#### Operability and reliability — where they meet

Recall §2's availability formula:

```
A = MTBF / (MTBF + MTTR)
```

Operability **directly attacks MTTR**. MTTR has four components:

```
MTTR = detect + respond + repair + verify
```

Operability shows up in all four:

- **Detect** — visibility (the canonical observability pillars: metrics, logs, traces; alerting on top).
- **Respond** — runbook quality, on-call routing, escalation paths.
- **Repair** — self-healing depth, the count of manual steps left in the recovery procedure.
- **Verify** — visibility again (how do you know the system is back?).

§2 noted that *"reducing MTTR is usually cheaper than raising MTBF."* Operability is the practical surface where that bet is paid in. The DORA metrics — Deploy Frequency, Lead Time, Change Failure Rate, **MTTR** — codify it: MTBF is conspicuously absent; the modern wager is on fast recovery, and operability is what makes fast recovery possible.

#### Observability ≠ operability — a quick disambiguation

Two terms that get conflated:

- **Monitoring** — watching for problems you already know about. *CPU > 80% → alert.* The list of things to watch is fixed.
- **Observability** — being able to explain problems you didn't anticipate, from the system's existing output, **without shipping new code** to the system. The phrase entered the system-design vocabulary via Charity Majors and the Honeycomb team in the late 2010s.

Observability is the foundation of operability's *visibility* component. Without observability, no system is fully operable. But observability is **not** operability — having logs and having *"the right log on hand within 30 seconds at 3 a.m."* are different problems. The full treatment of observability — three pillars (metrics, logs, traces), continuous profiling as a fourth — lives in chapter 11. This section uses observability only as one input to operability.

#### Automation, runbooks, self-healing — the decision tree

When a recurring operational task lands on the team, three options are evaluated in order:

1. **Can the work be eliminated entirely?** Make the system heal itself. Always ask first — automation that fixes a problem the system shouldn't have had in the first place is wasted code.
2. **If not, can it be automated?** Script + guardrail (dry-run, rate limit, cell-bounded blast radius — §2). The risk is the *automation paradox*: the surface left for humans grows harder, not easier.
3. **If not, write a runbook.** A runbook is the contract for *"a human runs this, but the steps are determined."* Every alert that pages a human must link to a runbook; *"page fired but no runbook"* is itself an operability bug.

The anti-pattern is **automation without a fallback runbook**. When the automation fails — and it will — a human has to step in; if that human has no runbook, the team improvises under pressure. Correct practice: automation runs by default, failure of automation pages a human *with a runbook*.

The general rule of thumb: any manual step performed **three times** earns either a script or a runbook. Leaving it in the gray zone is what lets tribal knowledge accumulate.

#### Cattle, not pets

The slogan attached to operability — already encountered in §3's stateless discussion — is short and load-bearing:

> Servers must be **cattle** (numbered, interchangeable, replaced when sick), not **pets** (named, special, individually nursed).

The operational test: *"Kill any node, bring up an identically-configured replacement. Does the system behave the same?"* If no, the system is not fully operable — every node carries unique knowledge, and the on-call must keep a map of *"which node does what."* The whole ergonomics of cloud-native runtimes (Kubernetes, ECS, serverless platforms) is built around making this test trivially pass.

---

### 4.2 Simplicity

If operability is *"how easy is the system to run,"* simplicity is *"how easy is it to understand."*

#### Definition — the cost of the mental model

Simplicity is **not** "less code," "fewer features," or "fewer files." Simplicity is **the low cost of the mental model** the system imposes on its readers. When a new engineer reads a feature, how much state do they have to load into their head before the feature makes sense?

The most-quoted formulation is Tony Hoare's:

> *"There are two ways of constructing a software design: one way is to make it so simple that there are obviously no deficiencies; the other is to make it so complicated that there are no obvious deficiencies."*

Simplicity is hard to define directly, so Kleppmann tracks its inverse — **complexity** — which is observable. Recurring symptoms:

- **State explosion** — global variables, implicit dependencies, hidden side-effects.
- **Tight coupling** — changing one module breaks unexpected places.
- **Tangled dependencies** — A depends on B depends on C depends on A.
- **Special cases** — long lists of *"except when X, then Y."*
- **Inconsistent naming** — same concept under three names; different concepts under one.
- **Performance hacks** — *"don't touch this line, it's slow if you do, no one knows why."*

Each symptom is a deferred cost on every future reader.

#### Essential vs accidental complexity

The single most useful distinction in the simplicity literature, from Fred Brooks' *No Silver Bullet* (1986):

- **Essential complexity** — complexity *intrinsic to the problem*. *"You are doing multi-currency invoicing, exchange rates fluctuate, every country's tax code is different."* This complexity is in the world, not the code. There is no way out.
- **Accidental complexity** — complexity *introduced by the way you chose to solve the problem*. Wrong abstractions, unnecessary frameworks, premature generalization, *"we might need this someday"* hooks. This complexity buys nothing; it is pure cost.

The simplicity discipline = **drive accidental complexity toward zero, accept the essential.** Brooks argued that no "silver bullet" tool can attack the essential side (no 10× productivity multiplier). Reducing accidental complexity is a daily engineering discipline, not a tool.

The practical test for any block of code: *"How would I justify this to someone who only knows what the product does?"* If the answer requires concepts that are not part of the problem, the code is accidental.

#### Abstraction — a double-edged tool

Abstraction is the primary lever of simplicity *and* the primary source of accidental complexity. The same instrument cuts both ways.

**A good abstraction** hides a complexity, presents a simple interface, and the user does not need to know what is underneath. Programming languages, file systems, SQL, HTTP, TCP — successful abstractions. *"You can do work without thinking about the layer below."*

**A bad abstraction** does not hide the underlying complexity; it merely *displaces* it or *adds another layer to it*. Joel Spolsky's **Law of Leaky Abstractions**: *"All non-trivial abstractions, to some degree, are leaky."* An ORM hides SQL — until a performance problem forces you to drop into SQL anyway. Kubernetes hides networking — until a DNS bug drags you through the entire CNI plugin stack. When an abstraction leaks, the user has to learn *two* things: the abstraction *and* what it was hiding.

The tests for an abstraction worth keeping:

1. **Does it reduce complexity, or just move it?** If the same complexity sits underneath and surfaces often, you have only added a layer.
2. **Does it cut at the right boundary?** Does the user's mental model match the interface the abstraction provides? Wrong boundary = constant fighting against the abstraction.
3. **Is it paying for genericity it doesn't need?** A single abstraction trying to serve three different users may fit none of them well.

The practical guideline: **concrete code first, abstraction second.** Two similar pieces of code are not yet candidates for abstraction; three similar pieces are (the *Rule of Three*). Abstracting for hypothetical future users — *speculative abstraction* — is the leading source of accidental complexity in real codebases.

#### "Clever code is the enemy"

Simplicity, like operability, requires a posture. The canonical formulation is Brian Kernighan's:

> *"Debugging is twice as hard as writing the code in the first place. Therefore, if you write the code as cleverly as possible, you are, by definition, not smart enough to debug it."*

This is not a stylistic preference; it is a **team-scale** argument. The author of clever code will not be at the company in two years; the next reader has to debug it. The principles related to it — *boring technology* (covered in §9), *YAGNI* (You Aren't Gonna Need It), *Rule of Least Astonishment* (the code should do what its reader expects) — are the daily practice of simplicity.

#### Measuring simplicity

Simplicity does not collapse to a single number. The proxies, in order of usefulness:

| Proxy | What it captures | Limit |
|---|---|---|
| **Onboarding time** — days to first PR | Mental-model cost of the codebase | High variance across people |
| **Cyclomatic complexity** — branches per function | Local complexity | Misses system-level complexity |
| **Coupling metrics** — between-module dependency degree | Structural complexity | Tooling immature |
| **Time-to-debug** — hours from incident to root cause | Operational shadow of understandability | Confounded by incident type |
| **PR review time** — minutes to comprehend a change | Code readability | Depends on reviewer discipline |

No single metric tells the whole story. Mature teams watch the **trend** of these proxies — onboarding *getting longer*, time-to-debug *climbing* — as the early signal.

#### Simplicity and the other axes

The tensions in §1's table land here concretely:

- **Reliability** — adding redundancy, failover, replication makes the system more complex. The 99.9 → 99.99 jump charges complexity as part of its bill.
- **Scalability** — distributed systems are a step more complex than single-node systems. *"Premature horizontal"* (§3) is exactly this trade-off, paid badly.
- **Performance** — clever optimization is the chief enemy of simplicity. Knuth's full quote (almost always truncated): *"Premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."* For the real 3% bottleneck, the complexity is worth it; everywhere else, it is not.
- **Evolvability** — **aligned**. Simple code is changeable code. Loose coupling, modular boundaries, and small abstractions feed both at once.

---

### 4.3 Evolvability

Operability is *"how do I run this system,"* simplicity is *"how does this system work,"* evolvability is *"how do I **change** this system?"*

#### Why evolvability is its own sub-axis

Change is not optional, it is **inevitable**:

- Requirements change — the product hypothesis was wrong; pivot.
- Scale changes — yesterday's architecture does not carry today's load (§3's *"premature/late horizontal"* dilemma).
- Laws change — GDPR, PCI-DSS, evolving regional regulation.
- Dependencies change — your database is deprecated, your framework released a breaking v2, your vendor was acquired.
- Your understanding changes — the *"correct solution"* you committed to two years ago is no longer correct.

No matter how well-designed the day-one system is, *"the design we made on day one"* will not stay valid forever. The evolvability question is sharper: **how painful will the inevitable change be?**

#### Agility vs evolvability — different scales

Two concepts that get conflated:

- **Agility** — change velocity at the **team** level. Scrum, lean, CI/CD, short feedback loops. *"How fast can the team take on new work?"*
- **Evolvability** — change velocity at the **system** level. Once an architectural decision has been made, how expensive is it to switch to a different one. *"Does the existing design accommodate the new requirement, or fight it?"*

An agile team on top of an inevolvable system has fake velocity — sprints close, but every feature takes twice as long because of *structural resistance*. The reverse exists too: a highly evolvable system carried by a slow team — only money is lost.

The practical consequence: agility is a people/process problem, evolvability is an **architectural** problem. Kleppmann separates evolvability from operability and simplicity for exactly this reason.

#### Reversibility — Bezos's one-way and two-way doors

The most useful operational framing of evolvability comes from Amazon — Jeff Bezos's *"two types of decisions"*:

> *Type 1 decisions — one-way doors. Walk through them and you cannot come back. Make them slowly, with lots of analysis.*
>
> *Type 2 decisions — two-way doors. Walk through, see the other side, walk back if you do not like it. Make them quickly.*

System architecture reduces to one question per decision: **what does it cost to change my mind on this?** Low — two-way door, move quickly. High — one-way door, deliberate carefully.

Improving evolvability = **converting one-way doors into two-way doors**. The more decisions can be reversed, the lower the risk of being trapped by a wrong call, the cheaper it becomes for the team to experiment, the less pressure to *"get it perfect now."*

Which architectural decisions tend to be one-way:

| One-way (hard to reverse) | Two-way (easy to reverse) |
|---|---|
| Programming-language choice | Library choice |
| Database engine (Postgres → Cassandra) | Schema change (with migration) |
| Public API contract (consumers in production) | Internal API |
| Data model (denormalized, replicated) | Adding an index |
| Microservice boundaries | Module boundary inside one service |
| Event payload schema (consumers in production) | Adding a new topic |

A discipline lives in the question: **is the thing I am calling a two-way door actually two-way?** *"We will split it later"* often becomes an 18-month refactor three years later — meaning it was one-way all along, just disguised.

#### Coupling — the structural enemy

Evolvability's one-sentence structural question: **when I change one place, how many other places break?**

This question's technical name is **coupling**. Two ends of a spectrum:

- **Tight coupling** — changing module A breaks B, C, D. Change propagates.
- **Loose coupling** — changing A only affects A as long as A's *contract* (interface, schema) is unchanged.

Loose coupling is the raw material of evolvability. The practical instruments:

- **Modular boundaries** — explicit interfaces, hidden implementation. Inside the module, anything goes; the contract to the outside is fixed.
- **Dependency direction** — dependency flow is deliberate; no anti-patterns like *"core domain depends on UI."*
- **Schema as contract** — the internal data structure and the externally-published contract are different objects. Inside the system, change freely; outside, change *versioned*.

The original theoretical argument for microservices was this — *"service boundaries enforce loose coupling."* The argument did not hold in many places in practice (network calls can re-introduce the same tight coupling at higher cost), but the framing of *"track coupling as the metric"* survived.

#### Schema evolution — the concrete touchpoint

The place evolvability shows up most often in daily practice: **changing the data format of a system already in production.** Three scenarios for changing an API request schema or an event payload:

1. **Backward compatibility** — *new server understands old clients*. New fields are optional, old fields stay around. Producers can be upgraded first.
2. **Forward compatibility** — *old server does not crash on what new clients send*. Unknown fields are silently ignored. Consumers can be upgraded first.
3. **Breaking change** — neither holds. All sides must be upgraded simultaneously — close to impossible in a large system.

The serialization formats designed for this (Protobuf, Avro, Thrift) use *tag-based encoding* — fields are referenced by **tag number**, not by name or position. Adding a new field cannot break old readers. The literature on *schema evolution* is built around this discipline. The detail belongs in chapter 4 (communication); the point here is that **evolvability is rhetoric until it has a concrete schema discipline behind it.** *"Our system is open to change"* and *"our event payloads are Protobuf with versioned schemas"* are different sentences.

#### Test coverage as enabler

What practically makes a system evolvable is the system that *tells you what broke when you changed something*. That system is **automated tests**.

A system with low test coverage is mechanically inevolvable — every change shrinks under the question *"did something break?"* Refactoring becomes risky. Ossification (next subsection) sets in.

The rule of thumb: *"If you do not trust that passing tests means you can deploy to production, the system cannot be refactored."* And a system that cannot be refactored cannot evolve, regardless of how elegantly it was designed on day one.

This is why modern *trunk-based development* and *continuous deployment* are, at root, an **evolvability posture**: take small changes, validate them quickly, roll them back quickly. All three of those make evolution practical.

#### Ossification — the maintenance discipline

Systems are not born evolvable; they do not stay evolvable. **Ossification** is the gradual stiffening of a system against change. The causes accumulate:

- New decisions sit on top of old decisions; the lower layers can no longer be changed.
- Test coverage decays; the courage to refactor decays with it.
- Tribal-knowledge holders leave; *"no one dares change this code"* zones appear.
- Production APIs accumulate consumers who cannot all be upgraded.
- Temporary hacks become permanent — *"we will fix it later"* never arrives.

Evolvability is therefore a **maintenance practice**, not an architectural property: *maintainability is not a state you reach, it is a discipline you sustain.*

The practical instruments against ossification:

- **Refactoring slack** — 10–20% of the team's time is spent on code health. Without this slot, technical debt only ever grows.
- **Strangler fig pattern** — instead of replacing a legacy system atomically, the new implementation grows around it; the old one is gradually starved out. Named by Martin Fowler in 2004; the dominant pattern in legacy migration today.
- **Architecture Decision Records (ADRs)** — *"why we made this decision"* is documented at the time of the decision. Three years later, when someone asks *"why is it like this,"* the record answers — and *"this decision is no longer valid"* becomes part of the record.
- **Killing things** — unused code, deprecated endpoints, dead features are *removed*. Keeping them is not free — it is permanent mental load and an attachment point for new coupling.

#### Evolvability and the other axes

- **Reliability** — tension. *"Ship changes constantly"* (evolvability) and *"keep production stable"* (reliability) appear to fight. Modern SRE practice reconciles them with **error budgets** (covered in §8): change as fast as the SLO allows; brake when the budget is spent.
- **Scalability** — tension. A scaling decision (sharding, denormalization) is usually a one-way door — expensive to undo. Evolvability argues for **postponing the decision** — *"do not shard until you have to."*
- **Simplicity** — **aligned**. Simple code is easier to change. Loose coupling, modular boundaries, small abstractions help both. The *"future-proof flexibility"* (factories, plugin systems, config-driven everything) added speculatively breaks both — too soon to know which generalization is correct, you generalize the wrong way.
- **Operability** — aligned. Cattle-not-pets, automated deploys, IaC make change *operationally* cheap. Evolvability's system-level claim resolves to operability's daily practice.

---

### What §4 covers, what §4 doesn't

§4 names the three sub-axes of maintainability and the conceptual lens for each. The day-to-day instruments — the patterns, tools, and operational disciplines that make a system actually maintainable — live in dedicated chapters across the rest of the handbook.

| Concern | Where in the handbook |
|---|---|
| Observability stack — metrics, logs, traces, profiling | Chapter 11 |
| Deployment topology, IaC, blue/green, canary, progressive delivery | Chapter 9 |
| Reliability patterns — circuit breaker, bulkhead, retry/backoff, graceful degradation | Chapter 10 |
| SLI / SLO / error budget — the operational language for trade-offs | §8 of this chapter |
| Schema evolution, API versioning, message format compatibility | Chapter 4 |
| Storage migrations, online schema change | Chapter 7 |
| Engineering principles — boring technology, YAGNI, simplicity as a feature | §9 of this chapter |

§4 supplies the language. The rest of the handbook supplies the techniques.

### §4 recap

In one sentence: **maintainability is the engineering of a system's day-to-day cost — the friction of running it, understanding it, and changing it over its lifetime.**

The three anchors:

1. **Operability** — *help the operators*. Visibility, predictability, automation, runbooks, self-healing, cattle-not-pets. The system is operable when *"any node, any time"* is replaceable and every alert links to a runbook. MTTR is the metric that moves with it.
2. **Simplicity** — *attack accidental complexity, accept the essential*. The mental-model cost of the codebase. Abstractions are double-edged; concrete code first, abstraction by Rule of Three. Onboarding time and time-to-debug are the proxies.
3. **Evolvability** — *make tomorrow's change cheap*. Convert one-way doors into two-way. Loose coupling, schema versioning, test coverage, refactoring slack, the strangler fig. The fight is against ossification, and the discipline is permanent.

The sentence to keep at the top of any maintainability discussion:

> *Most of a system's lifetime cost is paid by the engineers who arrive after launch. Maintainability is the engineering of how much that costs.*

---

## 5. Back-of-envelope estimation

§3 and §4 kept returning to the same point: design decisions are choices about *which order of magnitude the system will encounter*, not choices about which tool sounds clever. *"Should we cache this?"* is a numbers question — DB hit ~5 ms, cache hit ~50 µs, the cache wins by 100×. *"Cross-region replication?"* is a numbers question — round-trip 150 ms, p99 SLO 200 ms, sync replication is dead on arrival. *"Kafka or Postgres?"* is a numbers question — 50k events/s × 1 KB each is the calculation that decides.

An engineer who cannot run these numbers in their head does not have design intuition. The interview is scored on the same numbers; the daily job demands them if you do not want to spend a sprint on a wrong choice.

The good news: the list to memorize is small — about 15–20 lines. The better news: the numbers are useful at **order-of-magnitude precision**. *"Is this 100 ns or 80 ns"* does not matter; *"is it 100 ns, 100 µs, or 100 ms"* does — that is the 1000× decision. Engineering runs on this resolution; physics does not get a say.

This section walks the three sets of numbers that recur in capacity work — latency, throughput, and storage — plus the powers-of-two and time conversions that bind them, and the Fermi discipline that keeps results within useful tolerance. The full reference tables (more systems, more hardware, more scenarios) live alongside in [back-of-envelope-numbers.md](./back-of-envelope-numbers.md).

### Jeff Dean's table — the canonical reference

In 2009 Jeff Dean (one of Google's lead architects, behind MapReduce, BigTable, and Spanner) gave a Stanford talk: *"Designs, Lessons and Advice from Building Large Distributed Systems."* One slide held a small list — *"Numbers Everyone Should Know."* The list circulated and is now referenced across the industry as *"the Jeff Dean numbers."*

The classic table (with 2012 updates — some lines have improved with newer hardware):

| Operation | Time | As a multiple |
|---|---|---|
| L1 cache reference | 0.5 ns | — |
| Branch mispredict | 5 ns | 10× L1 |
| L2 cache reference | 7 ns | 14× L1 |
| Mutex lock/unlock | 25 ns | 50× L1 |
| **Main memory (DRAM) reference** | **100 ns** | **200× L1** |
| Compress 1 KB (Snappy) | 3 µs | 30× DRAM |
| Send 1 KB over 1 Gbps network | 10 µs | 100× DRAM |
| **Read 4 KB random from SSD** | **150 µs** | **1,500× DRAM** |
| Read 1 MB sequentially from RAM | 250 µs | 2,500× DRAM |
| **Round-trip within same datacenter** | **500 µs** | **5,000× DRAM** |
| Read 1 MB sequentially from SSD | 1 ms | 10,000× DRAM |
| **Disk seek (HDD)** | **10 ms** | **100,000× DRAM** |
| Read 1 MB sequentially from HDD | 30 ms | 300,000× DRAM |
| **Round-trip cross-continent (CA ↔ Netherlands)** | **150 ms** | **1,500,000× DRAM** |

The bold lines are the ones referenced in daily practice; the rest provide the ladder that lets the bold lines be reasoned about.

### Reading the table — the "two-orders-of-magnitude ladder"

The right way to learn the table is not line by line. The right way is to recognize **the ladder of tiers**, each tier roughly 100× the previous:

```
Tier 1   ~1 ns       inside the CPU (cache, register, branch)
Tier 2   ~100 ns     RAM (DRAM access)
Tier 3   ~10 µs      fast local I/O (network within rack, SSD random)
Tier 4   ~1 ms       slower local I/O (1 MB transfer, SSD sequential)
Tier 5   ~10 ms      HDD seek (mechanical disk)
Tier 6   ~100 ms     cross-region network (intercontinental)
```

The engineering instinct reduces to one sentence: **"if I move this operation from tier X to tier Y, what is the multiplier?"**

- Cache vs DB → tier 2 vs tier 3 → ~100× faster. Caching is usually justified.
- Local DC vs cross-region → tier 3 vs tier 6 → ~300,000× slower. This is why multi-region replication is *"hard."*
- HDD vs SSD random read → tier 5 vs tier 3 → ~70× faster. The reason SSDs paid for themselves.
- DRAM vs disk → tier 2 vs tier 5 → ~100,000× slower. The reason behind the *"working set must fit in RAM"* rule.

If the exact numbers slip away, the ladder still gives the answer: *"cross-region cache sync sits around 100 ms, ~100,000× slower than local — sync replication is unworkable."*

### Three numbers that force pause

Three facts the average mental model gets wrong:

1. **Network is not free.** The intuition is often *"a network call costs almost nothing."* In reality a local datacenter round-trip is ~500 µs — 5,000× slower than RAM. The full bill of *"every interaction is an RPC"* microservice designs is paid here.
2. **Sequential disk is fast; random disk is brutal.** Even SSD reads at ~3 GB/s sequentially but throttles under random 4 KB IOPS. On HDD the gap is 1,000×. Database design (LSM trees, log-structured storage) is fanatical about sequential writes for exactly this reason.
3. **Cross-region 150 ms = "you can't."** The number is not below 150 ms — it is physics. Light through fiber covers ~5,000 km in ~25 ms one-way; intercontinental round-trip plus router overhead lands around 150 ms. Sync replication across regions means accepting that latency on every write; no production user accepts it. Multi-region systems run async, with eventual consistency — physics decided, architecture followed.

### Has the list aged?

A frequent question. Short answer: **the orders of magnitude are unchanged; some absolute numbers improved.**

- **DRAM** ~100 ns — unchanged. Sits near a physical limit (capacitor row activation); no measurable jump in 15 years.
- **SSD random read** was ~150 µs in 2009; modern NVMe drops to ~20–50 µs. Still ~200× slower than DRAM.
- **Network within DC** can drop below 100 µs with RDMA and modern NICs in some environments, but "typical" is still around 500 µs.
- **Cross-region** unchanged — physics (light speed) is fixed. New fiber does not help; latency tracks distance.
- **HDD seek** unchanged (~10 ms) because it is mechanical. Most production environments no longer touch HDD; HDD is for cold data and backups now.

These numbers will be measured again in five years; most lines will not move, because physics has not.

### Throughput numbers — *"how much per second"*

Latency answers *"how long does one operation take."* Throughput answers *"how many can be done per unit time."* Half of capacity decisions hinge on throughput — *"can one node carry this, or do I need to shard?"*

| System | Typical throughput | Notes |
|---|---|---|
| Single HTTP server (modest JSON API) | **~10k QPS** | Synchronous, small payload. Async I/O reaches 50k+. |
| Single Postgres — read | **~10–50k QPS** | Indexed point reads. Aggregations or scans much lower. |
| Single Postgres — write | **~5–10k QPS** | Bounded by WAL fsync. Batch + group commit reaches 50k+. |
| Single Redis | **~100k QPS** | Single-threaded, in-memory; pipelining reaches 1M+. |
| Single Kafka broker | **~100 MB/s sustained** | Sequential disk write limit. |
| Single Cassandra node — write | **~10k/s** | LSM tree, sequential-write friendly. |
| Single Elasticsearch node — index | **~5k doc/s** | Indexing is CPU-bound; query throughput separate. |
| Load balancer (HAProxy / Envoy) — L7 | **~50k+ QPS** | Varies with connection count. |
| Load balancer — L4 | **~500k+ QPS** | Less work; much higher. |

Hardware tier:

| Hardware | Typical throughput | Notes |
|---|---|---|
| SATA SSD — sequential | **~500 MB/s** | Bus and controller limited. |
| NVMe SSD — sequential | **~3–7 GB/s** | PCIe 4.0; PCIe 5.0 reaches ~12 GB/s. |
| NVMe SSD — random 4 KB | **~100k IOPS** | NAND access; ~10× fewer bytes/s than sequential. |
| HDD — sequential | **~150 MB/s** | Mechanical; not changed much. |
| HDD — random | **~100 IOPS** | Bounded by seek time. |
| 1 Gbps NIC | **~125 MB/s** | Practical upper bound. |
| 10 Gbps NIC | **~1.25 GB/s** | Modern datacenter standard. |

The full per-system and per-hardware tables are in [back-of-envelope-numbers.md](./back-of-envelope-numbers.md).

**How to read these tables.** The numbers are **rough** and depend on CPU, disk, schema, and workload. The use is decisional, not contractual: *"my estimate says 100k write/s — a single Postgres will not carry it; I need sharding or a different engine."* Even if the rough number is off by 1×, the decision usually does not change.

**Two intuition-breakers:**

- **In-memory systems are 10–100× faster.** The Redis vs Postgres gap alone explains the popularity of cache architectures. The slogan *"cache anything you can cache"* lives on this number.
- **Sequential vs random gap is enormous.** The same SSD does ~3 GB/s sequentially, ~400 MB/s on random 4 KB reads — almost a tenfold gap. On HDD the gap is 1,000×. Hence the universal preference in storage engines for *append-only logs*, *LSM trees*, and *write-ahead logs*: all exist to keep disk access sequential.

### Storage and size numbers — typical object sizes

The other half of capacity planning: *"how much disk does this much data occupy?"* This requires knowing how big a single object typically is.

| Object | Typical size | Notes |
|---|---|---|
| UUID (binary) | 16 B | Identifier accounting. |
| Tweet (text) | ~200 B | 280 characters plus metadata; serialized payload larger. |
| Typical JSON API record | 1–5 KB | User profile, post, order, etc. |
| Single structured log line | 200 B – 1 KB | JSON / logfmt; raw text smaller. |
| Thumbnail JPEG (200×200) | 10–30 KB | — |
| Typical web image (1080p JPEG) | 200–500 KB | — |
| Smartphone photo (12 MP, JPEG) | 2–4 MB | RAW: 20–40 MB. |
| 1 minute 1080p video (H.264) | ~50 MB | Bitrate ~6 Mbps. |
| 1 minute 4K video (H.265) | ~250 MB | Bitrate ~30 Mbps. |
| Postgres row overhead | ~24 B | Header + null bitmap; per-row added cost. |

Capacity formula:

```
total_storage = N_objects × avg_size × replication_factor × overhead_multiplier

  - N_objects     : DAU (Daily Active Users) or total, whichever you are counting
  - avg_size      : the matching row from the table above
  - replication   : 3× (classic), 2× (cost-sensitive), 6× (mission-critical)
  - overhead      : 1.5–2× for indexes / metadata
```

A typical example: *"100M users, 50 posts per user, 1 KB average per post, 3× replication, 1.5× overhead"* → 100M × 50 × 1 KB × 3 × 1.5 = **22.5 TB**. Fits on one SSD; if you reach for sharding, the justification has to come from a different axis.

### Powers-of-two cheat sheet

All of the above arithmetic crosses between *power-of-two* and decimal. Equivalences worth knowing by heart:

| Power of 2 | Approx decimal | Name |
|---|---|---|
| 2¹⁰ | 10³ (thousand) | KB |
| 2²⁰ | 10⁶ (million) | MB |
| 2³⁰ | 10⁹ (billion) | GB |
| 2⁴⁰ | 10¹² (trillion) | TB |
| 2⁵⁰ | 10¹⁵ | PB |
| 2⁶⁰ | 10¹⁸ | EB |

Every 10 bits ≈ 3 decimal digits (×1024 ≈ ×1000). The approximation is intentional — the real gap is 1024 / 1000 = 1.024, about 2.4%. Back-of-envelope work ignores it. Where precision matters (vendor disk *"1 TB = 10¹² bytes"*, OS *"1 TiB = 2⁴⁰ bytes"* — IEC notation), it is called out explicitly.

**Time conversions** — capacity calculations usually need *"X events per second"* → *"per day / per year"*:

| Window | Seconds |
|---|---|
| 1 minute | 60 |
| 1 hour | 3,600 (~3.6K) |
| 1 day | 86,400 (~10⁵) |
| 1 month (30 days) | 2.6M |
| 1 year | 31.5M (~3 × 10⁷) |

Practical mnemonic: **1 year ≈ π × 10⁷ seconds** (3.14 × 10⁷ ≈ 31.4M, actual 31.5M — a quirky coincidence that keeps mental math fast).

A quick application: *"100 events per second — how many in a year?"* → 100 × 3 × 10⁷ = **3 × 10⁹ ≈ 3 billion**. *"At 1 KB each, total data?"* → 3 × 10⁹ × 10³ B = **3 TB / year**.

### Order-of-magnitude thinking — the Fermi discipline

Engineering arithmetic is not about avoiding 1.2× errors; it is about avoiding **10× errors**. *"50k QPS"* (Queries Per Second) vs *"47k QPS"* never changes a decision; *"50k"* vs *"500k"* changes the **architecture**. Hence the discipline: *"approximately right beats precisely wrong"* — the spirit of Enrico Fermi's atomic-test estimate (*"how far does this scrap of paper fly when the bomb goes off"*).

Three working rules:

1. **Round, then multiply.** You are not computing 7,234 × 412 in your head. 7K × 0.4K = 2.8M is enough. Error margin 5–10%, easily inside decision tolerance.
2. **Be precise enough to keep the order.** 100K QPS *vs* 1M QPS — that gap moves architecture. 100K *vs* 50K — moves node count, but not architecture.
3. **Sanity-check.** *"Math says I need 10 GB/s"* — then remember 10 GB/s is 80 Gbps, that exceeds a 10 Gbps NIC, and walk back to *"is this estimate right?"* Half of Fermi discipline is asking *"is this even plausible?"* on every result.

### *"Why these numbers"* — a brief physics primer

Three physical facts behind the table. Depth belongs to computer organization and networking textbooks; the goal here is to anchor the numbers to reasoning rather than memory:

- **DRAM ~100 ns.** DRAM stores each bit in a capacitor. A read involves *row activate* (capacitor charge driven onto the bitline), sense amplifier, *row buffer* load, and column select. The chain sums to nanoseconds; dropping below ~50–100 ns requires a different physical technology (HBM, persistent memory). CPU caches (~1 ns) are faster because they are **SRAM** — flip-flop based, much faster but much more expensive per byte.
- **Cross-region ~150 ms RTT.** Light in fiber moves at ~65% of vacuum speed (refractive index ~1.5). US east-to-west is ~5,000 km → ~25 ms one-way. Europe ↔ US is ~8,000 km → ~40 ms one-way. Round-trip plus router/switch jitter and minor overhead lands at 100–150 ms. **This number does not improve** — laying new fiber does not help unless distance shrinks. The argument for edge computing is to cross *under* this number by *bringing the computation to the user.*
- **HDD seek ~10 ms.** Mechanical. A 7,200 rpm disk completes one rotation in 8.3 ms → average rotational latency is half of that ≈ 4.2 ms. On top, the head moves to the right track (seek time) ≈ 5 ms. Total ≈ 10 ms. This number has not improved in 30 years — physical motor speed is the ceiling. The reason SSDs displaced HDDs in production is essentially this one number.

For the interested:

- Brendan Gregg, *Systems Performance* (2020, 2nd ed) — the canonical hardware-and-OS performance reference.
- Colin Scott's interactive *"Latency Numbers Every Programmer Should Know"* — Jeff Dean's table, kept up to date by year.

### What §5 covers, what §5 doesn't

§5 is the *vocabulary* of estimation — three sets of numbers, the powers-of-two and time conversions that connect them, and the Fermi discipline that keeps results within useful tolerance.

The full reference tables — the lines for systems and hardware not in the README's summary tables — live in [back-of-envelope-numbers.md](./back-of-envelope-numbers.md). Use the README to learn the structure; use the supplementary file as the lookup sheet for case-study work and capacity calculations.

The *application* of these numbers — turning *"100M DAU"* into *"peak QPS, peak bandwidth, total storage, hot-set size, working set"* — is §6's subject. §5 teaches the alphabet; §6 teaches the sentences.

### §5 recap

In one sentence: **back-of-envelope estimation is the discipline of binding engineering intuition to numbers — at order-of-magnitude precision, in the head, in 30 seconds.**

Three sets of numbers, all worth memorizing:

1. **Latency hierarchy** — five tiers (cache → DRAM → local I/O → disk → cross-region), each tier ~100× the last. Decision: *"which tier does this operation belong on?"*
2. **Throughput numbers** — single-node ceilings (~10k HTTP QPS, ~50k Postgres read QPS, ~100k Redis QPS, ~100 MB/s Kafka). Decision: *"will one node carry it, or do I shard?"*
3. **Storage / size numbers** — typical object size × N × replication × overhead = total disk. Decision: *"which storage tier (SSD / HDD / object store), how many nodes?"*

Helpers: powers-of-two (2¹⁰ ≈ 10³), time conversions (1 year ≈ π × 10⁷ s), and Fermi discipline (round → multiply → sanity check).

The sentence to keep at the top of any estimation:

> *Approximate is fine. Off by an order of magnitude is not. The discipline is to know which is which.*

---

## 6. Capacity planning ritual

§5 gave the alphabet — single-node QPS ceilings, latency tiers, storage object sizes. §6 turns those into sentences: *can this system carry the next six months of traffic at SLO?*

Capacity is not a single number. It is a *recurring decision* — measured against an SLO, recalculated as load grows, retriggered every quarter, every launch, every incident. This section is about that loop.

### Capacity ≠ performance

Performance and capacity are often confused, especially in interview answers. They are different questions:

- **Performance** — how fast is *one* request? (latency, p99)
- **Capacity** — how many concurrent requests does the system carry *while still meeting that performance*? (throughput @ SLO)

A system can serve 50 ms p99 at 1k QPS and 500 ms p99 at 12k QPS. Performance did not "drop" between those points — capacity was exhausted. Beyond a system's capacity, queues form, and latency rises super-linearly (the queueing math is below, in *Headroom*).

Two consequences:

1. Capacity numbers without an SLO are meaningless. *"This system handles 100k QPS"* is incomplete. *"100k QPS at p99 < 200 ms"* is a capacity claim.
2. The capacity number is bounded by the SLO, not by the hardware ceiling. Hardware can push further; the SLO refuses to follow.

### The five questions

Capacity planning is a process, not an arithmetic exercise. Each cycle answers five questions:

| # | Question | Output |
|---|---|---|
| 1 | Which metrics do we watch? | Load + saturation metric set |
| 2 | What is utilization right now? | Percentages (CPU 62%, DB pool 80%) |
| 3 | What is the headroom target? | Ceiling (e.g., 70%) |
| 4 | How is load growing? | Forecast curve |
| 5 | What happens when the ceiling is crossed? | Trigger + action |

The five answers — measured, written down, reviewed — *are* the capacity plan. Each subsection below covers one of them.

### What we measure — load and saturation

A capacity reading needs both kinds of metrics. Either alone misleads.

**Load metrics — what is arriving:**

- QPS, RPS
- concurrent users, concurrent connections
- writes/sec, reads/sec
- messages/sec (queue producer rate)

**Saturation metrics — how the system absorbs that load:**

- CPU utilization
- memory pressure (RSS, working set, page faults)
- disk IOPS, queue depth
- network bandwidth utilization
- DB connection pool fullness
- thread / file-descriptor exhaustion
- queue depth (Kafka consumer lag, RabbitMQ backlog, worker pool depth)

The two diverge in informative ways:

- **Load high, saturation low** → well-provisioned. No capacity concern yet.
- **Load low, saturation high** → bottleneck inside the system: memory leak, contention, slow downstream. Adding traffic makes it worse, not better.
- **Load high, saturation high** → at the edge. The capacity ceiling is what you measure here.

**The USE method** is the canonical frame for the saturation side, due to Brendan Gregg. For each resource (CPU, memory, disk, network, every pool), ask three questions:

- **U**tilization — how busy is this resource? (% of time)
- **S**aturation — how much extra work is queued for it?
- **E**rrors — how often does it return errors?

USE is a *checklist*, not a tool. Run it on every shared resource in the system. The first resource where U or S is high is the next capacity constraint.

### Headroom — why not 100%?

A natural question: if a CPU has spare cycles, why deliberately leave them empty? The answer is from queueing theory, not policy.

For an M/M/1 queue (single server, Poisson arrivals), average wait time is proportional to:

> **wait ∝ ρ / (1 − ρ)**, where ρ is utilization

Plotted, this is not "linear with a small bend." It is a wall:

| Utilization | Wait factor |
|---|---|
| 50% | 1× |
| 70% | 2.3× |
| 80% | 4× |
| 90% | 9× |
| 95% | 19× |
| 99% | 99× |

The cost of running 50% → 70% is small. The cost of running 90% → 95% is doubling the queue. **Headroom is not slack — it is latency insurance.**

Practical headroom targets, by resource type:

| Resource | Target ceiling |
|---|---|
| Stateless app/API CPU | 60–70% |
| Database CPU | 50–60% |
| Disk I/O | ~50% |
| Network bandwidth | ~50% (microburst risk) |
| Queue depth | should drain — sustained backlog is a saturation signal |

Two reasons headroom is held below the wall, not at it:

1. **Burst absorption.** Spikes (deploys, retries, partial outages, viral content) routinely add 20–30% load. The headroom catches them; the SLO holds.
2. **Provisioning lead time.** Cloud instances arrive in seconds; bare-metal in weeks. The longer the lead time, the larger the buffer needs to be.

### Forecasting — how load grows

Headroom is not a static number. It is held against a *growth rate*. The planning question is: at the current trajectory, *how long until the headroom is consumed?*

Four typical growth shapes:

1. **Linear** — predictable B2B, signup-driven. Forecast is straightforward extrapolation.
2. **Exponential / compounding** — viral consumer, network-effect products. Doubling time = **70 / monthly-percent-growth** months (rule of 70). 6%/month → ~12-month doubling; 15%/month → ~5-month doubling.
3. **Seasonal + spikes** — e-commerce (Black Friday), tax software, event-driven launches. Peak/average ratio of 5–20× is common; capacity is provisioned to peak, idle the rest of the time.
4. **S-curve** — a feature rollout that grows fast then saturates as the addressable user set fills.

**Practical rule:** take the trailing 3–6 months of trend, overlay known events (campaigns, launches, feature flag rollouts), add a 20% safety margin. The forecast itself will be wrong — the discipline is to recheck and replan, not to compute one number perfectly.

### Bottleneck — the first resource to fail

A system runs out of one of five things first: **CPU, memory, disk I/O, network, or a downstream dependency** (database, cache, external API). USE finds it; the capacity ceiling sits there.

Typical bottleneck patterns:

| System | First to saturate |
|---|---|
| Stateless API gateway | CPU or network |
| Read-heavy DB | CPU + cache miss → disk I/O |
| Write-heavy DB | disk I/O (WAL fsync), or replication lag |
| Cache layer | network bandwidth (large values), or CPU on eviction |
| Queue consumer | downstream throughput |
| Image / video pipeline | disk + CPU (encoding) |

The discipline is: **once the bottleneck is found, capacity equals that resource's ceiling — nothing else matters.** Doubling app-tier CPU when the DB connection pool is the bottleneck adds zero capacity; the bottleneck has just moved into sharper focus. Capacity planning is, in the end, bottleneck tracking.

This connects back to §3 — Amdahl says the serial fraction caps speedup, USL says contention and coherence eventually bend it down. The bottleneck is the physical embodiment of those laws inside *this* system. See [scalability-laws.md](./scalability-laws.md) for the full derivation.

### Trigger and action

Once a metric crosses a threshold, something must happen. The plan is written in three tiers:

| Tier | Threshold | Action |
|---|---|---|
| **Warning** | ~70% of ceiling | dashboard signal, team awareness |
| **Critical** | ~85% | on-call alarm, incident channel |
| **Saturation** | ~95% | automated action: auto-scale, load shed, circuit break |

Auto-scaling strategies, used singly or combined:

- **Reactive** — *if CPU > 70%, add an instance.* Simple; lags warm-up time.
- **Predictive** — scale ahead of known patterns (Netflix's evening peak, Uber's Friday-night surge).
- **Scheduled** — *every weekday at 08:00, +N instances.* Trivial, predictable, cheap.

Provisioning lead time governs the choice:

| Provisioning path | Lead time |
|---|---|
| AWS EC2 + ALB | ~30 s |
| Kubernetes pod (warm node) | ~5 s |
| New node from autoscaling group | 1–3 min |
| Bare-metal | days to weeks |

The longer the lead time, the more headroom must be held; reactive auto-scaling assumes seconds-to-minutes provisioning. Bare-metal capacity is *planned*, not reacted to.

### When the ritual runs

Three triggers, each producing a different output:

1. **Quarterly planning** — fold the latest forecast into the capacity ledger. *"Will current provisioning carry the next 90 days?"* If no, file the provisioning work.
2. **Pre-launch / pre-event** — Black Friday, product launch, major feature rollout. Run a load test, validate headroom against expected peak.
3. **Post-incident** — after a saturation incident. *"Why did we not see the ceiling? Which metric was missing?"* This is the most valuable trigger; it improves the ritual itself.

A capacity plan that runs only at moment 1 will be surprised by moment 2 and humiliated by moment 3.

### Anti-patterns

The classic mistakes — common in interviews, common in production:

- **"CPU is fine, so we have room."** CPU may be fine while the DB pool is exhausted. A single metric reading is not a capacity check.
- **Planning against averages.** Peak/average can be 10×. Plan against peak with margin, not against the mean.
- **Headroom set to 90%.** "Spare CPU is wasted CPU" reasoning. The first burst breaks the SLO; the ritual collapses.
- **Forecast frozen in time.** A forecast written six months ago is a plan against last quarter's traffic.
- **Single-tier auto-scale.** App tier scales, database does not — the bottleneck migrates and breaks at the seam.
- **Synthetic load test as proof.** Real traffic has long-tail key distributions, hot keys, bursty arrival patterns. Synthetic tests miss these and overstate capacity.

### What §6 covers, what §6 doesn't

§6 is the *operating loop*: the questions, the cadence, the math behind headroom, the framework (USE) for finding the bottleneck. It treats capacity as a continuous engineering practice rather than a one-shot calculation.

What §6 leaves to later chapters:

- **Auto-scalers as products** — HPA, KEDA, AWS Auto Scaling Groups, predictive scalers — get concept cards in chapter 09 (Deployment and Scaling). §6 names the strategy types; chapter 09 covers the tools.
- **SLI / SLO formalism + error budgets** — §8.
- **Observability stack** (Prometheus, dashboards, alerting wiring) — chapter 11. §6 names the metrics; chapter 11 covers their plumbing.
- **Per-storage-engine capacity tuning** (Postgres connection pooling, Redis maxmemory policies, Kafka partition sizing) — covered alongside each system in chapters 03 (Data) and 07 (Storage).

### §6 recap

In one sentence: **capacity is not a number; it is a recurring decision — *throughput at SLO*, measured against a forecast, bounded by a tracked bottleneck, defended with headroom whose size is dictated by queueing math and provisioning lead time.**

The five questions, restated:

1. What load and saturation metrics do we watch? (USE every shared resource.)
2. Where is utilization now?
3. What headroom does the SLO require? (Look up ρ / (1 − ρ); decide which utilization bend the SLO can survive.)
4. How is load growing? (Linear / exponential / seasonal / S-curve, with a 20% margin.)
5. When the ceiling is approached, what fires? (Warning → Critical → Saturation, with reactive / predictive / scheduled scaling matched to provisioning lead time.)

Bind the answers to a cadence — quarterly, pre-launch, post-incident — and the result is a capacity plan that survives contact with real traffic.

> *Capacity is not a number — it is throughput at SLO, replanned every cycle.*

---

## 7. Mental models

§5 gave the alphabet — back-of-envelope numbers. §6 gave the loop — the capacity ritual. §7 gives the *lenses*: the mental shapes that turn a system diagram into the right *questions*. A lens does not produce an answer. It forces a particular question to be asked, and reveals which side of a tradeoff the system has chosen.

Mental models earn their place in three ways:

- They give a checklist to enter any case study with.
- They keep tradeoffs from being forgotten under interview pressure.
- They produce, fast, the sentence *"this system is strong here, weak there."*

Four categories: **trade-off, architectural, failure, organizational.**

### A. Trade-off lenses

#### A.1 CAP theorem

Brewer, 2000. A distributed system, **during a network partition**, can hold at most two of:

- **C** — Consistency: every read returns the most recent write
- **A** — Availability: every request gets a response
- **P** — Partition tolerance: the system continues despite a partition

Partitions are inevitable in real systems, so P is given; the choice is **C vs A**. Systems are labeled **CP** (Spanner, etcd, ZooKeeper) or **AP** (Cassandra, DynamoDB, Riak). "CA" is shorthand for *"I have not designed for partitions."*

**Why the lens matters:** it forces the question *"what does this system do during a partition?"* Either it serves stale data (AP) or it returns errors (CP). Neither is wrong — the answer depends on the use-case (banking → CP; shopping cart → AP).

**Trap:** CAP only describes behavior during a partition. It says nothing about normal-operation behavior. That gap is closed by PACELC.

#### A.2 PACELC

Abadi, 2010. The mature form of CAP:

- **P**artition: choose **A** vs **C**
- **E**lse: choose **L**atency vs **C**onsistency

A system carries a tradeoff in *normal* operation too. Synchronous replication gives strong consistency at the cost of latency; asynchronous replication gives low latency at the risk of stale reads.

| Class | Examples |
|---|---|
| **PA/EL** | Cassandra, DynamoDB (default reads) |
| **PC/EC** | Spanner, FoundationDB |
| **PA/EC** | DynamoDB strong-consistent reads |

**Why the lens matters:** *"What's the system's full PACELC class?"* is sharper than *"is it CP or AP?"* CAP is the entry; PACELC is the working tool.

#### A.3 Latency vs throughput

Two independent readings of the same capacity:

- **Latency** — time for *one* request to complete
- **Throughput** — requests completed per unit time

They do not improve together. Batching raises throughput at the cost of per-item latency. Stream processing lowers latency at the cost of throughput per node.

**Why the lens matters:** *"Is this operation latency-sensitive or throughput-sensitive?"* drives the architecture. The user-facing path is latency-sensitive; the analytics path is throughput-sensitive — the same data flows through two different systems.

#### A.4 Consistency spectrum

Strong vs eventual is not a binary; it is a gradient:

| Level | Guarantee |
|---|---|
| Linearizable | Behaves as if there is one node |
| Sequential | All nodes see the same order |
| Causal | Causally-related writes preserve order |
| Read-your-writes | A user reads their own writes |
| Monotonic reads | Reads do not go backwards in time |
| Eventual | Given enough time, everyone agrees |

**Why the lens matters:** *"Which level is enough?"* A bank balance demands linearizable; a social-media like counter is content with eventual. Stronger consistency is more expensive and slower — pay for what is needed, not more.

### B. Architectural lenses

#### B.1 Push vs pull

Where is the initiative?

- **Push** — producer sends when ready (webhooks, server-sent events, Kafka producer→broker)
- **Pull** — consumer asks when ready (polling, Kafka consumer→broker, Prometheus scrape)

Push gives low latency but demands backpressure; pull gives consumer control at the cost of overhead and delay.

**Why the lens matters:** notifications are push (instant); metrics scrape is pull (Prometheus model); email is pull (the client fetches). Naming the choice forces awareness of why it was made.

#### B.2 Sync vs async

Does the caller wait?

- **Sync** — caller blocks until response; simple, easy to debug
- **Async** — caller dispatches and continues; response delivered via callback, queue, or future

Sync↔async is *orthogonal* to push↔pull. HTTP request/response is a sync push. A webhook is an async push. A blocking poll is a sync pull. A queue consumer is an async pull.

**Why the lens matters:** *"Is the user waiting on this call?"* If yes, sync fast path; if no, async + queue. Over-sync produces cascading failure; over-async produces debugging hell.

#### B.3 Stateful vs stateless

Does the server hold session/data between requests?

- **Stateless** — every request is self-contained; any node serves the same answer
- **Stateful** — the server keeps history (session, connection, in-process cache)

Covered in §3 as the prerequisite for horizontal scaling. As a lens: *push state outside* — to a database, cache, or client. State-bearing nodes are hard to scale, lose data on restart, and demand sticky routing.

**Why the lens matters:** *"Is this component stateful? Does it need to be?"* If yes (database, cache, queue), its scaling story is fundamentally different.

#### B.4 Read path vs write path

The same data has two paths through the system, and they are rarely symmetric:

- **Read path** — cache → replica → primary
- **Write path** — validate → primary → cache invalidate → replicate → fan-out

They are asymmetric because reads outnumber writes 10–100×, reads tolerate staleness, and writes demand atomicity.

**Why the lens matters:** once *"write-heavy or read-heavy?"* is answered, the system is drawn as **two systems**. The Twitter timeline is canonical: fan-out-on-write (the Bieber problem — one celebrity tweet pushes to millions) versus fan-out-on-read (the Kim Kardashian problem — millions of followers pulling on demand) — same feature, different architectures.

### C. Failure lenses

#### C.1 Idempotency

An operation is **idempotent** if calling it N times with the same input has the same effect as calling it once.

- `SET balance = 100` → idempotent
- `INCREMENT balance BY 100` → not idempotent
- `DELETE WHERE id=42` → idempotent (second call is a no-op)
- `INSERT INTO orders VALUES (...)` → not idempotent

In a distributed system, retries are unavoidable (timeouts, partial failures, dropped responses). Idempotency makes retries *safe*.

**Why the lens matters:** *"Can I make this call idempotent? If not, can I attach an idempotency key + dedup table?"* Stripe, AWS, Square — every modern API supports idempotency keys. Idempotency is the smallest safe building block in distributed systems.

#### C.2 Delivery semantics

How many times does a message get through?

- **At-most-once** — delivered zero or one times. Messages can be lost; no duplicates. (UDP, fire-and-forget.)
- **At-least-once** — delivered at least one time. Duplicates possible. (Kafka default, SQS.)
- **Exactly-once** — delivered exactly once. *No pure form exists in distributed delivery* — it is produced by combining at-least-once delivery with idempotent consumers.

**The motto:** *exactly-once **delivery** does not exist; exactly-once **processing** does — at-least-once delivery + idempotent consumer.* For money movement and similar, the transactional outbox pattern wires the two together.

#### C.3 Failure domains and blast radius

- **Failure domain** — the boundary of things that die together (rack, AZ, region, account, deployment unit)
- **Blast radius** — the area affected when something fails

| Architecture | Failure domains | Blast radius |
|---|---|---|
| Single DB | 1 | the entire system |
| Multi-AZ replicas | 3 | one third of traffic |
| Cell-based (Slack, AWS) | many cells | one cell's customers |

**Why the lens matters:** *"What goes down when this dies?"* If the answer is "everything," isolation (bulkhead, cell, sharding) is the next architectural move.

#### C.4 Backpressure

What happens when production exceeds consumption?

- The buffer fills, then bursts
- A "slow down" signal travels back to the producer (TCP flow control, gRPC streaming, Reactive Streams)
- Load is shed

A pipeline without backpressure produces **runaway queues** and **cascading failure**: fast producer + slow consumer → queue grows → memory pressure → consumer slows further → queue grows more → collapse.

**Why the lens matters:** *"Is there backpressure between this producer and this consumer?"* If not, the bottleneck must be tracked (§6) and producers throttled deliberately.

### D. Organizational lens

#### D.1 Conway's Law

Conway, 1968:

> *Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure.*

Four teams produce a four-component system. Team boundaries become system boundaries. The reverse — the *Inverse Conway Maneuver* — restructures teams to *force* the architecture you want.

**Why the lens matters:** *"Why is this system carved this way?"* is often not a technical question. Where two teams communicate poorly, a friction surface forms — and that surface becomes an API.

### Anti-patterns when applying the lenses

- **Treating a lens as dogma.** *"AP is better,"* *"push is more modern"* — these are slogans, not lens use. A lens forces a tradeoff; the answer depends on the system.
- **Looking through a single lens.** A system can be AP under CAP but PA/EL or PC/EC under PACELC; every lens leaves a blind spot.
- **Applying CAP outside a partition.** CAP only fires during partition. Normal-operation behavior is the EL/EC side of PACELC.
- **Believing in exactly-once delivery.** Any system claiming it is, in fact, doing at-least-once + idempotency underneath.
- **Using Conway's Law as a complaint.** *"The org is messy"* is an explanation, not a fix. The Inverse Conway Maneuver demands the org change too.

### Summary table

| Category | Lens | Question it asks |
|---|---|---|
| Trade-off | CAP | C or A during partition? |
| Trade-off | PACELC | Tradeoff during partition *and* in normal operation? |
| Trade-off | Latency vs Throughput | Which is being optimized? |
| Trade-off | Consistency spectrum | Which level is enough? |
| Architectural | Push vs Pull | Where is the initiative? |
| Architectural | Sync vs Async | Is the caller waiting? |
| Architectural | Stateful vs Stateless | Does the node hold state? |
| Architectural | Read path vs Write path | Which direction is being optimized? |
| Failure | Idempotency | Is retry safe? |
| Failure | Delivery semantics | How many times is the message delivered? |
| Failure | Failure domain / Blast radius | What goes down when this dies? |
| Failure | Backpressure | How is producer–consumer balance maintained? |
| Organizational | Conway's Law | Why is the system carved this way? |

### What §7 covers, what §7 doesn't

§7 is a *catalogue* — short, practical definitions and the question each lens forces onto the table. The *deep* treatment of each subject lives in the chapter where the mental model is most operational:

- CAP, PACELC, consistency spectrum, idempotency, delivery semantics → chapter 06 (Consistency and Consensus)
- Push vs pull, sync vs async, backpressure, delivery semantics → chapter 04 (Communication)
- Read path vs write path, stateful vs stateless → chapter 03 (Data) and chapter 05 (Caching)
- Failure domains, blast radius → chapter 10 (Reliability Patterns)
- Conway's Law → chapter 09 (Deployment and Scaling)

§7 carries the lens *names* so later chapters can refer back to them.

### §7 recap

In one sentence: **mental models are the lenses through which a system is interrogated — they do not produce answers, they force the right questions.**

The discipline:

> *Before describing a system, declare which lenses apply. For each lens, state which side this system is on.*

A sample interview cadence: *"Under CAP it is AP; under PACELC it is PA/EL. The read path is cache→replica, the write path is primary→fan-out. Failure domain is the AZ; idempotency is via a client-supplied key. Four teams, four services — Conway."* Not one sentence — a chain of sentences, each produced by a lens.

> *Lenses compose. A single lens is always half-wrong.*

---

## 8. SLI, SLO, SLA, and error budgets

These three terms are the most-confused trio in interview answers. Once the layering is clear, the rest follows from arithmetic.

### Three terms, three layers

- **SLI** *(Service Level Indicator)* — a **measurement**. A number. *"p99 latency = 187 ms"*, *"successful requests / total = 0.9994"*
- **SLO** *(Service Level Objective)* — an **internal target**. *"p99 < 200 ms, 99.9% of the time, over a 28-day rolling window"*
- **SLA** *(Service Level Agreement)* — an **external contract**, with a financial penalty. *"If monthly availability drops below 99.5%, customer receives a 25% credit"*

The layering is **SLI ⊂ SLO ⊂ SLA**: SLI is the raw number; SLO is the target the team holds itself to; SLA is what is promised to a customer. SLO is always *stricter* than SLA — typical pattern: SLA promises 99.5%, SLO targets 99.9%, leaving margin to repair before the contract is breached.

### Choosing a good SLI

A useful SLI:

- Measures the **user journey**, not the resource. CPU utilization is not an SLI; *"successful logins / login attempts"* is.
- Is a **proportion or rate**, not a raw count. Comparable across volumes.
- Is bound to a **window** (28-day rolling is standard).

Google SRE's **Four Golden Signals** are the canonical starting set: **latency, traffic, errors, saturation**. Most SLIs are formal expressions of one of these.

### The nines table

The cost of each additional 9 was already shown in §1; SLO targets sit on this ladder:

| SLO | Monthly downtime | Yearly |
|---|---|---|
| 99% | 7.2 h | 3.65 d |
| 99.9% | 43.2 min | 8.76 h |
| 99.99% | 4.32 min | 52.6 min |
| 99.999% | 26 s | 5.26 min |

Each 9 costs roughly 10× more in engineering, infrastructure, and architectural complexity (see §1). The SLO is the *budget statement*: how much reliability is the team willing to fund?

### Error budget — fuel for deploy velocity

**Error budget = 1 − SLO.** A 99.9% monthly SLO yields a 0.1% budget = 43 minutes/month of tolerated unavailability.

The insight from Google SRE: **a 100% reliability target is irrational** — it is expensive, unnecessary, and freezes feature work. A defined error budget turns reliability into **fuel for deployment**:

- **Budget burning fast** (e.g., 30 minutes already consumed by mid-month) → slow deploys, bug fixes only, reduce risk
- **Budget under-consumed** → ship features, run risky experiments, deploy more frequently

This converts the classic SRE-vs-Dev tension (*"don't break it"* vs *"ship faster"*) into a **numerical contract** instead of a political one.

### Interview cadence

The sentence that earns the marks:

> *"SLI is a number, SLO is an internal target, SLA is an external contract with a penalty. SLO is always stricter than SLA, leaving repair margin. Error budget = 1 − SLO; the budget governs deploy velocity — burn fast, freeze; burn slow, ship."*

Deeper material — multi-window burn-rate alerting, SLO ladders by service tier, the operational details — lives in chapter 11 (Observability).

### §8 recap

| Term | Owner | Output |
|---|---|---|
| SLI | Engineering | A number |
| SLO | Engineering + Product | Internal target; deploy discipline |
| SLA | Legal + Sales | Customer contract; financial penalty |
| Error budget | SRE + Dev jointly | 1 − SLO; the velocity governor |

> *Reliability is not a number; it is a contract — with whom, for how much, under what penalty.*

---

## 9. Engineering principles

A small set of well-worn rules that earn their place not by being clever but by being repeatedly *correct under pressure*. They rarely show up as direct interview questions; they show up as the **discipline behind a tasteful design answer** — the reason one diagram is judged "senior" and another "over-engineered."

Each principle is a *bias*, not a law. Together they pull a design toward simplicity, reliability, and reversibility.

### 9.1 YAGNI — *You Ain't Gonna Need It*

Build for the requirements you know, not for the ones you imagine. Speculative features carry the same maintenance cost as real ones, with none of the value.

**Where it bites:** the "extensible plugin system" that nobody plugs in to; the abstract base class with one concrete subclass forever.

### 9.2 KISS — *Keep It Simple*

Complexity is not free; every abstraction is a maintenance debt and a learning tax. Solve the simplest version first; let the second version emerge from real friction, not anticipated friction.

**Where it bites:** premature microservice decomposition; three-layer queue chains where one queue would have done.

### 9.3 End-to-end principle

Saltzer, Reed, and Clark (1984): reliability checks belong at the **endpoints**, not in the intermediate layers. Networks, queues, and middleware do their best; the truth is verified at the edges.

**Where it bites:** trusting that "the queue is reliable" instead of having the consumer verify and ack. Trusting TCP for application-level integrity. The sender writes a record; the receiver re-checks the checksum.

### 9.4 Principle of least surprise

A system's behavior should be predictable from its name and shape, not from its documentation. If a function called `getUser` writes to the database, somebody is going to be surprised — usually at 3 AM.

**Where it bites:** APIs whose verbs lie (`POST /users/refresh` that deletes accounts); side effects hidden behind pure-looking names; defaults that differ between dev and prod.

### 9.5 Fail fast, fail loudly

A failure that is visible at the second it happens costs minutes. A failure that is silenced costs days, and surfaces in unrelated alerts. Errors should propagate, log, and alert — not be swallowed by a `catch` that returns zero.

**Where it bites:** catch-all blocks that hide schema mismatches; retry loops that mask a downstream outage; default values returned in place of errors. The cost compounds: every silent failure is a future incident with no breadcrumbs.

### 9.6 Make the right thing easy, the wrong thing hard

Defaults should land in the safe place. The risky path should require explicit opt-in — a flag, a comment, a code review.

**Where it bites:** raw SQL string concatenation easier than a parameterized query (SQL injection); a `delete()` API with no soft-delete option; a config that ships with `debug: true`. If safety requires discipline, it eventually fails; if safety is the default, the principle scales.

### 9.7 Boring technology — Dan McKinley's "innovation tokens"

A team has a fixed budget of *innovation tokens* per year. Spend them on the parts of the system where novelty is worth it; for everything else, choose the boring, well-understood option. *Postgres + Redis + a queue* covers more designs than any new database does.

**Where it bites:** adopting three new datastores at once and burning the on-call team; choosing a fashionable framework whose Stack Overflow page is empty; trading hiring leverage for technical novelty.

### 9.8 Reversibility — one-way vs two-way doors

Bezos (referenced in §4 Maintainability): decisions split into two-way doors (cheap to reverse — try, observe, adjust) and one-way doors (cheap to make, expensive to undo). Make two-way decisions *fast*; make one-way decisions *slow* and with deliberation.

**Where it bites:** treating a public API contract as a two-way door (it isn't — once shipped, customers depend on it); treating a feature flag as a one-way door (it isn't — flip it, observe, flip back).

### Anti-patterns when applying principles

- **Treating a principle as a law.** *"YAGNI says don't add it"* is not an argument; the question is whether the requirement is real *now*. Principles are biases under uncertainty.
- **Stacking principles to win an argument.** Selecting the principle that supports a preferred answer is post-hoc reasoning; pick the design first, then check whether the principles applied honestly.
- **Quoting the slogan without the *why*.** *"KISS"* is not a design review comment. *"This is two layers of indirection for one caller — KISS"* is.

### §9 recap

| Principle | Bias |
|---|---|
| YAGNI | Build the known, not the imagined |
| KISS | Simple first; complexity earns its place |
| End-to-end | Verify at the edges, not in the middle |
| Least surprise | Behavior matches the name |
| Fail fast, fail loudly | Visibility beats silence |
| Right thing easy | Defaults are safe; risk is opt-in |
| Boring technology | Spend novelty where it pays |
| Reversibility | Cheap to undo → fast; expensive → slow |

> *Principles are not laws. They are the gravity a good design falls toward when nobody is watching.*

---

## Next

- §10 — Interview-ready summary

Sections are added as they are studied. Some will graduate into standalone supplementary files in this folder.
