https://www.hellointerview.com/learn/system-design/deep-dives/numbers-to-know


# System Components: Key Metrics and Scale Triggers

| Component | Key Metrics | Scale Triggers |
|-----------|------------|----------------|
| Caching | - ~1 millisecond latency<br>- 100k+ operations/second<br>- Memory-bound (up to 1TB) | - Hit rate < 80%<br>- Latency > 1ms<br>- Memory usage > 80%<br>- Cache churn/thrashing |
| Databases | - Up to 50k transactions/second<br>- Sub-5ms read latency (cached)<br>- 64 TiB+ storage capacity | - Write throughput > 10k TPS<br>- Read latency > 5ms uncached<br>- Geographic distribution needs |
| App Servers | - 100k+ concurrent connections<br>- 8-64 cores @ 2-4 GHz<br>- 64-512GB RAM standard, up to 2TB | - CPU > 70% utilization<br>- Response latency > SLA<br>- Connections near 100k/instance<br>- Memory > 80% |
| Message Queues | - Up to 1 million msgs/sec per broker<br>- Sub-5ms end-to-end latency<br>- Up to 50TB storage | - Throughput near 800k msgs/sec<br>- Partition count ~200k per cluster<br>- Growing consumer lag |