# DUNGEON-T11 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution:** LOCKED / NOT RUN  
**Bug:** `DINFO::BUG-DUNGEON-011`

## Static basis
Current `dungeon_info.txt` documents `QUEST <quest> <flag> <type>` as `0=PC, 1=GLOBAL`.

Current parser does not implement that numeric contract:
```cpp
if (strcmp(szValue3, "GLOBAL") == 0)
    byType = QUEST_FLAG_GLOBAL;
else
    byType = QUEST_FLAG_PC;
```

Current Blue Dragon row:
```
QUEST dragon_lair_access dragon_lair_time 1
```

The active quest implementation reads/writes `dragon_lair_time` through:
- `game.get_event_flag("dragon_lair_time")`
- `game.set_event_flag("dragon_lair_time", ...)`

Therefore Dungeon Info reads a per-character quest flag while the dungeon logic uses a global event flag.

## Future live action
After T09 is resolved and runtime is explicitly unlocked:
1. open Dungeon Info normally;
2. select Blue Dragon / map 208;
3. note the displayed cooldown/status;
4. compare it to the actual global `dragon_lair_time` state from normal server/admin observability;
5. capture evidence and stop.

## Reproduced
Mark REPRODUCED when Dungeon Info behavior reflects the PC flag path and disagrees with the actual global event-flag state.

## Inconclusive
Use when deployment config/source differs, global state cannot be observed safely, or another error blocks the comparison.

No source/config/DB modification is authorized by this handoff.
