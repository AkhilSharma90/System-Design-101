---
slug: game-days-and-dirt-testing
title: Game Days and Dirt Testing
readTime: 20 min
orderIndex: 2
premium: false
---




# Game Days and DiRT Testing (Chaos Engineering in Production)

> Audience: site reliability engineers and platform teams running critical production systems.

This article assumes:
- Your system will fail in production - it's a question of when, not if.
- Incidents happen at the worst possible times: holidays, weekends, during launches.
- Your runbooks are outdated: they describe the system from 6 months ago.
- Team knowledge is unevenly distributed: only 2 people know how the payment flow really works.

---

##  Challenge: Your perfectly designed system fails anyway

### Scenario
It's 3 AM. Your payment processor is down. The on-call engineer:

- Has never seen this failure mode before
- Can't find the runbook
- Doesn't know who owns the payment service
- Accidentally makes it worse by restarting the wrong components

Your monitoring says everything is green. Customers say they can't pay.

### Interactive question (pause and think)
What failed here?

1. The monitoring system
2. The on-call engineer's skills
3. The organization's preparation for failure
4. All of the above

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (3), which contributes to (4).

The engineer is competent. The monitoring works (for known failure modes). But the organization never practiced this scenario.

### Real-world analogy (fire drills)
Buildings don't just have fire extinguishers and exit signs. They run fire drills. Why? Because panic creates tunnel vision, people forget procedures, and muscle memory matters during chaos.

Your production system is no different.

### Key insight box
> Game Days and DiRT (Disaster Recovery Testing) are controlled chaos experiments that expose gaps before real incidents do.

### Challenge question
If you could only practice one failure scenario per quarter, which category would reveal the most organizational gaps: network failures, data corruption, or complete region outages?

---

##  Mental model - Failure is a learning opportunity, not a risk

### Scenario
Your VP asks: "Why would we intentionally break production?"

Two philosophies collide:

- Traditional: minimize all risk, avoid disruption at all costs
- Resilience engineering: controlled small failures prevent catastrophic large failures

### Interactive question (pause and think)
Which statement reflects reality?

A. "Our system is too critical to intentionally break."
B. "If we're afraid to test failures, we're not ready for real failures."
C. "Chaos engineering is just for Netflix and Amazon."

### Progressive reveal
Answer: B.

- A reflects risk aversion that leads to fragility.
- C is a dangerous myth - chaos engineering scales to any system that must be reliable.

### Mental model
Think of Game Days as:

- **Rehearsals** before the performance
- **Flight simulators** for your engineering team
- **Stress tests** for both systems and people

The goal isn't to break things - it's to learn what breaks, how teams respond, and where documentation/automation gaps exist.

### Real-world parallel (hospital emergency drills)
Hospitals run mass casualty drills. They don't wait for a real disaster to find out that:
- The backup generator is in the wrong location
- Nurses don't know the evacuation protocol
- The emergency contact list is outdated

Your production system deserves the same rigor.

### Key insight box
> Game Days reveal unknown unknowns. Runbooks describe known knowns. The gap between them is where incidents live.

### Challenge question
What's more dangerous: a failure you've practiced recovering from, or a failure you've never seen but your monitoring claims to detect?

---

##  What "Game Days" vs "DiRT" vs "Chaos Engineering" actually mean

### Scenario
Your team debates what to call these exercises. The terms are often confused.

### Definitions

**Game Days:**
- Broader organizational exercise
- Includes incident response, communication, decision-making
- Often involves multiple teams
- Simulates complete failure scenarios (e.g., "AWS us-east-1 is down")

**DiRT (Disaster Recovery Testing):**
- Focuses on disaster recovery capabilities
- Tests: backups, failover, data restoration, region evacuation
- Validates recovery time objectives (RTO) and recovery point objectives (RPO)
- Often scheduled maintenance windows

**Chaos Engineering:**
- Continuous, automated failure injection
- Hypothesis-driven experiments
- Production environment (often)
- Smaller blast radius, higher frequency

### Visual: scope and frequency

```text
                         │  Scheduled   │   Continuous
─────────────────────────┼──────────────┼────────────────
Full org simulation      │  Game Days   │      (rare)
(multi-team)             │              │
                         │              │
─────────────────────────┼──────────────┼────────────────
Infrastructure failover  │  DiRT        │   Chaos Eng
(technical focus)        │              │   (automated)
                         │              │
─────────────────────────┼──────────────┼────────────────

Blast Radius:  LARGE ←──────────────────→ SMALL
Frequency:     RARE  ←──────────────────→ CONTINUOUS
```

