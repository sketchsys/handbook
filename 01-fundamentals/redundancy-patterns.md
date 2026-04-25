# Redundancy patterns

The structural toolkit for converting fault tolerance into actual availability. Redundancy means having more than one of a component; the patterns differ on four axes:

1. **How many spares?** (1, N, percentage, quorum-sized.)
2. **Is the spare working?** (Active vs passive.)
3. **How ready is the spare?** (Cold / warm / hot.)
4. **How is it geographically distributed?** (Rack / AZ / region / multi-cloud.)

This page catalogs the canonical patterns. Conceptual context — fault model, the metrics, the math that makes redundancy work — is in [§2 of the chapter](./README.md#2-reliability--deep-dive) and [availability-math.md](./availability-math.md).

## Active-passive (failover)

One primary handles all traffic. One or more standbys wait. On primary failure, a standby is promoted.

Three flavors based on standby readiness:

| Flavor | Standby state | Failover time (RTO) | Typical use |
|---|---|---|---|
| **Cold** | Off. Boot, configure, restore data | Minutes to hours | DR sites, low-cost workloads |
| **Warm** | Running, periodic data sync, no traffic | Seconds to ~1 minute | Most RDBMS replicas, batch systems |
| **Hot** | Running, sync replication, ready to serve | Sub-second to seconds | High-availability databases, financial systems |

**Pros**

- Simple mental model — single-writer invariant preserved, no conflicts.
- Standby capacity is held in reserve, not racing.
- Consistency story is clean for stateful primaries.

**Cons**

- **Idle resource cost.** You pay for capacity you don't use.
- **The failover path is rarely exercised** and can fail when needed. *Untested redundancy is no redundancy.*
- **Split-brain risk.** If the old primary returns after promotion, you have two writers. Production-grade systems use *fencing* — physically or logically isolating the old primary before promoting the standby.
- **Replication lag.** Under async replication, failover loses data — RPO ≠ 0.

Examples: Postgres streaming replication, MySQL replication, Redis Sentinel, Patroni.

## Active-active

All replicas serve traffic simultaneously. Failure of one means survivors absorb the load.

**Pros**

- No idle resource — all replicas are productive.
- Continuously validated — every replica is in production all the time, no cold-path surprises.
- Lower latency — clients can route to the nearest replica.
- A single replica's death is a *capacity* problem, not a *failover* problem.

**Cons**

- **Consistency is hard with shared state.** Multiple writers require conflict resolution: last-write-wins, CRDTs, vector clocks. (Chapter 6.)
- **Headroom over-provisioning.** With N replicas serving N replicas' worth of traffic, you must run at < (N−1)/N utilization to absorb a failure. Running at 50% load is common, meaning half the capacity is reserved for failure.
- **Coordination overhead** — leader election, distributed locking, or conflict resolution costs CPU and latency.

Examples: stateless API tiers, DNS resolvers, CDN edges, search read replicas, stateless function executors.

## N+1, N+M, 2N, 2N+1 — capacity-budget patterns

These describe how much spare capacity you reserve relative to demand:

| Pattern | Meaning | Tolerates |
|---|---|---|
| **N+1** | Need N units, run N+1 | 1 concurrent failure |
| **N+M** | Need N units, run N+M | M concurrent failures |
| **2N** | Need N, run 2N | N concurrent failures (50% of fleet) |
| **2N+1** | Consensus systems requiring odd quorum | N failures while preserving majority |

The choice depends on the **size of the failure mode you defend against**:

- Single-server failures → N+1 is enough.
- Whole-AZ outages, when one AZ holds 33% of fleet → effective need for **2N** distribution across 3 AZs at 50% headroom each.
- Multi-region outages → 2N at the region level.

**2N+1 and consensus.** Systems that need majority quorum (Raft, Paxos, ZooKeeper) prefer odd counts. A 5-node cluster tolerates 2 failures; a 3-node cluster tolerates 1. A 4-node cluster tolerates only 1 (since losing 2 of 4 destroys the majority), so the 4th node is dead weight — it costs as much as the 5th but doesn't increase tolerance.

## Geographic redundancy

Failure domains nest:

```
process → server → rack → AZ → region → cloud provider → planet
```

(The last layer isn't a joke — undersea cable cuts can reduce intercontinental availability.)

Each level adds:

- **Cost.** Cross-region replication, network egress, doubled storage.
- **Complexity.** Weaker consistency models, latency tolerance, divergent control planes.
- **Independence.** Higher levels fail less often but blow bigger when they do.

Practical patterns:

| Pattern | Tolerates | Cost | Typical use |
|---|---|---|---|
| **Single region, multi-AZ** | AZ outage | Low (intra-region network is cheap) | Default for most production |
| **Active-passive multi-region** | Region outage | Moderate (warm secondary + cross-region replication) | Stateful systems — RDBMS, payments |
| **Active-active multi-region** | Region outage + low global latency | High (engineered consistency, dual writes, conflict resolution) | Hyperscaler products — DynamoDB Global Tables, Spanner, S3 |
| **Multi-cloud** | Cloud-provider-wide outage | Very high (operational duplication, no shared tooling) | Rarely worthwhile; a vendor-leverage argument more than an availability one |

The hyperscale reality — S3, Spanner, DynamoDB Global Tables — runs active-active multi-region with engineered consistency contracts (S3 strong read-after-write within region, eventual cross-region; Spanner external consistency via TrueTime). This is chapter 6 territory.

A note on multi-cloud: it is usually oversold. Running two cloud providers at equal operational maturity exhausts teams; tooling, IAM, audit, and observability all double. Most providers already deliver 99.99%+; the marginal availability gain from multi-cloud is typically less than the marginal complexity cost. The honest argument for multi-cloud is vendor leverage, not availability.

## Untested redundancy is no redundancy

Redundancy is structural potential. It becomes actual availability only when four pieces are in place:

1. **Detection** — you notice the failure (resilient to gray failure: real-work health checks, p99 outlier detection).
2. **Failover mechanism** — you can actually shift traffic (manual or automatic, in seconds or minutes).
3. **Tested failover** — you exercise the mechanism regularly. A standby that has never been promoted has unknown reliability.
4. **Independence** — replicas fail independently in practice, not just in the diagram.

This is why mature reliability teams run **chaos engineering** — deliberately killing nodes, AZs, even regions in production. Netflix's Chaos Monkey, the Simian Army, AWS's Fault Injection Simulator. The goal is not to claim redundancy works but to **prove it does**, on a schedule.

Operational slogan: *Untested redundancy is no redundancy.* Game days, DR drills, failover rehearsals — all manifestations of the same idea.
