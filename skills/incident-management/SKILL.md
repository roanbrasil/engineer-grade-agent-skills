---
name: incident-management
description: Guide production incident response, SRE practices, on-call runbooks, postmortems, SLOs, and error budget management
---

# Incident Management & SRE Practices

## Severity Levels

```
+-------+------------------+---------------------+------------------+------------------------+
| Level | Name             | Definition          | Response Time    | Who Responds           |
+-------+------------------+---------------------+------------------+------------------------+
| SEV1  | Critical Outage  | Total service down  | Immediate        | All hands, IC required |
| SEV2  | Major Degradation| Core feature broken | < 15 min         | On-call + lead         |
| SEV3  | Partial Impact   | Subset of users hit | < 1 hour         | On-call engineer       |
| SEV4  | Minor Issue      | Low-impact, no data | Next business day| Ticket + backlog       |
+-------+------------------+---------------------+------------------+------------------------+
```

**SEV1 declaration criteria (any one triggers it):**
- Error rate > 5% for 5+ minutes on critical path
- P99 latency > 10x baseline for 5+ minutes
- Data loss confirmed or suspected
- Security breach confirmed or suspected
- Complete service unavailability for any region

---

## Incident Response Phases

```
  +----------+     +---------+     +----------+     +---------+     +------------+
  |  DETECT  | --> |  TRIAGE | --> | MITIGATE | --> | RESOLVE | --> | POSTMORTEM |
  +----------+     +---------+     +----------+     +---------+     +------------+
       |                |               |               |                 |
  Alert fires      Assign SEV       Stop the         Fix root         Blameless
  Dashboards       Declare IC        bleeding          cause           review
  User reports     Notify team      Rollback?         Verify          Action items
                   Open channel     Feature flag?     Monitor         Owners + dates
```

### Phase Details

**1. Detect**
- Automated alerting fires (never wait for users to report)
- On-call engineer acknowledges within SLA
- Checks: dashboards, logs, error trackers, status page

**2. Triage**
- Assign severity (default to higher severity if uncertain — downgrade is cheaper than escalation delay)
- Declare incident commander (IC)
- Open dedicated Slack channel: `#inc-YYYYMMDD-short-description`
- Post initial message: what is known, what is unknown, severity

**3. Mitigate (PRIORITY OVER ROOT CAUSE)**
```
RULE: Restore service first. Investigate why second.
      A 2-hour outage hunting root cause is worse than a 20-minute mitigation + later RCA.
```
- Options in order of preference:
  1. Rollback last deployment
  2. Toggle feature flag off
  3. Redirect traffic (DNS, load balancer)
  4. Scale out horizontally
  5. Disable non-critical subsystems (graceful degradation)
  6. Manual workaround for users

**4. Resolve**
- Error rate back to baseline for 30+ minutes
- All monitoring indicators green
- Root cause identified (not just mitigated)
- Update status page: "Resolved"
- Close incident channel with summary

**5. Postmortem**
- Write within 5 business days
- Blameless, systems-focused
- Action items with owners and deadlines

---

## Incident Commander (IC) Role

```
                        INCIDENT COMMANDER
                               |
           +-------------------+-------------------+
           |                   |                   |
    TECHNICAL LEAD       COMMUNICATIONS        SCRIBE
    (fixes the thing)    (talks to world)    (writes everything)
```

**IC responsibilities:**
- Single decision-maker — no committee decisions during SEV1/SEV2
- Delegates investigation to technical lead; keeps technical lead off Slack comms
- Makes the call: rollback, scale, escalate, declare resolved
- Never goes heads-down in code themselves
- Runs every 15-minute update cycle

**What the IC does NOT do:**
- Debug code
- Write Slack updates to customers (delegates to Comms)
- Attend to non-incident Slack pings

---

## Communication During Incidents

### Internal Slack Channel Protocol

```
[HH:MM UTC] [UPDATE] Error rate at 12%. Team investigating.
[HH:MM UTC] [THEORY] Spike correlates with deploy abc123. Rolling back now.
[HH:MM UTC] [ACTION] Rollback initiated. ETA 8 minutes.
[HH:MM UTC] [RESOLVED] Error rate nominal. Rollback confirmed good. RCA TBD.
```

**Rules:**
- Every message gets a UTC timestamp
- Tag message type: `[UPDATE]`, `[ACTION]`, `[THEORY]`, `[RESOLVED]`
- Update every 15 minutes even if nothing has changed: `[HH:MM UTC] [UPDATE] No change. Still investigating X.`
- Do not delete or edit messages — corrections go in new messages

