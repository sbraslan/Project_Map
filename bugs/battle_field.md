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
