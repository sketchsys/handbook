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

## Next

- §4 — Maintainability deep dive
- §5 — Back-of-envelope estimation
- §6 — Capacity planning ritual
- §7 — Mental models
- §8 — SLI / SLO / SLA and error budgets
- §9 — Engineering principles
- §10 — Interview-ready summary

Sections are added as they are studied. Some will graduate into standalone supplementary files in this folder.
