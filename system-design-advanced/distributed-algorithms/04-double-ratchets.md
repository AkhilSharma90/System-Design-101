---
slug: double-ratchets
title: "Double Ratchet Algorithm"
readTime: 20 min
orderIndex: 4
premium: false
---

# Double Ratchet Algorithm

> The algorithm that gives Signal, WhatsApp, and Matrix end-to-end encryption — where compromising one message key reveals nothing about past or future messages.




---

## Key Takeaways

1. **The Double Ratchet algorithm provides forward and backward secrecy for messaging** — compromising one message key doesn't reveal past or future messages
2. **Two ratchets operate together** — a Diffie-Hellman ratchet generates new shared secrets, a symmetric ratchet derives per-message keys
3. **Each message uses a unique encryption key** — even if an attacker captures one key, only that single message is compromised
4. **Signal, WhatsApp, and Matrix all use the Double Ratchet** — it's the standard for end-to-end encrypted messaging
