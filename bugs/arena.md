# Arena / PvP Duel — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 1 VERIFIED BUG  
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

## Shadowed implementation candidates
### Candidate A — map112 has no MAP_ALLOW owner
All four classic arenas are registered on map112 by loaded `settings.lua`; no tracked core CONFIG hosts map112. StartDuel ignores both WarpSet results.

### Candidate B — timeout reset packet targets A twice
`duel_time_out` sends the reset/end `TPacketGCDuelStart` twice to A and never to B.

### Candidate C — observer Arena pointer/mode not explicitly cleared on EndDuel
Current deployment/start path prevents ordinary validation; revisit after Arena001/map112 repair.

These candidates are intentionally unnumbered until current-player reachability is established.
