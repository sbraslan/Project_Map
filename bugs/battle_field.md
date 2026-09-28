# Battle Field System — Bug Registry

**Phase:** Detection / Mapping Only
**Source repos:** read-only

### BUG-BFIELD-001 — exit_battle_field is callable outside Battle Field and becomes a general town-warp command
- Statik durum: **doğrulandı**
- Sınıf: command authorization / map-boundary bypass

The player command table exposes:
`exit_battle_field -> do_exit_battle_field`
with `GM_PLAYER` and minimum position `POS_DEAD`.

The command interpreter permits any character whose current position is >= the command minimum, so this is not restricted to the Battle Field UI or Battle Field map.

`do_exit_battle_field` directly calls `CBattleField::RequestExit(ch)`.

`RequestExit` validates PC/event state and `CanWarp()`, but never checks:
`IsBattleZoneMapIndex(ch->GetMapIndex())`.

It creates the normal Battle Field exit event, which later calls `ExitCharacter`.

`ExitCharacter` also has no Battle Field map check and:
- optionally converts temporary Battle Field points to persistent points/ranking;
- sets the 600-second `battlefield.cooldown`;
- warps the character to the empire start.

Therefore a normal player outside Battle Field can invoke the Battle Field exit command anywhere `CanWarp()` permits and obtain the Battle Field town-warp behavior without being a Battle Field participant.

### BUG-BFIELD-002 — exit_battle_field_on_dead 1 bypasses map/death/CanWarp checks and immediately teleports
- Statik durum: **doğrulandı**
- Sınıf: command authorization / warp restriction bypass

The player command:
`exit_battle_field_on_dead`
is also registered as `GM_PLAYER`, minimum `POS_DEAD`.

For argument `1`, `do_exit_battle_field_on_dead` directly executes:
`CBattleField::Instance().ExitCharacter(ch)`.

There is no validation that:
- the character is on `BATTLE_FIELD_MAP_INDEX`;
- the character is dead;
- the character is in the Battle Field death/exit flow;
- `CanWarp()` is true;
- a Battle Field exit event/affect exists.

Unlike BUG-BFIELD-001, this path also skips the 3/120-second `RequestExit` delay and its `CanWarp()` gate.

A normal player can therefore invoke argument 1 outside Battle Field and immediately execute Battle Field exit semantics, including empire-start warp and Battle Field cooldown assignment.

The intended client path reaches this command after a Battle Field death, but server authorization does not enforce that provenance.

### BUG-BFIELD-003 — anti-repeat kill cooldown is not renewed after the first expiry
- Statik durum: **doğrulandı**
- Sınıf: score integrity / anti-farming logic
- Configured interval: `BATTLE_FIELD_KILL_TIME = 60` seconds

`CHARACTER::SetBattleKill(victimPID)` checks an existing victim entry:
- if current time is still before stored expiry -> reject;
- if expiry has passed -> continue.

It then calls:
`m_BattleFieldKillMap.emplace(victimPID, now + TIME_BETWEEN_KILLS)`.

For an already-existing PID, `std::map::emplace` does not replace the old value.

Sequence:
1. first kill inserts victim -> now+60;
2. kills during the next 60 seconds are rejected;
3. after expiry, one kill is accepted;
4. the attempted `emplace` fails because the key already exists;
5. stored expiry remains the old past timestamp;
6. every subsequent kill of that victim in the same character session passes the time check immediately.

Thus the intended per-victim repeat-kill cooldown is enforced only for the first interval. After the first expiry, repeated kills can award Battle Field score without the configured spacing.

The map is cleared only with the character object's lifecycle/initialization, not after each accepted post-expiry kill.


### BUG-BFIELD-004 — Battle Field calls an unqualified/nonexistent LoadRanking symbol
- Statik durum: **doğrulandı**
- Sınıf: build/integration failure
- Build koşulu: `ENABLE_BATTLE_FIELD` + `ENABLE_RANKING_SYSTEM` (both enabled in current server CommonDefines snapshot)

`CBattleField::CloseEnter` and the weekly-ranking branch in `CBattleField::Update` both call:
`LoadRanking(RK_CATEGORY_BF);`

`CBattleField` declares no `LoadRanking` member.

The mapped headers included by `battle_field.cpp` expose only:
`CRankingSystem::LoadRanking(uint8_t)`.

No included header declares a matching global/free `LoadRanking`, and no macro alias was found in the mapped include set.

Therefore the active preprocessor path contains an unresolved unqualified function call at both reload sites. This is a Battle Field <-> Ranking integration compile defect in the checked-in source snapshot.

### BUG-BFIELD-005 — weekly rollover leaves stale winners when the new week has fewer than three scorers
- Statik durum: **doğrulandı**
- Sınıf: ranking lifecycle / stale DB state

