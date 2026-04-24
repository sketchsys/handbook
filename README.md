# handbook

Curated, opinionated notes on system design fundamentals.

Each topic answers:

- What is it?
- Why does it exist?
- How does it work?
- Trade-offs (pros, cons, when to use, when not to)
- Real-world examples
- Interview-ready summary

## Start here

- [**Overview & learning path**](./00-overview/README.md) — the 12 topics this handbook covers, with representative tools for each
- [**How to approach a system design question**](./00-overview/how-to-approach-a-design-question.md) — clarification playbook + anti-patterns

## Topics

1. [**Fundamentals**](./01-fundamentals/) — reliability, scalability, maintainability; back-of-envelope estimation; latency numbers
2. **Networking & traffic routing** — DNS, load balancers (L4 vs L7), CDN, API gateway, reverse proxy, TLS
3. **Data** — SQL vs NoSQL, replication, sharding, indexing, transactions
4. **Communication** — REST vs RPC vs GraphQL, message queues, pub/sub, streaming
5. **Caching** — cache patterns, eviction policies, CDNs
6. **Consistency & consensus** — CAP, PACELC, consensus algorithms, distributed transactions
7. **Storage** — object storage, distributed file systems, time-series
8. **Search** — full-text search, indexing strategies
9. **Deployment & scaling** — horizontal vs vertical, stateless services, containers, Kubernetes, multi-region
10. **Reliability patterns** — circuit breakers, retries, timeouts, bulkheads, graceful degradation
11. **Observability** — logs, metrics, tracing
12. **Security** — authentication, authorization, rate limiting, secrets management

## Status

Work in progress. See commit history for what has landed.

## Scope note

This handbook is about *system* design — the architecture of distributed services. Code-level patterns (MVC, hexagonal, Clean Architecture, tactical DDD) are intentionally out of scope. See [Overview](./00-overview/README.md#adjacent-topics-intentionally-out-of-scope) for canonical resources on those.

## License

MIT — see [LICENSE](./LICENSE).
