# Horse / Mount / Riding — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.

## Entry roots

Server:
- `game/src/horse_rider.cpp/.h` — persistent horse level/health/stamina, ride-state events;
- `game/src/char_horse.cpp` — summon/dismiss/ride/unride/death/revive, horse character ownership;
- `game/src/questlua_horse.cpp` — Lua horse API;
- `game/src/char_item.cpp` — mount item/use/affect integration;
- `game/src/item.cpp` — ride-item classification, mount proto affects, ChangeLook mount expiry helpers;
- `game/src/char.cpp` — mount VNUM, stat recomputation, warp/show/login persistence.

Client/data surfaces to close:
- horse/mount packet and render state;
- mount costume/item proto families;
- horse NPC/race assets;
- horse appearance persistence;
- mount ChangeLook paths.

## Initial lifecycle map

### Classic horse state
`CHorseRider` owns:
- level;
- health;
- stamina;
- riding flag;
- stamina consume/regeneration events;
- health-drop timing.

`StartRiding()` requires positive horse level, health and stamina, then starts stamina consumption.

`StopRiding()` clears the riding flag and returns to stamina regeneration.

The horse-rider destructor cancels both stamina events.

### Character horse actor
`CHARACTER::HorseSummon(true)` spawns a horse actor, binds rider/horse pointers and sends horse-state UI data.

`CHARACTER::StartRiding()`:
1. validates map/state restrictions;
2. derives current horse/mount race;
3. enters CHorseRider riding state;
4. dismisses the standing horse actor;
5. sets character mount VNUM.

`StopRiding()` clears the mount VNUM and normally respawns the previously ridden horse actor.

Dead-horse summon creates a delayed dead-event which later dismisses the horse actor.

### Mount item/costume layer
`CItem::IsRideItem()` covers:
- UNIQUE_SPECIAL_RIDE;
- UNIQUE_SPECIAL_MOUNT_RIDE;
- COSTUME_MOUNT under the active mount-proto-affect system.

Mount item equip/unequip integrates with `POINT_MOUNT`, `MountVnum()` and full point recomputation.

### ChangeLook mount expiry candidate
Mount/horse-summon ChangeLook has a dedicated event using socket2.

`CItem::IsExpireTimeItem()` currently uses:
`if (GetType() != ITEM_COSTUME && GetSubType() != COSTUME_MOUNT) return false;`

That predicate is broader than the apparent intended exact type/subtype check. It remains an **unpromoted candidate** until a live caller/reachable consequence is proven in this subsystem.

## Initial cross-system observations
- Achievement data contains SUMMON_MOUNT tasks, while prior Achievement audit found no mapped horse/mount caller invoking that task family.
- Costume/Appearance audit already identified mount ChangeLook helper gaps but intentionally left them unpromoted pending a closed mount call chain.
- Horse/mount state affects character combat stats and therefore needs explicit recompute/cleanup auditing across dismount, item expiry, death, warp and logout.

## Exact next work
1. close horse persistence/login/logout and level bounds;
2. audit stamina/health events for lifetime and duplicate-event state;
3. map quest horse API authorization and range checks;
4. trace mount item/costume use -> affect -> `MountVnum` lifecycle;
5. trace item expiry/unequip/death/warp cleanup;
6. close ChangeLook mount expiry caller graph;
7. map Achievement SUMMON_MOUNT producer gap in this owning subsystem;
8. close client race/proto/appearance asset coverage;
9. create bugs/tests only for verified reachable paths.

No Horse/Mount runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Checkpoint — persistence/login + stamina/health event lifecycle closed

### Persistence / login
- `TPlayerTable::horse` persists level, riding, stamina, health and health-drop timestamp.
- DB save/load covers `horse_level`, `horse_riding`, `horse_hp`, `horse_hp_droptime` and `horse_stamina`.
- `CHARACTER::SetPlayerProto()` restores the raw `THorseInfo` and applies offline stamina regeneration through `UpdateHorseDataByLogoff()`.
- Enter-game then calls `EnterHorse()`; when persisted `bRiding` is true, `EnterHorse()` normalizes the flag and re-enters `StartRiding()`, recreating the consume-event side of the state machine.
- `SetHorseLevel()` clamps API-driven horse levels to `0..HORSE_MAX_LEVEL` (30).
- DB-loaded `THorseInfo::bLevel` itself is copied without a clamp before horse-stat table access. No tracked in-repository malformed producer was established, so this remains a persistence-integrity candidate rather than a promoted bug.

### Current-build health/stamina semantics
- `ENABLE_INFINITE_HORSE_HEALTH_STAMINA` is enabled in the active common defines.
- In this build, `GetHorseHealth()` and `GetHorseStamina()` always return the level maximum.
- Therefore classic consume/drop exhaustion is intentionally masked at the public accessor layer; the stamina consume event continues scheduling while riding, but cannot force a zero-stamina dismount through the accessor in the current build.
- Regen/consume event switching is single-owner: starting one cancels the opposite event, and `CHorseRider::Destroy()` cancels both.
- No duplicate-event or lifetime bug was verified in the current build.

### Login normalization note
- Later login setup calls `SetHorseLevel(GetHorseLevel())`, which resets underlying horse HP/stamina/drop-time to level maxima/new drop time.
- With infinite horse health/stamina enabled this has no distinct current gameplay consequence; if that feature is disabled later, this path must be reopened because persisted HP/stamina semantics would change materially.

