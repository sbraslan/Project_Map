# Mining / Pickaxe

**Status:** MAPPING IN PROGRESS

## Scope
Independent static audit of the classic Mining / Pickaxe subsystem.

### Mapped roots
- `Project_ServerSRC/game/src/mining.cpp/.h`
- `Project_ServerSRC/game/src/char.cpp` — mining start/cancel/take, warp lifecycle, ore-vein lifetime
- `Project_ServerSRC/game/src/char_state.cpp` — normal movement -> `Move()` -> `OnMove()` cancellation path
- `Project_ServerSRC/game/src/char_battle.cpp::Dead`
- `Project_ServerSRC/game/src/questlua_pc.cpp` — `pc.mining`, `pc.ore_refine`, `pc.diamond_refine`
- `Project_ServerSRC/game/src/questlua_global.cpp` — `__refine_pick`
- `Project_Game/share/locale/europe/quest/n_npc/mining.quest`
- `Project_Game/share/locale/europe/quest/g_guild/guild_building_melt.quest`
- `Project_DumpProto/{tr,en,de}/item_names.txt` — pickaxe family 29101..29110
- `Project_ServerSRC/common/CommonDefines.h` — `ENABLE_PICKAXE_RENEWAL`

## Core mining flow
`pc.mining()`
-> `CHARACTER::mining(current NPC/ore vein)`
-> validates same map, <=1000 distance, ore-vein VNUM and equipped ITEM_PICK
-> random work count 5..15
-> `CreateMiningEvent` for 10..30 seconds
-> event re-resolves player PID + vein VID
-> pick validation
-> mining success chance
-> `OreDrop`
-> raw ore ground drop with normal ownership
-> `PracticePick`.

Normal player movement is already covered correctly:
`StateMove -> Move -> OnMove -> mining_cancel`.

Character destruction also cancels `m_pkMiningEvent`.

Ore veins use the same per-character event field on the vein object for a 15-minute self-destroy timer; this is entity-local and is not itself a bug.

## Success arithmetic
Base mining chance: 20%.

Mining skill contribution is table-driven for skill 0..40. Pickaxe grade contribution for +0..+9 is:
`3, 5, 8, 11, 15, 20, 26, 32, 40, 50`.

Nominal maximum mapped chance is 81%.

Raw ore count uses the nine-entry fraction table and its probability weights sum to 100%.

## BUG-MINE-001 — pickaxe NPC refine path is logically unreachable
Current `mining.quest` calls `__refine_pick(item.get_cell())` only when:
`item.get_socket(0) == item.get_value(2)`.

But server `Pick_Refinable()` returns false while:
`Pick_GetCurExp(item) <= Pick_GetMaxExp(item)`.

Therefore `RealRefinePick()` accepts only `socket0 > value2`.

The two conditions do not overlap:
- at `socket0 == value2`, quest calls C++ but C++ rejects;
- once practice advances to `socket0 > value2`, the quest enters its not-ready branch and never calls C++.

Promoted as `BUG-MINE-001`.

## BUG-MINE-002 — active mining survives same-character warp
`CanWarp()` does not consider `m_pkMiningEvent`.
`WarpSet()` calls `Stop()`, but `Stop()` does not call `mining_cancel()`.
`WarpEnd()/Show()` also do not cancel mining.

The mining completion event does not re-check the player's current map or distance from the original vein.

If a same-process/same-character warp completes while the original vein still exists, the event can resolve afterward. `OreDrop()` uses the player's current map/current coordinates, so successful ore can be dropped at the destination rather than the original vein.

Promoted as `BUG-MINE-002`.

## BUG-MINE-003 — active mining survives player death
`CHARACTER::Dead()` does not cancel `m_pkMiningEvent`.
The mining event callback does not check `IsDead()`.

As long as the character object, equipped pickaxe and source vein remain valid, the countdown can finish after death, perform the normal success roll, drop ore and practice the pickaxe.

Promoted as `BUG-MINE-003`.

## Ore refinement status
Current deployed caller is `guild_building_melt.quest`.

`OreRefine()` itself subtracts 100 raw ore before its internal Yang check. That ordering is dangerous in isolation, but the current quest checks the matching guild/empire-adjusted fee before invoking the Lua binding.

The quest's `GetOreRefineCost` currently matches `ComputeRefineFee` semantics:
- same guild: 90%;
- foreign empire: 3x;
- otherwise base cost.

Therefore the low-Yang deletion branch is **not promoted as a current normal-path bug** at this checkpoint. It remains a defense-in-depth/race candidate pending quest-yield and transaction-boundary audit.

The non-diamond path validates the selected catalyst in quest as VNUM 28000..28299 before invoking `pc.ore_refine`.

## Current next work
1. Audit ore-refine quest yield/transaction boundaries and catalyst lifetime.
2. Map pickaxe proto Value0..Value4 / RefinedVnum semantics and current grade data.
3. Audit `ENABLE_MINING_EVENT` modifiers/rewards and mining-specific event hooks.
4. Audit SKILL_MINING book progression and any Battle Pass/Achievement integration.
5. Map client click/dig animation to quest/server trust boundaries.
6. Close Battle Field ownership behavior and multiplayer ore pickup semantics.
7. Audit channel-change versus same-process warp event ownership.
8. Promote only verified reachable findings.

## Runtime
Execution remains locked. `MINE-T01..MINE-T03` are documentation-only until explicit phase change.

Global first future live gate remains `DUNGEON-T09`.
