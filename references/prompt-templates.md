# Agent Team Prompt Templates

Copy-paste prompts for the Team Lead. Replace `[bracketed sections]` with your specifics.

---

## Pattern 1: Multi-Lens Code Review (Perspective)

```
Create an agent team to review [PR #XXX / module at path/to/module].

Spawn 3 reviewers:
1. Security reviewer: Focus on [auth, input validation, injection, XSS].
   Check [src/auth/, src/api/middleware/]. Report with severity ratings.
2. Performance reviewer: Audit [DB queries, response times, memory].
   Check [src/db/, src/services/]. Flag N+1 queries and unindexed lookups.
3. Test coverage reviewer: Validate coverage for [login, payment, export].
   Check [tests/]. Report untested edge cases and missing integration tests.

Each reviewer reports independently.
Wait for all teammates to complete their tasks before proceeding.
```

## Pattern 2: Parallel Module Development (Throughput)

```
Create an agent team to build [feature name].

Spawn teammates by layer:
1. API teammate: Implement [endpoints] in [src/api/]. Contract: [request/response shapes].
2. Frontend teammate: Build [components] in [src/frontend/]. Mock API until ready.
   Same contract as API teammate.
3. Test teammate: Write [unit + integration tests] in [tests/]. Cover [critical paths].

Each teammate owns their directory exclusively. No cross-directory edits.
Use Sonnet for all teammates.
Wait for all teammates to complete their tasks before proceeding.
```

## Pattern 3: Competing Hypotheses — Debugging (Perspective)

```
[Describe bug: symptoms, when it happens, what you've tried].

Create an agent team of [3-5] investigators. Assign each a hypothesis:
1. Hypothesis A: [e.g., Race condition in WebSocket handler]
2. Hypothesis B: [e.g., Memory leak in cache layer]
3. Hypothesis C: [e.g., Auth token expiry not handled]

Rules:
- Each investigator gathers EVIDENCE (logs, traces, reproduction steps).
- Investigators message each other to share findings and challenge theories.
- Only update shared findings when evidence supports/refutes a hypothesis.
- Converge on root cause through evidence, not speculation.

Wait for all teammates to complete their tasks before proceeding.
```

## Pattern 4: Research & Implement (Throughput)

```
Create an agent team with 2 teammates:

1. Researcher: Investigate [topic/approaches] for [goal].
   Document in [docs/research/topic.md]: pros/cons, recommendation, implementation outline.

2. Implementer: Wait for Researcher to complete. Then implement recommended approach
   in [target directory]. Ask Researcher via DM for clarification if needed.

Researcher goes first. Implementer starts only after research is marked complete.
Wait for all teammates to complete their tasks before proceeding.
```

## Pattern 5: QA Swarm (Perspective)

```
The application is running at [http://localhost:PORT].

Create an agent team for QA:
1. Functional tester: Test [core user flows]. Verify expected outputs.
2. Edge case tester: Test [boundary conditions, error states, empty inputs, max limits].
3. Accessibility tester: Check [ARIA labels, keyboard nav, contrast, screen reader].

Each tester documents: what was tested, reproduction steps for issues,
severity (blocker/major/minor/cosmetic), evidence.

Wait for all teammates to complete their tasks before proceeding.
```

---

## Lead Control Snippets

Append these to any team prompt as needed:

**Force delegation:**
```
Activate delegate mode. Do not write any code yourself.
Only coordinate, assign tasks, and synthesize results from teammates.
```

**Cost control:**
```
Use Sonnet for all teammates. Maximum [N] teammates.
Shut down teammates immediately after completion.
```

**Staged execution:**
```
Phase 1: Spawn [teammates A, B]. Wait for completion.
Phase 2: Based on Phase 1 results, spawn [teammates C, D] only if needed.
Do not spawn all teammates at once.
```

**Convergence forcing:**
```
After all teammates report, identify areas of agreement and disagreement.
For disagreements, ask the relevant teammates to provide additional evidence.
Only synthesize when consensus is reached or evidence clearly favors one position.
```