### Interactive question
Your payment service handles $10M/hour. Which approach do you start with?

1. Monthly Game Days with full region failover
2. DiRT testing quarterly during maintenance windows
3. Automated chaos experiments on 1% traffic daily

### Progressive reveal
Answer: Start with (2), graduate to (3), use (1) for organizational readiness.

Starting with Game Days (full region failover) on a critical service without DiRT foundation is reckless.

### Key insight box
> Start small, automate, then scale. Chaos maturity is a journey: manual → scheduled → continuous.

### Challenge question
What happens if you run chaos experiments but never practice the human incident response process?

---

##  Core components of a Game Day

### Scenario
You're planning your first Game Day. What do you actually need?

Typical components:

1. **Scenario design** (what will we break?)
2. **Blast radius definition** (how much can we afford to impact?)
3. **Success criteria** (what does "handled well" look like?)
4. **Observers and facilitators** (who's running the exercise?)
5. **Communication plan** (how do we coordinate?)
6. **Rollback plan** (when do we stop?)
7. **Post-mortem and remediation tracking**

### Visual: Game Day phases

```text
┌────────────────────────────────────────────────────────────┐
│                    GAME DAY LIFECYCLE                      │
└────────────────────────────────────────────────────────────┘

  BEFORE                 DURING                    AFTER
    │                      │                         │
    ├─ Define scenario    ├─ Inject failure         ├─ Debrief
    │                      │                         │
    ├─ Set success        ├─ Observe response       ├─ Document gaps
    │  criteria            │   - Technical           │
    │                      │   - Organizational      │
    ├─ Brief team          │                         ├─ Create tickets
    │  (optional)          ├─ Monitor blast radius   │
    │                      │                         │
    ├─ Prepare rollback    ├─ Communicate status    ├─ Update runbooks
    │                      │                         │
    └─ Notify stakeholders └─ Rollback (if needed)  └─ Schedule retest

    Failure modes to watch:
      - Cascading failures (blast radius exceeds plan)
      - Silent degradation (no alerts fire)
      - Runbook doesn't match reality
      - Team paralysis / indecision
      - Communication breakdown
```

### Interactive question
You're designing a Game Day: "Primary database fails over to replica."

Which element is most critical to define first?

A. The exact failure mechanism (network partition vs process crash)
B. Rollback criteria (when to abort)
C. Success metrics (what good looks like)

### Progressive reveal
Answer: B, then C, then A.

Without rollback criteria, you risk turning a Game Day into a real incident. Without success metrics, you can't learn. The failure mechanism matters less than knowing when to stop.

### Failure catalog (organizational reality)
At scale, Game Days expose:

- **Runbook rot**: documentation is 3 versions behind
- **Tribal knowledge**: only senior engineers know critical workflows
- **Coordination gaps**: teams don't know who to call
- **Alert fatigue**: important alerts buried in noise
- **Hidden dependencies**: service A fails, but service C dies (undocumented coupling)
- **Tooling friction**: dashboards don't show the right data
- **Authority confusion**: who has permission to make rollback decisions?

### Key insight box
> Game Days fail fast in controlled environments so your real incidents don't fail slow in production.

### Challenge question
If your Game Day runs perfectly and nothing breaks, what went wrong with your scenario design?

---

##  Designing safe chaos experiments - the blast radius game

### Scenario
You want to test "what if Kafka loses a partition."

But your Kafka cluster handles:

- User authentication events (critical)
- Analytics events (non-critical)
- Payment authorizations (ultra-critical)
- Recommendation updates (non-critical)

### Think about it
How do you test Kafka failures without taking down payments?

### Interactive question (pause and think)
What's the right blast radius strategy?

1. Test in production, but only on 0.1% of traffic
2. Test in staging with full traffic
3. Test in production, but only on non-critical topics
4. Shadow test: duplicate traffic to an isolated Kafka cluster

Pause.

### Progressive reveal
Answer depends on maturity, but typically: start with (2), then (4), then (3), finally (1).

### Blast radius control techniques

**Traffic-based:**
- Canary: 1% → 5% → 25% → 100%
- Geography: test in low-traffic regions first
- User segment: employees, beta users, then general population

**Infrastructure-based:**
- Shadow environments (duplicate traffic, isolated infrastructure)
- Feature flags (degrade gracefully without full failure)
- Read replicas (test non-critical reads, not writes)

**Time-based:**
- Business hours vs off-hours
- Low-traffic days (Sunday 2 AM)
- Scheduled maintenance windows

### Visual: blast radius expansion

```text
Chaos Maturity Progression:

Level 1: Staging environment, full blast
         ├─ Learn tooling and basic mechanics
         └─ Zero production risk, but low realism

Level 2: Production, non-critical paths only
         ├─ Test analytics, caching, recommendations
         └─ Real traffic patterns, limited user impact

Level 3: Production, critical paths, small %
         ├─ Canary regions or user segments
         └─ Real risk, measurable, contained

Level 4: Production, critical paths, automated
         ├─ Continuous chaos with auto-rollback
         └─ Full resilience validation

Anti-pattern: Skip levels 1-3 and YOLO level 4
```

### Production insight
Netflix's Chaos Monkey started simple: randomly kill instances, nothing more sophisticated. They didn't start with region-level failures.

### Key insight box
> Blast radius discipline is not cowardice - it's engineering rigor. Start small, measure, expand.

### Challenge question
How would you design a chaos experiment for a system where "safe" and "critical" paths share the same database?

---

##  DiRT testing - validating disaster recovery for real

### Scenario
Your RTO (Recovery Time Objective) SLA says "4 hours to restore service."

But when was the last time you actually tested a full restore from backup?

### Interactive question (pause and think)
Your database backup runs nightly. Which statement is true?

A. "Backups that run successfully are restorable."
B. "Backups must be tested separately from creation."
C. "Backup validation is too risky to do often."

### Progressive reveal
Answer: B.

Countless companies have learned the hard way: successful backups ≠ restorable backups.

### DiRT testing dimensions

**Data recovery:**
- Full restore from backup
- Point-in-time recovery (PITR)
- Corrupted data rollback
- Incremental vs full backup validation

**Infrastructure recovery:**
- Region failover (active-passive)
- Zone failover within region
- Bare-metal restore (for on-prem)
- Kubernetes cluster rebuild

**Application recovery:**
- Stateless service redeploy
- Stateful service migration
- Database schema migration after recovery
- Configuration restore (IaC validation)

**Organizational recovery:**
- Runbook walkthrough
- Incident command structure
- Communication tree (who calls whom?)
- Third-party vendor coordination

### Visual: DiRT test matrix

```text
                 │ Quarterly │ Monthly │ Weekly │ Automated
─────────────────┼───────────┼─────────┼────────┼──────────
Full region      │     ✓     │         │        │
failover         │           │         │        │
─────────────────┼───────────┼─────────┼────────┼──────────
Database         │           │    ✓    │        │
restore          │           │         │        │
─────────────────┼───────────┼─────────┼────────┼──────────
Service          │           │         │   ✓    │
redeployment     │           │         │        │
─────────────────┼───────────┼─────────┼────────┼──────────
Backup           │           │         │        │    ✓
validation       │           │         │        │
─────────────────┼───────────┼─────────┼────────┼──────────

Key metrics per test:
  - Actual recovery time (vs RTO target)
  - Data loss amount (vs RPO target)
  - Manual steps count (automation gap)
  - Undocumented steps discovered
  - Cross-team coordination breakdowns
```

### Real-world DiRT failure stories

**The backup that wasn't:**
Company had 7 years of database backups. During a ransomware attack, they discovered:
- Backup files were corrupted (silent failures for 2 years)
- Restore scripts referenced decommissioned servers
- No one had actually restored in 4 years

**The runbook fiction:**
"Step 3: SSH to the failover database."
Problem: failover database IP changed 6 months ago, runbook never updated.

**The permission trap:**
During region failover drill, discovered: junior on-call engineers don't have AWS IAM permissions to create Route53 records.

### Key insight box
> DiRT testing answers: "Can we actually do what we claim we can do?" Most organizations are surprised by the answer.

### Testing strategy (pragmatic)

```go
// Pseudocode for DiRT test automation
type DiRTTest struct {
    Name           string
    Frequency      Duration
    MaxDowntime    Duration
    SuccessCriteria []Criterion
}

func (d *DiRTTest) Execute() TestResult {
    // 1. Snapshot current state
    preState := CaptureSystemState()

    // 2. Inject disaster
    disaster := d.InjectFailure()

    // 3. Start recovery timer
    recoveryStart := time.Now()

    // 4. Execute recovery procedure
    recovered := d.AttemptRecovery()

    recoveryDuration := time.Since(recoveryStart)

    // 5. Validate correctness
    postState := CaptureSystemState()
    dataLoss := CalculateDataLoss(preState, postState)

    // 6. Compare against SLOs
    return TestResult{
        RTOmet:      recoveryDuration < d.MaxDowntime,
        RPOmet:      dataLoss < d.MaxDataLoss,
        Correctness: ValidateDataIntegrity(postState),
        GapsFound:   d.DocumentGaps(),
    }
}

// Example DiRT suite
var DiRTSuite = []DiRTTest{
    {
        Name:           "Database PITR",
        Frequency:      Monthly,
        MaxDowntime:    30 * time.Minute,
        SuccessCriteria: []Criterion{
            {Type: DataLoss, Threshold: "< 5 minutes"},
            {Type: Downtime, Threshold: "< 30 minutes"},
        },
    },
    {
        Name:           "Full region failover",
        Frequency:      Quarterly,
        MaxDowntime:    4 * time.Hour,
        SuccessCriteria: []Criterion{
            {Type: DataLoss, Threshold: "zero"},
            {Type: CustomerImpact, Threshold: "< 0.1%"},
        },
    },
}
```

### Challenge question
Your DiRT test succeeds: you restored from backup in 30 minutes. But your RTO is 4 hours. Should you update your SLA to 30 minutes, or keep the 4-hour buffer? Why?

---

##  The "we'll just manually fix it" fallacy

### Scenario
During a Game Day, the team manually recovers in 20 minutes.

Engineers conclude: "We're good. Our runbook works."

### Interactive question (pause and think)
What's wrong with this conclusion?

1. Nothing - manual recovery is fine if it's fast
2. Manual steps don't scale to 3 AM on a holiday
3. Different on-call engineer might not know the trick
4. Both 2 and 3

### Progressive reveal
Answer: 4.

### The automation maturity ladder

**Level 0: No runbook**
- Recovery depends on who's on call
- Institutional knowledge in people's heads
- Each incident is novel

**Level 1: Documentation exists**
- Runbook with manual steps
- Better than nothing, but:
  - Runbooks go stale
  - Steps are error-prone under pressure
  - Requires specific knowledge/access

**Level 2: Semi-automated scripts**
- Some steps scripted
- Engineer still makes decisions
- Faster, less error-prone

**Level 3: Push-button recovery**
- Single command triggers full recovery
- Engineer validates but doesn't execute
- Consistent outcomes

**Level 4: Automated detection + recovery**
- System self-heals
- Human notified but not required
- SRE dream state

### Real-world failure: The 3 AM scenario

```text
Manual runbook step: "SSH to db-primary-03 and run failover script"

3 AM problems:
  ├─ db-primary-03 was decomissioned last month
  ├─ Failover script moved to /opt/scripts/v2/
  ├─ New script requires 2FA auth (on-call doesn't have laptop)
  ├─ After failed attempts, on-call escalates
  ├─ Senior engineer wakes up, tries to help over phone
  ├─ Turns out the "new" script has a bug
  └─ 90 minutes elapsed before workaround found

Automated equivalent: 3 minutes
```

### Key insight box
> If your recovery depends on heroics, it's not recovery - it's luck.

### Challenge question
You've automated 90% of your incident response. The remaining 10% requires human judgment calls. How do you practice that 10%?

---

##  Game Day scenario library - what to practice

### Scenario
You have limited time. Which failures matter most?

### Core scenarios (start here)

**Infrastructure failures:**
1. Primary database crash
2. Entire availability zone down
3. Full region outage
4. DNS provider failure
5. CDN/load balancer failure

**Application failures:**
6. Memory leak OOM kills
7. Dependency timeout (third-party API down)
8. Configuration pushed breaks app
9. Deployment rollout stuck halfway
10. Infinite retry loop (thundering herd)

**Data failures:**
11. Corrupted data written to database
12. Accidental table drop
13. Replication lag spike
14. Disk full (logs or data)
15. Backup restoration needed

**Organizational failures:**
16. On-call engineer unavailable
17. Subject matter expert on vacation
18. Escalation tree outdated
19. Cross-team dependency broken
20. Third-party vendor unresponsive

### Interactive question
You can only practice 3 scenarios this quarter. Your system is an e-commerce checkout flow. Which 3?

A. DNS failure, database crash, CDN down
B. Region outage, payment provider timeout, on-call unavailable
C. Memory leak, configuration bug, disk full

Pause and think about your actual blast radius.

### Progressive reveal
Answer: B reveals more organizational gaps.

- A tests infrastructure (important but often well-automated)
- B tests both technical and human response
- C tests application issues (valuable but narrower scope)

### Scenario design template

```yaml
game_day_scenario:
  name: "Payment provider timeout cascade"

  description: |
    Primary payment provider (Stripe) experiences degraded
    performance. API calls timeout after 30s instead of 200ms.

  failure_injection:
    - Add network delay: 30s to api.stripe.com
    - Scope: 100% of traffic
    - Duration: 45 minutes

  expected_system_behavior:
    - Circuit breaker opens after 3 failures
    - Requests queue (up to 1000)
    - Alert fires: "Payment provider degraded"
    - Automatic failover to secondary provider (PayPal)

  expected_team_behavior:
    - On-call acknowledges alert within 5 min
    - Incident channel created
    - Payment team paged
    - Status page updated
    - Fallback provider confirmed healthy

  success_criteria:
    - Checkout success rate > 95%
    - Customer-visible errors < 5%
    - Recovery time < 10 minutes
    - No manual intervention required

  failure_modes_to_observe:
    - Circuit breaker doesn't open (infinite retries)
    - Queue overflows (OOM)
    - Secondary provider also fails (untested)
    - Status page not updated (customer support overwhelmed)

  remediation_tracking:
    - Document gaps found
    - Create tickets for each gap
    - Retest in 30 days after fixes
```

### Production insight
Companies often test "big obvious" failures (region down) but miss "slow degradation" failures (API timeouts, connection pool exhaustion, gradual memory leaks).

Slow failures are harder to detect and often cause worse cascading effects.

### Key insight box
> The best Game Day scenario is the one you're most afraid to run. That's where your knowledge gaps hide.

### Challenge question
Design a Game Day that tests your team's response to a security incident (compromised credentials) rather than an infrastructure failure. What changes?

---

##  Measuring chaos engineering success

### Scenario
You've run 10 Game Days. How do you know if you're getting more resilient?

### Interactive question (pause and think)
Which metric best indicates chaos engineering maturity?

A. Number of Game Days run
B. Percentage of scenarios that "pass"
C. Time to recovery improvement over time
D. Number of issues found and fixed

### Progressive reveal
Answer: C and D together.

- A is vanity (quantity ≠ quality)
- B is misleading (passing means you're not testing hard enough)
- C shows improving resilience
- D shows learning velocity

### Metrics to track

**System resilience metrics:**
- Mean Time To Detect (MTTD)
- Mean Time To Recover (MTTR)
- Blast radius size (customers impacted)
- Automated vs manual recovery ratio

**Organizational metrics:**
- Runbook accuracy rate
- Cross-team coordination time
- Escalation path effectiveness
- Post-incident action item completion rate

**Learning metrics:**
- Unknown-unknowns discovered per Game Day
- Automation coverage increase
- Documentation freshness score
- Team confidence survey results

### Visual: chaos maturity scorecard

```text
┌─────────────────────────────────────────────────────────┐
│              CHAOS ENGINEERING SCORECARD                │
└─────────────────────────────────────────────────────────┘

Dimension               Before    After 6mo   Target
──────────────────────────────────────────────────────
MTTD (detection)        45 min    12 min      < 5 min
MTTR (recovery)         4 hours   30 min      < 15 min
Automated recovery      20%       65%         > 80%
Runbook accuracy        40%       85%         > 95%
Confidence (1-10)       4         7           8+

Scenarios tested        │█████░░░░░░░░░░│  5/15 (33%)
Issues found            │██████████████░│  28 total
Issues remediated       │█████████░░░░░░│  18/28 (64%)
Automation added        │███████░░░░░░░░│  7 workflows

Trend: ↗ Improving (MTTR reduced 87% in 6 months)
```

### Continuous improvement loop

```go
type ChaosProgram struct {
    Scenarios []Scenario
    Results   []GameDayResult
}

func (c *ChaosProgram) ContinuousImprovement() {
    for {
        // 1. Run scenario
        result := c.RunGameDay(c.NextScenario())

        // 2. Analyze gaps
        gaps := result.IdentifyGaps()

        // 3. Prioritize remediation
        tickets := c.CreateRemediationTickets(gaps)

        // 4. Track to completion
        c.TrackToCompletion(tickets)

        // 5. Retest after fixes
        if tickets.AllClosed() {
            c.ScheduleRetest(result.Scenario, in: 30*Day)
        }

        // 6. Expand scope
        if result.PassRate > 0.90 {
            c.AddHarderScenario()
        }

        time.Sleep(c.CadenceInterval)
    }
}
```

### Key insight box
> Chaos engineering success is not "zero failures" - it's "faster learning and recovery from inevitable failures."

### Challenge question
Your Game Day found 15 issues. You have budget to fix 5 this quarter. How do you prioritize?

---

##  Final synthesis - Build your own chaos program

### Synthesis challenge
You're the SRE lead for a fintech company's payment processing platform.

Requirements:

- Handle $500M transactions/day
- 99.99% uptime SLA (52 minutes downtime/year)
- Multi-region active-active
- PCI compliance requires audit trail of all recovery tests
- Team: 10 SREs, 40 developers across 6 teams
- Current state: no Game Days ever run, some monitoring, manual runbooks

### Your tasks (pause and think)
1. Design your chaos maturity roadmap (6-month plan)
2. Choose first 3 scenarios to test (and justify)
3. Define success metrics and tracking
4. Build org buy-in plan (how to convince leadership?)
5. Define blast radius controls for production chaos
6. Create remediation process for gaps found

Write down your answers.

### Progressive reveal (one possible solution)

**1. Chaos maturity roadmap:**
- Month 1-2: DiRT testing in staging (database restore, failover)
- Month 3-4: First production Game Day (read replica failure, off-hours)
- Month 5-6: Continuous chaos (automated, 1% traffic, analytics paths first)

**2. First 3 scenarios:**
- Database primary failover (tests infra + runbooks)
- Payment provider timeout (tests circuit breakers + fallbacks)
- Region failure (tests multi-region claims + org response)

Justification: Cover data layer, external dependencies, and geography - the three highest-risk domains.

**3. Success metrics:**
- MTTR trending down month-over-month
- Automated recovery % increasing
- Post-Game-Day action item closure rate
- SRE confidence score (survey)

**4. Org buy-in:**
- Start with "We almost lost $2M last outage - Game Days prevent that"
- Show insurance industry parallel: they practice disasters
- Pitch as "SLA protection" not "breaking things for fun"
- Get executive sponsor from engineering leadership

**5. Blast radius controls:**
- Start with off-hours, low-traffic regions
- Implement kill-switch (auto-rollback if error rate > 1%)
- Feature flag critical paths (degrade, don't fail)
- Shadow environment for aggressive tests

**6. Remediation process:**
```
Game Day → Document gaps → Create tickets → Assign owners →
Weekly review → Track to close → Retest → Expand scope
```

### Key insight box
> A chaos program is not a one-time project - it's a cultural shift from "avoid failures" to "practice failures."

### Final challenge question
Your CEO asks: "How much will this chaos engineering program cost us in downtime and lost revenue?" How do you answer?

---

## Appendix: Quick checklist (printable)

**Before your first Game Day:**
- [ ] Define success criteria and rollback triggers
- [ ] Get stakeholder sign-off (leadership, legal, compliance)
- [ ] Prepare communication plan (who to notify, when)
- [ ] Set up monitoring dashboards for the test
- [ ] Review runbooks that will be tested
- [ ] Assign observer roles (don't test and observe simultaneously)

**During Game Day:**
- [ ] Document everything: timelines, decisions, blockers
- [ ] Observe both system behavior AND human response
- [ ] Note what's NOT in the runbook
- [ ] Track blast radius continuously
- [ ] Execute rollback if criteria breached

**After Game Day:**
- [ ] Immediate debrief (same day while fresh)
- [ ] Document gaps found (technical and organizational)
- [ ] Create remediation tickets with owners
- [ ] Update runbooks based on learnings
- [ ] Share results with leadership (build program momentum)
- [ ] Schedule retest after remediations complete

**Chaos maturity progression:**
- [ ] Level 1: Manual Game Days in staging (quarterly)
- [ ] Level 2: Manual Game Days in production (monthly, off-hours)
- [ ] Level 3: Automated chaos in production (weekly, scoped)
- [ ] Level 4: Continuous chaos with auto-recovery (daily, broad)

**Red flags (stop and reassess):**
- [ ] Game Days always pass (scenarios too easy)
- [ ] Same scenarios repeated without variation
- [ ] Remediation tickets never close
- [ ] Team dreads Game Days (psychological safety issue)
- [ ] No executive support or visibility
