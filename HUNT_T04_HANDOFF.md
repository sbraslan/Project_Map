# HUNT-T04 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-HUNT-004`  
**Canonical test:** `HUNT-T04`  
**Order:** Hunting normal-path candidate after Dungeon Info and Ticket normal-path gates.

> Handoff only. It does not authorize runtime execution.

## Static basis

Active build:
```cpp
#define HUNTING_MISSION_COUNT 90
```

Tables:
```cpp
THuntingMissions[HUNTING_MISSION_COUNT + 1][2][2]
THuntingRewardItem[HUNTING_MISSION_COUNT + 1][2][4][2]
```
Valid mission indices are therefore 0..90.

Reward claim ends unconditionally with:
```cpp
SetQuestFlag("hunting_system.level",
    GetQuestFlag("hunting_system.level") + 1);
```

A legitimate mission-90 reward claim therefore stores level 91.

When Hunting is next opened:
- if character level < hunting level, server sends the zero-data main packet;
- if character level >= hunting level, `OpenHuntingWindowSelect()` is called;
- that function indexes:
  - `THuntingMissions[actLevel][type]`
  - `THuntingRewardItem[actLevel][type][race]`

At hunting level 91 this is outside the declared 0..90 table domain.

## Critical precondition

The observing character must be **character level >= 91** after the mission-90 claim.

A character below 91 will take the safe-looking zero-data branch and will not reach the level-91 table access. Such a run is not a valid reproduction attempt.

## Future live action

When runtime is explicitly unlocked and prior gates are resolved:

1. Use an isolated/dev character that has legitimately reached Hunting mission 90.
2. Ensure character level is >=91.
3. Complete mission 90 through the ordinary Hunting flow.
4. Claim mission-90 reward normally.
5. Record that `hunting_system.level` advances to 91, using existing safe observability only.
6. Open the Hunting window normally.
7. Capture GAME/client behavior and logs.
8. Stop immediately after the first observation.

## Expected result

The next open reaches `OpenHuntingWindowSelect()` with `actLevel = 91` and attempts out-of-bounds reads from the mission/reward tables.

Observable effects may vary by build/runtime:
- crash;
- sanitizer/debug report;
- corrupted/garbage mission data;
- apparently benign output due to undefined adjacent memory.

A lack of crash alone does not mean NOT REPRODUCED if instrumentation proves the index 91 table access occurred.

## Result classification

### REPRODUCED
Use when level 91 is stored after legitimate mission-90 claim and the next eligible open reaches/observes the index-91 table access.

### NOT REPRODUCED
Use only if a mapped-equivalent deployment contains a real terminal guard/clamp and prevents level-91 table access.

### INCONCLUSIVE
Use when:
- character level is below 91;
- mission-90 state was not reached legitimately;
- deployment differs materially;
- another error prevents determining whether index 91 was accessed;
- no suitable debug/sanitizer evidence exists and behavior is ambiguous.

## Stop conditions

Stop immediately on:
- client crash/disconnect;
- GAME/core crash;
- sanitizer OOB report;
- corrupted Hunting packet/UI data;
- any need to edit source, table data, quest flags or packets to proceed.

Do not continue directly into HUNT-T01/T02/T03/T05+.

## Evidence template

- Test: `HUNT-T04`
- Bug: `BUG-HUNT-004`
- Result:
- Date/time:
- Client/server build:
- Character level:
- Hunting level before claim:
- Hunting level after claim:
- Mission 90 completed normally: yes/no
- Hunting window reopened: yes/no
- Observed packet/UI behavior:
- GAME/client errors:
- ASan/debug evidence:
- Deployment drift:
- Evidence reference:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
