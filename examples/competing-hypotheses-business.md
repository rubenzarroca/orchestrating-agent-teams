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

## Step 2: Team Spawn

```
Our SaaS conversion (signup → paid) dropped from 12% to 7% in 3 weeks.
No deploys, no marketing changes. Codebase: Next.js + Node + Postgres.

Create an agent team of 5 investigators. Each defends a hypothesis:

1. "Onboarding Regression" investigator:
   Check package-lock.json and yarn.lock for dependency updates in the last 4 weeks.
   Inspect src/components/onboarding/ for any 3rd-party widget changes (Intercom,
   Appcues, etc.). Check if CDN-loaded scripts changed versions. Look at
   src/pages/onboarding/ for conditional rendering that might have shifted.
   Evidence needed: a specific change that altered the onboarding experience.

2. "Trial Experience Degraded" investigator:
   Analyze src/api/ for error handling patterns. Check if any external service
   (Stripe, SendGrid, analytics) has timeout/retry changes. Look at
   src/middleware/rate-limit.ts for threshold changes. Check docker-compose.yml
   and infra/ for resource limit changes. Search logs/ for error rate trends.
   Evidence needed: measurable degradation in trial-period user experience.

3. "Pricing Page Confusion" investigator:
   Inspect src/pages/pricing/ and src/components/pricing/. Check for active
   feature flags in src/config/feature-flags.ts or PostHog/LaunchDarkly configs.
   Look for A/B test variants still running. Check if pricing tiers, CTAs, or
   copy changed. Inspect the checkout flow at src/pages/checkout/.
   Evidence needed: a pricing/checkout change that could confuse or deter users.

4. "Lead Quality Shift" investigator:
   Analyze src/lib/analytics/ and src/middleware/tracking.ts for UTM handling.
   Check src/pages/api/webhooks/ for signup source data. Look at seed data or
   migration files for any changes to user segmentation. Search for referral
   or attribution logic changes.
   Evidence needed: a shift in traffic source composition correlated with the drop.

5. "Competitor Disruption" investigator:
   Use web search to check competitor pricing pages (list top 5 competitors in
   the space). Search Product Hunt, Hacker News, and G2 for launches in the
   last 30 days. Check if any competitor announced a free tier or aggressive
   pricing change. Look at our public-facing pages for competitive positioning gaps.
   Evidence needed: a specific competitor move that explains user hesitation.

Rules:
- Each investigator MUST present evidence, not speculation.
- Investigators message each other when findings overlap or contradict.
- Rate your own hypothesis: CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
- If you find evidence supporting ANOTHER hypothesis, message that investigator.

Activate delegate mode. Do not investigate anything yourself.
Wait for all teammates to complete their tasks before proceeding.
```

## Step 3: Parallel Investigation

**H1 — Onboarding Regression** (~4 min):
- Scans package-lock.json → finds `@intercom/messenger-js-sdk` bumped from 0.4.x to 0.5.x
  three weeks ago via dependabot auto-merge
- Reads changelog → v0.5.0 changed default launcher position from bottom-right to
  bottom-left, AND added a mandatory cookie consent banner for EU users
- Checks src/components/onboarding/IntercomWidget.tsx → no pinned version, uses `latest`
- Messages H3 investigator: "Cookie consent banner might be covering the pricing CTA
  on mobile. Check if pricing page has overlap issues."
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
- Receives message from H1 about cookie consent banner overlap
- Checks pricing page mobile layout → confirms the Intercom cookie banner overlaps
  the "Start Free Trial" CTA on viewports < 768px
- Messages H1: "Confirmed. Cookie banner covers CTA on mobile. This is your evidence."
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

## Step 4: Lead Synthesis

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

**Impact estimate**: 55% mobile × 40% EU = ~22% of signup traffic sees a
blocked CTA. Combined with the disorienting launcher repositioning, this
plausibly accounts for the 12% → 7% drop.

### Hypotheses Summary

| Hypothesis | Rating | Key Finding |
|-----------|--------|-------------|
| H1: Onboarding regression | ✅ CONFIRMED | Intercom v0.5.0 cookie banner blocks mobile CTA |
| H2: Trial degraded | ❌ REFUTED | All API/infra metrics stable |
| H3: Pricing confusion | ➡️ REDIRECTED | Found H1 corroborating evidence (banner overlap) |
| H4: Lead quality shift | ❌ REFUTED | Traffic sources unchanged |
| H5: Competitor disruption | ❌ REFUTED | No market changes detected |

### Recommended Actions
1. **Immediate**: Pin `@intercom/messenger-js-sdk` to 0.4.x in package.json
2. **This week**: Add Intercom widget to dependabot ignore list or require manual review
3. **Follow-up**: Add visual regression tests for mobile onboarding + pricing flows
4. **Process**: Audit all auto-merge rules for user-facing 3rd-party SDKs
```

## Why This Worked

- **Rival framing**: Investigators were motivated to find evidence, not confirm bias
- **Cross-pollination**: H1 messaged H3 with a lead, H3 confirmed it — the truth
  emerged from collaboration between competing theories
- **Fast elimination**: H2, H4, H5 refuted quickly, focusing attention on the real cause
- **Non-obvious root cause**: No human deploy caused this — a dependabot auto-merge
  of a 3rd-party SDK created a UX regression invisible to traditional monitoring
- **Actionable output**: Immediate fix (pin version) + systemic fix (audit auto-merge rules)
