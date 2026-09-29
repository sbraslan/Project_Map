# Mining / Pickaxe System

**Status:** MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Runtime:** EXECUTION LOCKED  
**Started:** 2026-09-29

## Scope
Primary server roots:
- `game/src/mining.cpp`
- `game/src/mining.h`
- `game/src/char.cpp::mining/mining_cancel/mining_take`
- `game/src/questlua_global.cpp::_refine_pick`
- `game/src/questlua_pc.cpp::pc_mining/pc_diamond_refine/pc_ore_refine`

Current game/config roots:
- `share/locale/europe/quest/n_npc/mining.quest`
- `share/locale/europe/quest/quest_list`
- current pickaxe names VNUM 29101..29110
- `ENABLE_PICKAXE_RENEWAL` active

## Normal mining flow
vein click / quest `pc.mining`
-> `CHARACTER::mining(load)`
-> same-map + <=1000 distance check
-> valid ore race check
-> equipped ITEM_PICK + subtype 0 check
-> DIG_MOTION broadcast
-> `CreateMiningEvent` delayed 10..30 seconds
-> event resolves player/load by PID/VID
-> current equipped pick is checked
-> current mining skill + current pick grade determine ore chance
-> `OreDrop`
-> ground item ownership
-> `PracticePick`.

Normal movement calls `OnMove()`, which calls `mining_cancel()`.

## Pickaxe refinement flow
Current deployed quest:
`n_npc/mining.quest`
-> NPC 20015 take
-> only when pick VNUM 29101..<29110 and socket0 == value2
-> `__refine_pick(item.get_cell())`
-> `questlua_global::_refine_pick`
-> `mining::RealRefinePick`
-> `Pick_Refinable`.

`ENABLE_PICKAXE_RENEWAL` means failed refine retains grade and subtracts 10% mastery.

## BUG-MINE-001 — quest and C++ pickaxe-refine thresholds are mutually incompatible
Current quest allows refine only when:
`item.get_socket(0) == item.get_value(2)`.

Server `Pick_Refinable` returns false when:
`Pick_GetCurExp(item) <= Pick_GetMaxExp(item)`.

Therefore the exact quest-approved state (`cur == max`) is rejected by C++.

If practice later pushes mastery above max, the quest's first take branch (`socket0 != value2`) handles the item and does not invoke `__refine_pick`.

The normal current quest path therefore has no mastery value that satisfies both layers.

Promoted as `BUG-MINE-001`.

## BUG-MINE-002 — mining event survives death/warp without completion revalidation
Movement cancellation is implemented only via `OnMove()->mining_cancel()`.

`Dead()` does not cancel `m_pkMiningEvent`.
`CanWarp()` does not consider `m_pkMiningEvent`.
`WarpSet()` calls `Stop()`, but `Stop()` does not invoke `OnMove()` or `mining_cancel()`.

At delayed completion, `mining_event` does not check:
- `IsDead()`;
- current map equality with the original vein;
- current distance to the vein.

If the original vein still exists and a pick is equipped, the normal ore roll continues. `OreDrop` drops the ore at the player's current map/current coordinates.

Consequences:
- a player can complete a mining roll while dead;
- a same-character warp can carry the pending mining result to another map and drop ore at the destination.

Promoted as `BUG-MINE-002`.

## Open candidates

### OreRefine consumes ore before affordability check
`OreRefine` executes:
1. validate >=100 ore;
2. `item->SetCount(item->GetCount() - 100)`;
3. compute fee;
4. only then test `GetGold() < iCost`.

If insufficient Yang, it returns false after the 100 ore are already removed.

The Lua bindings `pc.diamond_refine` / `pc.ore_refine` exist, but no current Project_Game quest caller was found in this pass. Keep as a statically real routine defect but do not promote as a current reachable deployment bug until a live caller is established.

### Unreachable MINING_LOCATION hack log
`CHARACTER::mining` first returns when distance >1000, then later contains a >2500 `HackLog("MINING_LOCATION")` branch. The later branch is unreachable under the same coordinates. This is an observability/security-logging defect candidate, not a mining reward bypass.

## Exact next work
1. map click/quest entry ownership and packet trust boundary;
2. audit death/warp/logout/equipment-change cleanup in full;
3. map pick mastery/refine item cell and extended-inventory behavior;
4. establish current reachability of ore refining and validate fee/material atomicity;
5. audit mining skill-book progression and cooldown;
6. audit ore drop ownership, Battle Field behavior, and mining event interactions;
7. promote only statically reachable findings.

## Runtime
Do not execute Mining runtime tests while execution lock is active. Global first future live gate remains `DUNGEON-T09`.
