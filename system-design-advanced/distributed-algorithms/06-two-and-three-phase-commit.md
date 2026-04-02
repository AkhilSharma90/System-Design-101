---
slug: two-and-three-phase-commit
title: "Two-Phase and Three-Phase Commit"
readTime: 25 min
orderIndex: 6
premium: false
---

# Two-Phase and Three-Phase Commit




---

## Key Takeaways

1. **2PC coordinates atomic commits across multiple databases** — a coordinator asks all participants to prepare, then commit or abort
2. **2PC is blocking** — if the coordinator fails after prepare, participants hold locks indefinitely until it recovers
3. **3PC adds a pre-commit phase to reduce blocking** — but still can't handle network partitions correctly
4. **Modern systems prefer Sagas over 2PC/3PC** — Sagas trade atomicity for availability, using compensating transactions instead of distributed locks
