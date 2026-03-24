---
name: orchestrating-agent-teams
description: >
  Orchestrate Claude Code Agent Teams (multi-agent swarms). Use when: (1) spawning
  or coordinating multiple Claude teammates, (2) designing team structures for parallel
  or perspective work, (3) the user asks for a "team", "swarm", "multi-agent", or
  "parallel agents" approach, (4) debugging stuck teammates, file conflicts, idle states,
  or team coordination issues, (5) evaluating whether a task needs a team vs. subagents
  vs. single agent, (6) the user wants to "divide the work", do a "multi-perspective
  review", or "parallelize" a task across agents. Also triggers for Spanish equivalents:
  "equipo de agentes", "trabajo en paralelo", "dividir el trabajo", "revisión
  multi-perspectiva", "enjambre de agentes". Even if the user doesn't say "team"
  explicitly, use this skill whenever the intent is clearly multi-agent coordination.
compatibility:
  tools: [TeamCreate, TeamDelete, Agent, SendMessage, TaskCreate, TaskUpdate, TaskList]
---

# Agent Teams Orchestration

Agent Teams let multiple Claude Code instances work together on a shared project —
but "together" doesn't always mean "in parallel." Sometimes the goal is speed
(divide the work), sometimes it's depth (multiply the perspectives). And sometimes
the right answer is not to use a team at all.

This skill covers when to use teams, when to use cheaper alternatives, how to
structure them, and how to govern them so they don't burn tokens without delivering
value. Read the Gatekeeper Rule before spawning anything.

## Prerequisites

1. Claude Code CLI installed (v2.x+)
2. The following tools must be available: `TeamCreate`, `TeamDelete`, `Agent`,
   `SendMessage`, `TaskCreate`, `TaskUpdate`, `TaskList`
