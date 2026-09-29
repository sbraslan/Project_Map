# Zodiac Temple / 12ZI — Static System Map

**Status:** STATIC MAPPING CLOSED / 9 VERIFIED BUGS  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source policy:** Project_ClientSrc, Project_ServerSRC, Project_Binary, Project_Game and Project_DumpProto are read-only.

## Why this subsystem is separate
`ENABLE_12ZI` is active and the tracked corpus contains a dedicated Zodiac runtime rather than only generic dungeon helpers:
- server manager/instance: `game/src/zodiac_temple.cpp/.h`;
- character/reward state: `game/src/char_zodiac_temple.cpp`;
- Zodiac combat/events: `game/src/char_battle_zodiac.cpp`;
- Lua API: `game/src/questlua_zodiac_temple.cpp`;
- player commands: `game/src/cmd.cpp` + `cmd_general.cpp`;
- client UI: `Project_Binary/root/ui12zi.py`;
- game data: `share/data/dungeon/zodiac/` and `share/locale/europe/map/metin2_12zi_stage/`.

## Initial call-flow roots
```
quest / NPC entry
  -> zodiac_temple.starttemple(portal)
  -> CZodiacManager::StartTemple
  -> CZodiacManager::Create
  -> SECTREE_MANAGER::CreatePrivateMap(MAP_12ZI_STAGE)
  -> CZodiac::Jump / JumpParty
  -> CZodiac::StartLogin
  -> floor event / mob progression
```

```
client Reward12ziWindow
  -> chat command /cz_check_box or /cz_reward
  -> command table (GM_PLAYER)
  -> do_cz_check_box / do_cz_reward
  -> CHARACTER::ZTT_CHECK_BOX / ZTT_REWARD
  -> quest flags + item consumption/reward
  -> ZTT_LOAD_INFO
  -> OpenUI12zi command
```

```
client floor controls
  -> /jumpfloor or /nextfloor
  -> do_jump_floor / do_next_floor
  -> CZodiacManager::FindByMapIndex
  -> CZodiac::NewFloor
```

## Mapped deployment roots
- `CommonDefines.h::ENABLE_12ZI`
- `zodiac_temple.cpp/.h`
- `char_zodiac_temple.cpp`
- `char_battle_zodiac.cpp`
- `questlua_zodiac_temple.cpp`
- `cmd.cpp`
- `cmd_general.cpp`
- `Project_Binary/root/ui12zi.py`
- `Project_Binary/root/game.py`
- `Project_Game/share/data/dungeon/zodiac/days/*.txt`
- `Project_Game/share/data/dungeon/zodiac/zodiac_seller.txt`
- `Project_Game/share/locale/europe/map/metin2_12zi_stage/*`

## Audit coverage completed
1. Verify deployment/quest entry path and portal validation.
2. Map instance ownership, party membership, reconnect/logout and destruction lifecycle.
3. Audit `/cz_check_box`, `/cz_reward`, revive and floor commands as player-authorized server entry points.
4. Audit reward item lifetime, quest-flag arithmetic and duplicate/replay behavior.
5. Audit floor/event timers, mob event raw pointers and private-map destruction.
6. Map client/server UI parity and packet/command boundaries.


## Verified bug cluster — first pass
- `BUG-ZOD-001`: player-triggerable temporary item-object leak in check-box/reward helpers.
- `BUG-ZOD-002`: asymmetric reward counters can pass a zero pair count into item creation and yield one gold box.
- `BUG-ZOD-003`: replayed check-box uses arithmetic addition and corrupts/forges the intended bitmask.
- `BUG-ZOD-004`: revive validates only Zodiac map *range*, not the same private instance.
- `BUG-ZOD-005`: `m_pkZodiacSkill1..11` event handles are raw/uninitialized.
- `BUG-ZOD-006`: delayed Zodiac skill events are not cancelled on character destruction and retain raw character pointers.
- `BUG-ZOD-007`: non-channel-99 Zodiac manager initialization falls off a non-void function.
- `BUG-ZOD-008`: the `zodiac_disconnect_member_2` branch erases the current set element and then increments an invalidated iterator.
- `BUG-ZOD-009`: bead catch-up resets the regeneration timestamp to now, discards the elapsed-hour remainder and sends a stale negative remaining-time value.

## Reward/command boundary findings
The Python UI applies useful client-side state (disabling completed cells and enabling gold reward only when both color counters exceed the displayed paired value), but all authoritative actions are plain player chat commands. Server code therefore must independently enforce those UI invariants. It currently does not for duplicate cell selection, gold-pair eligibility, or same-instance revive.

## Closing findings
- **Deployment/entry ownership:** channel 99 owns portal spawning through the server-time scheduler and day regen files. The Lua binding `zodiac_temple.starttemple(portal)` is present, but the tracked Game snapshot has no Zodiac entry in `quest_list`, no portal-NPC quest source, and no compiled `quest/object` path for NPCs 20439-20450. This is recorded as a deployment/source-completeness gap, not promoted as a runtime bug.
- **Party/reconnect lifecycle:** the normal/default path is internally coherent. `CHARACTER::Destroy -> SetZodiac(nullptr)` removes membership, and `CParty::Link` restores Zodiac association only when the reconnecting character is on the same private map. Missing event flags default to zero. The alternate `zodiac_disconnect_member_2` branch is separately promoted as `BUG-ZOD-008`.
- **Private-map/timer teardown:** manager-owned floor/remaining/exit events are cancelled before teardown; manager lookup entries are removed before the private map is destroyed, so later map-index callbacks fail closed. Character teardown occurs while the `CZodiac` object is still alive. No additional source-proven defect was promoted here.
- **Regen lifetime:** `ClearRegen()` is not called, but the tracked Zodiac caller uses `SpawnRegen("...zodiac_seller.txt")` with the default one-shot mode, so the persistent heap/event regen path is not reached by the mapped deployment. No bug promoted.
- **Bead persistence:** login executes `BeadTime()`; catch-up accounting discards the modulo-hour remainder and sends the pre-reset negative timer after elapsed time exceeds one hour. Promoted as `BUG-ZOD-009`.
- **12ZI shop accounting:** `ENABLE_12ZI_SHOP_LIMIT` is disabled in the tracked source snapshot, so the active path is the legacy DB-backed `zodiac_npc / zodiac_npc_sold` path. The spawned Zodiac seller is tracked, but its authoritative shop inventory/count configuration is not present in the tracked Game shop table, so no additional defect is promoted without deployment evidence.
- **Client/server parity:** `PythonNetworkStreamCommand.cpp` routes `ZodiacTime`, `ZodiacTimeClear`, `Bead_count`, `Bead_time` and `OpenReviveDialog` to the corresponding Python game/UI callbacks. No additional parity defect was found.

## Static-map result
All five closing cursor items are resolved for the pinned source snapshot. Zodiac Temple / 12ZI is ready to transition to CLOSED; runtime tests remain locked.
