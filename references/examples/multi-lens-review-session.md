# Example: Multi-Lens Code Review Session (Perspective Paradigm)

A complete walkthrough of a successful 3-teammate code review. This example shows the
expected flow, communication patterns, and output format. This is a Perspective team:
all agents examine the same code through different lenses (security, performance, tests).

---

## Context

Project: E-commerce API. Reviewing a new payment module at `src/payments/` before merge.
The module handles Stripe integration, webhook processing, and refund logic.

## Step 1: Planning (Single Agent, Plan Mode)

Before spawning the team, the developer decomposes the review:

```
I need a thorough review of src/payments/ before merging to main.
Use plan mode. Break this into independent review tracks.
```

**Lead output (plan):**
- Track 1 — Security: Token handling, PCI compliance, webhook signature verification
- Track 2 — Performance: DB queries in webhook handler, idempotency key lookups
- Track 3 — Test coverage: Missing edge cases, error path testing, integration tests

All three tracks can run independently on different aspects of the same directory.
No file edits needed — read-only review. Safe for parallel work.

## Step 2: Team Setup and Spawn

**Create the team and tasks:**
```
TeamCreate(team_name: "payments-review", description: "Multi-lens review of src/payments/")

TaskCreate(title: "Security audit of payments module", description: "...")
TaskCreate(title: "Performance audit of payments module", description: "...")
TaskCreate(title: "Test coverage analysis of payments module", description: "...")
```

**Spawn 3 reviewers** (all use Explore since this is read-only):

```
Agent(
  description: "Security review of payments",
  team_name: "payments-review",
  name: "security-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the security reviewer for the payment module at src/payments/.
    Focus on: Stripe API key handling in config.ts, webhook signature verification
    in webhooks.ts, PCI compliance in checkout.ts. The app stores no card data
    directly — all tokenized through Stripe. Report findings with severity ratings
    (critical/high/medium/low) and remediation steps.
    If you find evidence relevant to performance or testing, DM the relevant
    reviewer via SendMessage.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Performance review of payments",
  team_name: "payments-review",
  name: "perf-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the performance reviewer for the payment module at src/payments/.
    Focus on: DB queries in webhooks.ts (check for N+1 on order lookups),
    idempotency implementation in idempotency.ts, connection pooling in stripe-client.ts.
    Report with estimated impact (high/medium/low) and optimization suggestions.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Test coverage review of payments",
  team_name: "payments-review",
  name: "test-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the test coverage reviewer. Analyze tests/payments/ against src/payments/.
    Focus on: webhook failure scenarios, partial refund edge cases, concurrent
    payment attempts. Identify untested code paths and missing integration tests.
    Report with priority for which tests to add first.
    When done, mark your task as completed via TaskUpdate."
)
```

## Step 3: Parallel Execution

Each teammate works independently in their own context window:

**Security reviewer** (working ~3 min):
- Reads config.ts → flags API key loaded from env but no rotation mechanism
- Reads webhooks.ts → verifies signature check is present, but finds timing-safe
  comparison is missing (medium severity)
- Reads checkout.ts → confirms no raw card data stored
- Marks task complete via TaskUpdate

**Performance reviewer** (working ~4 min):
- Reads webhooks.ts → finds N+1 query pattern on line 47 (order.items loaded in loop)
- Reads idempotency.ts → confirms idempotency keys have TTL, no issues
- Reads stripe-client.ts → finds no connection reuse, new client per request (medium impact)
- Marks task complete via TaskUpdate

**Test coverage reviewer** (working ~3 min):
- Compares test files against source → 73% line coverage
- Identifies gap: no tests for webhook retry scenarios
- Identifies gap: refund partial amount boundary (0, negative, > original)
- Identifies gap: no integration test for full checkout→webhook→confirmation flow
- Marks task complete via TaskUpdate

## Step 4: Lead Synthesis and Shutdown

After all three teammates report, the Lead synthesizes:

```markdown
## Payment Module Review — Synthesis

### Critical Findings (address before merge)
None identified.

### High Priority (address in this sprint)
1. [PERF] N+1 query in webhooks.ts:47 — order items loaded in loop.
   Fix: eager-load with JOIN or batch query. Est. impact: webhook p99 latency.
2. [TEST] No integration test for checkout→webhook→confirmation flow.
   This is the critical path. Must have coverage before production.

### Medium Priority (address next sprint)
3. [SEC] Webhook signature verification uses string equality instead of
   timing-safe comparison. Risk: timing attack on signature.
4. [PERF] stripe-client.ts creates new Stripe client per request.
   Fix: singleton or connection pool. Est. impact: cold start latency.
5. [TEST] Missing edge cases: partial refund boundaries (0, negative, >original).

### Low Priority (backlog)
6. [SEC] No API key rotation mechanism. Current setup works but no rotation
   path without downtime.
7. [TEST] Webhook retry scenario testing (idempotency under concurrent retries).

### Recommended Merge Decision
Proceed with merge after addressing items 1 and 2. Items 3-5 tracked as
follow-up tickets. Items 6-7 added to backlog.
```

**Shutdown and cleanup:**
```
SendMessage(to: "security-reviewer", message: { type: "shutdown_request", reason: "Review complete" })
SendMessage(to: "perf-reviewer", message: { type: "shutdown_request", reason: "Review complete" })
SendMessage(to: "test-reviewer", message: { type: "shutdown_request", reason: "Review complete" })
TeamDelete()
```

## Why This Worked

- **Clear scope per teammate**: Each owned a specific lens, no overlap in analysis
- **Explore agent type**: Read-only access prevented accidental "fixes" during review
- **Self-contained briefs**: Each teammate had enough context to work without asking
- **Lead stayed in coordinator role**: Did not review code itself, only synthesized
- **Synthesis added value**: Lead identified cross-cutting priorities, not just concatenation
- **Actionable output**: Clear severity, fix suggestions, and merge recommendation
- **Clean shutdown**: Formal shutdown_request + TeamDelete left no orphaned resources
