---
slug: causal-consistency
title: "Causal Consistency"
readTime: 20 min
orderIndex: 14
premium: false
---

# Causal Consistency




---

## Key Takeaways

1. **Causal consistency preserves the order of causally related operations** — if A causes B, everyone sees A before B
2. **Concurrent (unrelated) operations may be seen in any order** — this is weaker than linearizability but much cheaper to implement
3. **Vector clocks or Lamport timestamps track causal dependencies** — enabling the system to enforce ordering only where it matters
4. **Causal consistency is a practical middle ground** — stronger than eventual consistency, cheaper than linearizability
