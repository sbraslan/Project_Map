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