`CBattleField::UpdateWeekRanking` selects up to three current-week players and writes:
`REPLACE INTO log.battle_week (pos, pid, score, last_update) ...`
only for rows actually returned.

It never clears `log.battle_week` positions that are not replaced.

If the new week has:
- zero qualifying scorers -> all old winner rows remain;
- one scorer -> previous positions 2/3 remain;
- two scorers -> previous position 3 remains.

`CRankingSystem::LoadRankingWeekWinners` later reads:
`SELECT p.id FROM log.battle_week ... ORDER BY r.score DESC LIMIT 3`
with no week/timestamp filter.

Old rows can therefore be interpreted as winners of the new week and feed `GetBFRankingPosition` / winner-affect assignment.

This defect is independent of the separate Ranking module bugs; it originates in Battle Field's weekly rollover writer.


### BUG-BFIELD-006 — battle_set_event silently writes event date to the wrong core
- Statik durum: **doğrulandı**
- Sınıf: multi-core state / operator command routing

`battle_set_event` is registered as an implementor command and validates only month/day.

Unlike `battle_force_open` and `battle_force_close`, it does **not** require:
`g_bChannel == BATTLE_FIELD_MAP_CHANNEL`.

It only executes:
`CBattleField::Instance().SetEventInfo(month - 1, day)`.

The event month/day fields are process-local members of the Battle Field singleton; no P2P/state replication is performed by this command.

The server heartbeat calls `CBattleField::Update()` only when:
`g_bChannel == CHANNEL_99`.

Therefore issuing `battle_set_event` from an implementor character connected to any other channel reports no error but stores the event date in a process that never evaluates the Battle Field schedule. Channel 99 retains its old/default event date, so the intended event-mode opening and score multiplier do not activate there.

This is a deterministic multi-core administration/state-routing defect.

### BUG-BFIELD-007 — weekly winner affect flags are not refreshed for online players
- Statik durum: **doğrulandı**
- Sınıf: ranking state lifecycle / stale online state

Battle Field weekly winner status is applied by:
`CBattleField::SetWeakRankingPosition(pChar)`.

It:
- looks up the player in `CRankingSystem::vecBattleFieldWeekRankingWinners`;
- directly sets one of `AFF_BATTLE_RANKER_1..3` through `m_afAffectFlag.Set()`.

This is not a normal `CAffect` object and is not represented in the affect list.

The winner list is refreshed by `LoadRankingWeekWinners`, but neither that function nor the weekly rollover iterates currently online characters to:
- remove old Battle Field rank flags;
- assign new rank flags.

`SetWeakRankingPosition` is mapped on the Battle Field `Connect`/login path, not as a post-rollover refresh.

Consequences at rollover:
- a previously ranked player who remains online can retain the old winner flag after losing its ranking;
- a new winner who remains online does not receive the new winner flag until a later reconnect/path that calls `SetWeakRankingPosition`.

The raw flag can also survive ordinary affect-list recomputation because it was not installed as a `CAffect`.

### BUG-BFIELD-008 — open/close countdown arithmetic uses current seconds with the wrong sign
- Statik durum: **doğrulandı**
- Sınıf: schedule calculation / UI timing

For a same-day target time, both `GetOpenTime` and `GetCloseTime` calculate:
`hour_delta + minute_delta + pTimeInfo->tm_sec`.

The mathematically correct remaining time to an HH:MM:00 target is:
`hour_delta + minute_delta - current_seconds`.

Therefore same-day values are overstated by:
`2 * current_seconds`
(0..118 seconds).

The next-day branch similarly builds a day/minute offset and then **adds** current seconds. Relative to the true remaining time, its error is:
`2 * current_seconds - 60`
(-60..58 seconds).

These functions feed Battle Field timing commands/UI and are also compared by the open/close scheduler. The defect is deterministic for any call made with nonzero seconds.

A separate algorithm limitation remains unnumbered: the fallback searches only the immediate next day, so sparse schedules with a gap longer than one day can return 0. The repository does not contain the live `common.battlefield_open_info` rows, so current-data reachability for that limitation is not established.


### BUG-BFIELD-009 — Battle Point cap rejection still clears temporary points and credits ranking
- Statik durum: **doğrulandı**
- Sınıf: reward/currency consistency / ranking divergence

`CBattleField::ExitCharacter` handles temporary Battle Field score as:

1. read `dwBattleFieldPoints`;
2. `PointChange(POINT_BATTLE_FIELD, dwBattleFieldPoints)`;
3. unconditionally `SetBattleFieldPoint(0)`;
4. unconditionally `RegisterBattleRanking(..., dwBattleFieldPoints)`.

