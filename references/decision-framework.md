# Decision Framework: Teams vs. Subagents vs. Single Agent

## First Question: What Do You Need?

```
What's the bottleneck?
├── SPEED (I need to finish faster)
│   └── Is the work divisible into independent pieces?
│       ├── NO → Single Agent
│       └── YES
│           ├── Do pieces need coordination? → Agent Team (Throughput)
│           └── Fire-and-forget is fine? → Subagents (cheaper)
│
├── PERSPECTIVE (I need to see what I'm missing)
│   └── Can I define distinct, non-overlapping lenses?
│       ├── NO → Single Agent (clarify the angles first)
│       └── YES → Agent Team (Perspective)
│
└── UNSURE → Start with a single agent in plan mode.
    The plan will reveal whether you need speed, perspective, or both.
```

## Operational Guardrails (both paradigms)

Regardless of paradigm, these rules apply before spawning:

```
Do workers edit the same files?
├── YES → Single Agent (no file locking in Agent Teams)
└── NO → Safe to proceed with team or subagents
```

## Subagents vs. Agent Teams: The Architectural Difference

These are not two flavors of the same thing. They are fundamentally different
coordination models, like the difference between a manager delegating tasks and
a team of peers collaborating.

**Subagents are vertical — delegation with a boss.** The parent agent spawns
workers, each receives a brief, executes independently, and reports back. Workers
never see each other. The parent collects all reports and synthesizes. Think of it
as a manager sending three people to do three separate errands — they don't need
to talk to each other, the manager just collects the results. Cost: 1.5-2x.
Session resumption works normally.

**Agent Teams are horizontal — peers with a coordinator.** Teammates are fully
independent Claude instances with their own context windows. They share a task
list and a mailbox. They can message each other directly, challenge each other's
findings, and self-organize. The Lead coordinates but doesn't micromanage. Think
of it as a war room where specialists work the same problem from different angles
and talk to each other. Cost: 3-10x. Session resumption is broken (experimental).

The key question is: **do the workers need to interact with each other?**

If the answer is no — each worker does their thing and reports back — subagents
are cheaper and sufficient. If the answer is yes — workers need to share evidence,
challenge assumptions, or coordinate interfaces — you need a team.

A common mistake is using Agent Teams when subagents would do. Running three
independent linting tasks? Subagents. Generating docs for three separate modules?
Subagents. The moment you catch yourself thinking "but they don't really need to
talk to each other," that's your signal to downgrade to subagents and save 2-5x
in token cost.

## Comparison Matrix

| Dimension          | Single Agent    | Subagents            | Agent Team                  |
|--------------------|-----------------|----------------------|-----------------------------|
| Context            | One window      | Report to parent     | Fully independent windows   |
| Communication      | N/A             | Up only (→ parent)   | Any direction (broadcast/DM)|
| Coordination       | Manual          | Parent micromanages  | Self-managed via task list  |
| File safety        | No conflicts    | No conflicts         | Race condition risk         |
| Token cost         | 1x              | 1.5-2x               | 3-10x                      |
| Session resumption | Full            | Full                 | Broken (experimental)       |

## When Perspective Teams Excel

The investment pays off when a single viewpoint is the bottleneck:

- Code review across lenses (security + perf + tests + architecture)
- Root cause investigation with rival hypotheses that challenge each other
- QA across quality dimensions (functional, edge cases, accessibility)
- Architecture decisions where trade-offs need adversarial evaluation

Key signal: if you catch yourself thinking "a single agent would probably
miss something here," you need a Perspective team.

## When Throughput Teams Excel

The investment pays off when wall-clock time is the bottleneck:

- Large feature builds with clear directory boundaries
- Cross-module refactoring where layers are independent
- Parallel test suites across different environments
- Research + implementation where research unblocks building

Key signal: if you catch yourself thinking "this would take one agent
too long, but each piece is straightforward," you need a Throughput team.

## When Teams Are Wasteful

- Sequential pipelines (parse → transform → validate → deploy)
- Single-file intensive refactoring
- Simple "run these 3 scripts" delegation (subagents are cheaper)
- Tasks a single agent can finish in under 5 minutes
- Unclear scope (if you can't define task or lens boundaries, don't team)

## Cost Estimation

| Team Size | Cost Multiplier | Typical Use Case                        |
|-----------|-----------------|-----------------------------------------|
| 2         | 2.5-3x          | Research + Implement (Throughput)        |
| 3         | 4-5x            | Multi-lens review (Perspective)         |
| 5         | 7-10x           | Competing hypotheses (Perspective)      |
| 5+        | 10x+            | Rarely justified                        |

Multiplier includes coordination overhead (task list, mailbox, Lead synthesis).
Perspective teams tend toward the higher end because cross-communication is essential.

## Pre-Team Checklist

Before spawning, verify:

- [ ] Paradigm identified: Throughput or Perspective (or hybrid)
- [ ] Work divides into independent tracks with zero file overlap
- [ ] Each track has a self-contained brief (scope + context + output format)
- [ ] For Perspective: each agent has a distinct, named lens
- [ ] For Perspective: cross-communication is explicitly enabled in the brief
- [ ] Plan is solid (done in plan mode first)
- [ ] Expected value justifies the token cost
- [ ] tmux/iTerm2 ready for monitoring
- [ ] Model allocation decided (Opus Lead, Sonnet teammates)