3. If Agent Teams are behind a feature flag in your version, enable with
   `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json or shell

## Before You Spawn: The Gatekeeper Rule

**When a user asks for an agent team, do not spawn one automatically.** First,
evaluate whether the work actually requires a team. Agent Teams are the most
expensive tool in the toolkit (3-10x token cost). Most tasks don't need them.

Run this check silently before proceeding:

1. Can a single agent handle this in under 10 minutes? → **Single agent.** Don't team.
2. Is the work parallelizable but each piece is fire-and-forget? → **Subagents.**
   Cheaper (1.5-2x), no coordination overhead, the parent collects results.
3. Do workers need to coordinate, share evidence, or challenge each other? → **Agent Team.**
   Only now is the cost justified.

If the answer is subagents or single agent, tell the user: explain why a team is
overkill for their case, what you recommend instead, and the cost difference. Be
direct — "this doesn't need a team, here's why" is better than burning tokens to
be agreeable.

Read `references/decision-framework.md` for the full decision tree and the
architectural differences between subagents and teams.

## Why Teams: Two Paradigms

Agent Teams are not just "more agents going faster." There are two fundamentally
different reasons to spawn a team, and confusing them leads to waste.

**Throughput — divide to accelerate.** Split a large task into independent pieces so
multiple agents work simultaneously. Each agent does a different part of the whole.
The result is the sum of the parts, delivered faster. Use when the bottleneck is
time and the work has clear boundaries (different directories, layers, or modules).

**Perspective — multiply to see more.** Point multiple agents at the same problem,
each through a different lens. Every agent examines the same code, data, or situation
but from a distinct angle. The result is greater than the sum of the parts — cross-
pollination between viewpoints surfaces insights no single agent would reach, regardless
of how much time it had. Use when the bottleneck is not speed but blind spots.

The first question before spawning a team is not "can I parallelize this?" but
**"what do I need — to go faster, or to see what I'm missing?"**

| Paradigm | Core question | Agents work on | Result is |
|----------|--------------|----------------|-----------|
| Throughput | "How do I finish faster?" | Different parts of the whole | Sum of the parts |
| Perspective | "What am I not seeing?" | The same thing, different angles | Greater than the sum |

Both paradigms require task independence at the operational level (no shared file
edits, self-contained briefs). But the design intent — and therefore the team
structure, prompts, and synthesis — is fundamentally different.

Read `references/decision-framework.md` for the full decision matrix and cost estimation.

## Team Lifecycle — The Real Workflow

Agent Teams follow a formal lifecycle using specific tools. Understanding this
sequence is essential — skipping steps leads to orphaned teams or lost coordination.

```
1. TeamCreate        → Creates team + shared task list
2. TaskCreate        → Define tasks in the shared list
3. Agent             → Spawn teammates (with team_name + name params)
4. TaskUpdate        → Teammates claim and complete tasks
5. SendMessage       → Communicate (DM, broadcast, shutdown requests)
6. SendMessage       → shutdown_request to each teammate when done
7. TeamDelete        → Clean up team + task directories
```

**Creating the team:**
```
TeamCreate(team_name: "my-feature", description: "Build notification preferences")
```
This creates `~/.claude/teams/my-feature/` and `~/.claude/tasks/my-feature/`.

**Spawning a teammate:**
```
Agent(
  description: "API layer implementation",
  prompt: "...",           # The full brief (see Briefing Protocol)
  team_name: "my-feature", # Joins this team
  name: "api-dev",         # Addressable name for messaging
  model: "sonnet",         # Optional: sonnet for workers, opus for lead
  mode: "plan"             # Optional: requires plan approval before edits
)
```

**Shutting down teammates** (formal protocol — teammates can accept or reject):
```
SendMessage(
  to: "api-dev",
  message: { type: "shutdown_request", reason: "All tasks complete" }
)
```

**Cleaning up:**
```
TeamDelete()  # Fails if teammates are still active — shut them down first
```

## Briefing Protocol

The spawn prompt is the single most important factor in a team's success or failure.
Teammates start cold — they load CLAUDE.md and skills but NOT the Lead's conversation.
Everything they need to know must be in their spawn prompt. Think of it as writing a
job description: if you wouldn't hire someone with a brief that says "work on the
frontend," don't spawn a teammate with one either.

**Bad**: "Review the auth module"
**Good**: "Review src/auth/ for security vulnerabilities. Focus on token handling in jwt.ts
and session management in session.ts. App uses JWT in httpOnly cookies, 24h expiry.
Report findings with severity ratings (critical/high/medium/low) and suggested fixes."

Every spawn prompt must answer five questions:

1. **What is your role?** Name the lens or responsibility. "You are the security
   reviewer" or "You own the API layer." This anchors the teammate's identity and
   prevents scope drift.

2. **What is your scope?** Exact files, directories, or systems. Be explicit about
   boundaries: "Work exclusively in src/api/. Do not touch src/frontend/ or tests/."
   File ownership prevents conflicts — if two teammates can edit the same file,
   you've already lost.

3. **What context do you need?** Relevant architecture decisions, tech stack, business
   rules, constraints. The teammate has zero prior context. Anything you don't include,
   they'll either guess wrong or waste time discovering.

4. **What output do you expect?** Format, structure, level of detail. "Report findings
   as a markdown list with severity ratings and remediation steps" is actionable.
   "Let me know what you find" is not.

5. **What is your definition of done?** When should the teammate stop and mark complete?
   "Done when all files in src/payments/ have been reviewed and findings documented"
   is clear. Without this, teammates either stop too early or spiral indefinitely.

### Task Sizing

Right-size what you give each teammate. Too large and the teammate loses focus or
runs out of context window. Too small and the coordination overhead exceeds the work.

A well-sized task for a teammate is one that takes 3-15 minutes of focused work with
a clear deliverable. If a task would take a single agent under 3 minutes, it's overhead
to spawn a teammate for it. If it would take over 20 minutes, consider splitting it
into subtasks or breaking it across two teammates.

### File Ownership

This is the #1 source of team failures. Agent Teams have no file-level locking. If
two teammates edit the same file, one will overwrite the other's changes silently.

**Primary solution:** Always assign exclusive file/directory ownership in the spawn
prompt. Make it explicit and non-negotiable: "You own src/api/. No other teammate
will touch these files. Do not edit files outside your scope." The Lead should verify
there is zero overlap in file ownership across all teammates before spawning.

**Alternative — worktree isolation:** For cases where file overlap is hard to avoid,
spawn the teammate with `isolation: "worktree"`. This creates a temporary git worktree
so the agent works on an isolated copy of the repo. If the agent makes changes, the
worktree path and branch are returned in the result for manual merge. This eliminates
race conditions at the cost of requiring a merge step afterward.

### Agent Types for Teammates

The Agent tool supports different `subagent_type` values, each with different tool access:

- **general-purpose** (default): Full tool access including edit/write/bash. Use for
  implementation work.
- **Explore**: Read-only. Can search and read files but cannot edit. Use for research
  or investigation tasks.
- **Plan**: Read-only. Designs implementation plans. Use for architecture/planning tasks.

Match the agent type to the task. A review teammate doesn't need write access — use
Explore. An implementer needs full access — use general-purpose. This prevents
accidental scope creep (a reviewer accidentally "fixing" what they found).

### Supervision and Governance

Not every teammate needs the same level of oversight. Calibrate based on risk:

**Low risk (read-only tasks)**: Reviews, investigations, research. Spawn with
`subagent_type: "Explore"`. Let the teammate run independently.

**Medium risk (new code in isolated scope)**: Tests, docs, new modules with clear
contracts. Spawn with `mode: "plan"` to require plan approval. The teammate will
present a plan and wait for your explicit `plan_approval_response` via SendMessage
before making any edits. This is a formal API mechanism, not just a prompt instruction.

**High risk (changes to existing production code)**: Use `mode: "plan"` AND
`isolation: "worktree"`. Review the plan, approve, and then review the worktree
changes before merging.

**Plan Approval flow:**
1. Spawn teammate with `mode: "plan"`
2. Teammate works in plan mode — can read but not edit
3. Teammate calls ExitPlanMode → system sends you a `plan_approval_request`
4. You review and respond via SendMessage:
   ```
   SendMessage(to: "teammate-name", message: {
     type: "plan_approval_response",
     request_id: "...",    # from the request
     approve: true         # or false with feedback
   })
   ```
5. On approval, teammate exits plan mode and can implement

**Lead as coordinator, not implementer.** The Lead should coordinate, assign tasks,
monitor progress, and synthesize results — not write code itself. This is a prompt
convention, not a system enforcement: include "Do not write any code yourself. Only
coordinate, assign tasks, and synthesize results from teammates." in the initial
team setup. Without this, the Lead tends to "help" by implementing, and the team
loses its coordinator.

## Communication: SendMessage

Teammates communicate via the `SendMessage` tool. There are two modes:

**Direct message** — send to a specific teammate by name:
```
SendMessage(to: "researcher", message: "Check if the auth module uses JWT or sessions", summary: "Ask about auth approach")
```

**Broadcast** — send to all teammates at once:
```
SendMessage(to: "*", message: "Critical blocker found — stop all work", summary: "Critical blocker")
```

**Broadcast is expensive** — it sends a separate message to every teammate. Use it
only for critical announcements that genuinely affect everyone. Default to direct
messages for normal coordination.

How much communication you encourage depends on the paradigm:

| Mode | Throughput teams | Perspective teams |
|------|-----------------|-------------------|
| DM | Minimal — only for blockers | Heavy — cross-communication is where the value emerges |
| Broadcast | Rarely needed | Only for critical shared findings |

In Perspective teams, explicitly encourage DM use in the spawn prompt (e.g.,
"message other investigators when you find evidence that supports or refutes their
hypothesis"). Without this, agents default to silos and you lose the cross-pollination
that justifies the team.

Teammates discover each other by reading `~/.claude/teams/{team-name}/config.json`,
which lists all members with their names.

## Understanding Idle State

**Teammates go idle after every turn. This is normal, not an error.** When a
teammate's turn ends (they sent a message, completed a task, or are waiting for
input), the system marks them idle and sends a notification to the Lead.

Key points:
- Idle means "waiting for input," not "stuck" or "done"
- A teammate sending a message and then going idle is the normal flow
- Sending a message to an idle teammate wakes them up immediately
- Do not treat idle notifications as errors or try to "fix" them
- Only be concerned about idleness if a teammate has been idle for an extended
  period AND has unfinished tasks assigned to them

## Team Patterns

### Perspective patterns — multiply viewpoints

| Pattern | Teammates | Use When |
|---------|-----------|----------|
| Multi-Lens Review | 3-5 | Same code, different lenses (security, perf, tests) |
| Competing Hypotheses | 3-5 | Same problem, rival theories tested with evidence |
| QA Swarm | 3 | Same app, different quality angles (functional, edge, a11y) |

Perspective teams are most valuable when a single agent would converge on one
interpretation too quickly. The power comes from agents challenging each other's
findings — encourage cross-communication and evidence sharing.

### Throughput patterns — divide work

| Pattern | Teammates | Use When |
|---------|-----------|----------|
| Parallel Modules | 2-4 | Separate dirs/layers (frontend, backend, tests) |
| Research & Implement | 2 | One researches, one builds on findings |

Throughput teams are most valuable when work has clear boundaries and zero overlap.
The power comes from wall-clock time savings — keep agents isolated and focused.

Read `references/prompt-templates.md` for ready-to-use prompts for each pattern.

**Perspective examples:**
See `references/examples/multi-lens-review-session.md` for a 3-agent code review walkthrough.
See `references/examples/competing-hypotheses-business.md` for a 5-agent business investigation.

**Throughput examples:**
See `references/examples/parallel-modules-feature-build.md` for a 3-agent feature build.
See `references/examples/research-and-implement.md` for a 2-agent research→build workflow.

## Operating Rules

**Cost control:**
- Plan first in plan mode (cheap), then hand plan to team (expensive)
- Use Sonnet for teammates (`model: "sonnet"`), Opus for Lead
- Shut down teammates via `shutdown_request` immediately after completion
- Start with 2-3 teammates; add more only if parallelism genuinely helps
- Each teammate = full Claude instance; 5 teammates ~ 7-10x token cost
- Use `run_in_background: true` on Agent when you have independent work to do
  while a teammate runs — avoids blocking the Lead

**Monitoring:**
- Messages from teammates are delivered automatically — no need to poll
- Monitor task list: teammates sometimes forget to mark tasks completed
- Always instruct: "Wait for all teammates to complete before synthesizing"
- Idle notifications are informational — don't react unless a teammate is stuck

## Anti-Patterns (Quick Check)

Before spawning, scan this list. If any apply, stop and fix first.

1. Same-file editing across teammates → assign exclusive ownership or use `isolation: "worktree"`
2. Vague briefs → answer the 5 questions (see Briefing Protocol)
3. More than 5 teammates → reduce; coordination overhead exceeds gains
4. Sequential dependencies disguised as parallel work → use subagents or single agent
5. Spawning before planning → plan in plan mode first (see Gatekeeper Rule)
6. Forgetting TeamCreate → always create the team before spawning teammates
7. Forgetting shutdown → always send `shutdown_request` before `TeamDelete`

## Troubleshooting

Read `references/troubleshooting.md` when teammates appear stuck, idle state is
confusing, the Lead starts coding instead of coordinating, or task states fall
out of sync.
