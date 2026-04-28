# Back-of-envelope numbers

A reference card for the numbers that recur in capacity estimation, design discussions, and case-study work. The structure mirrors §5 of the chapter README but goes wider — more systems, more hardware, more scenarios.

This page is a lookup sheet. The reasoning behind these numbers and how to apply them in design lives in [§5 of the chapter README](./README.md#5-back-of-envelope-estimation).

## Latency hierarchy

The Jeff Dean numbers, extended:

| Operation | Time | Notes |
|---|---|---|
| L1 cache reference | 0.5 ns | Per-CPU; effectively free. |
| Branch mispredict | 5 ns | Pipeline flush penalty. |
| L2 cache reference | 7 ns | Larger, still per-CPU. |
| L3 cache reference | 15–25 ns | Shared across cores. |
| Mutex lock/unlock (uncontended) | 25 ns | Lockless atomic 5–10 ns. |
| Mutex lock (contended) | 1–10 µs | OS-mediated wakeup. |
| Function call (typical) | 1–5 ns | Inlining eliminates. |
| Indirect / virtual call | 2–10 ns | Branch predictor helps. |
| **Main memory (DRAM) reference** | **100 ns** | DRAM access. |
| Persistent memory (Optane class) | 200–400 ns | Now discontinued by Intel; rarely encountered. |
| Compress 1 KB (LZ4) | 1–2 µs | Fastest mainstream codec. |
| Compress 1 KB (Snappy) | 2–3 µs | Comparable to LZ4. |
| Compress 1 KB (zstd, level 1) | 3–10 µs | Better ratio, slower. |
| Compress 1 KB (gzip, default) | 20–50 µs | Tuned for ratio, not speed. |
| Send 1 KB over 1 Gbps network | 10 µs | Wire time only; software stack adds. |
| Send 1 KB over 10 Gbps network | 1 µs | Wire-only. |
| TCP handshake (same-DC) | 0.5–1 ms | Three-way; mostly RTT. |
| TLS 1.2 handshake (same-DC) | 1–4 ms | Two RTT plus crypto. |
| TLS 1.3 handshake (same-DC) | 0.5–2 ms | One RTT. |
| **Read 4 KB random from SATA SSD** | **150 µs** | NAND access; varies by SSD class. |
| Read 4 KB random from NVMe SSD | 20–50 µs | Modern NVMe much faster than SATA. |
| Read 1 MB sequentially from RAM | 250 µs | DDR4. |
| **Round-trip within same datacenter** | **500 µs** | TCP, typical. |
| Read 1 MB sequentially from SATA SSD | 1 ms | — |
| Read 1 MB sequentially from NVMe SSD | 0.15–0.3 ms | PCIe 4.0 / 5.0. |
| **Disk seek (HDD, 7,200 rpm)** | **10 ms** | Average. |
| Read 1 MB sequentially from HDD | 30 ms | ~30 MB/s sustained. |
| **Round-trip same continent (US east ↔ west)** | **70–80 ms** | ~5,000 km. |
| **Round-trip cross-continent (US ↔ EU)** | **150 ms** | ~8,000 km. |
| Round-trip cross-Pacific (US ↔ Asia) | 200–250 ms | ~12,000 km plus router overhead. |
| 4G mobile RTT (median) | 50–100 ms | Variable; tail much worse. |
| 5G mobile RTT (median) | 10–30 ms | Closer to fixed broadband. |
| Satellite (geostationary) RTT | 600–800 ms | ~36,000 km × 4 hops. |
| Satellite (LEO, Starlink) RTT | 30–50 ms | Low orbit; competitive with fiber. |

## Throughput — software systems

Per-node ceilings on commodity hardware, for a representative workload. Real numbers vary with version, configuration, schema, and access pattern; these are decision-making approximations.

| System | Throughput | Notes |
|---|---|---|
| Single HTTP server (sync, JSON API) | 10k QPS | Modest payload. |
| Single HTTP server (async I/O) | 50–100k QPS | Node.js, Go, async Python. |
| HTTPS termination (TLS 1.3) | 5–20k QPS | TLS handshake CPU-bound. |
| Single nginx — static file | 50k+ QPS | sendfile + epoll. |
| Single nginx — reverse proxy | 30k+ QPS | Adds upstream RTT. |
| Single Postgres — indexed point read | 10–50k QPS | Working set in memory. |
| Single Postgres — write | 5–10k QPS | WAL fsync; group commit raises. |
| Single MySQL — indexed point read | 10–50k QPS | Similar to Postgres. |
| Single MySQL (InnoDB) — write | 5–20k QPS | Same WAL pattern. |
| Single Redis | 100k+ QPS | Single-thread; pipelining → 1M+. |
| Single Memcached | 200k+ QPS | Multi-thread. |
| Single Cassandra — write | 10–20k/s | LSM tree. |
| Single Cassandra — read | 5–10k/s | Bloom filter + SSTable. |
| Single MongoDB — read | 10–30k QPS | Indexed. |
| Single MongoDB — write | 5–15k QPS | WiredTiger. |
| Single Kafka broker — produce | 100 MB/s sustained | Sequential disk. |
| Single Kafka broker — consume | 200+ MB/s | Page cache helps. |
| Single Pulsar broker | 100 MB/s | Comparable to Kafka. |
| Single Elasticsearch — index | 5k doc/s | CPU-bound. |
| Single Elasticsearch — query | 100–1,000 QPS | Highly query-dependent. |
| Single ClickHouse — analytical query | varies | Column-store; scan-heavy. |
| Single Spanner / CockroachDB node | ~5k write/s | Geo-replicated; consensus overhead. |
| Single S3 prefix | ~3.5k PUT/s, 5.5k GET/s | Per partition; hot keys throttle. |
| Load balancer (HAProxy, L7) | 50k+ QPS | Limited by connection count. |
| Load balancer (HAProxy, L4) | 500k+ QPS | Less work. |
| Load balancer (Envoy, L7) | 30–80k QPS | Filter chain affects. |

## Throughput — hardware

| Component | Throughput | Notes |
|---|---|---|
| SATA SSD — sequential read/write | ~500 MB/s | SATA III bus limit. |
| SATA SSD — random 4 KB read | 50–80k IOPS | NAND access. |
| NVMe SSD (PCIe 4.0) — sequential read | 5–7 GB/s | — |
| NVMe SSD (PCIe 5.0) — sequential read | 10–14 GB/s | Latest gen. |
| NVMe SSD — random 4 KB read | 100k–1M IOPS | High-end NVMe. |
| HDD (7,200 rpm) — sequential | ~150 MB/s | — |
| HDD — random 4 KB | ~100 IOPS | Bounded by seek. |
| 1 Gbps NIC | ~125 MB/s | Practical. |
| 10 Gbps NIC | ~1.25 GB/s | DC standard. |
| 25 Gbps NIC | ~3 GB/s | Newer. |
| 40 Gbps NIC | ~5 GB/s | Spine / aggregation. |
| 100 Gbps NIC | ~12 GB/s | Hot path / storage. |
| DRAM bandwidth (DDR4) | 20–25 GB/s | Per channel. |
| DRAM bandwidth (DDR5) | 40–50 GB/s | Per channel. |
| L3 cache bandwidth | 50–100 GB/s | Per CPU. |
| PCIe 4.0 x16 | 32 GB/s | GPU / NVMe slot. |
| PCIe 5.0 x16 | 64 GB/s | — |
| NVLink (NVIDIA) | 600 GB/s | GPU-to-GPU. |

## Object size reference

| Object | Typical size | Notes |
|---|---|---|
| UUID (binary) | 16 B | — |
| UUID (string, hex with dashes) | 36 B | — |
| ULID | 26 B (string) | Sortable. |
| IPv4 (binary) | 4 B | — |
| IPv6 (binary) | 16 B | — |
| Unix timestamp (seconds) | 4 B | 32-bit. |
| Unix timestamp (nanoseconds) | 8 B | 64-bit. |
| Single character (UTF-8, ASCII) | 1 B | — |
| Single character (UTF-8, BMP) | 2–3 B | Most non-Latin scripts. |
| Tweet (text only) | ~200 B | 280 chars + envelope. |
| Tweet (full record with metadata) | 1–3 KB | User, timestamps, entities. |
| Typical JSON API record | 1–5 KB | User, post, order. |
| Single structured log line (JSON) | 200 B – 2 KB | Trace IDs, fields. |
| Single binary log line (Protobuf) | 100 B – 1 KB | Smaller than JSON. |
| HTTP request (no body, with cookies) | 0.5–2 KB | — |
| HTTP response (small JSON) | 0.5–5 KB | — |
| HTTP/2 frame | 9 B header + payload | — |
| Thumbnail image (JPEG, 200×200) | 10–30 KB | — |
| Web image (JPEG, 1080p) | 200–500 KB | — |
| Smartphone photo (JPEG, 12 MP) | 2–4 MB | — |
| Smartphone photo (RAW) | 20–40 MB | — |
| 1 minute audio (MP3, 192 kbps) | ~1.5 MB | — |
| 1 minute audio (Opus, 64 kbps) | ~500 KB | — |
| 1 minute video (H.264, 1080p) | ~50 MB | Bitrate ~6 Mbps. |
| 1 minute video (H.265, 1080p) | ~30 MB | Better codec. |
| 1 minute video (H.265, 4K) | ~250 MB | Bitrate ~30 Mbps. |
| Email (text only) | 5–20 KB | With headers. |
| Email (typical, with HTML) | 50–500 KB | Embedded images via reference. |
| PDF (text-only document) | 100–500 KB | — |
| PDF (image-heavy) | 2–20 MB | — |
| Postgres row overhead | ~24 B | Header + null bitmap. |
| Postgres BTree index entry | ~40 B | Key + pointer. |
| Cassandra SSTable per-row overhead | ~20–40 B | Tombstones add. |
| Parquet per-column metadata | depends on column | Encoded; usually small. |

## Powers-of-two and time conversions

| Power | Decimal | Name |
|---|---|---|
| 2¹⁰ | ≈ 10³ | KB |
| 2²⁰ | ≈ 10⁶ | MB |
| 2³⁰ | ≈ 10⁹ | GB |
| 2⁴⁰ | ≈ 10¹² | TB |
| 2⁵⁰ | ≈ 10¹⁵ | PB |
| 2⁶⁰ | ≈ 10¹⁸ | EB |

Each 10 bits ≈ 3 decimal digits. Real ratio 1024 / 1000 = 1.024 — a 2.4% gap, ignored in back-of-envelope work.

| Window | Seconds | Mnemonic |
|---|---|---|
| 1 minute | 60 | — |
| 1 hour | 3,600 | ~3.6K |
| 1 day | 86,400 | ~10⁵ |
| 1 week | 604,800 | ~6 × 10⁵ |
| 1 month (30 days) | 2,592,000 | ~2.6M |
| 1 year | 31,536,000 | π × 10⁷ |

## The Fermi discipline — three rules

1. **Round, then multiply.** 7,234 × 412 → 7K × 0.4K = 2.8M. Error 5–10%, well inside decision tolerance.
2. **Be precise enough to keep the order.** 100K vs 1M moves architecture; 100K vs 50K moves only node count.
3. **Sanity-check.** Cross every result against a real-world bound — NIC throughput, DRAM size, light speed, single-node ceiling. Half of estimation is asking *"is this even plausible?"*

## How to read these tables

These numbers are **rough**. They depend on hardware generation, workload pattern, payload size, schema design, and a hundred other variables. They are useful at order-of-magnitude precision — for binary decisions like *"can a single node carry this?"*, *"which storage tier is feasible?"*, *"is this latency budget plausible?"*

For a benchmark you can cite as evidence, run the actual measurement on the actual workload. For a back-of-envelope estimate that decides which architecture to even prototype, these numbers are the starting point.

For canonical references and updates:

- Colin Scott, *Latency Numbers Every Programmer Should Know* — interactive page that updates Jeff Dean's table by year.
- Brendan Gregg, *Systems Performance* (2020, 2nd ed) — hardware and OS performance.
- Vendor benchmarks (Postgres, Redis, Kafka, Cassandra) — per-system numbers vary across versions and configurations.