### Status Page Updates (customer-facing)

```
Template for incident in progress:
"We are investigating reports of [symptoms]. Our team is actively working to
resolve this issue. We will provide an update within 15 minutes."

Template for mitigation:
"We have identified the cause of [symptoms] and have implemented a mitigation.
We are monitoring to confirm resolution. Some users may still experience issues."

Template for resolution:
"This incident has been resolved. [One sentence on what was affected and for how long].
We will publish a full postmortem within 5 business days."
```

**Tone rules:**
- Never blame a vendor by name (say "a third-party dependency")
- Never speculate about financial impact
- Avoid technical jargon in customer-facing copy
- Always apologize sincerely, once, briefly

---

## The Four Golden Signals (Detection Foundation)

```
+-------------+------------------------------------------+---------------------------+
| Signal      | What it measures                         | Alert example             |
+-------------+------------------------------------------+---------------------------+
| Latency     | How long requests take (p50, p99, p999)  | p99 > 2s for 5 min        |
| Traffic     | Volume (RPS, events/sec, messages/sec)   | RPS drops 50% from avg    |
| Errors      | Rate of failed requests (5xx, timeouts)  | Error rate > 1% for 5 min |
| Saturation  | Resource utilization (CPU, mem, disk, Q) | CPU > 85% for 10 min      |
+-------------+------------------------------------------+---------------------------+
```

**Alert design principles:**
- Alert on symptoms (high latency, high error rate), not causes (high CPU)
- Every alert must have a runbook link
- If an alert cannot be acted on, delete it (alert fatigue kills response quality)

---

## SLOs and Error Budgets

### Definitions

```
SLI (Service Level Indicator): the measurement
    Example: "percentage of HTTP requests returning 2xx within 500ms"

SLO (Service Level Objective): the target
    Example: "99.9% of requests succeed per rolling 30-day window"

Error Budget: the allowed failure space
    Formula: Error Budget = 1 - SLO
    Example: 99.9% SLO = 0.1% budget = 43.8 minutes/month allowed downtime

SLA (Service Level Agreement): the contract with customers (set below SLO)
```

### Error Budget Policy

```
Budget consumed:   < 50%   → Feature development proceeds normally
                  50-75%   → Reduce deploy frequency; review risky changes
                  75-100%  → Freeze feature deploys; focus on reliability
                   > 100%  → Full reliability sprint; no features until replenished
```

### Burn Rate Alerting

**The problem with threshold-only alerts:** a 2x burn rate drains budget silently for weeks.

**Solution: multi-window, multi-burn-rate alerts**

```
+----------------+------------+-------------+---------------------+-------------------+
| Alert          | Window     | Burn Rate   | Budget consumed     | Urgency           |
+----------------+------------+-------------+---------------------+-------------------+
| Page (urgent)  | 1 hour     | > 14.4x     | 2% in 1 hour        | Wake someone now  |
| Page (urgent)  | 5 minutes  | > 14.4x     | Confirms fast burn  | (both must fire)  |
+----------------+------------+-------------+---------------------+-------------------+
| Ticket (slow)  | 6 hours    | > 6x        | 5% in 6 hours       | Fix by morning    |
| Ticket (slow)  | 1 hour     | > 6x        | (both must fire)    |                   |
+----------------+------------+-------------+---------------------+-------------------+
| Warning        | 3 days     | > 1x        | On track to miss    | Investigate       |
+----------------+------------+-------------+---------------------+-------------------+

Burn Rate formula: (error_rate / (1 - SLO)) 
Example: 1% error rate with 99.9% SLO = 1% / 0.1% = 10x burn rate
```

**Prometheus alert example:**
```yaml
# Fast burn — wake someone up
- alert: HighErrorBudgetBurnRate
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[1h]) /
      rate(http_requests_total[1h])
    ) > 14.4 * 0.001
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Fast error budget burn — page IC immediately"
    runbook: "https://runbooks.internal/high-error-rate"

# Slow burn — file a ticket
- alert: SlowErrorBudgetBurnRate
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[6h]) /
      rate(http_requests_total[6h])
    ) > 6 * 0.001
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Slow error budget burn — investigate before EOD"
```

---

## On-Call Runbooks

### Required Sections in Every Runbook

