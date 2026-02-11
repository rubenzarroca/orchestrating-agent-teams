# Orchestrating Agent Teams

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) for orchestrating multi-agent teams (swarms) in Claude Code.

## What it does

This skill teaches Claude Code **when and how** to use Agent Teams — multiple Claude instances collaborating on a shared project. It covers:

- **The Gatekeeper Rule** — evaluate whether a task actually needs a team before spawning one (most don't)
- **Two paradigms** — Throughput (divide work to go faster) vs. Perspective (multiply viewpoints to see more)
- **Briefing Protocol** — how to write spawn prompts that don't waste tokens
- **Coordination** — shared task lists and mailbox messaging between agents
- **Governance** — plan approval and delegate mode to keep teams on track
- **Cost control** — teams are 3-10x more expensive; the skill enforces discipline

## Team Patterns

### Perspective patterns (multiply viewpoints)

| Pattern | Agents | Use When |
|---------|--------|----------|
| Multi-Lens Review | 3-5 | Same code, different lenses (security, perf, tests) |
| Competing Hypotheses | 3-5 | Same problem, rival theories tested with evidence |
| QA Swarm | 3 | Same app, different quality angles |

### Throughput patterns (divide work)

| Pattern | Agents | Use When |
|---------|--------|----------|
| Parallel Modules | 2-4 | Separate dirs/layers (frontend, backend, tests) |
| Research & Implement | 2 | One researches, one builds on findings |

## Installation

Copy the `orchestrating-agent-teams` folder into your Claude Code skills directory:

```
~/.claude/skills/orchestrating-agent-teams/
```

Or clone this repo directly:

```bash
git clone https://github.com/rubenzarroca/orchestrating-agent-teams.git ~/.claude/skills/orchestrating-agent-teams
```

## Prerequisites

1. Enable agent teams: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json or shell
2. Install tmux or iTerm2 for split-pane monitoring (recommended for 3+ teammates)
3. Claude Code >= v2026.2.x

## Structure

```
orchestrating-agent-teams/
├── SKILL.md                 # Main skill definition
├── examples/
│   ├── competing-hypotheses-business.md
│   ├── multi-lens-review-session.md
│   ├── parallel-modules-feature-build.md
│   └── research-and-implement.md
└── references/
    ├── decision-framework.md
    ├── prompt-templates.md
    └── troubleshooting.md
```

## License

MIT
