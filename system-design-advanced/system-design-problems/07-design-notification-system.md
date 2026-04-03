---
slug: design-notification-system
title: Design a Notification System
readTime: 30 min
orderIndex: 7
premium: false
---

# Design a Notification System

Design a multi-channel notification system that delivers push notifications, emails, and SMS to users at scale — reliably, with user preference controls, and without duplicates.

---

## Step 1: Requirements

### Functional Requirements
- Support three channels: push (mobile), email, SMS
- Notifications can be triggered by events (payment received, new message) or scheduled (weekly digest)
- User preference management: opt-out per channel, quiet hours, notification frequency limits
- Templates with dynamic personalization
- Delivery tracking: sent, delivered, failed

### Non-Functional Requirements
- Reliability: critical notifications (payment alerts) must be delivered
- Scale: 10 billion notifications per day across all channels
- Latency: high-priority notifications within 5 seconds
- Deduplication: never send the same notification twice
- Rate limiting: no more than N notifications per user per hour

---

## Step 2: Estimation

```
Volume: 10B notifications/day = ~115,000/second
  Push:  7B/day  (70%) = 81,000/sec
  Email: 2B/day  (20%) = 23,000/sec
  SMS:   1B/day  (10%) = 11,500/sec

Storage per notification: ~500 bytes (content + metadata)
Storage: 10B × 500 bytes = 5 TB/day
```

---

## Step 3: High-Level Architecture

```
Event Sources                   Notification System
                                         │
 ┌──────────┐  payment.succeeded ──▶  ┌──▼──────────────────────┐
 │ Payment  │                         │    Notification API      │
 │ Service  │                         │  (receive notification   │
 └──────────┘                         │   requests)             │
 ┌──────────┐  message.received ──▶   └──────────┬──────────────┘
 │  Chat    │                                    │
 │ Service  │                                    ▼
 └──────────┘                         ┌─────────────────────┐
 ┌──────────┐  scheduled trigger ──▶  │  Notification       │
 │ Cron Job │                         │  Processor          │
 └──────────┘                         │                     │
                                      │ - Validate          │
                                      │ - Check preferences │
                                      │ - Deduplicate       │
                                      │ - Rate limit        │
                                      │ - Render template   │
                                      └──────────┬──────────┘
                                                 │
                         ┌───────────────────────┼───────────────┐
                         ▼                       ▼               ▼
                  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
                  │  Push Queue  │   │ Email Queue  │   │  SMS Queue   │
                  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                         │                  │                  │
                         ▼                  ▼                  ▼
                  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
                  │ Push Workers │   │ Email Workers│   │  SMS Workers │
                  │  (APNs/FCM)  │   │  (SendGrid,  │   │  (Twilio,    │
                  │              │   │   Amazon SES)│   │   Vonage)    │
                  └──────────────┘   └──────────────┘   └──────────────┘
```

---

## Step 4: Data Model

```sql
-- User notification preferences
CREATE TABLE user_preferences (
  user_id          UUID NOT NULL,
  channel          ENUM('push', 'email', 'sms') NOT NULL,
  notification_type VARCHAR(100) NOT NULL,  -- e.g., "payment", "marketing", "security"
  enabled          BOOLEAN DEFAULT true,
  quiet_hours_start TIME,                   -- e.g., "22:00"
  quiet_hours_end  TIME,                    -- e.g., "08:00"
  PRIMARY KEY (user_id, channel, notification_type)
);

-- Device tokens for push notifications
CREATE TABLE device_tokens (
  id         UUID PRIMARY KEY,
  user_id    UUID NOT NULL,
  token      TEXT UNIQUE NOT NULL,         -- APNs or FCM device token
  platform   ENUM('ios', 'android') NOT NULL,
  active     BOOLEAN DEFAULT true,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Notification log (for deduplication and delivery tracking)
CREATE TABLE notifications (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  dedup_key        VARCHAR(255) UNIQUE,    -- prevents duplicate sends
  user_id          UUID NOT NULL,
  channel          ENUM('push', 'email', 'sms'),
  template_id      VARCHAR(100),
  template_data    JSONB,
  status           ENUM('pending', 'sent', 'delivered', 'failed'),
  scheduled_at     TIMESTAMP,
  sent_at          TIMESTAMP,
  error_message    TEXT,
  created_at       TIMESTAMP DEFAULT NOW()
);

-- Notification templates
CREATE TABLE templates (
  id          VARCHAR(100) PRIMARY KEY,    -- e.g., "payment_received"
  channel     ENUM('push', 'email', 'sms'),
  subject     TEXT,                        -- for email
  body        TEXT,                        -- Handlebars/Jinja template
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

---

## Step 5: Deep Dives

### Deep Dive 1: Multi-Channel Delivery

**Push Notifications (APNs for iOS, FCM for Android):**

```
User opens app → device registers with APNs/FCM → receives token
App sends token to our server → stored in device_tokens table

Sending:
  POST https://api.push.apple.com/3/device/{device_token}
  Headers: Authorization: Bearer {JWT}
  Body: {
    "aps": {
      "alert": { "title": "Payment received", "body": "You got $50 from Alice" },
      "badge": 1,
      "sound": "default"
    }
  }

APNs/FCM responses:
  200 OK → delivered to device (device may be offline; queued)
  410 Gone → device token invalid (user uninstalled app) → mark token as inactive
  429 → rate limited → back off
```

**Email (SendGrid/Amazon SES):**

```
POST https://api.sendgrid.com/v3/mail/send
{
  "to": [{"email": "user@example.com"}],
  "from": {"email": "hello@algoroq.io"},
  "subject": "Payment received",
  "html": "<p>You received $50 from Alice</p>",
  "custom_args": { "notification_id": "notif_abc" }
}

