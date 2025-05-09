https://www.hellointerview.com/learn/system-design/deep-dives/numbers-to-know

# System Design Conversion Cheat Sheet

## Data Units (Storage/Bandwidth)

| Unit | Equivalent in Bytes           | Notes                    |
|------|-------------------------------|--------------------------|
| 1 KB | 1,024 Bytes                   | Use binary (not 1,000)   |
| 1 MB | 1,024 KB = 1,048,576 Bytes    |                          |
| 1 GB | 1,024 MB ≈ 1 Billion Bytes    |                          |
| 1 TB | 1,024 GB                      |                          |
| 1 PB | 1,024 TB                      |                          |

---

## Number Units (Users, Operations, Money)

| Unit       | Value         |
|------------|---------------|
| 1 Thousand | 10³ = 1,000   |
| 1 Million  | 10⁶ = 1,000,000 |
| 1 Billion  | 10⁹ = 1,000,000,000 |
| 1 Trillion | 10¹² = 1,000,000,000,000 |

---

## Quick Mental Conversions

| Concept                        | Approximation / Formula             |
|--------------------------------|-------------------------------------|
| 1 GB                           | ≈ 1 Billion Bytes (10⁹)             |
| 1 TB                           | ≈ 1,000 GB = 8 Trillion bits        |
| 1 GBps                         | ≈ 8 Gbps                            |
| 1 Gbps                         | ≈ 125 MB/s                          |
| Requests per Second (RPS)     | = Requests per Day / 86,400         |
| 1 Million req/day              | ≈ 11.6 RPS                          |
| 100 Million req/day           | ≈ 1,157 RPS                         |

---

## Handy Estimations

- **1 KB ≈ 1,000 Bytes**
- **1 MB ≈ 1 Million Bytes**
- **1 GB ≈ 1 Billion Bytes**
- **Multiply MB/s by 8 to get Mbps**




# System Components: Key Metrics and Scale Triggers

| Component | Key Metrics | Scale Triggers |
|-----------|------------|----------------|
| Caching | - ~1 millisecond latency<br>- 100k+ operations/second<br>- Memory-bound (up to 1TB) | - Hit rate < 80%<br>- Latency > 1ms<br>- Memory usage > 80%<br>- Cache churn/thrashing |
| Databases | - Up to 50k transactions/second<br>- Sub-5ms read latency (cached)<br>- 64 TiB+ storage capacity | - Write throughput > 10k TPS<br>- Read latency > 5ms uncached<br>- Geographic distribution needs |
| App Servers | - 100k+ concurrent connections<br>- 8-64 cores @ 2-4 GHz<br>- 64-512GB RAM standard, up to 2TB | - CPU > 70% utilization<br>- Response latency > SLA<br>- Connections near 100k/instance<br>- Memory > 80% |
| Message Queues | - Up to 1 million msgs/sec per broker<br>- Sub-5ms end-to-end latency<br>- Up to 50TB storage | - Throughput near 800k msgs/sec<br>- Partition count ~200k per cluster<br>- Growing consumer lag |