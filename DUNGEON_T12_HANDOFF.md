# DUNGEON-T12 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution:** LOCKED / NOT RUN  
**Bug:** `DINFO::BUG-DUNGEON-012`

## Static basis
Current QUEST-backed dungeon blocks contain no explicit `COOLDOWN` directive, so `pSDungeonData->dwCooldown` remains 0.

Current expired branch:
```cpp
uint32_t dwRemainSec =
    (dwFlagValue + pSDungeonData->dwCooldown) - get_global_time();

if (dwRemainSec > 0)
    dwCooldown = dwRemainSec;
```

When the flag is 0 or otherwise expired, the mathematical result is negative and is assigned to `uint32_t`, wrapping to a large positive value.

## Future live action
After T09/T11 sequencing permits and runtime is explicitly unlocked:
1. use a QUEST-backed dungeon with unset/expired cooldown state;
2. open Dungeon Info normally;
3. observe the shown cooldown;
4. capture any abnormally huge timer plus client/server logs;
5. stop before any fix or state mutation.

## Reproduced
Mark REPRODUCED when an unset/expired state is shown as a very large positive cooldown rather than available/zero.

## Inconclusive
Use when the selected flag is still active, deployment source/config differs, or another error prevents observation.

No config edit, packet craft or forced DB mutation is authorized by this handoff.
