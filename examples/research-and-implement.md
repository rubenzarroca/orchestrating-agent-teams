# Example: Research & Implement — "Choose and Build a Caching Layer" (Throughput Paradigm)

A 2-teammate workflow with controlled sequencing: the Researcher evaluates options
and documents a recommendation, the Implementer builds it. Unlike Perspective teams,
these agents don't challenge each other — one informs, the other executes. The value
comes from overlapping research time with preparation time, and from separating the
"think" phase from the "build" phase into specialized roles.

---

## Context

Project: Node.js API serving a React dashboard. API response times have grown from
50ms to 400ms as the dataset scaled. The team needs a caching layer but hasn't
decided between Redis, in-memory (node-cache), or HTTP caching (CDN + ETags).
Codebase: Express API in `src/api/`, data access in `src/db/`, no existing cache layer.

## Step 1: Planning (Single Agent, Plan Mode)

```
Our API latency went from 50ms to 400ms. We need a caching strategy.
Options: Redis, in-memory (node-cache), HTTP caching (CDN + ETags).
Use plan mode. Break this into research and implementation tracks.
```

**Lead output (plan):**
- Track 1 — Researcher: Evaluate all three options against our specific constraints
  (data freshness requirements, multi-instance deployment, memory budget). Produce a
  recommendation document with clear rationale.
- Track 2 — Implementer: Wait for research to complete, then build the recommended
  approach. Implement in `src/cache/`, integrate with existing data access layer.

Research goes first. Implementation starts only when research is marked complete.

## Step 2: Team Spawn

```
Our Express API latency grew from 50ms to 400ms as data scaled. We need caching.
Options on the table: Redis, in-memory (node-cache), HTTP caching (CDN + ETags).

Constraints:
- App runs 3 instances behind a load balancer (sticky sessions off)
- Data changes every ~15 minutes (batch import from external source)
- Dashboard has 12 API endpoints, 4 are heavy (aggregation queries, 300ms+)
- Memory budget per instance: 512MB
- Team has no Redis operational experience

Create an agent team with 2 teammates:

1. Researcher: Evaluate Redis, node-cache, and HTTP caching for our situation.
   Analyze the codebase: check src/api/ for endpoint patterns, src/db/ for query
   patterns, and package.json for existing dependencies. Consider our constraints
   (multi-instance, 15-min freshness, 512MB memory, no Redis ops experience).
   Document your analysis in docs/research/caching-strategy.md with:
   - Pros/cons of each option against our specific constraints
   - Clear recommendation with rationale
   - Implementation outline (what files to create, what to modify)
   - Cache invalidation strategy for the 15-minute refresh cycle
   You own docs/research/ exclusively. Do not write implementation code.
   Done when: recommendation document is complete with actionable implementation plan.

2. Implementer: Wait for Researcher to complete and read their recommendation
   at docs/research/caching-strategy.md. Then implement the recommended approach
   in src/cache/. Integrate with the existing data access layer in src/db/.
   If the recommendation is unclear, DM the Researcher for clarification.
   You own src/cache/ exclusively. You may modify src/db/ files to add cache
   integration but do not restructure existing query logic.
   Done when: caching is integrated, the 4 heavy endpoints use the cache,
   and cache invalidation is wired to the 15-minute import cycle.

Researcher goes first. Implementer starts only after research is marked complete.
Use Sonnet for both teammates.
Activate delegate mode. Do not write any code yourself.
Wait for all teammates to complete their tasks before proceeding.
```

## Step 3: Sequential Execution with Overlap

**Researcher** (working ~5 min):
- Reads src/api/ → identifies 12 endpoints, 4 heavy ones hit aggregation queries
- Reads src/db/ → queries use a shared pool, no caching layer exists
- Reads package.json → no Redis client, already has express middleware setup

Analysis:
- **Redis**: Best for multi-instance consistency. But team has no ops experience,
  adds infrastructure dependency, overkill for 15-minute TTL with batch invalidation.
  Would need new infra provisioning.
- **node-cache**: Simple, zero deps. But each instance has its own cache — with 3
  instances and no sticky sessions, cache hit rate drops to ~33%. Memory risk if
  dataset grows.
- **HTTP caching (CDN + Cache-Control)**: Leverages existing infrastructure. Set
  `Cache-Control: public, max-age=900` (15 min) on the 4 heavy endpoints. Zero
  code for cache invalidation — TTL matches data refresh. Works across all instances.
  No memory overhead on the app.

Recommendation: HTTP caching with Cache-Control headers. Lowest complexity, matches
the 15-minute freshness perfectly, no new infrastructure, no memory cost. Add ETags
for conditional requests on the remaining 8 endpoints.

- Documents recommendation in docs/research/caching-strategy.md
- Marks task complete

**Implementer** (starts after Researcher completes, working ~6 min):
- Reads docs/research/caching-strategy.md
- Creates src/cache/http-cache.ts — Express middleware that sets Cache-Control headers
- Configures: heavy endpoints get `max-age=900`, light endpoints get `max-age=60` + ETag
- Creates src/cache/etag.ts — ETag generator based on response content hash
- Modifies src/api/router.ts to apply cache middleware to relevant routes
- Tests with curl — verifies Cache-Control headers present, ETag conditional responses work
- Marks task complete

## Step 4: Lead Synthesis

```markdown
## Caching Layer — Implementation Summary

### Decision
HTTP caching via Cache-Control headers + ETags. Chosen over Redis (unnecessary
complexity for our use case) and node-cache (multi-instance inconsistency).

### What Was Built
- src/cache/http-cache.ts: Express middleware, Cache-Control headers
- src/cache/etag.ts: ETag generation for conditional requests
- src/api/router.ts: Modified to apply cache middleware

### Cache Strategy
- 4 heavy endpoints: Cache-Control max-age=900 (matches 15-min import cycle)
- 8 light endpoints: Cache-Control max-age=60 + ETag for conditional requests
- No explicit invalidation needed — TTL aligns with data refresh

### Expected Impact
- Heavy endpoints: ~300ms → ~0ms for cached responses (CDN serves directly)
- Light endpoints: conditional 304 responses reduce payload transfer
- Zero memory overhead on application instances
- No new infrastructure required

### Follow-up
- Monitor CDN cache hit rate after deploy (target: >80% for heavy endpoints)
- If data refresh frequency changes from 15 min, update max-age accordingly
- Consider adding Vary headers if user-specific data is added later
```

## Why This Worked

- **Controlled sequencing**: Unlike Parallel Modules, these tasks have a dependency
  (Implementer needs Researcher's output). The team structure enforces the sequence
  explicitly rather than hoping for the right order.
- **Separation of concerns**: The Researcher explored options without implementation
  bias. The Implementer focused on building without decision fatigue. Each role did
  what it's best at.
- **DM as escape valve**: The Implementer could DM the Researcher for clarification
  if needed, but didn't have to — the recommendation document was specific enough.
  This is the minimal mailbox usage typical of Throughput teams.
- **Research prevented over-engineering**: A single agent might have jumped to Redis
  (the "obvious" choice) without evaluating whether simpler options fit the constraints.
  The dedicated research phase forced a thorough evaluation that landed on the simplest
  sufficient solution.
- **Actionable handoff**: The research document included an implementation outline, not
  just a recommendation. This is what makes the handoff work — the Implementer received
  a buildable plan, not just a "go use HTTP caching" verdict.
