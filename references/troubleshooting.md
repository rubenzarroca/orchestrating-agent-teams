# Troubleshooting Agent Teams

Common failure modes and recovery procedures.

---

## Problem: Lead Starts Coding Instead of Delegating

**Symptoms**: The Lead writes implementation code directly instead of assigning to teammates.

**Cause**: Without explicit instruction, the Lead defaults to "helpful assistant" mode and tries
to solve problems itself rather than coordinating. There is no system-level enforcement of a
"delegate-only" mode — it's a prompt convention.

**Fix**: Interrupt the Lead and send:
```
Stop. You are the coordinator, not an implementer.
Do not write any code yourself. Assign this work to a teammate.
```

**Prevention**: Always include in the initial team setup prompt:
"Do not write any code yourself. Only coordinate, assign tasks, and synthesize results
from teammates."

---

## Problem: Teammate Idle — Doesn't Mean Stuck

**Symptoms**: The system sends idle notifications for a teammate. The Lead panics or
tries to "fix" the idle state.

**Cause**: Teammates go idle after every turn — this is completely normal. Idle means
"waiting for input," not "broken" or "done." The system sends an idle notification
automatically whenever a teammate's turn ends.

**Not a problem if**: The teammate just sent a message or completed a task. They're
simply waiting for a response or new work.

**Actually a problem if**: The teammate has been idle for an extended period AND has
unfinished tasks assigned AND hasn't sent any messages. In this case, send them a
direct message to check status:
```
SendMessage(to: "teammate-name", message: "Status check: are you blocked on anything?")
```

---

## Problem: Teammate Stuck — Task Not Marked Complete

**Symptoms**: Task list shows "in progress" but the teammate has stopped producing output
and is idle. The Lead is waiting and won't proceed.

**Cause**: Teammates sometimes finish work but forget to update the shared task list.

**Fix**:
1. Send a message to the teammate:
   ```
   SendMessage(to: "teammate-name", message: "Your work looks complete. Please mark your task as completed via TaskUpdate.")
   ```
2. If the teammate is unresponsive after the message, the Lead can mark the task
   complete directly via TaskUpdate and proceed with synthesis.

---

## Problem: File Conflicts Between Teammates

**Symptoms**: One teammate's changes overwrite another's. Build breaks after integration.

**Cause**: Two teammates edited the same file. There is no file-level locking in Agent Teams.

**Fix**:
1. Identify which teammate's changes are correct
2. Have that teammate re-apply their changes
3. Tell the other teammate to work in a different file

**Prevention (two options)**:
- **Directory ownership**: Divide work by directory, never by function within the same file.
  In the spawn prompt: "You own src/api/ exclusively. No cross-directory edits."
- **Worktree isolation**: Spawn with `isolation: "worktree"` for any teammate that might
  touch overlapping files. Changes land on a separate branch for manual merge.

---

## Problem: Teammates Duplicate Work

**Symptoms**: Two teammates produce overlapping implementations or investigate the same area.

**Cause**: Vague spawn briefs with unclear scope boundaries.

**Fix**: Send messages to reassign with narrower scope:
```
SendMessage(to: "teammate-a", message: "Narrow your scope to [specific files/dirs] only.")
SendMessage(to: "teammate-b", message: "Narrow your scope to [different files/dirs] only.")
```

**Prevention**: Spawn briefs must include explicit file/directory ownership.

---

## Problem: Lead Synthesizes Before Teammates Finish

**Symptoms**: Lead produces a summary while teammates are still working. Final synthesis
is incomplete or ignores findings from slower teammates.

**Cause**: The Lead gets "impatient" — a known behavioral pattern where it proceeds
before all inputs are ready.

**Fix**: Send a message to the Lead (or remind yourself):
```
Stop. Teammates [X, Y] have not completed their tasks yet.
Wait for ALL teammates to mark their tasks as completed before synthesizing.
Do not proceed until the task list shows all tasks completed.
```

**Prevention**: Always include "Wait for all teammates to complete their tasks before
proceeding" as the last line of the team prompt.

---

## Problem: Shutdown Rejected

**Symptoms**: You send a `shutdown_request` but the teammate rejects it.

**Cause**: Teammates can reject shutdown if they believe they still have work to do.
The `shutdown_response` will include `approve: false` and a reason.

**Fix**: Read the rejection reason. If the teammate genuinely has remaining work,
let them finish. If they're confused, send a clarifying message:
```
SendMessage(to: "teammate-name", message: "All tasks are complete. Please accept the shutdown.")
```
Then resend the shutdown request.

---

## Problem: TeamDelete Fails

**Symptoms**: `TeamDelete()` returns an error about active members.

**Cause**: You must shut down all teammates before deleting the team.

**Fix**: Send `shutdown_request` to each active teammate, wait for acceptance,
then call TeamDelete again.

---

## Problem: Token Budget Exploding

**Symptoms**: Session costs far exceed expectations. Teammates run much longer than planned.

**Diagnosis checklist**:
1. Are teammates idle but still alive? → Send `shutdown_request`
2. Did you spawn more than 3 teammates? → Consider reducing
3. Are teammates using Opus? → Switch to Sonnet (`model: "sonnet"`) for execution work
4. Did you skip plan mode? → The plan is your cost checkpoint
5. Are you broadcasting when DMs would suffice? → Each broadcast costs N messages

**Cost rule of thumb**: If the total team work would take a single agent < 15 minutes,
the coordination overhead makes teams more expensive than sequential work.