### ChangeLook mount follow-up
- `StartChangeLookExpireEvent()` supports both costume mounts and horse-summon items.
- Current automatic event-start sites found in `ITEM_MANAGER::CreateItem()` and `CItem::OnAfterCreatedItem()` only start it for `IsHorseSummonItem()`.
- This is now the lead ChangeLook-mount candidate. It is not promoted until the socket2 producer/load path and a reachable costume-mount consequence are closed.

## Exact next work
1. audit quest horse API authorization/range handling and current quest producers;
2. trace mount item/costume -> affect -> `MountVnum` lifecycle;
3. trace expiry/unequip/death/warp cleanup;
4. close ChangeLook mount socket2 producer + automatic event-start caller graph;
5. resolve Achievement SUMMON_MOUNT producer gap;
6. close client race/proto/appearance asset coverage;
7. create bugs/tests only for verified reachable paths.

No Horse/Mount runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Checkpoint — active quest API + mount state synchronization

### Active horse quest deployment
The current `quest_list` loads these eight Horse/Mount scripts:
- `horse_exchange_ticket.quest`
- `horse_guard.quest`
- `horse_menu.quest`
- `horse_revive.quest`
- `horse_ride.quest`
- `horse_summon.quest`
- `ride_ticket_change.quest`
- `training_mount.quest`

Verified current producers:
- `horse_menu` -> `horse.ride/unride/summon/unsummon/revive/feed/set_name`;
- `horse_summon` -> `horse.summon()` with no custom VNUM argument;
- `horse_ride` -> `pc.mount(20030, 600)` and item 71241 -> `pc.mount(20030, 10)`;
- `training_mount` consumes horse-level gates but does not change horse level.

No active h_horse producer was found for:
- `horse.set_level`;
- `horse.advance`;
- `horse.set_appearance`;
- `horse.set_stat0`.

The current package therefore depends on an already-existing horse level for grade-gated flows. This is kept as a deployment/progression candidate until all non-h_horse active quest producers are excluded.

### BUG-HORSE-001 — pc.mount affect does not synchronize MountVnum
Current build enables `ENABLE_MOUNT_COSTUME_SYSTEM` and therefore `ENABLE_MOUNT_PROTO_AFFECT_SYSTEM`.

Live path:
`horse_ride.quest`
-> `pc.mount(20030, duration)`
-> `questlua_pc.cpp::pc_mount`
-> `AddAffect(AFFECT_MOUNT, POINT_MOUNT, 20030, ...)`
-> `ComputeAffect`
-> `PointChange(POINT_MOUNT, 20030)`.

But `CHARACTER::PointChange(POINT_MOUNT)` only updates the point value; its `MountVnum(val)` call is commented out.

No later call in `pc_mount` synchronizes `m_dwMountVnum`.

Consequences in the reachable active quest path:
- `GetPoint(POINT_MOUNT)` becomes non-zero;
- `GetMountVnum()` remains unchanged;
- `pc.is_mount()` reads `GetMountVnum()`, so it does not reflect the new affect-backed mount;
- character packet/render riding state also continues to use `GetMountVnum()`;
- the active rental horse path therefore has split authoritative state.

Promoted as `BUG-HORSE-001`.

### Item/costume mount lifecycle closure
Normal equipped ride-item flow is separate from `pc.mount` and does explicitly synchronize:
- `CItem::ModifyPoints(true)` applies `APPLY_MOUNT -> POINT_MOUNT`;
- `CHARACTER::EquipItem()` then calls `MountVnum(GetPoint(POINT_MOUNT))`;
- unequip removes the apply and then calls `MountVnum(GetPoint(POINT_MOUNT))` again.

Death cleanup:
- `Dead()` clears riding state;
- `UnEquipSpecialRideUniqueItem()` covers special-ride UNIQUE items and `WEAR_COSTUME_MOUNT`.

Restricted-map cleanup:
- `WarpSet()` itself does not unmount;
- post-warp EnterGame checks `IS_MOUNTABLE_ZONE()` and calls `Unmount()` when required.

No separate death/warp stale-mount bug was verified in these paths.

### ChangeLook mount candidate strengthened
`CTransmutation::Accept()` writes only:
`left->SetChangeLookVnum(right->GetVnum())`
and then destroys the right material.

It does not:
- call `right->IsExpireTimeItem()`;
- copy `right->GetRealExpireTime()` into the target's socket2;
- call `StartChangeLookExpireEvent()`.

Additionally, automatic ChangeLook expire-event start sites currently found in `ITEM_MANAGER::CreateItem()` and `CItem::OnAfterCreatedItem()` gate on `IsHorseSummonItem()`, not `COSTUME_MOUNT`.

This is a strong lifetime-laundering candidate, but promotion is deferred until a current deployed time-limited `COSTUME_MOUNT` proto is verified.

## Exact next work
1. verify current time-limited COSTUME_MOUNT proto/data reachability and close the ChangeLook lifetime candidate;
2. exclude/locate non-h_horse active producers for horse level progression;
3. close timer-based/real-time mount item expiry paths;
4. resolve Achievement SUMMON_MOUNT producer gap;
5. close client race/proto/horse-appearance asset coverage;
6. promote only verified reachable additional Horse/Mount bugs/tests.

No runtime execution is authorized. Global first future live gate remains `DUNGEON-T10`.
