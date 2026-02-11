# Troubleshooting Agent Teams

Common failure modes and recovery procedures.

---

## Problem: Lead Starts Coding Instead of Delegating

**Symptoms**: The Lead writes implementation code directly instead of assigning to teammates.

**Cause**: Without explicit instruction, the Lead defaults to "helpful assistant" mode and tries
to solve problems itself rather than coordinating.

**Fix**: Interrupt the Lead and send:
```
Stop. You are the coordinator, not an implementer.
Activate delegate mode (Shift+Tab). Assign this work to a teammate.
Do not write any code yourself for the rest of this session.
```

**Prevention**: Always include "Activate delegate mode" in the initial team prompt.

---

## Problem: Teammate Stuck — Task Not Marked Complete

**Symptoms**: Task list shows "in progress" but the teammate has stopped producing output.
The Lead is waiting and won't proceed.

**Cause**: Teammates sometimes finish work but forget to update the shared task list.
This is a known experimental limitation.

**Fix**:
1. Navigate to the teammate's session (Shift+Up/Down)
2. Check if the work is actually done
3. If done, tell the teammate: "Mark your current task as completed"
4. If the teammate is unresponsive, tell the Lead: "Task X is complete. Proceed with synthesis."

---

## Problem: File Conflicts Between Teammates

**Symptoms**: One teammate's changes overwrite another's. Build breaks after merge.

**Cause**: Two teammates edited the same file. There is no file-level locking in Agent Teams.

**Fix**:
1. Identify which teammate's changes are correct
2. Have that teammate re-apply their changes
3. Tell the other teammate to work in a different file

**Prevention**: Always divide work by directory, never by function within the same file.
In the team prompt, explicitly state: "Each teammate owns their directory exclusively.
No cross-directory edits."

---

## Problem: Teammates Duplicate Work

**Symptoms**: Two teammates produce overlapping implementations or investigate the same area.

**Cause**: Vague spawn briefs with unclear scope boundaries.

**Fix**: Tell the Lead to reassign with narrower scope:
```
Teammate A: you own [specific files/dirs]. Do not touch anything outside this scope.
Teammate B: you own [different files/dirs]. Do not touch anything outside this scope.
```

**Prevention**: Spawn briefs must include explicit file/directory ownership.

---

## Problem: Lead Synthesizes Before Teammates Finish

**Symptoms**: Lead produces a summary while teammates are still working. Final synthesis
is incomplete or ignores findings from slower teammates.

**Cause**: The Lead gets "impatient" — a known behavioral pattern where it proceeds
before all inputs are ready.

**Fix**: Interrupt and send:
```
Stop. Teammates [X, Y] have not completed their tasks yet.
Wait for ALL teammates to mark their tasks as completed before synthesizing.
Do not proceed until the task list shows all tasks completed.
```

**Prevention**: Always include "Wait for all teammates to complete their tasks before
proceeding" as the last line of the team prompt.

---

## Problem: Session Resume Fails

**Symptoms**: After `/resume`, the Lead tries to message teammates that no longer exist.

**Cause**: `/resume` and `/rewind` do not restore in-process teammates. This is a
known limitation of the experimental feature.

**Fix**:
```
Your previous teammates no longer exist after session resume.
Spawn new teammates to continue the remaining work.
Here is the current state: [describe what was completed and what remains].
```

---

## Problem: Token Budget Exploding

**Symptoms**: Session costs far exceed expectations. Teammates run much longer than planned.

**Diagnosis checklist**:
1. Are teammates idle but still alive? → Shut them down
2. Did you spawn more than 3 teammates? → Consider reducing
3. Are teammates using Opus? → Switch to Sonnet for execution work
4. Did you skip plan mode? → The plan is your cost checkpoint

**Cost rule of thumb**: If the total team work would take a single agent < 15 minutes,
the coordination overhead makes teams more expensive than sequential work.
