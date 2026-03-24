# Agent Team Prompt Templates

Ready-to-use prompts showing the full lifecycle: TeamCreate, Agent spawning with
correct parameters, and shutdown. Replace `[bracketed sections]` with your specifics.

---

## Pattern 1: Multi-Lens Code Review (Perspective)

**Step 1 — Create team:**
```
TeamCreate(team_name: "code-review", description: "Multi-lens review of [module]")
```

**Step 2 — Create tasks:**
```
TaskCreate(title: "Security review of [module]", description: "...")
TaskCreate(title: "Performance review of [module]", description: "...")
TaskCreate(title: "Test coverage review of [module]", description: "...")
```

**Step 3 — Spawn reviewers** (all in parallel, using Explore since read-only):

Agent 1 — Security:
```
Agent(
  description: "Security review",
  team_name: "code-review",
  name: "security-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the security reviewer for [module at path/].
    Focus on [auth, input validation, injection, XSS].
    Check [src/auth/, src/api/middleware/].
    Report findings with severity ratings (critical/high/medium/low)
    and remediation steps. When done, mark your task as completed
    via TaskUpdate. If you find evidence relevant to performance or
    testing, DM the relevant reviewer via SendMessage."
)
```

Agent 2 — Performance:
```
Agent(
  description: "Performance review",
  team_name: "code-review",
  name: "perf-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the performance reviewer for [module at path/].
    Audit [DB queries, response times, memory].
    Check [src/db/, src/services/].
    Flag N+1 queries and unindexed lookups.
    Report with estimated impact (high/medium/low) and optimization
    suggestions. When done, mark your task as completed via TaskUpdate."
)
```

Agent 3 — Test Coverage:
```
Agent(
  description: "Test coverage review",
  team_name: "code-review",
  name: "test-reviewer",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are the test coverage reviewer for [module].
    Validate coverage for [critical paths].
    Check [tests/]. Report untested edge cases and missing
    integration tests with priority. When done, mark your task
    as completed via TaskUpdate."
)
```

**Step 4 — After all complete, synthesize and shut down:**
```
SendMessage(to: "security-reviewer", message: { type: "shutdown_request" })
SendMessage(to: "perf-reviewer", message: { type: "shutdown_request" })
SendMessage(to: "test-reviewer", message: { type: "shutdown_request" })
TeamDelete()
```

---

## Pattern 2: Parallel Module Development (Throughput)

**Step 1 — Create team:**
```
TeamCreate(team_name: "feature-build", description: "Build [feature name]")
```

**Step 2 — Create tasks:**
```
TaskCreate(title: "API endpoints", description: "Implement [endpoints] in src/api/")
TaskCreate(title: "Frontend components", description: "Build [components] in src/frontend/")
TaskCreate(title: "Test suite", description: "Write tests in tests/")
```

**Step 3 — Spawn implementers** (general-purpose with plan approval for safety):

```
Agent(
  description: "API layer implementation",
  team_name: "feature-build",
  name: "api-dev",
  model: "sonnet",
  mode: "plan",
  prompt: "You own src/api/[feature]/ and src/db/migrations/ exclusively.
    Do not touch any other directory. Implement [endpoints].
    Contract: [request/response shapes].
    Present your implementation plan first and wait for approval.
    When done, mark your task as completed via TaskUpdate."
)
```

```
Agent(
  description: "Frontend implementation",
  team_name: "feature-build",
  name: "frontend-dev",
  model: "sonnet",
  mode: "plan",
  prompt: "You own src/frontend/[feature]/ exclusively.
    Do not touch any other directory. Build [components].
    Mock API using this contract: [shapes]. Follow existing patterns
    in src/frontend/settings/ for layout.
    Present your implementation plan first and wait for approval.
    When done, mark your task as completed via TaskUpdate."
)
```

