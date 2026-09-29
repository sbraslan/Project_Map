# ACH-T04 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-ACH-007`  
**Canonical test:** `ACH-T04`

> Handoff only. It does not authorize runtime execution.

## Static basis

`CAchievementSystem::OnLogin()` explicitly calls:
```cpp
OnVisitMap(player);
```

`OnVisitMap()` progresses `TYPE_EXPLORE` tasks by comparing the task's map-index restriction against `player->GetMapIndex()`.

Current `achievements.xml` contains **29 TYPE_EXPLORE task rows**.

The mapped normal warp/map-transition paths do not call `OnVisitMap()`. Therefore entering a target map during an existing session does not immediately evaluate exploration progress; the next login on that map does.

## Future live action

When runtime is explicitly unlocked:
1. choose an unfinished TYPE_EXPLORE target map from current XML;
2. log in outside that map;
3. record task progress;
4. warp/travel normally into the target map without relogging;
5. check Achievement progress immediately;
6. logout/login while remaining on the target map;
7. check progress again;
8. stop after evidence capture.

## Expected result

- immediately after normal map transition: no exploration progress;
- after relog on the target map: `OnLogin -> OnVisitMap` progresses the task.

## Result classification

**REPRODUCED:** transition alone does not progress, relog on target map does.  
**NOT REPRODUCED:** normal map transition invokes equivalent progression in deployed code.  
**INCONCLUSIVE:** wrong/already-complete target, map restriction mismatch, or deployment drift.

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
