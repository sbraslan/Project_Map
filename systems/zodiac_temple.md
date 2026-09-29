# Zodiac Temple / 12ZI — Static System Map

**Status:** STATIC MAPPING OPEN / 7 VERIFIED BUGS  
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

## Current audit cursor
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

## Reward/command boundary findings
The Python UI applies useful client-side state (disabling completed cells and enabling gold reward only when both color counters exceed the displayed paired value), but all authoritative actions are plain player chat commands. Server code therefore must independently enforce those UI invariants. It currently does not for duplicate cell selection, gold-pair eligibility, or same-instance revive.

## Current audit cursor
1. Resolve actual Zodiac deployment/entry ownership: server-time portals vs missing tracked quest source.
2. Audit `DecMember` event-flag branch and party/reconnect teardown.
3. Audit private-map/floor timer destruction ordering.
4. Audit bead regeneration/persistence and 12ZI shop-limit accounting.
5. Close remaining client/server parity and decide whether additional bugs are promoted.
