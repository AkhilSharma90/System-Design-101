---
slug: transactional-outbox-pattern
title: "Transactional Outbox Pattern"
readTime: 20 min
orderIndex: 3
premium: false
---

# Transactional Outbox Pattern




---

## Key Takeaways

1. **The outbox pattern ensures database writes and event publishing happen atomically** — write both to the same database transaction, then publish events asynchronously
2. **Solves the dual-write problem** — without it, a crash between database commit and event publish leaves the system in an inconsistent state
3. **A background process (poller or CDC) reads from the outbox table and publishes events** — Debezium with CDC is the most reliable approach
4. **Events in the outbox must be published in order and at-least-once** — consumers must be idempotent to handle duplicate deliveries