```
1. ALERT NAME & LINK
   What alert fires, how to reach the dashboard

2. SYMPTOMS
   What users experience, what metrics look abnormal

3. SEVERITY GUIDE
   Conditions that make this SEV1 vs SEV2 vs SEV3

4. IMMEDIATE ACTIONS (first 5 minutes)
   Ordered steps that any on-call can execute without deep knowledge

5. DIAGNOSTIC COMMANDS
   Copy-paste ready commands with expected vs abnormal output

6. COMMON CAUSES
   Top 3-5 causes with links to past incidents

7. MITIGATION OPTIONS
   Rollback, feature flag, config change — ordered by preference

8. ESCALATION PATH
   Who to call when: primary → secondary → team lead → VP

9. ROLLBACK PROCEDURE
   Exact steps to revert the last deployment

10. LINKS
    Dashboard, logs, deployment history, architecture diagram
```

### Diagnostic Commands Example Block

```bash
# Check error rate last 5 minutes
kubectl logs -l app=api --since=5m | grep "ERROR" | wc -l

# Check pod status
kubectl get pods -n production -l app=api

# Check recent deployments
kubectl rollout history deployment/api -n production

# Quick rollback to previous version
kubectl rollout undo deployment/api -n production

# Check database connection pool
psql $DATABASE_URL -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Check queue depth
aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```

---

## Blameless Postmortem

### Culture Principles

```
WHAT WE ASK:          "What conditions made this failure possible?"
NOT:                  "Who made this mistake?"

WHAT WE BELIEVE:      Engineers make reasonable decisions with the information available.
                      Systems, not people, are the unit of improvement.

WHAT WE AVOID:        "Human error" as a root cause (it's always a system that allowed
                      the human to cause impact).
```

### Postmortem Template

```markdown
## Incident Postmortem: [Short Title]

**Date:** YYYY-MM-DD
**Severity:** SEV[N]
**Duration:** HH:MM (HH:MM UTC to HH:MM UTC)
**Author(s):** [names]
**Reviewers:** [names]
**Status:** Draft | In Review | Final

---

### Summary
[2-3 sentences: what happened, user impact, how it was resolved]

### Impact
- Users affected: [number or percentage]
- Duration: [minutes/hours]
- Data loss: [yes/no — details]
- Revenue/SLA impact: [if known]
- Error budget consumed this incident: [X% of monthly budget]

### Timeline
| Time (UTC) | Event |
|------------|-------|
| HH:MM      | Alert fires / first detection |
| HH:MM      | IC declared, incident channel opened |
| HH:MM      | [Key investigation step] |
| HH:MM      | [Mitigation applied] |
| HH:MM      | Service restored |
| HH:MM      | Incident declared resolved |

### Root Cause
[One clear paragraph. Not "human error." The systemic condition that allowed this to happen.]

### Contributing Factors
- [Factor 1: e.g., no automated rollback trigger]
- [Factor 2: e.g., alert threshold too high to detect slow burn]
- [Factor 3: e.g., runbook outdated after last refactor]

### 5 Whys Analysis
1. Why did users see errors? → Database connections exhausted
2. Why were connections exhausted? → Connection pool not sized for new traffic pattern
3. Why wasn't the pool resized? → Load test didn't model the new access pattern
4. Why did load test miss this? → Load test script not updated with new feature traffic
5. Why was load test script outdated? → No process to update load tests on feature changes

### Action Items
| Item | Owner | Due Date | Priority |
|------|-------|----------|----------|
| Add automated rollback to deploy pipeline | @engineer | 2024-02-01 | P1 |
| Update runbook for DB exhaustion | @engineer | 2024-01-25 | P1 |
| Add slow-burn alert for connection pool | @sre | 2024-02-08 | P2 |
| Update load test to cover new feature | @qa | 2024-02-15 | P2 |

### What Went Well
- [Specific thing that worked]

### What Could Be Improved
- [Specific gap to address]
```

### 5 Whys Rules
- Ask "why" until you hit a process/system gap, not a person
- Each "why" must be grounded in evidence from the timeline
- Stop at 5 — going deeper produces diminishing returns and speculation
- Multiple causal chains are valid; follow all of them

---

## Runbook Automation

### MTTD and MTTR Reduction

```
MTTD = Mean Time To Detect
MTTR = Mean Time To Recover

Goal: MTTD < 5 minutes (alerting + dashboards)
      MTTR < 30 minutes (runbook automation + rollback)
```

**Automation checklist:**
- [ ] Automated rollback on error rate spike post-deploy
- [ ] Auto-scaling triggers before saturation alerts fire
- [ ] Circuit breakers trip before cascading failures
- [ ] Canary analysis gates deployments automatically
- [ ] Runbook links embedded in every alert
- [ ] Diagnostic dashboards pre-built for top 5 incident types
- [ ] Slack bot can post current status on command (`/incident status`)

---

## Game Days & Chaos Engineering

