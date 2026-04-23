# How to approach a system design question

Whether you're in an interview or kicking off a real project, the first 5–10 minutes are not about design. They're about **reducing uncertainty**: turning a vague ask into a concrete, scoped problem. If you start drawing boxes before you've done this, you'll design the wrong system.

Use this as a checklist. Not every question applies every time, but most applies most of the time.

## 1. Scope — what exactly are we building?

- What are the **core user-facing features**? List 3–5.
- What is explicitly **out of scope**? (Interviews: always ask. Real projects: always document.)
- Who are the **actors**? End users, admins, other services?
- Is this a **greenfield** design or an extension of an existing system?

> **Interview tip:** state the scope out loud and get agreement. "I'm going to design X, Y, Z. I'll skip W unless you want to include it — OK?"

## 2. Scale — how much, how fast?

- **DAU / MAU** — daily and monthly active users
- **Peak traffic multiplier** — usually 2–5× average
- **Read vs write ratio** — read-heavy (100:1), balanced, or write-heavy (1:1)?
- **Request rate (QPS)** at peak
- **Payload size** — average and p99
- **Storage growth** — per day, per year
- **Retention** — how long do we keep data?

If the interviewer gives no numbers, **make reasonable assumptions and state them**. "I'll assume 100M DAU, 10:1 read:write, avg payload 1 KB."

## 3. Performance & availability targets

- **Latency budget** — p50, p95, p99 for core operations
- **Availability target** — 99.9%? 99.99%? (each extra nine costs an order of magnitude more)
- **Consistency requirements** — must reads always see the latest write, or is a few seconds of staleness OK?
- **Durability** — can we tolerate data loss? Ever? How much?

## 4. Data shape

- What **entities** exist and how do they relate?
- Are relationships deep (many joins) or shallow?
- **Read patterns** — by ID? by range? full-text? geographic?
- **Write patterns** — one row at a time, or bulk ingest?
- Is data **structured** (fits a schema) or **unstructured** (blobs, media)?

## 5. Special requirements

Ask about each explicitly — don't assume.

- **Geographic distribution** — single region, multi-region, global?
- **Real-time needs** — push (websocket, SSE) or is polling fine?
- **Full-text search** — needed? fuzzy matching? faceting?
- **Media handling** — images, video, file uploads? processing pipelines?
- **Offline support** — mobile clients that sync?
- **Compliance** — GDPR, HIPAA, PCI? data residency?
- **Cost sensitivity** — "cheap" vs "performance at any cost"?

## 6. Failure expectations

- What happens if a single **node** dies? A **zone**? A **region**?
- Is **downtime** ever acceptable? (Scheduled maintenance?)
- Are there **hot paths** that must never be down (login, checkout) vs cold paths that can degrade?

## 7. The shape of your answer

After clarification, narrate your approach. Typical structure:

1. **Capacity estimation** — derive QPS, storage, bandwidth from the numbers in step 2
2. **API design** — main endpoints and their contracts
3. **Data model** — tables/collections/keys
4. **High-level architecture** — draw the boxes (client → LB → API → cache → DB → …)
5. **Deep dives** — for each bottleneck the interviewer cares about, explain the choice and the alternatives
6. **Trade-offs & what you'd do differently** — show you know your design isn't perfect

## Mini-checklist to keep in your head

> **Scope → Scale → Performance → Data → Specials → Failure → Design**

If you remember nothing else, remember this sequence. It prevents the #1 interview failure: jumping to a shiny architecture before understanding the problem.

## Anti-patterns — common mistakes to avoid

These are the recurring failure modes in both interviews and real projects. If you catch yourself doing one, stop and reconsider.

### Design process

- **Jumping to architecture without clarifying.** Drawing boxes before you've pinned down scope, scale, and special requirements. The #1 cause of designing the wrong system.
- **Not narrating trade-offs out loud.** Presenting a design as if it's the only option. Interviewers are grading your *reasoning*, not your final diagram.
- **Resume-driven development.** Picking Kafka because it sounds impressive when a cron job would do. Pick tools for the problem, not the CV.

### Over-engineering

- **Designing for scale you don't have.** Building for 1B users when you have 10K. Complexity is a tax; pay it only when you need to.
- **Premature microservices.** Splitting a small monolith into ten services before you understand the domain boundaries. Often makes everything worse.
- **Sharding too early.** Adding sharding complexity before vertical scaling + read replicas have been exhausted.
- **Distributed transactions everywhere.** Reaching for 2PC when a saga, outbox pattern, or just eventual consistency would work.

### Data and consistency

- **Cache as a band-aid.** Using caching to hide a slow query or a bad data model instead of fixing the root cause. Your cache *will* miss eventually.
- **Wrong consistency model.** Using eventual consistency for a bank ledger, or strong consistency for a like counter. Consistency costs — spend it where it matters.
- **Ignoring the simplest solution.** Reaching for Cassandra before asking if Postgres with read replicas would work. A boring, well-understood tool beats a shiny one you'll operate poorly.

### Operations

- **Single points of failure.** One load balancer, one primary DB without failover, one region. If it can fail, it will — design for redundancy at every tier.
- **Synchronous when async would work.** Making the user wait for an email to send, a thumbnail to generate, or a report to render. Push slow work off the request path.
- **"We'll add observability later."** Adding logs, metrics, and traces *after* a production fire instead of before. By then you're debugging blind.
- **Ignoring operational cost.** Designing for peak performance without considering what it costs to run, monitor, and maintain at 3 AM on a Sunday.
