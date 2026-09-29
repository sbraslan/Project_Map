# Arena / PvP Duel — Static Map

**Status:** STATIC COMPLETE / 3 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
This subsystem covers two distinct implementations:
1. classic player-vs-player `CArenaManager / CArena / CArenaMap`;
2. legacy GM-driven monster invasion `CBattleArena`, tracked as a companion until caller/deployment closure.

## Classic Arena verified roots
- `Project_ServerSRC/game/src/arena.cpp`
- `Project_ServerSRC/game/src/arena.h`
- `Project_ServerSRC/game/src/questlua_arena.cpp`
- `Project_ServerSRC/game/src/input_login.cpp`
- `Project_ServerSRC/game/src/char.cpp`
- `Project_Game/share/locale/europe/settings.lua`
- `Project_Game/share/locale/europe/quest/n_npc/arena_manager.quest`
- `Project_Game/share/locale/europe/arena_forbidden_items.txt`
- `Project_Game/share/locale/europe/map/metin2_map_duel/*`

## Deployment
- deployed quest list contains `n_npc/arena_manager.quest`;
- `settings.lua` is loaded directly during Lua initialization;
- settings registers four duel areas on map 112:
  - (8534,101) vs (8564,101)
  - (8584,101) vs (8614,101)
  - (8534,155) vs (8564,155)
  - (8584,155) vs (8614,155)
- map index file: `112 metin2_map_duel`.
- current tracked CONFIG files contain **no MAP_ALLOW 112 owner**.
- `MAP_PVP_ARENA = 90` is a separate map (`metin2_map_pvp_arena`) and is not the classic CArena map configured by settings.lua.

## Normal duel quest flow
`NPC 20017`
→ arena_close / minimum-level checks
→ input opponent name
→ find local opponent VID
→ opponent level / proximity checks
→ `arena.is_in_arena(opp_vid)`
→ confirmation
→ `arena.start_duel(name,3)`
→ Lua `arena_start_duel`
→ manager member checks
→ `CArenaManager::StartDuel`
→ first empty registered arena
→ `CArena::StartDuel`
→ warp both players to duel start points
→ 10-second ready event
→ SetArena pointers / potion limit
→ duel start packet
→ death rounds until first to 3 or timeout.

### BUG-ARENA-001 — deployed availability precheck rejects every normal idle opponent
The deployed quest expects `arena.is_in_arena(opp_vid)` to return non-zero for an opponent eligible to start a duel and rejects when it returns 0.

Lua implementation:
- resolves target character;
- for a normal idle player, `GetArena() == nullptr`;
- asks `CArenaManager::IsMember(current_map,pid)`;
- a normal idle player is `MEMBER_NO`;
- Lua pushes 0.

Therefore the ordinary eligible opponent is rejected before confirmation / `arena.start_duel`.

The only state that can return 1 is a character already classified as `MEMBER_DUELIST` while entering the first branch; but `arena_start_duel` independently rejects existing members. There is no successful normal state transition through the deployed quest.

Promoted as `BUG-ARENA-001`.

## Classic Arena lower-layer findings currently shadowed by BUG-ARENA-001

### Candidate A — map112 has no deployed host
`settings.lua` registers all classic arenas on map112, but none of the tracked game-core CONFIG files include map112 in MAP_ALLOW.

`map_allow_find` is a strict allow-list lookup. `CArena::StartDuel` calls `WarpSet(startX*100,startY*100)` for both players and ignores the boolean results.

If BUG-ARENA-001 were removed without changing deployment, the duel manager can reserve the arena and schedule start/timeout events while the target duel map has no advertised server route.

Keep candidate until the current normal start path is restored or another deployed caller of `arena.start_duel` is proven.

### Candidate B — timeout end packet is sent to A twice, never B
Normal `duel_time_out` state 0 builds an empty `TPacketGCDuelStart` end/reset packet, then executes:
- `chA->GetDesc()->Packet(...)`
- `chA->GetDesc()->Packet(...)`

There is no equivalent send to chB before the 10-second delayed EndDuel.

This is a concrete packet asymmetry but is currently shadowed by the unreachable normal duel start.