```
PURPOSE: Practice incident response before real incidents happen.
         Find weaknesses in systems AND weaknesses in process.
```

### Game Day Structure

```
1. PLANNING (1 week before)
   Define failure scenario, success criteria, safety boundaries

2. HYPOTHESIS
   "If we kill 2 of 3 database replicas, we expect automatic failover
    within 30 seconds with < 0.1% error rate impact."

3. EXECUTION
   Inject failure in staging (or production with tight blast radius)
   Team runs incident response as if real

4. OBSERVE
   Did systems behave as expected?
   Did the on-call team respond correctly?
   Were runbooks accurate?

5. LEARN
   Document gaps; create action items; update runbooks
```

### Chaos Experiments by Category

```
+------------------+------------------------------------------+
| Category         | Experiment examples                      |
+------------------+------------------------------------------+
| Compute          | Kill random pods; CPU stress test        |
| Network          | Add 200ms latency; packet drop 10%       |
| Dependencies     | Block calls to external API              |
| Data layer       | Kill replica; corrupt cache              |
| Deployment       | Bad deploy; config change rollout        |
| People           | Primary on-call unavailable              |
+------------------+------------------------------------------+
```

**Chaos tools:** Chaos Monkey, Litmus, Gremlin, AWS Fault Injection Simulator, `tc netem` (Linux)

---

## On-Call Health

### Rotation Design

```
Minimum viable rotation: 5 engineers
  - Prevents single-person burnout
  - Allows context to build across team
  - Each person on-call ~1 week per 5 weeks

Escalation tiers:
  Primary → Secondary (15 min no response) → Team Lead (30 min) → Director (SEV1 only)
```

### Burnout Prevention

```
SIGNALS OF ON-CALL BURNOUT:
  - > 2 pages per night on average
  - Engineers hesitating to deploy on Fridays
  - Runbooks not being updated after incidents
  - Alert fatigue (engineers silencing alerts)

INTERVENTIONS:
  - Alert audit: delete noisy alerts; tune thresholds
  - Runbook investment: top 20 alerts must have runbooks
  - Post-on-call debrief: 30-min sync after every SEV1/SEV2
  - Compensation: on-call pay, comp time, or both
  - "No more SEV1s at 3am" reliability sprints
```

### On-Call Handoff Template

```
HANDOFF [date] [outgoing] → [incoming]

ACTIVE ISSUES:
  - [Any degraded systems, open tickets, monitoring concerns]

KNOWN RISKS:
  - [Upcoming deploys, maintenance windows, risky changes queued]

RECENT INCIDENTS:
  - [Brief summary of any incidents in past week]

ACTION ITEMS FOR INCOMING:
  - [Anything that needs follow-up]
```

---

## Anti-Patterns

```
+--------------------------------------------+------------------------------------------+
| Anti-Pattern                               | Why It Fails                             |
+--------------------------------------------+------------------------------------------+
| "All hands" for every alert               | Burnout; no clear ownership              |
| Root cause before mitigation              | Prolongs outage unnecessarily            |
| Post-incident blame                       | Reduces future incident reporting        |
| Postmortems without action items          | Same incident recurs in 6 months         |
| Action items with no owner               | Nothing gets done                        |
| Alerts with no runbooks                   | On-call paralyzed; escalates everything  |
| Status page updated only at resolution   | Customers lose trust during outage       |
| IC also doing technical investigation    | Neither role done well                   |
| "It was human error" as root cause       | System gap remains; failure repeats      |
| Game days in production without limits   | Cure worse than disease                  |
+--------------------------------------------+------------------------------------------+
```

---

## Quick Reference Checklist

### First 5 Minutes of a SEV1

- [ ] Acknowledge the alert
- [ ] Check Four Golden Signals dashboard
- [ ] Declare severity; open `#inc-YYYYMMDD-name` channel
- [ ] Post initial update: "Investigating [symptom]. Severity: SEV1. IC: @me."
- [ ] Assign roles: IC, Technical Lead, Comms
- [ ] Check: was there a recent deployment? (`git log --oneline -10` / deployment history)
- [ ] Consider rollback as first mitigation option
- [ ] Update status page: "Investigating"
- [ ] Set 15-minute timer for next update

### Postmortem Checklist

- [ ] Written within 5 business days
- [ ] Full timeline constructed from logs + Slack
- [ ] Root cause is systemic, not "human error"
- [ ] Every action item has one owner and a deadline
- [ ] Reviewed by the team before publishing
- [ ] Shared with stakeholders
- [ ] Action items tracked in issue tracker (not just the doc)
