---
slug: linearizability-and-sequential-consistency
title: "Linearizability and Sequential Consistency"
readTime: 25 min
orderIndex: 13
premium: false
---

# Linearizability and Sequential Consistency




---

## Key Takeaways

1. **Linearizability means operations appear to take effect at a single instant** — every read returns the most recent write, as if there's one copy of the data
2. **Sequential consistency means all nodes see operations in the same order** — but that order may not match real-time wall-clock ordering
3. **Linearizability is stronger and more expensive than sequential consistency** — it requires coordination across nodes on every operation
4. **Use linearizability for correctness-critical operations** — leader election, lock acquisition, unique constraint checks
5. **Most distributed databases offer configurable consistency levels** — letting you choose the trade-off per operation
