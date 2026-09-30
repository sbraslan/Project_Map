# Snake Lair / Queen Nethis — Bug Registry

**Status:** STATIC MAPPING OPEN / 5 VERIFIED BUGS
**Execution:** LOCKED / NOT RUN

## BUG-SNK-001 — wrong-order pillar use consumes the valid pillar key

**Class:** item transaction / validation order

### Proof
- Item-give dispatch for target race `PILAR_STEP_4` routes Snake-map interactions to `CSnk::OnKillPilar`.
- The global wrapper verifies the item is `VNUM_KILL_PILAR` (70422) and forwards to the registered `CSnkMap`.
- `CSnkMap::OnKillPilar` immediately calls `ITEM_MANAGER::Instance().RemoveItem(pkItem)`.
- Only after removal does the handler validate the required sequence for pillars 2-6.
- On an out-of-order target, the function sends the “unlock the other pillar first” notice and returns, with no compensation path.

### Consequence
A legitimate Snake pillar key is permanently lost when used on a later pillar before its required predecessor.

### Deferred validation
`SNK-T01`.

## BUG-SNK-002 — Snake statues consume arbitrary or wrong-element items before validation

**Class:** item transaction / input validation

### Proof
- `char_item.cpp` routes any item given to statue NPC races 4024-4027 into `CSnk::OnStatueSetRotation` when the player is on a Snake map.
- The outer filter is `itemVnum < SNAKE_STATUE1 && itemVnum > SNAKE_STATUE4`; this condition is mathematically impossible and therefore rejects nothing.
- It also compares item VNUMs to statue NPC VNUMs instead of the intended Snake element item VNUMs 70423-70426.
- `CSnkMap::OnStatueSetRotation` calls `ITEM_MANAGER::Instance().RemoveItem(pkItem)` before checking:
  - whether the clicked statue is already blocked/completed;
  - which statue race is present;
  - whether the supplied item is the matching fire/ice/wind/ground item.
- Every rejection after that point returns without restoring the item.

### Consequence
Wrong-element items, unrelated items, or an item handed to an already-completed statue can be destroyed without advancing the dungeon.

### Deferred validation
`SNK-T02`.

## BUG-SNK-003 — floor-4 Ice Siren phase completes after the first Siren kill

**Class:** progression counter / premature stage completion

### Proof
- Step-4 substep 11 chooses `spawnSirens = number(0,4)`.
- If nonzero, it stores that spawn count in `KillCountMonsters` and spawns exactly that many `SIREN_ICE` mobs.
- On the first Ice Siren death:
  - `remainSirens = GetKillCountMonsters()` stores N;
  - `SetKillCountMonsters(GetKillCountMonsters()+1)` changes it to N+1;
  - the test `GetKillCountMonsters() >= remainSirens` is therefore immediately true for every N from 1 to 4.
- The handler then resets counters and calls `SetDungeonStep(5)`.

### Consequence
When 2-4 Ice Sirens are spawned, killing only one is sufficient to skip the remaining required kills and advance the dungeon.

### Deferred validation
`SNK-T03`.


## BUG-SNK-004 — party destruction unregisters but does not destroy the Snake private instance

**Class:** lifecycle cleanup / orphaned private-map resources

### Proof
- `CParty::Destroy()` explicitly detects connected members on a Snake map.
- It calls `CSnk::LeaveParty(mapIndex)`, whose complete implementation is only `Remove(mapIndex)`.
- `Remove` erases the `m_dwRegGroups` entry and performs no instance teardown.
- `CParty::Destroy()` then calls `CSnk::Leave(member)`, which only warps the connected member out.
- The private `SECTREE_MAP`, `CSnkMap` object, spawned entities and Snake events are left intact.
- Their eventual cleanup depends on the already-running one-hour `r_snakelimit_event` reaching `CSnkMap::EndDungeonWarp()`.

### Consequence
A party destroyed during a Snake run can leave an inaccessible/unregistered private dungeon consuming a private-map slot, mobs and event processing for the remainder of the original one-hour lifetime.

### Deferred validation
`SNK-T04`.


## BUG-SNK-005 — delayed Queen Nethis skill event can dereference a destroyed player

**Class:** asynchronous lifetime / use-after-free

### Proof
- `FSkillQueenNethis` iterates Snake-map PCs every 25 seconds and calls `ComputeSnakeSkill(273, pkChar, 1)`.
- `ComputeSnakeSkill` allocates `r_snakeskill_info` and stores `pEventInfo->pkVictim = this` as a raw `LPCHARACTER`.
- The delayed callback runs two seconds later and dereferences `pEventInfo->pkVictim` to call `GetSectree()` and apply area damage.
- The event is assigned to CHARACTER member `m_pkSnakeSkillEvent`.
- `CHARACTER::Destroy()` cancels many owned events, but does not cancel `m_pkSnakeSkillEvent`.
- The event queue holds its own intrusive reference to the event, so destruction of the CHARACTER-side handle does not remove the queued callback.

### Consequence
If a player disconnects/is destroyed during the two-second delay, the queued Snake skill can execute with a freed CHARACTER pointer and crash or corrupt the game process.

### Deferred validation
`SNK-T05`.
