# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Mining / Pickaxe mapping in progress  
**Status:** 30 STATIC COMPLETE / MINING ACTIVE / 3 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Mining / Pickaxe  
**System:** `systems/mining.md`  
**Bugs:** `bugs/mining.md`  
**Tests:** `tests/mining.md`  
**Last completed subsystem:** Fishing Renewal  
**Effective completed/readiness-covered subsystems:** 30  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified Mining findings
- `BUG-MINE-001` — current NPC mining quest calls pickaxe refine only at `socket0 == value2`, while C++ accepts only `socket0 > value2`; the normal upgrade path is unreachable.
- `BUG-MINE-002` — same-character warp does not cancel active mining; completion can use the original vein and drop ore at the destination map.
- `BUG-MINE-003` — death does not cancel active mining; the completion event can still reward/practice while the character is dead.

## Closed/held observation
`OreRefine()` consumes 100 raw ore before its internal Yang check, but the current `guild_building_melt.quest` pre-checks the matching guild/empire-adjusted fee. Keep this as a candidate pending quest-yield/state analysis; do not count it as verified yet.

## Exact next work
1. Audit ore-refine quest yield/transaction boundaries and catalyst lifetime.
2. Map pickaxe proto Value0..Value4 / RefinedVnum semantics and current grade data.
3. Audit `ENABLE_MINING_EVENT` modifiers/rewards and mining-specific event hooks.
4. Audit SKILL_MINING progression and Battle Pass/Achievement integration.
5. Map client click/dig animation to server trust boundaries.
6. Close Battle Field ownership and multiplayer ore-pickup behavior.
7. Audit cross-core/channel warp behavior separately from same-process warp.
8. Promote only verified reachable findings.

Do not execute `MINE-T01..MINE-T03`. Global first live runtime gate remains `DUNGEON-T09`.
