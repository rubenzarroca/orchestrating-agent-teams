---
name: orchestrating-agent-teams
description: >
  Orchestrate Claude Code Agent Teams (multi-agent swarms). Use when: (1) spawning
  or coordinating multiple Claude teammates, (2) designing team structures for parallel
  or perspective work, (3) the user asks for a "team", "swarm", or "multi-agent"
  approach, (4) debugging stuck teammates, file conflicts, or team coordination issues,
  (5) evaluating whether a task needs a team vs. subagents vs. single agent.
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

1. Enable: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json or shell
2. Install tmux or iTerm2 for split-pane monitoring (required for 3+ teammates)
3. Claude Code >= v2026.2.x

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

Always assign exclusive file/directory ownership in the spawn prompt. Make it explicit
and non-negotiable: "You own src/api/. No other teammate will touch these files. Do
not edit files outside your scope." The Lead should verify there is zero overlap in
file ownership across all teammates before spawning.

### Supervision and Guidance

Not every teammate needs the same level of oversight. Calibrate based on risk:

**Low risk (read-only tasks)**: Reviews, investigations, research. Let the teammate
run independently. Check output when they mark complete.

**Medium risk (new code in isolated scope)**: Tests, docs, new modules with clear
contracts. Use plan approval for the approach, then let them execute.

**High risk (changes to existing production code)**: Require plan approval. Review
the plan carefully. Consider requiring the teammate to explain their changes before
marking complete. This is where the governance mechanisms from the next section earn
their cost.

## Coordination Primitives

Teammates share a **Task List** (state tracking) and a **Mailbox** (DM/broadcast
messaging). How much you use each depends on the paradigm:

| Primitive | Throughput teams | Perspective teams |
|-----------|-----------------|-------------------|
| Task List | Heavy — clear ownership, track progress per module | Moderate — track completion, but work is less divisible |
| Mailbox | Minimal — DM only for blockers | Heavy — cross-communication is where the value emerges |

Known limitation: teammates sometimes finish work but forget to mark tasks complete.
The Lead should check periodically and nudge.

In Perspective teams, explicitly encourage mailbox use in the spawn prompt (e.g.,
"message other investigators when you find evidence that supports or refutes their
hypothesis"). Without this, agents default to silos and you lose the cross-pollination
that justifies the team.

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

## Governance

Agent teams without governance are expensive chaos. These two mechanisms are
non-negotiable controls — the equivalent of code review and role separation
in an agile team.

**Plan Approval — the merge request before work starts.** For complex or risky
tasks, require the teammate to present a detailed plan of what they intend to do
before writing any code. The Lead (or the human) reviews and approves the plan.
Only after approval does the teammate execute. This is a merge request for AIs:
you review the intent before the implementation exists, catching misunderstandings
and scope creep early when they're cheap to fix. Use plan approval when the
teammate's task involves production code changes, architectural decisions, or any
work that would be costly to redo. For read-only tasks (reviews, investigations),
plan approval is optional — the risk of wrong execution is low.

To enforce plan approval, include in the spawn prompt:
```
Before writing any code, present a detailed plan of your approach:
what files you'll modify, what changes you'll make, and why.
Wait for explicit approval before proceeding with implementation.
```

**Delegate Mode — the Lead coordinates, never implements.** Activate with
Shift+Tab. This restricts the Lead to coordination only: assigning tasks,
monitoring progress, synthesizing results, and facilitating communication.
The Lead cannot write code. This prevents the most common failure mode: the
Lead gets "helpful," starts implementing instead of orchestrating, and the
team loses its coordinator. The role of the manager is to facilitate, not to
do the team's work. A Lead that codes is a Lead that isn't watching the task
list, isn't catching stuck teammates, and isn't synthesizing findings.

Always include in the initial team prompt:
```
Activate delegate mode. Do not write any code yourself.
Only coordinate, assign tasks, and synthesize results from teammates.
```

These two mechanisms work together. Plan approval gives the Lead visibility
into what each teammate will do before they do it. Delegate mode ensures the
Lead stays focused on that oversight instead of getting pulled into implementation.
Together they create a governance loop: plan → approve → execute → report →
synthesize — similar to sprint planning → review in agile.

## Operating Rules

**Monitoring:**
- Split-pane view (tmux/iTerm2) shows all teammate activity simultaneously
- Monitor task list: teammates sometimes forget to mark tasks completed
- Always instruct: "Wait for all teammates to complete before synthesizing"

**Navigation:**
- Shift+Up/Down to switch between teammate sessions
- Enter to view, Escape to interrupt

**Cost control:**
- Plan first in plan mode (cheap), then hand plan to team (expensive)
- Use Sonnet for teammates, Opus for Lead
- Kill idle teammates immediately after completion
- Start with 2-3 teammates; add more only if parallelism genuinely helps
- Each teammate = full Claude instance; 5 teammates ≈ 7-10x token cost

## Anti-Patterns (Quick Check)

Before spawning, scan this list. If any apply, stop and fix first.

1. Same-file editing across teammates → assign exclusive ownership (see Briefing > File Ownership)
2. Vague briefs → answer the 5 questions (see Briefing Protocol)
3. More than 5 teammates → reduce; coordination overhead exceeds gains
4. Sequential dependencies disguised as parallel work → use subagents or single agent
5. Spawning before planning → plan in plan mode first (see Gatekeeper Rule)

## Troubleshooting

Read `references/troubleshooting.md` when teammates appear stuck, the Lead starts
coding instead of coordinating, or task states fall out of sync.
