# Example: Competing Business Hypotheses — "Why Did Conversion Drop 40%?" (Perspective Paradigm)

A 5-agent investigation into a sudden conversion collapse in a SaaS signup funnel.
Agents work as rival analysts: each defends a theory, shares evidence, and challenges
the others. The Lead acts as a board moderator, not an analyst. This is a Perspective
team at its most powerful: the root cause emerged from cross-pollination between agents,
not from any single agent's analysis.

---

## Context

Product: B2B SaaS analytics platform. Freemium model with 14-day trial.
Conversion from signup → paid dropped from 12% to 7% over the last 3 weeks.
No deploys to production in that window. Marketing spend unchanged.
The codebase has full-stack TypeScript: Next.js frontend, Node API, Postgres.

## Step 1: Hypothesis Generation (Plan Mode, ~2 min)

```
Our signup-to-paid conversion dropped from 12% to 7% in 3 weeks.
No code deploys, no marketing changes. Use plan mode.
Generate 5 competing hypotheses and the evidence each would need.
```

**Lead output (plan):**

| # | Hypothesis | Key Evidence Needed |
|---|-----------|---------------------|
| H1 | Onboarding UX regression (3rd-party widget update?) | Check package-lock.json diffs, Intercom/widget changelogs, DOM changes in onboarding flow |
| H2 | Trial experience degraded (API latency, errors) | Query error logs, p95/p99 latency trends, failed API calls during trial period |
| H3 | Pricing page confusion (A/B test still running?) | Check feature flags, PostHog experiments, pricing page render variations |
| H4 | Lead quality shifted (different traffic source mix) | Analyze UTM params, referral breakdown, signup cohort demographics over 6 weeks |
| H5 | Competitor launched free tier (external market shift) | Scan competitor pricing pages, Product Hunt launches, G2/Capterra reviews last 30 days |

All five are independent. No shared files to edit — pure investigation.

## Step 2: Team Setup and Spawn

**Create the team and tasks:**
```
TeamCreate(team_name: "conversion-investigation", description: "Root cause analysis: conversion drop 12% → 7%")

TaskCreate(title: "H1: Onboarding regression investigation", description: "...")
TaskCreate(title: "H2: Trial experience degradation investigation", description: "...")
TaskCreate(title: "H3: Pricing page confusion investigation", description: "...")
TaskCreate(title: "H4: Lead quality shift investigation", description: "...")
TaskCreate(title: "H5: Competitor disruption investigation", description: "...")
```

**Spawn 5 investigators** (all Explore type — read-only investigation):

```
Agent(
  description: "Investigate onboarding regression",
  team_name: "conversion-investigation",
  name: "h1-onboarding",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: Onboarding UX regression caused by
    a 3rd-party widget update. Context: SaaS conversion dropped 12% → 7% in 3 weeks,
    no deploys, no marketing changes. Codebase: Next.js + Node + Postgres.
    Check package-lock.json and yarn.lock for dependency updates in the last 4 weeks.
    Inspect src/components/onboarding/ for any 3rd-party widget changes (Intercom,
    Appcues, etc.). Check if CDN-loaded scripts changed versions.
    Evidence needed: a specific change that altered the onboarding experience.
    Message other investigators via SendMessage when you find overlapping evidence.
    Rate your hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Investigate trial degradation",
  team_name: "conversion-investigation",
  name: "h2-trial",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: Trial experience degraded (API latency,
    errors). Context: SaaS conversion dropped 12% → 7% in 3 weeks, no deploys.
    Analyze src/api/ for error handling patterns. Check external service configs
    (Stripe, SendGrid). Look at src/middleware/rate-limit.ts and infra/ for changes.
    Search logs/ for error rate trends.
    Evidence needed: measurable degradation in trial-period user experience.
    Message other investigators via SendMessage when you find overlapping evidence.
    Rate your hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Investigate pricing confusion",
  team_name: "conversion-investigation",
  name: "h3-pricing",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: Pricing page confusion (A/B test
    still running?). Context: SaaS conversion dropped 12% → 7% in 3 weeks.
    Inspect src/pages/pricing/ and src/components/pricing/. Check feature flags in
    src/config/feature-flags.ts. Look for A/B test variants still running.
    Check if pricing tiers, CTAs, or copy changed. Inspect src/pages/checkout/.
    Evidence needed: a pricing/checkout change that could confuse or deter users.
    Message other investigators via SendMessage when you find overlapping evidence.
    Rate your hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Investigate lead quality shift",
  team_name: "conversion-investigation",
  name: "h4-leads",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: Lead quality shifted (different
    traffic source mix). Context: SaaS conversion dropped 12% → 7% in 3 weeks.
    Analyze src/lib/analytics/ and src/middleware/tracking.ts for UTM handling.
    Check src/pages/api/webhooks/ for signup source data. Search for referral
    or attribution logic changes.
    Evidence needed: a shift in traffic source composition correlated with the drop.
    Message other investigators via SendMessage when you find overlapping evidence.
    Rate your hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)

Agent(
  description: "Investigate competitor disruption",
  team_name: "conversion-investigation",
  name: "h5-competitor",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: Competitor launched free tier
    (external market shift). Context: SaaS conversion dropped 12% → 7% in 3 weeks.
    Use web search to check competitor pricing pages (top 5 competitors).
    Search Product Hunt, Hacker News, and G2 for launches in the last 30 days.
    Check if any competitor announced a free tier or aggressive pricing change.
    Evidence needed: a specific competitor move that explains user hesitation.
    Message other investigators via SendMessage when you find overlapping evidence.
    Rate your hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)
```