```
Agent(
  description: "Test suite",
  team_name: "feature-build",
  name: "test-dev",
  model: "sonnet",
  prompt: "You own tests/[feature]/ exclusively.
    Do not touch any other directory. Cover [critical paths].
    Write unit + integration tests.
    When done, mark your task as completed via TaskUpdate."
)
```

---

## Pattern 3: Competing Hypotheses — Debugging (Perspective)

**Step 1 — Create team:**
```
TeamCreate(team_name: "bug-investigation", description: "Root cause analysis for [bug]")
```

**Step 2 — Spawn investigators** (Explore type since this is investigation):

```
Agent(
  description: "Investigate hypothesis A",
  team_name: "bug-investigation",
  name: "investigator-a",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "You are investigating the hypothesis: [e.g., Race condition in WebSocket handler].
    Gather EVIDENCE (logs, traces, code paths). Not speculation — evidence.
    Message other investigators via SendMessage when you find something that
    supports or refutes their theory. Rate your hypothesis:
    CONFIRMED / LIKELY / INCONCLUSIVE / REFUTED.
    When done, mark your task as completed via TaskUpdate."
)
```

(Repeat for hypotheses B, C, etc.)

---

## Pattern 4: Research & Implement (Throughput)

**Step 1 — Create team:**
```
TeamCreate(team_name: "research-build", description: "Research and implement [topic]")
```

**Step 2 — Spawn researcher first** (can run in background):
```
Agent(
  description: "Research [topic]",
  team_name: "research-build",
  name: "researcher",
  model: "sonnet",
  subagent_type: "Explore",
  prompt: "Investigate [topic/approaches] for [goal].
    Analyze the codebase: check [relevant dirs] for patterns.
    Document in docs/research/topic.md: pros/cons, recommendation,
    implementation outline, and key decisions.
    You own docs/research/ exclusively.
    When done, mark your task as completed via TaskUpdate
    and DM 'implementer' that research is ready."
)
```

**Step 3 — Spawn implementer after research completes:**
```
Agent(
  description: "Implement recommendation",
  team_name: "research-build",
  name: "implementer",
  model: "sonnet",
  mode: "plan",
  prompt: "Read the research at docs/research/topic.md and implement
    the recommended approach in [target directory].
    If anything is unclear, DM 'researcher' for clarification.
    You own [target directory] exclusively.
    Present your plan first and wait for approval.
    When done, mark your task as completed via TaskUpdate."
)
```

---

## Pattern 5: QA Swarm (Perspective)

**Step 1 — Create team:**
```
TeamCreate(team_name: "qa-swarm", description: "QA for [app] at [URL]")
```

**Step 2 — Spawn testers:**

```
Agent(
  description: "Functional testing",
  team_name: "qa-swarm",
  name: "functional-tester",
  model: "sonnet",
  prompt: "Test [core user flows] at [http://localhost:PORT].
    Document: what was tested, reproduction steps for issues,
    severity (blocker/major/minor/cosmetic), evidence.
    When done, mark your task as completed via TaskUpdate."
)
```

(Repeat for edge-case tester, accessibility tester, etc.)

---

## Lead Control Snippets

Append these instructions to any spawn prompt as needed:

**Coordinator-only Lead** (prompt convention, not system enforcement):
```
Do not write any code yourself.
Only coordinate, assign tasks, and synthesize results from teammates.
```

**Cost control:**
```
Use Sonnet for all teammates. Maximum [N] teammates.
Send shutdown_request to teammates immediately after they complete.
```

**Staged execution:**
```
Phase 1: Spawn [teammates A, B]. Wait for completion.
Phase 2: Based on Phase 1 results, spawn [teammates C, D] only if needed.
Do not spawn all teammates at once.
```

**Convergence forcing (Perspective teams):**
```
After all teammates report, identify areas of agreement and disagreement.
For disagreements, ask the relevant teammates to provide additional evidence.
Only synthesize when consensus is reached or evidence clearly favors one position.
```