### Candidate C — observer cleanup depends on cross-process character rebuild
Observer entry sets `SetArena(pArena)`; target login enables observer modes. EndDuel warps observers and clears the arena observer map but does not explicitly clear each live observer's Arena pointer/mode. Current map112 is unhosted, so ordinary current reachability is already blocked. Re-evaluate after deployment repair.

## Duel combat / round flow
Arena attack permission:
`CArenaManager::CanAttack`
→ require same map
→ registered CArenaMap
→ matching pair PIDs.

Death:
`CArenaManager::OnDead(killer,victim)`
→ locate arena containing both PIDs
→ increment killer's set score
→ if target score reached: schedule EndDuel in 10s
→ otherwise schedule round restart in 10s, Show both at start points, restore HP/SP, resend duel packets.

Timeout:
- notice;
- send duel-reset packet;
- wait 10s;
- EndDuel.

EndDuel:
- cancel events;
- reset PK/recovery/HP/SP;
- clear duelist Arena pointers;
- warp duelists and observers to arena return points;
- clear observer map and arena PID/score state.

## BattleArena companion
`CBattleArena` is a separate GM-operated invasion event.

Deployed command:
`weeklyevent [empire]`
→ `do_weeklyevent`
→ permission `GM_LOW_WIZARD`
→ when not running: `CBattleArena::Start(empire)`
→ when running: `CBattleArena::ForceEnd()`.

Configured target maps:
- empire 1 -> 190
- empire 2 -> 191
- empire 3 -> 192

### BUG-ARENA-002 — deployed Weekly/BattleArena command targets nonexistent, unhosted maps
Current tracked deployment has:
- no map-index entries 190/191/192;
- no `metin2_map_battlearena01/02/03` map data;
- no core CONFIG with MAP_ALLOW 190/191/192.

`CBattleArena::Start` performs no map availability validation. It sets status BATTLE, schedules the event, writes `battle_arena = targetMap` and returns success.

The registered GM command consequently reports “Weekly Event Start” even though the target event map has no deployed route/data.

Promoted as `BUG-ARENA-002`.

### BUG-ARENA-003 — ForceEnd does not force the BattleArena state machine to end
`CBattleArena::ForceEnd`:
- sets `m_bForceEnd = true`;
- cancels the current event;
- creates a new `battle_arena_event` beginning at **state 3**.

But `m_bForceEnd` is never consulted by `battle_arena_event`.

State 3 is the normal monster-wave monitoring state. With monsters present it can continue 5-minute loops and spawn stones before eventually advancing to purge/end. Even when called through `weeklyevent`, the GM is immediately told “Weekly Event End” while the server event can remain running.

Promoted as `BUG-ARENA-003`.

Keep BattleArena lifecycle separate from classic duel correctness.

## Current audit cursor
1. audit duel disconnect/death/timeout and potion/item restrictions;
2. audit observer lifecycle/reconnect;
3. close classic map112 deployment candidate classification under BUG-ARENA-001 shadow;
4. finish BattleArena event-state edge cases after BUG-ARENA-002/003;
5. decide STATIC COMPLETE readiness.

## Runtime
No Arena runtime test may be executed while the global execution lock is active. First future live gate remains `DUNGEON-T09`.


## Final static closure
Arena / PvP Duel static mapping is complete for the tracked snapshot with `BUG-ARENA-001..003`.

Shadowed/deferred classic findings remain intentionally unpromoted:
- timeout end/reset packet is sent to player A twice and never B;
- observer packet/chat broadcast implementations are currently commented out/no-op;
- EndDuel clears the observer map but does not explicitly clear each live observer's Arena pointer/observer modes;
- map112 has no tracked MAP_ALLOW owner and StartDuel ignores WarpSet return values;
- settings.lua loads map112 arena slots into each game process, but no current deployed owner exists, so cross-process post-deployment behavior must be re-audited if map112 is restored.

BattleArena after-fix observations:
- empire2/3 random-stone region Y bounds are reversed in source, but current maps 191/192 are absent, so this remains shadowed by BUG-ARENA-002.