## Step 3: Parallel Investigation

**H1 — Onboarding Regression** (~4 min):
- Scans package-lock.json → finds `@intercom/messenger-js-sdk` bumped from 0.4.x to 0.5.x
  three weeks ago via dependabot auto-merge
- Reads changelog → v0.5.0 changed default launcher position from bottom-right to
  bottom-left, AND added a mandatory cookie consent banner for EU users
- Checks src/components/onboarding/IntercomWidget.tsx → no pinned version, uses `latest`
- Sends DM to h3-pricing via SendMessage: "Cookie consent banner might be covering the
  pricing CTA on mobile. Check if pricing page has overlap issues."
- Self-rating: **LIKELY** — timing matches perfectly, but needs conversion data by region

**H2 — Trial Degraded** (~5 min):
- Checks API error logs → error rate stable at 0.3%, no change
- Checks p95 latency → stable at 180ms, no degradation
- Checks external service configs → no timeout changes
- Checks infra/ → resource limits unchanged
- Self-rating: **REFUTED** — no measurable degradation in any metric

**H3 — Pricing Confusion** (~4 min):
- Checks feature-flags.ts → finds `pricing_v2_experiment` flag still active
- But flag was set to `control: 100%` 5 weeks ago (before the drop). No variant running.
- Receives DM from h1-onboarding about cookie consent banner overlap
- Checks pricing page mobile layout → confirms the Intercom cookie banner overlaps
  the "Start Free Trial" CTA on viewports < 768px
- Sends DM to h1-onboarding via SendMessage: "Confirmed. Cookie banner covers CTA on mobile."
- Self-rating: **INCONCLUSIVE** on original hypothesis, but **supporting evidence for H1**

**H4 — Lead Quality** (~3 min):
- Analyzes UTM tracking → traffic source mix is identical (±2%) to previous period
- Signup volume is stable. No new channels, no dropped channels.
- Self-rating: **REFUTED** — traffic composition unchanged

**H5 — Competitor Disruption** (~4 min):
- Searches competitor pricing pages → no changes in the last 30 days
- Checks Product Hunt → one competitor launched a minor feature, not a free tier
- Checks G2 reviews → no spike in competitor mentions
- Self-rating: **REFUTED** — no external market shift detected

## Step 4: Lead Synthesis and Shutdown

```markdown
## Investigation Results — Conversion Drop Root Cause

### Verdict: H1 CONFIRMED — Intercom SDK Auto-Update

**Root cause**: Dependabot auto-merged `@intercom/messenger-js-sdk` v0.5.0 three
weeks ago. This version introduced two breaking UX changes:
1. Mandatory cookie consent banner for EU users
2. Default launcher repositioned from bottom-right to bottom-left

**Smoking gun** (corroborated by H3 investigator): The cookie consent banner
overlaps the "Start Free Trial" CTA on mobile viewports (< 768px). Mobile
traffic is ~55% of signups. EU users represent ~40% of the user base.

**Impact estimate**: 55% mobile x 40% EU = ~22% of signup traffic sees a
blocked CTA. Combined with the disorienting launcher repositioning, this
plausibly accounts for the 12% → 7% drop.

### Hypotheses Summary

| Hypothesis | Rating | Key Finding |
|-----------|--------|-------------|
| H1: Onboarding regression | CONFIRMED | Intercom v0.5.0 cookie banner blocks mobile CTA |
| H2: Trial degraded | REFUTED | All API/infra metrics stable |
| H3: Pricing confusion | REDIRECTED | Found H1 corroborating evidence (banner overlap) |
| H4: Lead quality shift | REFUTED | Traffic sources unchanged |
| H5: Competitor disruption | REFUTED | No market changes detected |

### Recommended Actions
1. **Immediate**: Pin `@intercom/messenger-js-sdk` to 0.4.x in package.json
2. **This week**: Add Intercom widget to dependabot ignore list or require manual review
3. **Follow-up**: Add visual regression tests for mobile onboarding + pricing flows
4. **Process**: Audit all auto-merge rules for user-facing 3rd-party SDKs
```

**Shutdown and cleanup:**
```
SendMessage(to: "h1-onboarding", message: { type: "shutdown_request", reason: "Investigation complete" })
SendMessage(to: "h2-trial", message: { type: "shutdown_request", reason: "Investigation complete" })
SendMessage(to: "h3-pricing", message: { type: "shutdown_request", reason: "Investigation complete" })
SendMessage(to: "h4-leads", message: { type: "shutdown_request", reason: "Investigation complete" })
SendMessage(to: "h5-competitor", message: { type: "shutdown_request", reason: "Investigation complete" })
TeamDelete()
```

## Why This Worked

- **Rival framing**: Investigators were motivated to find evidence, not confirm bias
- **Cross-pollination via SendMessage**: H1 DM'd H3 with a lead, H3 confirmed it — the
  truth emerged from collaboration between competing theories
- **Fast elimination**: H2, H4, H5 refuted quickly, focusing attention on the real cause
- **Non-obvious root cause**: No human deploy caused this — a dependabot auto-merge
  of a 3rd-party SDK created a UX regression invisible to traditional monitoring
- **Actionable output**: Immediate fix (pin version) + systemic fix (audit auto-merge rules)
- **Clean lifecycle**: TeamCreate at start, shutdown_request to each, TeamDelete at end
