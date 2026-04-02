---
slug: hinted-handoff
title: "Hinted Handoff"
readTime: 15 min
orderIndex: 11
premium: false
---

# Hinted Handoff




---

## Key Takeaways

1. **Hinted handoff temporarily stores writes for unavailable replicas** — a nearby node holds the data and forwards it when the target recovers
2. **Improves write availability during temporary node failures** — writes succeed even if the designated replica is down
3. **Hints are stored with a TTL** — if the target node doesn't recover within the window, the hint is discarded
4. **Hinted handoff is not a replacement for anti-entropy repair** — it handles temporary failures, not permanent data divergence