Track events via webhook:
  SendGrid → POST our_server/webhooks/sendgrid
  Events: delivered, opened, clicked, bounced, unsubscribed
```

**SMS (Twilio):**

```
POST https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages
{
  "To": "+14155552345",
  "From": "+12125551234",
  "Body": "Payment of $50 received from Alice"
}
```

---

> **Pause and think:** If your Kafka consumer processes a "payment succeeded" event twice due to at-least-once delivery, the user gets two push notifications saying "Payment received." How do you prevent this?

### Deep Dive 2: Deduplication

Prevent sending the same notification twice:

```
Scenario: Kafka message processed twice (at-least-once delivery)
  → Notification worker picks up same "payment.succeeded" event twice
  → Without dedup: two "Payment received" push notifications!

Solution: Deduplication key
  dedup_key = MD5(user_id + channel + event_type + event_id)
  = "user_42:push:payment_received:evt_abc"

  Before sending:
    INSERT INTO notifications (dedup_key, ...) ON CONFLICT (dedup_key) DO NOTHING
    → If 0 rows inserted: already processed → skip
    → If 1 row inserted: new notification → proceed to send

  Redis alternative (faster):
    SET notif:dedup:{dedup_key} 1 EX 86400 NX
    → NX: only set if not exists
    → Returns 1 (new) or 0 (duplicate)
    → TTL cleans up after 24 hours
```

---

### Deep Dive 3: Rate Limiting Per User

Don't spam users. Cap notifications per user per channel per time window.

```
Per-user rate limits (Redis sliding window):
  Max 10 push notifications/hour
  Max 5 emails/day
  Max 3 SMS/day

Redis implementation:
  key = "notif_rate:{user_id}:{channel}:{hour_bucket}"
  count = INCR key
  if count == 1: EXPIRE key 3600   (set TTL on first increment)
  if count > limit: DROP notification (or queue for later)

Priority override:
  SECURITY notifications bypass rate limits (2FA codes, fraud alerts)
  MARKETING notifications are most aggressive rate limited
```

---

### Deep Dive 4: Template System with Personalization

```
Template "payment_received" (email):
  Subject: "💸 You received {{amount}} from {{sender_name}}"
  Body:    "Hi {{user_first_name}},
            {{sender_name}} sent you {{formatted_amount}}.
            View your balance at algoroq.io/wallet.
            ..."

Rendering at send time:
  template_data = {
    "user_first_name": "Alice",
    "sender_name": "Bob",
    "amount": "$50.00",
    "formatted_amount": "fifty dollars"
  }
  rendered = render_template(template_id="payment_received", data=template_data)

Template storage: database (versioned, A/B testable)
Rendering: server-side (Jinja2, Handlebars)
```

---

### Deep Dive 5: Priority Queues

Not all notifications are equal:

```
Priority Queues in Kafka (or separate Kafka topics):
  HIGH:    Security alerts, 2FA codes, fraud alerts
           → Delivered within 5 seconds
  MEDIUM:  Payment notifications, direct messages
           → Delivered within 30 seconds
  LOW:     Marketing emails, weekly digests, announcements
           → Delivered within hours (batch processing)

Worker allocation:
  HIGH consumers: 50 workers (always running, auto-scale up to 200)
  MEDIUM consumers: 20 workers (auto-scale 10–100)
  LOW consumers: 5 workers (batch during off-peak hours)
```

---

### Deep Dive 6: Retry and Failure Handling

```
Channel-specific retry strategies:

Push (APNs/FCM):
  Transient failure (network error): retry 3× with 1s, 5s, 30s backoff
  Token expired: mark token invalid, attempt email if available
  Device offline: APNs queues for 30 days; let it handle delivery

Email:
  Soft bounce (mailbox full): retry for 72 hours
  Hard bounce (invalid address): mark email invalid, don't retry
  Spam complaint: unsubscribe user from marketing emails immediately

SMS:
  Network failure: retry 3× with 1min, 5min, 15min backoff
  Invalid number: mark as invalid, don't retry
  Carrier blacklisted: try alternate SMS provider

All failures:
  After max retries: update notification status to "failed"
  Store failure reason for debugging
  Alert on-call if failure rate exceeds 1% for HIGH priority
```

---

## Step 6: Scaling

### Horizontal Scaling by Channel
Each channel's workers scale independently:

```
Black Friday sale email blast:
  Normal: 5 email workers, 1,000 emails/sec
  Black Friday: 500 email workers, 100,000 emails/sec (auto-scaled)
  Return to normal: 5 workers
```

### Partitioning the Notification Queue
```
Kafka topic: "notifications-push"
Partitioned by user_id (or device_id):
  → All notifications for user X go to same partition
  → Guarantees order: user sees notifications in chronological order
  → Prevents parallel delivery causing race conditions
```

---

## Key Takeaways

1. **Multi-channel routing decouples notification logic from delivery** — one API, many channels
2. **Deduplication keys prevent double-sends** — use idempotency at the DB or Redis layer
3. **Priority queues ensure critical notifications aren't delayed by bulk sends** — separate topics/workers by priority
4. **User preferences are checked before every send** — opt-outs must be respected instantly
5. **Templates with personalization** — render at send time with user-specific data
6. **Retry strategies are channel-specific** — push handles offline users differently from email bounces
7. **Expired device tokens must be cleaned up** — invalid tokens waste APNs/FCM quota and add latency

---