The persistent point handler computes:
`GetBattlePoint() + amount`
and returns without applying the amount when:
`BATTLE_POINT_MAX <= total`
or total is negative.

`BATTLE_POINT_MAX` is 2,000,000,000.

`PointChange` returns `void`, so `ExitCharacter` cannot detect that the addition was rejected.

Therefore when a player exits with temporary points that would reach/exceed the cap:
- persistent Battle Point balance receives none of those temporary points;
- temporary Battle Field score is still erased;
- `log.battle_score` is still credited with the full temporary amount through `RegisterBattleRanking`.

The player's spendable Battle Point balance and ranking score deterministically diverge at the cap boundary.

### BUG-BFIELD-010 — event-mode open state is not propagated to Battle Field clients
- Statik durum: **doğrulandı**
- Sınıf: multi-core/client-state integration

The Battle Field client has separate commands/state:
- `battle_field_event enable start end` -> `SetBattleFieldEventInfo`;
- `battle_field_event_open open` -> `SetBattleFieldEventOpen`.

The minimap chooses the special event-open visuals only when:
`IsBattleFieldOpen() == true && IsBattleFieldEventOpen() == true`.

Server `OpenEnter(isEvent=true)`:
- sets global Battle Field status open;
- broadcasts `battle_field_open 1`;
- sets only the channel-99 process-local `bEventStatus=true`;
- does **not** broadcast `battle_field_event 1 ...`;
- does **not** broadcast `battle_field_event_open 1`.

On ordinary channel processes, `Connect` reads their own process-local `GetEventStatus()`, which remains false, so it also does not send the event-info command to those clients.

P2P `HEADER_GG_COMMAND` forwarding calls `SendCommand` to clients; it does not mutate the remote `CBattleField` singleton event fields.

Thus Battle Field opening in event mode propagates the generic open state but not the event-open state required by the client event UI. The dedicated client `battle_field_event_open` callback exists, but the mapped Battle Field opening path never feeds it.

Related latent client defect, not separately numbered:
`CPythonPlayer::GetBattleFieldEventEnable()` returns `bBattleFieldIsEventOpen` instead of `bBattleFieldIsEventEnable`. In the current minimap implementation the returned `IsEventEnable` local is assigned but not used, so no additional active failure is attributed to that getter yet.


### BUG-BFIELD-011 — reconnect resets Battle Field anti-abuse session state while preserving map position
- Statik durum: **doğrulandı**
- Sınıf: reconnect/session-state bypass

Two Battle Field controls are character-instance memory only:
- `m_BattleFieldKillMap` — per-victim repeat-kill cooldown state;
- `m_bBattleDeadLimit` — accumulated Battle Field death penalty used by restart timing.

Character initialization executes:
- `m_BattleFieldKillMap.clear()`;
- `m_bBattleDeadLimit = 0`.

On disconnect, the normal player save persists current map/position, but neither of these fields is part of `TPlayerTable`.

If the Battle Field remains open, login restores the character on the Battle Field map and `CBattleField::Connect` leaves the player there.

Consequences after reconnect:
- a victim PID that was still inside the 60-second repeat-kill block is forgotten, allowing the same target to score again immediately;
- accumulated death penalty is reset, so subsequent Battle Field restart wait is calculated from the initial low death-limit state again.

This is independent of BUG-BFIELD-003:
- 003 breaks cooldown renewal after the first expiry without reconnecting;
- 011 clears the cooldown entry entirely on reconnect, even before its first 60-second expiry.

Temporary unbanked Battle Field score is also RAM-only and is forfeited on reconnect, but whether that forfeit is intentional is not classified separately.


## Registry/readiness normalization — 2026-09-28

Cross-system consistency correction:
- `BUG-BFIELD-004` is **RETRACTED / RESERVED**. The earlier Battle Field audit missed the same transitive declaration path later closed in Ranking: global `LoadRanking(uint8_t)` is declared through the mapped include chain and implemented in `cmd_general.cpp`. This is the same false-positive class as retracted `BUG-RANK-006`.
- `BUG-BFIELD-005` is not retained as an independent current Battle Field bug. Its stale weekly-winner-row defect is canonically owned by `BUG-RANK-003`.
- `BUG-BFIELD-007` is not retained as an independent current Battle Field bug. Its online winner-flag refresh defect is canonically owned by `BUG-RANK-007`.

Battle Field historical text is preserved for audit history. Runtime-readiness ownership uses:
- unique active Battle Field bugs: `BUG-BFIELD-001`, `002`, `003`, `006`, `008`, `009`, `010`, `011`;
- retracted/reserved: `BUG-BFIELD-004`;
- cross-system aliases: `BUG-BFIELD-005 -> BUG-RANK-003`, `BUG-BFIELD-007 -> BUG-RANK-007`.

No new bug IDs are created by this normalization.
