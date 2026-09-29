# Arena / PvP Duel — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 3 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

## BUG-ARENA-001 — deployed arena quest rejects normal eligible opponents before duel creation

**Class:** quest/Lua contract inversion / unreachable gameplay  
**Reachability:** VERIFIED through deployed `arena_manager.quest`.

### Proof
1. Deployed quest resolves a nearby opponent and calls `arena.is_in_arena(opp_vid)`.
2. Quest rejects when the return is 0.
3. Lua `arena_is_in_arena` resolves the character.
4. A normal idle character has no Arena pointer.
5. It then calls `CArenaManager::IsMember(currentMap,pid)`.
6. A normal idle character returns `MEMBER_NO`.
7. Lua pushes 0.
8. Therefore the quest rejects the exact normal state required before starting a duel.
9. A state returning MEMBER_DUELIST cannot help: `arena.start_duel` independently rejects any character already registered as a member.

### Consequence
The deployed NPC flow cannot reach a successful normal `arena.start_duel` for two idle eligible players. Classic duel creation is effectively unavailable through its tracked ordinary quest.

### Deferred validation
`ARENA-T01`.

## BUG-ARENA-002 — Weekly BattleArena starts against maps absent from the deployed map topology

**Class:** deployment/configuration / false-success event start  
**Reachability:** VERIFIED through registered `weeklyevent` GM command (`GM_LOW_WIZARD`).

### Proof
1. `CBattleArena` hardcodes maps 190, 191 and 192 for empires 1..3.
2. Current `share/locale/europe/map/index` contains no 190/191/192 entries.
3. Current Project_Game tree contains no `metin2_map_battlearena01/02/03` data referenced by `strRegen`.
4. No tracked core CONFIG contains MAP_ALLOW 190/191/192.
5. `CBattleArena::Start` does not validate map availability or map data.
6. It schedules `battle_arena_event`, sets STATUS_BATTLE and writes the target map to the global event flag.
7. `do_weeklyevent` reports “Weekly Event Start” after calling Start.

### Consequence
The administrative event can enter running state and advertise a target map that the deployed game topology cannot host or populate.

### Deferred validation
`ARENA-T02`.

## BUG-ARENA-003 — BattleArena ForceEnd does not terminate the event promptly

**Class:** event-state machine / administrative stop semantics  
**Reachability:** VERIFIED through the same registered `weeklyevent` toggle.

### Proof
1. When BattleArena is running, `do_weeklyevent` calls `ForceEnd()` and immediately reports “Weekly Event End”.
2. `ForceEnd` sets `m_bForceEnd=true`.
3. It cancels current `m_pEvent`.
4. It creates a replacement `battle_arena_event` with `state=3`.
5. The event state machine never checks `m_bForceEnd`.
6. State 3 is normal battle monitoring: with monsters present it loops at 5-minute intervals, increments wait_count and spawns stones before eventually reaching purge/end.
7. Repeated ForceEnd returns immediately because `m_bForceEnd` is already true.

### Consequence
The command can report an ended event while the BattleArena scheduler and battle lifecycle continue for many minutes.

### Deferred validation
`ARENA-T03`.

## Shadowed classic-Arena implementation candidates

### Candidate A — map112 has no MAP_ALLOW owner
All four classic arenas are registered on map112 by loaded `settings.lua`; no tracked core CONFIG hosts map112. `CArena::StartDuel` ignores both WarpSet results. Currently shadowed by BUG-ARENA-001.

### Candidate B — timeout reset packet targets A twice
`duel_time_out` sends the reset/end `TPacketGCDuelStart` twice to A and never to B. Currently shadowed by the unreachable normal duel start.

### Candidate C — observer Arena pointer/mode not explicitly cleared on EndDuel
Current deployment/start path prevents ordinary validation; revisit after Arena001/map112 repair.

These classic candidates remain intentionally unnumbered until current-player reachability is established.
