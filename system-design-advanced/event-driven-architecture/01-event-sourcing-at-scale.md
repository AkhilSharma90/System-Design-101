---
slug: event-sourcing-at-scale
title: "Event Sourcing at Scale"
readTime: 25 min
orderIndex: 1
premium: false
---

# Event Sourcing at Scale

> Instead of storing current state, store every change that ever happened — then derive any view of the data by replaying the event log.




---

## Key Takeaways

1. **Event sourcing stores every state change as an immutable event** — the current state is derived by replaying the event log
2. **The event log is the source of truth** — not the current state, which is just a projection (a materialized view of the events)
3. **Projections rebuild read models from events** — different views of the same data for different use cases, without affecting the event log
4. **Event sourcing provides a complete audit trail** — every change is recorded with who, what, and when, which is required in financial and compliance systems
5. **Snapshots prevent replay from becoming too slow** — periodically save the current state so replay starts from the snapshot, not from the beginning
