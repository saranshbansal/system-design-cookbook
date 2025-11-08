# Numbers to Know

You don't need to do back-of-the-envelope calculations at the start of an interview. That's not what interviewers care about. What matters is doing them when you need to make a decision. Should you shard the database? Can a single Redis instance handle the cache load? You can't answer these questions without rough numbers.

The trick is knowing which numbers to use. Modern hardware is way more powerful than most candidates realize. A well-tuned database server handles tens of thousands of queries per second. A single Redis instance handles hundreds of thousands of operations per second. If you're using 2010-era numbers in your head, you'll propose sharding and caching way earlier than you need to.

Start with the latency numbers because they affect almost every design decision. Memory access takes nanoseconds. SSD reads take microseconds. Network calls within a data center take 1-10 milliseconds. Cross-continent calls take tens to hundreds of milliseconds. When you're deciding whether to cache something or whether geographic distribution is worth the complexity, these gaps are what matter.

> Do your capacity calculations in context when you need them. If your interviewer asks "how many servers do we need," that's when you pull out the numbers. Walk through it. "We're expecting 50K requests per second, each server can handle maybe 5K requests, so we need around 10 servers plus some headroom." The interviewer wants to see you think through the math, not recite memorized facts.

Storage capacity matters for sharding decisions. A single Postgres instance handles a few terabytes comfortably. You don't need sharding until you're hitting tens or hundreds of terabytes. If someone proposes sharding at 500GB, they're adding massive complexity for no reason.


| Component      | Key Metrics                                                                                              | Scale Triggers                                                                                                  |
| -------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Caching        | - ~1 millisecond latency <br> - 100k+ operations/second <br> - Memory-bound (up to 1TB)	- Hit rate < 80% | - Latency > 1ms <br> - Memory usage > 80% <br> - Cache churn/thrashing                                          |
| Databases      | - Up to 50k transactions/second <br> - Sub-5ms read latency (cached) <br>- 64 TiB+ storage capacity      | - Write throughput > 10k TPS <br> - Read latency > 5ms uncached <br> - Geographic distribution needs            |
| App Servers    | - 100k+ concurrent connections <br> - 8-64 cores @ 2-4 GHz <br> - 64-512GB RAM standard, up to 2TB       | - CPU > 70% utilization <br> - Response latency > SLA <br> - Connections near 100k/instance <br> - Memory > 80% |
| Message Queues | - Up to 1 million msgs/sec per broker <br> - Sub-5ms end-to-end latency <br> - Up to 50TB storage        | - Throughput near 800k msgs/sec <br> - Partition count ~200k per cluster <br> - Growing consumer lag            |
