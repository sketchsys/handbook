# Overview — what this handbook covers

This handbook walks through the foundations of designing distributed systems, with a bias toward what shows up in senior interviews and what matters in real production systems. Each topic is self-contained but they build on each other.

## The core idea

Every design decision is a trade-off. You are rarely picking "the best" tool — you are picking the one whose trade-offs fit your constraints: how much data, how fast, how consistent, how available, how cheap to operate. This handbook is organized around the **questions** you ask, not the tools themselves. Tools are the answers.

## Learning path

| #  | Topic                              | What you'll learn                                                                                                                               | Representative tools                                            |
|----|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| 01 | **Fundamentals**                   | Reliability, scalability, maintainability; back-of-envelope estimation; latency numbers every engineer should know                              | —                                                               |
| 02 | **Networking & traffic routing**   | DNS, load balancers (L4 vs L7), CDN, API gateway, reverse proxy, TLS                                                                            | Nginx, HAProxy, Envoy, Cloudflare, AWS ALB/NLB, Kong            |
| 03 | **Data**                           | SQL vs NoSQL; replication (leader-follower, multi-leader, leaderless); sharding (hash, range, directory); indexes; transactions and isolation   | PostgreSQL, MySQL, MongoDB, Cassandra, DynamoDB, Redis          |
| 04 | **Communication**                  | REST vs RPC vs GraphQL; message queues for async decoupling; pub/sub; streaming                                                                 | gRPC, GraphQL, Kafka, RabbitMQ, NATS, SQS                       |
| 05 | **Caching**                        | Cache-aside, write-through, write-behind; eviction (LRU, LFU, TTL); CDN patterns                                                                | Redis, Memcached, Cloudflare, Fastly, CloudFront                |
| 06 | **Consistency & consensus**        | CAP and PACELC; strong vs eventual consistency; quorum reads/writes; consensus algorithms (Raft, Paxos); distributed transactions (2PC, sagas)  | etcd, ZooKeeper, Consul                                         |
| 07 | **Storage**                        | Object storage, distributed file systems, time-series storage; durability vs availability                                                       | S3, HDFS, GFS, InfluxDB, Prometheus TSDB                        |
| 08 | **Search**                         | Inverted indexes; relevance scoring; when DB full-text search is enough vs when you need a dedicated engine                                     | Elasticsearch, OpenSearch, Meilisearch, Typesense, Postgres FTS |
| 09 | **Deployment & scaling**           | Horizontal vs vertical scaling, stateless services, containers, Kubernetes basics, multi-region deployment                                      | Docker, Kubernetes, Terraform, AWS/GCP/Azure primitives         |
| 10 | **Reliability patterns**           | Circuit breakers, retries with backoff, timeouts, bulkheads, graceful degradation, chaos engineering                                            | Resilience4j, Polly, Istio, Hystrix (legacy)                    |
| 11 | **Observability**                  | The three pillars (metrics, logs, traces); sampling; alerting                                                                                   | Prometheus, Grafana, Loki, Tempo, OpenTelemetry, Jaeger         |
| 12 | **Security**                       | AuthN vs authZ; OAuth2 and OIDC; rate limiting (token bucket, leaky bucket, sliding window); secrets management; common attack surfaces         | Auth0, Keycloak, Vault, JWT                                     |

## Companion docs

- [How to approach a system design question](./how-to-approach-a-design-question.md) — the clarification playbook for interviews and real projects: which questions to ask, in what order, and why. Includes anti-patterns to avoid.

## How to read this handbook

- Each topic is short prose + a trade-off table + an "interview-ready summary" at the bottom.
- No filler. If a concept fits in three sentences, it gets three sentences.
- Every tool mentioned has (or will have) its own card: what it's for, what it's bad at, what you'd use instead.

## Adjacent topics (intentionally out of scope)

This handbook is about *system* design — the architecture of distributed services. It is **not** about code-level architecture within a single service. Patterns like MVC, hexagonal / ports-and-adapters, layered architecture, Clean Architecture, and tactical DDD (aggregates, repositories) belong to a different discipline and have excellent dedicated resources:

- *Clean Architecture* — Robert C. Martin
- *Implementing Domain-Driven Design* — Vaughn Vernon
- *Hexagonal Architecture* — Alistair Cockburn (original paper, 2005)

Where code-level patterns genuinely intersect with distributed systems — **CQRS**, **event sourcing**, **saga** — we cover them in the relevant chapters (Data, Consistency).
