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

## Step 2: Team Setup and Spawn

**Create the team and tasks:**
```
TeamCreate(team_name: "caching-layer", description: "Research and implement API caching strategy")

TaskCreate(title: "Research caching options", description: "Evaluate Redis, node-cache, HTTP caching")
TaskCreate(title: "Implement recommended caching", description: "Build chosen approach in src/cache/")
```

**Spawn researcher first** (Explore type — read-only research):
```
Agent(
  description: "Research caching options",
  team_name: "caching-layer",
  name: "researcher",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "Evaluate Redis, node-cache, and HTTP caching for our API.
    Constraints:
    - App runs 3 instances behind a load balancer (sticky sessions off)
    - Data changes every ~15 minutes (batch import from external source)
    - Dashboard has 12 API endpoints, 4 are heavy (aggregation queries, 300ms+)
    - Memory budget per instance: 512MB
    - Team has no Redis operational experience
    Analyze the codebase: check src/api/ for endpoint patterns, src/db/ for query
    patterns, and package.json for existing dependencies.
    Document your analysis in docs/research/caching-strategy.md with:
    - Pros/cons of each option against our specific constraints
    - Clear recommendation with rationale
    - Implementation outline (what files to create, what to modify)
    - Cache invalidation strategy for the 15-minute refresh cycle
    You own docs/research/ exclusively. Do not write implementation code.
    Done when: recommendation document is complete with actionable implementation plan.
    Mark your task as completed via TaskUpdate, then DM 'implementer' that research is ready."
)
```

**Spawn implementer after researcher completes** (general-purpose with plan approval):
```
Agent(
  description: "Implement caching layer",
  team_name: "caching-layer",
  name: "implementer",
  model: "sonnet",
  mode: "plan",
  prompt: "Read the caching research at docs/research/caching-strategy.md and implement
    the recommended approach in src/cache/. Integrate with the existing data access
    layer in src/db/. If the recommendation is unclear, DM 'researcher' for clarification
    via SendMessage.
    You own src/cache/ exclusively. You may modify src/db/ files to add cache
    integration but do not restructure existing query logic.
    Present your implementation plan first and wait for approval.
    Done when: caching is integrated, the 4 heavy endpoints use the cache,
    and cache invalidation is wired to the 15-minute import cycle.
    Mark your task as completed via TaskUpdate when done."
)
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
- Marks task complete via TaskUpdate
- DMs implementer via SendMessage: "Research complete. Recommendation: HTTP caching."

**Implementer** (starts after Researcher completes, working ~6 min):
- Reads docs/research/caching-strategy.md
- Presents plan → Lead approves via `plan_approval_response`
- Creates src/cache/http-cache.ts — Express middleware that sets Cache-Control headers
- Configures: heavy endpoints get `max-age=900`, light endpoints get `max-age=60` + ETag
- Creates src/cache/etag.ts — ETag generator based on response content hash
- Modifies src/api/router.ts to apply cache middleware to relevant routes
- Tests with curl — verifies Cache-Control headers present, ETag conditional responses work
- Marks task complete via TaskUpdate

## Step 4: Lead Synthesis and Shutdown

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

**Shutdown and cleanup:**
```
SendMessage(to: "researcher", message: { type: "shutdown_request", reason: "Implementation complete" })
SendMessage(to: "implementer", message: { type: "shutdown_request", reason: "Implementation complete" })
TeamDelete()
```

## Why This Worked

- **Controlled sequencing**: The team structure enforces Research → Implement order
  explicitly. The implementer waited for the researcher's TaskUpdate completion.
- **Separation of concerns**: The Researcher explored options without implementation
  bias. The Implementer focused on building without decision fatigue.
- **DM as escape valve**: The Implementer could DM the Researcher for clarification
  via SendMessage if needed, but didn't have to — the recommendation was specific enough.
- **Plan approval for implementation**: `mode: "plan"` ensured the Lead reviewed the
  implementation approach before any code was written.
- **Research prevented over-engineering**: A single agent might have jumped to Redis
  (the "obvious" choice) without evaluating whether simpler options fit the constraints.
- **Actionable handoff**: The research document included an implementation outline, not
  just a recommendation — the Implementer received a buildable plan.
- **Clean lifecycle**: TeamCreate → sequential work → shutdown_request → TeamDelete.
